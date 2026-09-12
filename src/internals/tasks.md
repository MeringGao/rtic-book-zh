# 软件任务

RTIC 支持软件任务与硬件任务. 每个硬件任务都绑定到一个不同的中断处理函数. 而多个软件任务可以由同一个中断处理函数派发 —— 这是为了尽量减少框架所使用的中断处理函数数量.

框架按优先级对可 `spawn` 的任务进行分组, 并为每个优先级生成一个*任务派发器*. 每个任务派发器运行在不同的中断处理函数上, 该中断处理函数的优先级被设置为与该派发器所管理任务的优先级相匹配.

每个任务派发器都维护一个就绪任务的*队列*, 该队列被称为*就绪队列*. 派生一个软件任务的过程包括: 往该队列中添加一个条目, 并触发运行相应任务派发器的中断. 队列中的每个条目都包含一个用于标识待执行任务的标签 (`enum`) 和一个传递给该任务的*消息指针*.

就绪队列是一个 SPSC ( 单生产者单消费者 ) 无锁队列. 任务派发器拥有该队列的消费者端点; 生产者端则被视为可 `spawn` 其他任务的那些任务所争夺的资源.

## 任务派发器

我们先来看一下框架为派发任务所生成的代码. 考虑下面的示例:

``` rust,noplayground
#[rtic::app(device = ..)]
mod app {
    // ..

    #[interrupt(binds = UART0, priority = 2, spawn = [bar, baz])]
    fn foo(c: foo::Context) {
        foo.spawn.bar().ok();

        foo.spawn.baz(42).ok();
    }

    #[task(capacity = 2, priority = 1)]
    fn bar(c: bar::Context) {
        // ..
    }

    #[task(capacity = 2, priority = 1, resources = [X])]
    fn baz(c: baz::Context, input: i32) {
        // ..
    }

    extern "C" {
        fn UART1();
    }
}
```

框架会生成下面的任务派发器, 它由一个中断处理函数与一个就绪队列组成:

``` rust,noplayground
fn bar(c: bar::Context) {
    // .. user code ..
}

mod app {
    use heapless::spsc::Queue;
    use cortex_m::register::basepri;

    struct Ready<T> {
        task: T,
        // ..
    }

    /// `spawn`-able tasks that run at priority level `1`
    enum T1 {
        bar,
        baz,
    }

    // ready queue of the task dispatcher
    // `5-1=4` represents the capacity of this queue
    static mut RQ1: Queue<Ready<T1>, 5> = Queue::new();

    // interrupt handler chosen to dispatch tasks at priority `1`
    #[no_mangle]
    unsafe UART1() {
        // the priority of this interrupt handler
        const PRIORITY: u8 = 1;

        let snapshot = basepri::read();

        while let Some(ready) = RQ1.split().1.dequeue() {
            match ready.task {
                T1::bar => {
                    // **NOTE** simplified implementation

                    // used to track the dynamic priority
                    let priority = Cell::new(PRIORITY);

                    // call into user code
                    bar(bar::Context::new(&priority));
                }

                T1::baz => {
                    // we'll look at `baz` later
                }
            }
        }

        // BASEPRI invariant
        basepri::write(snapshot);
    }
}
```

## 派生任务

`spawn` API 以 `Spawn` 结构体方法的形式暴露给用户. 每个任务都有一个对应的 `Spawn` 结构体.

针对上面这个示例, 框架生成的 `Spawn` 代码形如:

``` rust,noplayground
mod foo {
    // ..

    pub struct Context<'a> {
        pub spawn: Spawn<'a>,
        // ..
    }

    pub struct Spawn<'a> {
        // tracks the dynamic priority of the task
        priority: &'a Cell<u8>,
    }

    impl<'a> Spawn<'a> {
        // `unsafe` and hidden because we don't want the user to tamper with it
        #[doc(hidden)]
        pub unsafe fn priority(&self) -> &Cell<u8> {
            self.priority
        }
    }
}

mod app {
    // ..

    // Priority ceiling for the producer endpoint of the `RQ1`
    const RQ1_CEILING: u8 = 2;

    // used to track how many more `bar` messages can be enqueued
    // `3-1=2` represents the capacity of this queue
    // this queue is filled by the framework before `init` runs
    static mut bar_FQ: Queue<(), 3> = Queue::new();

    // Priority ceiling for the consumer endpoint of `bar_FQ`
    const bar_FQ_CEILING: u8 = 2;

    // a priority-based critical section
    //
    // this run the given closure `f` at a dynamic priority of at least
    // `ceiling`
    fn lock(priority: &Cell<u8>, ceiling: u8, f: impl FnOnce()) {
        // ..
    }

    impl<'a> foo::Spawn<'a> {
        /// Spawns the `bar` task
        pub fn bar(&self) -> Result<(), ()> {
            unsafe {
                match lock(self.priority(), bar_FQ_CEILING, || {
                    bar_FQ.split().1.dequeue()
                }) {
                    Some(()) => {
                        lock(self.priority(), RQ1_CEILING, || {
                            // put the taks in the ready queue
                            RQ1.split().1.enqueue_unchecked(Ready {
                                task: T1::bar,
                                // ..
                            })
                        });

                        // pend the interrupt that runs the task dispatcher
                        rtic::pend(Interrupt::UART0);
                    }

                    None => {
                        // maximum capacity reached; spawn failed
                        Err(())
                    }
                }
            }
        }
    }
}
```

用 `bar_FQ` 来限制可被派生的 `bar` 任务数量看似人为, 但当我们讨论任务容量时, 这样做的意义就会变得更加清晰.

## 消息

我们之前省略了消息传递的具体机制, 现在我们重新审视 `spawn` 的实现, 但这次关注接收 `u64` 消息的任务 `baz`.

``` rust,noplayground
fn baz(c: baz::Context, input: u64) {
    // .. user code ..
}

mod app {
    // ..

    // Now we show the full contents of the `Ready` struct
    struct Ready {
        task: Task,
        // message index; used to index the `INPUTS` buffer
        index: u8,
    }

    // memory reserved to hold messages passed to `baz`
    static mut baz_INPUTS: [MaybeUninit<u64>; 2] =
        [MaybeUninit::uninit(), MaybeUninit::uninit()];

    // the free queue: used to track free slots in the `baz_INPUTS` array
    // this queue is initialized with values `0` and `1` before `init` is executed
    static mut baz_FQ: Queue<u8, 3> = Queue::new();

    // Priority ceiling for the consumer endpoint of `baz_FQ`
    const baz_FQ_CEILING: u8 = 2;

    impl<'a> foo::Spawn<'a> {
        /// Spawns the `baz` task
        pub fn baz(&self, message: u64) -> Result<(), u64> {
            unsafe {
                match lock(self.priority(), baz_FQ_CEILING, || {
                    baz_FQ.split().1.dequeue()
                }) {
                    Some(index) => {
                        // NOTE: `index` is an ownining pointer into this buffer
                        baz_INPUTS[index as usize].write(message);

                        lock(self.priority(), RQ1_CEILING, || {
                            // put the task in the ready queue
                            RQ1.split().1.enqueue_unchecked(Ready {
                                task: T1::baz,
                                index,
                            });
                        });

                        // pend the interrupt that runs the task dispatcher
                        rtic::pend(Interrupt::UART0);
                    }

                    None => {
                        // maximum capacity reached; spawn failed
                        Err(message)
                    }
                }
            }
        }
    }
}
```

现在我们来看一下任务派发器的真实实现:

``` rust,noplayground
mod app {
    // ..

    #[no_mangle]
    unsafe UART1() {
        const PRIORITY: u8 = 1;

        let snapshot = basepri::read();

        while let Some(ready) = RQ1.split().1.dequeue() {
            match ready.task {
                Task::baz => {
                    // NOTE: `index` is an ownining pointer into this buffer
                    let input = baz_INPUTS[ready.index as usize].read();

                    // the message has been read out so we can return the slot
                    // back to the free queue
                    // (the task dispatcher has exclusive access to the producer
                    // endpoint of this queue)
                    baz_FQ.split().0.enqueue_unchecked(ready.index);

                    let priority = Cell::new(PRIORITY);
                    baz(baz::Context::new(&priority), input)
                }

                Task::bar => {
                    // looks just like the `baz` branch
                }

            }
        }

        // BASEPRI invariant
        basepri::write(snapshot);
    }
}
```

`INPUTS` 加上 `FQ` ( 空闲队列 ) 实际上就是一个内存池. 不过, 我们并没有使用通常的*空闲链表* ( 链表 ) 来跟踪 `INPUTS` 缓冲区中的空闲槽位, 而是使用了 SPSC 队列; 这让我们能够减少临界区的数量. 事实上, 正是这一选择使得任务派发代码成为无锁的.

## 队列容量

RTIC 框架使用了若干队列, 比如就绪队列与空闲队列. 当空闲队列为空时, 尝试 `spawn` 一个任务会导致失败; 该条件会在运行时被检查. 并不是框架对这些队列执行的所有操作都会检查队列是否为空 / 已满. 例如, 将一个槽位归还给空闲队列 ( 参见任务派发器 ) 是不做检查的, 因为在系统中循环使用的槽位数量是固定的, 它等于空闲队列的容量. 类似地, 向就绪队列添加条目 ( 参见 `Spawn` ) 也不做检查, 这是因为队列的容量由框架选定.

用户可以指定软件任务的容量; 该容量是高优先级任务在 `spawn` 返回错误之前, 可以向该任务投递消息的最大数量. 这个用户指定的容量就是该任务空闲队列 ( 例如 `foo_FQ` ) 的容量, 也是存放该任务输入的数组 ( 例如 `foo_INPUTS` ) 的大小.

就绪队列 ( 例如 `RQ1` ) 的容量被选为该派发器所管理所有任务容量之*和*; 这个和也是在所有可能的场景下, 在任务派发器获得运行机会之前, 该队列在最坏情况下所需容纳的消息数量. 因此, 在任何 `spawn` 操作中, 一旦从空闲队列中拿到了一个槽位, 就意味着就绪队列尚未填满, 因此向就绪队列插入条目时可以省略 "是否已满?" 的检查.

在我们这个贯穿始终的示例中, 任务 `bar` 不接收任何输入, 所以理论上我们可以同时省略 `bar_INPUTS` 与 `bar_FQ`, 让用户能够向该任务投递任意数量的消息. 但如果真的这样做, 就无法为 `RQ1` 选定一个能够在派生 `baz` 任务时省略 "是否已满?" 检查的容量. 在 [定时器队列](timer-queue.md) 一节中, 我们将看到空闲队列是如何被那些没有输入的任务所使用的.

## 上限分析

`spawn` API 内部使用的队列会被当作普通资源, 并纳入上限分析. 需要注意的是, 这些都是 SPSC 队列, 因此只有一个端点会受资源保护; 另一个端点则归任务派发器所有.

考虑下面的示例:

``` rust,noplayground
#[rtic::app(device = ..)]
mod app {
    #[idle(spawn = [foo, bar])]
    fn idle(c: idle::Context) -> ! {
        // ..
    }

    #[task]
    fn foo(c: foo::Context) {
        // ..
    }

    #[task]
    fn bar(c: bar::Context) {
        // ..
    }

    #[task(priority = 2, spawn = [foo])]
    fn baz(c: baz::Context) {
        // ..
    }

    #[task(priority = 3, spawn = [bar])]
    fn quux(c: quux::Context) {
        // ..
    }
}
```

上限分析的过程如下:

- `idle` ( prio = 0 ) 与 `baz` ( prio = 2 ) 争夺 `foo_FQ` 的消费者端点; 因此得到的优先级上限为 `2`.

- `idle` ( prio = 0 ) 与 `quux` ( prio = 3 ) 争夺 `bar_FQ` 的消费者端点; 因此得到的优先级上限为 `3`.

- `idle` ( prio = 0 )、`baz` ( prio = 2 ) 与 `quux` ( prio = 3 ) 都争夺 `RQ1` 的生产者端点; 因此得到的优先级上限为 `3`.

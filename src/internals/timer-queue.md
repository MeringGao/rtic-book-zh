# 定时器队列

定时器队列功能允许用户将任务调度到未来的某个时刻运行. 不出所料, 该功能同样是通过队列实现的: 它是一个优先级队列, 被调度的任务在其中按最早调度时间排序. 该功能需要一个能够设置超时中断的定时器. 该定时器用于在任务的调度时刻到达时触发一次中断; 此时, 该任务会从定时器队列中移除, 并被放入相应的就绪队列.

下面我们看一下在代码中是如何实现的. 考虑下面的程序:

``` rust,noplayground
#[rtic::app(device = ..)]
mod app {
    // ..

    #[task(capacity = 2, schedule = [foo])]
    fn foo(c: foo::Context, x: u32) {
        // schedule this task to run again in 1M cycles
        c.schedule.foo(c.scheduled + Duration::cycles(1_000_000), x + 1).ok();
    }

    extern "C" {
        fn UART0();
    }
}
```

## `schedule`

我们先来看 `schedule` API.

``` rust,noplayground
mod foo {
    pub struct Schedule<'a> {
        priority: &'a Cell<u8>,
    }

    impl<'a> Schedule<'a> {
        // unsafe and hidden because we don't want the user to tamper with this
        #[doc(hidden)]
        pub unsafe fn priority(&self) -> &Cell<u8> {
            self.priority
        }
    }
}

mod app {
    type Instant = <path::to::user::monotonic::timer as rtic::Monotonic>::Instant;

    // all tasks that can be `schedule`-d
    enum T {
        foo,
    }

    struct NotReady {
        index: u8,
        instant: Instant,
        task: T,
    }

    // The timer queue is a binary (min) heap of `NotReady` tasks
    static mut TQ: TimerQueue<U2> = ..;
    const TQ_CEILING: u8 = 1;

    static mut foo_FQ: Queue<u8, U2> = Queue::new();
    const foo_FQ_CEILING: u8 = 1;

    static mut foo_INPUTS: [MaybeUninit<u32>; 2] =
        [MaybeUninit::uninit(), MaybeUninit::uninit()];

    static mut foo_INSTANTS: [MaybeUninit<Instant>; 2] =
        [MaybeUninit::uninit(), MaybeUninit::uninit()];

    impl<'a> foo::Schedule<'a> {
        fn foo(&self, instant: Instant, input: u32) -> Result<(), u32> {
            unsafe {
                let priority = self.priority();
                if let Some(index) = lock(priority, foo_FQ_CEILING, || {
                    foo_FQ.split().1.dequeue()
                }) {
                    // `index` is an owning pointer into these buffers
                    foo_INSTANTS[index as usize].write(instant);
                    foo_INPUTS[index as usize].write(input);

                    let nr = NotReady {
                        index,
                        instant,
                        task: T::foo,
                    };

                    lock(priority, TQ_CEILING, || {
                        TQ.enqueue_unchecked(nr);
                    });
                } else {
                    // No space left to store the input / instant
                    Err(input)
                }
            }
        }
    }
}
```

这与 `Spawn` 的实现非常相似. 事实上, 同一个 `INPUTS` 缓冲区和空闲队列 (`FQ`) 被 `spawn` 与 `schedule` 两个 API 共享. 两者之间的主要区别在于 `schedule` 还在一个单独的缓冲区 ( 此例中的 `foo_INSTANTS` ) 中存储任务被调度运行的 `Instant`.

`TimerQueue::enqueue_unchecked` 所做的工作比仅仅把条目加入小顶堆要更多一些: 如果新加入的条目最终排到了队列首位, 它还会触发系统定时器中断 (`SysTick`).

## 系统定时器

系统定时器中断 (`SysTick`) 负责两件事: 把已变为就绪的任务从定时器队列转移到正确的就绪队列, 以及为下一个任务的调度时刻到来时设置一个超时中断.

我们看一下相关的代码.

``` rust,noplayground
mod app {
    #[no_mangle]
    fn SysTick() {
        const PRIORITY: u8 = 1;

        let priority = &Cell::new(PRIORITY);
        while let Some(ready) = lock(priority, TQ_CEILING, || TQ.dequeue()) {
            match ready.task {
                T::foo => {
                    // move this task into the `RQ1` ready queue
                    lock(priority, RQ1_CEILING, || {
                        RQ1.split().0.enqueue_unchecked(Ready {
                           task: T1::foo,
                           index: ready.index,
                        })
                    });

                    // pend the task dispatcher
                    rtic::pend(Interrupt::UART0);
                }
            }
        }
    }
}
```

这看起来与任务派发器很像, 只不过它并不直接运行就绪任务, 而只是把任务放入相应的就绪队列, 以便任务能够以正确的优先级运行.

`TimerQueue::dequeue` 在返回 `None` 时会设置一个新的超时中断. 这与 `TimerQueue::enqueue_unchecked` 配合使用, 后者会触发此处理函数; 也就是说, `enqueue_unchecked` 把设置新超时中断的工作委托给了 `SysTick` 处理函数.

## `cyccnt::Instant` 与 `cyccnt::Duration` 的分辨率与范围

RTIC 提供了一个基于 `DWT` ( 数据观察点与跟踪 ) 周期计数器的 `Monotonic` 实现. `Instant::now` 返回该定时器的一个快照; 这些 DWT 快照 (`Instant`) 用于在定时器队列中对条目进行排序. 周期计数器是一个按核心时钟频率计数的 32 位计数器. 每经过 `(1 << 32)` 个时钟周期, 该计数器就会回绕一次; 该计数器不关联任何中断, 因此回绕发生时不会产生任何需要关注的事件.

为了在队列中对 `Instant` 进行排序, 我们需要比较两个 32 位整数. 为了正确处理回绕行为, 我们使用两个 `Instant` 之差 `a - b`, 并将结果视作一个有符号 32 位整数. 如果结果小于零, 则说明 `b` 处于更晚的 `Instant`; 如果结果大于零, 则说明 `b` 处于更早的 `Instant`. 这意味着, 如果某个任务被调度到的 `Instant` 比起队列中第一个 ( 最早的 ) 条目的调度 `Instant` 大 `(1 << 31) - 1` 个周期, 该任务会被插入到队列的错误位置上. 我们已经设置了一些调试断言来防止这种用户错误, 但这种情况无法完全避免, 因为用户可以写出 `(instant + duration_a) + duration_b` 这样的表达式并导致 `Instant` 溢出.

系统定时器 `SysTick` 是一个同样按核心时钟频率计数的 24 位计数器. 当下一个被调度任务的时刻距离现在超过 `1 << 24` 个时钟周期时, 会设置一个在 `1 << 24` 个周期后触发的事件. 这个过程可能需要重复多次, 直到下一个被调度任务的时刻落入 `SysTick` 计数器的范围之内.

综上所述, `Instant` 与 `Duration` 的分辨率均为 1 个核心时钟周期, 而 `Duration` 的有效范围 ( 半开区间 ) 为 `0..(1 << 31)` ( 不含端点 ) 个核心时钟周期.

## 队列容量

定时器队列的容量被选为所有可 `schedule` 任务容量之和. 与就绪队列的情况一样, 这意味着一旦我们在 `INPUTS` 缓冲区中成功申请到了一个空闲槽位, 就一定能够把任务插入定时器队列; 这让我们得以省略运行时检查.

## 系统定时器优先级

系统定时器的优先级无法由用户设置, 它由框架选定. 为了确保低优先级任务不会阻碍高优先级任务的运行, 我们将系统定时器的优先级选为所有可 `schedule` 任务中的最高优先级.

为了说明为什么必须这样做, 考虑这样一种情况: 之前被调度的优先级分别为 `2` 和 `3` 的两个任务几乎同时就绪, 但低优先级任务先被移入就绪队列. 如果系统定时器优先级是 `1`, 那么在移入优先级为 `2` 的任务之后, 该任务会因优先级高于系统定时器而一直运行到结束, 从而延迟优先级为 `3` 的任务的执行. 为了避免此类情形, 系统定时器的优先级必须匹配可 `schedule` 任务中的最高优先级; 在本例中即为 `3`.

## 上限分析

定时器队列是一个被所有可 `schedule` 任务以及 `SysTick` 处理函数共享的资源. 此外, `schedule` API 与 `spawn` API 还会共同争夺空闲队列. 所有这些都必须纳入上限分析的考虑之中.

为了具体说明, 考虑下面的示例:

``` rust,noplayground
#[rtic::app(device = ..)]
mod app {
    #[task(priority = 3, spawn = [baz])]
    fn foo(c: foo::Context) {
        // ..
    }

    #[task(priority = 2, schedule = [foo, baz])]
    fn bar(c: bar::Context) {
        // ..
    }

    #[task(priority = 1)]
    fn baz(c: baz::Context) {
        // ..
    }
}
```

上限分析的过程如下:

- `foo` ( prio = 3 ) 与 `baz` ( prio = 1 ) 都是可 `schedule` 的任务, 因此 `SysTick` 必须以这两者中的最高优先级运行, 也就是 `3`.

- `foo::Spawn` ( prio = 3 ) 与 `bar::Schedule` ( prio = 2 ) 争夺 `baz_FQ` 的消费者端点; 因此得到的优先级上限为 `3`.

- `bar::Schedule` ( prio = 2 ) 对 `foo_FQ` 的消费者端点拥有独占访问权; 因此 `foo_FQ` 的优先级上限实际上就是 `2`.

- `SysTick` ( prio = 3 ) 与 `bar::Schedule` ( prio = 2 ) 争夺定时器队列 `TQ`; 因此得到的优先级上限为 `3`.

- `SysTick` ( prio = 3 ) 与 `foo::Spawn` ( prio = 3 ) 都对就绪队列 `RQ3` ( 它保存 `foo` 条目 ) 拥有无锁访问权; 因此 `RQ3` 的优先级上限实际上就是 `3`.

- `SysTick` 对就绪队列 `RQ1` ( 它保存 `baz` 条目 ) 拥有独占访问权; 因此 `RQ1` 的优先级上限实际上就是 `3`.

## `spawn` 实现中的变化

当使用 `schedule` API 时, `spawn` 的实现会有一些变化, 以跟踪任务的时间基线. 正如你在 `schedule` 实现中所看到的, 那里使用了一个 `INSTANTS` 缓冲区来存储任务被调度运行的时刻; 这个 `Instant` 会在任务派发器中被读取, 并作为任务上下文的一部分传递给用户代码.

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
                    let input = baz_INPUTS[ready.index as usize].read();
                    // ADDED
                    let instant = baz_INSTANTS[ready.index as usize].read();

                    baz_FQ.split().0.enqueue_unchecked(ready.index);

                    let priority = Cell::new(PRIORITY);
                    // CHANGED the instant is passed as part the task context
                    baz(baz::Context::new(&priority, instant), input)
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

相应地, `spawn` 的实现也需要向 `INSTANTS` 缓冲区写入一个值. 该值保存在 `Spawn` 结构体中, 它要么是硬件任务的 `start` 时刻, 要么是软件任务的 `scheduled` 时刻.

``` rust,noplayground
mod foo {
    // ..

    pub struct Spawn<'a> {
        priority: &'a Cell<u8>,
        // ADDED
        instant: Instant,
    }

    impl<'a> Spawn<'a> {
        pub unsafe fn priority(&self) -> &Cell<u8> {
            &self.priority
        }

        // ADDED
        pub unsafe fn instant(&self) -> Instant {
            self.instant
        }
    }
}

mod app {
    impl<'a> foo::Spawn<'a> {
        /// Spawns the `baz` task
        pub fn baz(&self, message: u64) -> Result<(), u64> {
            unsafe {
                match lock(self.priority(), baz_FQ_CEILING, || {
                    baz_FQ.split().1.dequeue()
                }) {
                    Some(index) => {
                        baz_INPUTS[index as usize].write(message);
                        // ADDED
                        baz_INSTANTS[index as usize].write(self.instant());

                        lock(self.priority(), RQ1_CEILING, || {
                            RQ1.split().1.enqueue_unchecked(Ready {
                                task: Task::foo,
                                index,
                            });
                        });

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

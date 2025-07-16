+++
title = "Rust 进阶学习笔记（八）：异步编程"
slug = "rust_learn_note_adv_8"
date = 2025-07-16
updated = 2025-07-17
description = ""
# draft = true
[taxonomies]
tags = ["Rust", "Learn"]
[extra]
pinned = false
post_listing_date = "both"
+++

本节出处：[圣经-4.11异步编程](https://course.rs/advance/async/intro.html)

异步比多线程难多了，头都要看炸了。

## 异步简介
Rust 同时支持多线程和异步编程，并且选择了基于`async/await`的异步编程[^1]。Rust 的异步编程性能很高，可以认为没有额外消耗。

async 底层也是基于线程实现，但它基于线程封装了一个运行时，让多个任务映射到少量线程上，然后线程切换就变成了任务切换，这就变得十分高效。对于 IO 密集型任务，如果使用多线程，那么大多数线程都被阻塞，处于等待状态，一旦唤醒又有昂贵的上下文切换代价，此时使用 async 就十分合适。

一般地，建议这样选择：
- 有大量 IO 任务需要并发运行时，选 async 模型
- 有部分 IO 任务需要并发运行时，选多线程，如果想要降低线程创建和销毁的开销，可以使用线程池
- 有大量 CPU 密集任务需要并行运行时，例如并行计算，选多线程模型，且让线程数等于或者稍大于 CPU 核心数
- 无所谓时，统一选多线程
```rust
// 来看一组对比
fn get_two_sites() {
    // 创建两个新线程执行任务
    let thread_one = thread::spawn(|| download("https://course.rs"));
    let thread_two = thread::spawn(|| download("https://fancy.rs"));

    // 等待两个线程的完成
    thread_one.join().expect("thread one panicked");
    thread_two.join().expect("thread two panicked");
}
// 一旦下载文件的并发请求多起来，那一个下载任务占用一个线程的模式就太重了，会很容易成为程序的瓶颈。
async fn get_two_sites_async() {
    // 创建两个不同的`future`，你可以把`future`理解为未来某个时刻会被执行的计划任务
    // 当两个`future`被同时执行后，它们将并发的去下载目标页面
    let future_one = download_async("https://www.foo.com");
    let future_two = download_async("https://www.bar.com");

    // 同时运行两个`future`，直至完成
    // 函数静态分发，减少内存分配，性能更好
    join!(future_one, future_two);
}
```
事实上，async 和多线程并不是二选一，在同一应用中，可以根据情况两者一起使用。

## 异步入门
async 的底层实现非常复杂，且会导致编译后文件体积显著增加。因此 Rust 内置的异步特性并不完整。要完整的使用 async 异步编程，你需要依赖以下特性和外部库：
- 必须的特质(例如`Future`)、类型和函数，由标准库提供实现
- 关键字`async/await`由 Rust 语言提供，并进行了编译器层面的支持
- 众多实用的类型、宏和函数由官方开发的 futures 包提供（不属于 std），它们可以用于任何 async 应用中。
- async 代码的执行、IO 操作、任务创建和调度等等复杂功能由社区的 async 运行时提供，例如 tokio 和 async-std

Rust 为异步增加了很多限制，导致异步编程中很容易遇到在同步中没见过的问题。

作为入门，先来看看最简单的`async/await`。

第一件事是需要引入 futures 包：
```toml,name=Cargo.toml
[dependencies]
futures = "0.3"
```

### async
使用`async fn`语法来创建一个异步函数。异步函数的返回值是一个 `Future`，若直接调用该函数，不会输出任何结果，因为`Future`还未被执行，需要使用一个执行器（executor）。
```rust
// `block_on`会阻塞当前线程直到指定的`Future`执行完成，这种阻塞当前线程以等待任务完成的方式较为简单、粗暴，
// 好在其它运行时的执行器会提供更加复杂的行为，例如将多个`future`调度到同一个线程上执行。
use futures::executor::block_on;

async fn hello_world() {
    println!("hello, world!");
}

fn main() {
    let future = hello_world(); // 返回一个Future, 因此不会打印任何输出
    block_on(future); // 执行`Future`并等待其运行完成，此时"hello, world!"会被打印输出
}
```
`block_on`执行器使得这段异步代码看起来就和同步代码一样。

### await
如果要在一个`async fn`函数中去调用另一个`async fn`并等待其完成后再执行后续的代码，该如何做？
```rust
use futures::executor::block_on;

async fn hello_world() {
    // 不加 await 编译器会告警，因为无人运行 hello_cat 这个 Future
    // futures do nothing unless you `.await` or poll them
    hello_cat().await;
    println!("hello, world!");
}

async fn hello_cat() {
    println!("hello, kitty!");
}

fn main() {
    let future = hello_world();
    block_on(future);
    // // 输出：
    // hello, kitty!
    // hello, world!
    // 输出的顺序跟代码定义的顺序完全符合
}
```
在`async fn`函数中使用`await`方法可以等待另一个异步调用的完成。`await`并不会阻塞当前的线程，而是允许我们在同一个线程内并发地运行多个任务，而不是一个一个先后完成，最终实现了并发处理的效果。因此，我们在上面代码中使用同步的代码顺序实现了异步的执行效果，非常简单、高效，而且很好理解，未来也绝对不会有回调地狱的发生。

## Future 的执行器和任务调度
`Future`是异步函数的返回值，是异步函数执行的关键。因此，`Future`特质是异步编程的核心。

### Future 特质
`Future`是一个是一个能产出值的异步计算。下面是一个简化的特质：
```rust
trait SimpleFuture {
    type Output;
    fn poll(&mut self, wake: fn()) -> Poll<Self::Output>;
}

enum Poll<T> {
    Ready(T),
    Pending,
}
```
`Future`需要被执行器轮询（poll）后才能运行。通过调用`poll`方法，可以推进`Future`的执行直到（`Future`不保证轮询一次就执行完）。如果在当前轮询中`Future`能够完成，就会返回一个`Poll::Ready(result)`；反之则是`Poll::Pending`加`wake`。`wake`函数的作用是，当未来`Future`准备好进一步执行时，`wake`将被调用，然后`Future`的执行器会再次调用`poll`方法，让`Future`继续执行。也就是说，`wake`能够让`Future`主动通知执行器，让执行器精确地执行这个`Future`，而不是不断进行全遍历。

来看一个例子：
```rust
pub struct SocketRead<'a> {
    socket: &'a Socket,
}

// 为 SocketRead 结构体实现一个 Future
impl SimpleFuture for SocketRead<'_> {
    type Output = Vec<u8>;

    fn poll(&mut self, wake: fn()) -> Poll<Self::Output> {
        if self.socket.has_data_to_read() {
            // socket 有数据，写入 buffer 中并返回
            Poll::Ready(self.socket.read_buf())
        } else {
            // socket中还没数据
            //
            // 注册一个`wake`函数，当数据可用时，该函数会被调用，
            // 然后当前 Future 的执行器会再次调用`poll`方法，此时就可以读取到数据
            self.socket.set_readable_callback(wake);
            Poll::Pending
        }
    }
}
```
这种`Future`模型允许将多个异步操作组合在一起，同时还无需任何内存分配。不仅仅如此，如果你需要同时运行多个`Future`或链式调用多个`Future`，也可以通过无内存分配的状态机实现：
```rust
/// 一个 SimpleFuture, 它使用顺序的方式，一个接一个地运行两个 Future
//
// 注意: 由于本例子用于演示，因此功能简单，`AndThenFut` 会假设两个 Future 在创建时就可用了.
// 而真实的`Andthen`允许根据第一个`Future`的输出来创建第二个`Future`，因此复杂的多。
pub struct AndThenFut<FutureA, FutureB> {
    first: Option<FutureA>,
    second: FutureB,
}

impl<FutureA, FutureB> SimpleFuture for AndThenFut<FutureA, FutureB>
where
    FutureA: SimpleFuture<Output = ()>,
    FutureB: SimpleFuture<Output = ()>,
{
    type Output = ();
    fn poll(&mut self, wake: fn()) -> Poll<Self::Output> {
        if let Some(first) = &mut self.first {
            match first.poll(wake) {
                // 我们已经完成了第一个 Future， 可以将它移除， 然后准备开始运行第二个
                Poll::Ready(()) => self.first.take(),
                // 第一个 Future 还不能完成
                Poll::Pending => return Poll::Pending,
            };
        }

        // 运行到这里，说明第一个 Future 已经完成，尝试去完成第二个
        self.second.poll(wake)
    }
}
```
上面是简化版的`Future`。真实的`Future`如下：
```rust
trait Future {
    type Output;
    fn poll(
        // 首先值得注意的地方是，`self`的类型从`&mut self`变成了`Pin<&mut Self>`:
        // `Pin`可以创建一个无法被移动的`Future`，也就是说在内存中拥有固定地址，可以存储其指针来访问。
        self: Pin<&mut Self>,
        // 其次将`wake: fn()` 修改为 `cx: &mut Context<'_>`:
        // Context 类型通过提供一个 Waker 类型的值，用于携带数据，唤醒特定的任务，避免歧义
        cx: &mut Context<'_>,
    ) -> Poll<Self::Output>;
}
```

### 唤醒任务
`Waker`提供了一个`wake()`方法可以用于告诉执行器：相关的任务可以被唤醒了，此时执行器就可以对相应的`Future`再次进行`poll`操作。

想象一个简单场景：实现一个计时装置。当计时器创建时，启动一个线程并让该线程进入睡眠，等睡眠结束后再通知给`Future`。

```rust
use std::{
    future::Future,
    pin::Pin,
    sync::{Arc, Mutex},
    task::{Context, Poll, Waker},
    thread,
    time::Duration,
};

pub struct TimerFuture {
    // 共享状态，用于在新线程和 Future 定时器间共享。
    shared_state: Arc<Mutex<SharedState>>,
}

/// 在Future和等待的线程间共享状态
struct SharedState {
    /// 定时是否结束
    completed: bool,

    /// 当睡眠结束后，线程可以用`waker`通知`TimerFuture`来唤醒任务
    waker: Option<Waker>,
}

impl Future for TimerFuture {
    type Output = ();
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        // 通过检查共享状态，来确定定时器是否已经完成
        let mut shared_state = self.shared_state.lock().unwrap();
        // 只要新线程设置了 shared_state.completed = true ，那任务就能顺利结束。
        // 如果没有设置，会为当前的任务克隆一份 Waker ，这样新线程就可以使用它来唤醒当前的任务。
        if shared_state.completed {
            Poll::Ready(())
        } else {
            // 设置`waker`，这样新线程在睡眠(计时)结束后可以唤醒当前的任务，接着再次对`Future`进行`poll`操作,
            //
            // 下面的`clone`每次被`poll`时都会发生一次，实际上，应该是只`clone`一次更加合理。
            // 选择每次都`clone`的原因是：`TimerFuture`可以在执行器的不同任务间移动，
            // 如果只克隆一次，那么获取到的`waker`可能已经被篡改并指向了其它任务，最终导致执行器运行了错误的任务
            shared_state.waker = Some(cx.waker().clone());
            Poll::Pending
        }
    }
}

// 构建定时器和启动计时线程的 new 方法
impl TimerFuture {
    /// 创建一个新的`TimerFuture`，在指定的时间结束后，该`Future`可以完成
    pub fn new(duration: Duration) -> Self {
        let shared_state = Arc::new(Mutex::new(SharedState {
            completed: false,
            waker: None,
        }));

        // 创建新线程
        let thread_shared_state = shared_state.clone();
        thread::spawn(move || {
            // 睡眠指定时间实现计时功能
            thread::sleep(duration);
            let mut shared_state = thread_shared_state.lock().unwrap();
            // 通知执行器定时器已经完成，可以继续`poll`对应的`Future`了
            shared_state.completed = true;
            if let Some(waker) = shared_state.waker.take() {
                waker.wake()
            }
        });

        TimerFuture { shared_state }
    }
}
```
此时有了一个定时器`Future`，但是还没有调用者。自然的，调用者就是一个执行器。

### 执行器
`Future`是懒的，不会主动启动。`await`可以唤醒`async`函数，但是`await`本身也是一个`async`函数的方法，最外层的`async`函数就需要执行器来调用。

执行器（Executor）管理一批`Future`，然后通过不停地`poll`推动它们直到完成。执行器会先`poll`一次，如果未能成功，后面就会等待`Future`通过调用`wake`函数来通知它可以继续，直到`Future`完成。

执行器需要从一个消息通道（channel）中拉取事件，然后运行它们。当一个任务准备好后（可以继续执行），它会将自己放入消息通道中，然后等待执行器`poll`。
```rust
use {
    futures::{
        future::{BoxFuture, FutureExt},
        task::{waker_ref, ArcWake},
    },
    std::{
        future::Future,
        sync::mpsc::{sync_channel, Receiver, SyncSender},
        sync::{Arc, Mutex},
        task::{Context, Poll},
        time::Duration,
    },
    // 引入之前实现的定时器模块
    timer_future::TimerFuture,
};

/// 任务执行器，负责从通道中接收任务然后执行
struct Executor {
    ready_queue: Receiver<Arc<Task>>,
}

/// `Spawner`负责创建新的`Future`然后将它发送到任务通道中
#[derive(Clone)]
struct Spawner {
    task_sender: SyncSender<Arc<Task>>,
}

/// 一个Future，它可以调度自己（将自己放入任务通道中），然后等待执行器去`poll`
struct Task {
    // 进行中的Future，在未来的某个时间点会被完成
    // 按理来说`Mutex`在这里是多余的，因为我们只有一个线程来执行任务。
    // 但是由于 Rust 编译器无法知道`Future`只会在一个线程内被修改，并不会被跨线程修改。
    // 因此需要使用`Mutex`来满足编译器对线程安全的执着。
    // 如果是生产级的执行器实现，不会使用`Mutex`，因为会带来性能上的开销，取而代之的是使用`UnsafeCell`
    future: Mutex<Option<BoxFuture<'static, ()>>>,

    // 可以将该任务自身放回到任务通道中，等待执行器的 poll
    task_sender: SyncSender<Arc<Task>>,
}

fn new_executor_and_spawner() -> (Executor, Spawner) {
    // 任务通道允许的最大缓冲数，即任务队列的最大长度
    const MAX_QUEUED_TASKS: usize = 10_000;
    let (task_sender, ready_queue) = sync_channel(MAX_QUEUED_TASKS);
    (Executor { ready_queue }, Spawner { task_sender })
}

// spawn 方法生成 Future , 然后将它放入任务通道中:
impl Spawner {
    fn spawn(&self, future: impl Future<Output = ()> + 'static + Send) {
        let future = future.boxed();
        let task = Arc::new(Task {
            future: Mutex::new(Some(future)),
            task_sender: self.task_sender.clone(),
        });
        self.task_sender.send(task).expect("任务队列已满");
    }
}

// 为任务实现 ArcWake 特征，这样它们就能被转变成 Waker 然后被唤醒:
impl ArcWake for Task {
    fn wake_by_ref(arc_self: &Arc<Self>) {
        // 通过发送任务到任务管道的方式来实现`wake`，这样`wake`后，任务就能被执行器`poll`
        let cloned = arc_self.clone();
        arc_self
            .task_sender
            .send(cloned)
            .expect("任务队列已满");
    }
}

// 执行器将从通道中获取任务，然后进行 poll 执行
impl Executor {
    fn run(&self) {
        while let Ok(task) = self.ready_queue.recv() {
            // 获取一个 future，若它还没有完成，即仍然是 Some、不是 None，则对它进行一次 poll 并尝试完成它
            let mut future_slot = task.future.lock().unwrap();
            if let Some(mut future) = future_slot.take() {
                // 基于任务自身创建一个 `LocalWaker`
                let waker = waker_ref(&task);
                let context = &mut Context::from_waker(&*waker);
                // `BoxFuture<T>`是`Pin<Box<dyn Future<Output = T> + Send + 'static>>`的类型别名
                // 通过调用`as_mut`方法，可以将上面的类型转换成`Pin<&mut dyn Future + Send + 'static>`
                if future.as_mut().poll(context).is_pending() {
                    // Future 还没执行完，因此将它放回任务中，等待下次被poll
                    *future_slot = Some(future);
                }
            }
        }
    }
}

// 使用执行器运行定时器
fn main() {
    let (executor, spawner) = new_executor_and_spawner();

    // 生成一个任务
    spawner.spawn(async {
        println!("howdy!");
        // 创建定时器Future，并等待它完成
        TimerFuture::new(Duration::new(2, 0)).await;
        println!("done!");
    });

    // drop掉任务，这样执行器就知道任务已经完成，不会再有新的任务进来
    drop(spawner);

    // 运行执行器直到任务队列为空
    // 任务运行后，会先打印`howdy!`, 暂停2秒，接着打印 `done!`
    executor.run();
}
```

### 执行器和 IO 复用
之前的 SocketRead 的例子中，`Future`将从 Socket 读取数据，若当前还没有数据，则会让出当前线程的所有权，允许执行器去执行其它的`Future`。当数据准备好后，会调用`wake()`函数将该`Future`的任务放入任务通道中，等待执行器的`poll`。这个例子中有一个 callback 方法是关键，只有它知道 socket 中的数据已经可以被读取了。那它的原理是什么？

其中一个简单粗暴的方法就是使用一个新线程不停的检查`socket`中是否有了数据，但 Rust 显然是不会这么干的，因为这样性能太低了。操作系统会提供 IO 多路复用机制，借助 IO 多路复用机制，可以实现一个线程同时阻塞地去等待多个异步 IO 事件，一旦某个事件完成就立即退出阻塞并返回数据。这是一个例子：
```rust
// rust 有一个 mio 包
struct IoBlocker {
    /* ... */
}

struct Event {
    // Event的唯一ID，该事件发生后，就会被监听起来
    id: usize,

    // 一组需要等待或者已发生的信号
    signals: Signals,
}

impl IoBlocker {
    /// 创建需要阻塞等待的异步IO事件的集合
    fn new() -> Self { /* ... */ }

    /// 对指定的IO事件表示兴趣
    fn add_io_event_interest(
        &self,

        /// 事件所绑定的socket
        io_object: &IoObject,

        event: Event,
    ) { /* ... */ }

    /// 进入阻塞，直到某个事件出现
    fn block(&self) -> Event { /* ... */ }
}

let mut io_blocker = IoBlocker::new();
io_blocker.add_io_event_interest(
    &socket_1,
    Event { id: 1, signals: READABLE },
);
io_blocker.add_io_event_interest(
    &socket_2,
    Event { id: 2, signals: READABLE | WRITABLE },
);
let event = io_blocker.block();

// 当socket的数据可以读取时，打印 "Socket 1 is now READABLE"
println!("Socket {:?} is now {:?}", event.id, event.signals);
```
这样，我们只需要一个执行器线程，它会接收 IO 事件并将其分发到对应的`Waker`中，接着后者会唤醒相关的任务，最终通过执行器`poll`后，任务可以顺利地继续执行, 这种 IO 读取流程可以不停的循环，直到 socket 关闭。














{{ admonition(type="warning", title="注意", text="施工中") }}

***
[^1]: 异步编程的模型有一大堆，包括线程、协程、actor模型等等，具体可以看原文
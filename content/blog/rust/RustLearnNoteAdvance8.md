+++
title = "Rust 进阶学习笔记（八）：异步编程"
slug = "rust_learn_note_adv_8"
date = 2025-07-16
updated = 2025-07-18
description = "aysnc、await和future，异步编程调度和执行器，Pin和UnPin，join和select"
# draft = true
[taxonomies]
tags = ["Rust", "Learn"]
[extra]
pinned = false
post_listing_date = "both"
+++

本节出处：[圣经-4.11异步编程](https://course.rs/advance/async/intro.html)

异步比多线程难多了，各种离谱的命名、离谱的设计给我看吐了，纯粹是靠强迫症坚持看下去的。后面会有一节是基于 [tokio](https://github.com/tokio-rs/tokio) 的实践，强迫自己看完本节后再去看 tokio 洗洗眼睛能舒服一些。

***

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

// 为任务实现 ArcWake 特质，这样它们就能被转变成 Waker 然后被唤醒:
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

## Pin
在之前的例子中看到过`Pin`，它可以防止一个类型在内存中被移动。还有一个`UnPin`表示类型可以在内存中安全地移动。

> 吐槽一下，这一小节简直难炸了，实在是恶心……辅助理解，可以看看这个 [Rust 的 Pin 与 Unpin](https://folyd.com/blog/rust-pin-unpin/) 和 [Rust Async: Pin概念解析](https://zhuanlan.zhihu.com/p/67803708)

### Why Pin？
<details>
<summary>这段意义有限，为防止理解出现偏差，先折叠了</summary>
一个`async`会创建一个实现了`Future`的匿名类型，并提供了一个`poll`方法：

```rust
let fut_one = /* ... */; // Future 1
let fut_two = /* ... */; // Future 2
async move {
    fut_one.await;
    fut_two.await;
}

// `async { ... }`语句块创建的 `Future` 类型
struct AsyncFuture {
    fut_one: FutOne,
    fut_two: FutTwo,
    state: State,
}

// `async` 语句块可能处于的状态
enum State {
    AwaitingFutOne,
    AwaitingFutTwo,
    Done,
}

impl Future for AsyncFuture {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        loop {
            match self.state {
                State::AwaitingFutOne => match self.fut_one.poll(..) {
                    Poll::Ready(()) => self.state = State::AwaitingFutTwo,
                    Poll::Pending => return Poll::Pending,
                }
                State::AwaitingFutTwo => match self.fut_two.poll(..) {
                    Poll::Ready(()) => self.state = State::Done,
                    Poll::Pending => return Poll::Pending,
                }
                State::Done => return Poll::Ready(()),
            }
        }
    }
}
```
正常情况下，当 poll 第一次被调用时，它会去查询 fut_one 的状态，若 fut_one 无法完成，则 poll 方法会返回。未来对 poll 的调用将从上一次调用结束的地方开始。该过程会一直持续，直到 Future 完成为止。

然而，如果我们的 async 语句块中使用了引用类型
```rust
async {
    let mut x = [0; 128];
    let read_into_buf_fut = read_into_buf(&mut x);
    read_into_buf_fut.await;
    println!("{:?}", x);
}
```
这段代码会编译成下面的形式：
```rust
struct ReadIntoBuf<'a> {
    buf: &'a mut [u8], // 指向下面的`x`字段
}

struct AsyncFuture {
    x: [u8; 128],
    read_into_buf_fut: ReadIntoBuf<'what_lifetime?>,
}
```
这里，ReadIntoBuf 拥有一个引用字段，指向了结构体的另一个字段 x ，一旦 AsyncFuture 被移动，那 x 的地址也将随之变化，此时对 x 的引用就变成了不合法的，也就是 read_into_buf_fut.buf 会变为不合法的。

若能将 Future 在内存中固定到一个位置，就可以避免这种问题的发生，也就可以安全的创建上面这种引用类型。
</details>

### Unpin
大多数类型都自动实现了`Unpin`特质，它表明一个类型可以随意被移动。`Unpin`是一个标记特质（不定义任何行为），与之相对，`Pin`是一个结构体：
```rust
#[stable(feature = "pin", since = "1.33.0")]
#[lang = "pin"]
#[fundamental]
#[repr(transparent)]
#[derive(Copy, Clone)]
// 包裹一个指针，并且能确保该指针指向的数据不会被移动
pub struct Pin<P> {
    pointer: P,
}

#[stable(feature = "pin", since = "1.33.0")]
impl<P: Deref> Deref for Pin<P> {
    type Target = P::Target;
    fn deref(&self) -> &P::Target {
        Pin::get_ref(Pin::as_ref(self))
    }
}
// !Unpin 不符合 Target: Unpin，故无法获取到可变引用
#[stable(feature = "pin", since = "1.33.0")]
impl<P: DerefMut<Target: Unpin>> DerefMut for Pin<P> {
    fn deref_mut(&mut self) -> &mut P::Target {
        Pin::get_mut(Pin::as_mut(self))
    }
}
```
可以被`Pin`住的类型会实现一个`!Unpin`特质，这个特质说明类型没有实现`Unpin`特质。对于实现了`Unpin`的类型，还是可以使用`Pin`，但是没有任何效果。因此，一个类型如果不能被移动，它必须实现`!Unpin`特质。

### 理解 Pin
`Pin`虽然名字叫“钉住”，但他的原理根本不是固定住。`Pin`的作用就是在实现了`!Unpin`的情况下阻止调用`get_mut`。

`Pin<&mut T>`，`Pin<&T>`，`Pin<Box<T>>`这样的数据结构，都能够确保 T 不会被移动。可以看出来，`Pin`是一个智能指针，包裹一个值，这个指针无法安全地获取到可变引用，那自然就不可移动了。至于为什么获取不到，可以看下上一节中`Pin`定义里`DerefMut`方法的签名。

`Pin`经常用于处理一些自引用的类型。自引用指的是了类型内部的某个成员是另一个成员的引用。来看一个例子：
```rust
#[derive(Debug)]
// 自引用结构体，b 是 a 的一个引用
struct Test {
    a: String,
    // 使用裸指针，因为其他方式会被 Rust 借用规则限制
    b: *const String,
}

// Test 提供方法用于获取字段 a 和 b 的值的引用。
impl Test {
    fn new(txt: &str) -> Self {
        Test {
            a: String::from(txt),
            b: std::ptr::null(),
        }
    }

    fn init(&mut self) {
        let self_ref: *const String = &self.a;
        self.b = self_ref;
    }

    fn a(&self) -> &str {
        &self.a
    }

    fn b(&self) -> &String {
        assert!(!self.b.is_null(), "Test::b called without Test::init being called first");
        unsafe { &*(self.b) }
    }
}

fn main() {
    let mut test1 = Test::new("test1");
    test1.init();
    let mut test2 = Test::new("test2");
    test2.init();

    println!("a: {}, b: {}", test1.a(), test1.b());
    // swap 将会交换结构体的所有字段，也就是把 B 的数据复制到 A 的地址，把 A 的数据复制到 B 的地址，也就是值拷贝
    std::mem::swap(&mut test1, &mut test2);
    println!("a: {}, b: {}", test2.a(), test2.b());
    // // 输出：
    // a: test1, b: test1
    // a: test1, b: test2
    // // test2.b 指针依然指向了旧的地址，而该地址对应的值现在在 test1 里，最终会打印出意料之外的值。
    test1.a = "I've totally changed now!".to_string();
    println!("a: {}, b: {}", test2.a(), test2.b());
    // a: test1, b: I've totally changed now! 
    // test2.b() 返回的是 &*test2.b，而 test2.b 指向原来的 test1.a，也就是现在的 test2.a
}
```
下面来用`Pin`解决问题。注意，一旦类型实现了`!Unpin`，那将它的值固定到栈上就是不安全的行为。
```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

#[derive(Debug)]
struct Test {
    a: String,
    b: *const String,
    // 这个标记可以让类型自动实现特质`!Unpin`
    // 这是一个“虚类型”
    _marker: PhantomPinned,
}

impl Test {
    fn new(txt: &str) -> Self {
        Test {
            a: String::from(txt),
            b: std::ptr::null(),
            _marker: PhantomPinned, 
        }
    }

    // 在栈上分配，通过 Pin<&mut Self> 固定引用
    fn init(self: Pin<&mut Self>) {
        let self_ptr: *const String = &self.a;
        let this = unsafe { self.get_unchecked_mut() };
        this.b = self_ptr;
    }

    fn a(self: Pin<&Self>) -> &str {
        &self.get_ref().a
    }

    fn b(self: Pin<&Self>) -> &String {
        assert!(!self.b.is_null(), "Test::b called without Test::init being called first");
        // !Unpin 是不安全的，需要进行标记
        unsafe { &*(self.b) }
    }
}

pub fn main() {
    // 此时的`test1`可以被安全的移动
    let mut test1 = Test::new("test1");
    // 新的`test1`由于使用了`Pin`，因此无法再被移动，这里的声明会将之前的`test1`遮蔽掉(shadow)
    let mut test1 = unsafe { Pin::new_unchecked(&mut test1) };
    Test::init(test1.as_mut());

    let mut test2 = Test::new("test2");
    let mut test2 = unsafe { Pin::new_unchecked(&mut test2) };
    Test::init(test2.as_mut());

    println!("a: {}, b: {}", Test::a(test1.as_ref()), Test::b(test1.as_ref()));
    // 此时编译时就会报错，保障代码正确性
    // the trait `Unpin` is not implemented for `PhantomPinned`
    std::mem::swap(test1.get_mut(), test2.get_mut());
    println!("a: {}, b: {}", Test::a(test2.as_ref()), Test::b(test2.as_ref()));
}

// fn main() {
//    let mut test1 = Test::new("test1");
//    // 如果忘记遮蔽，就可以 drop 掉 Pin，然后在生命周期结束后继续移动数据
//    let mut test1_pin = unsafe { Pin::new_unchecked(&mut test1) };
//    Test::init(test1_pin.as_mut());

//    drop(test1_pin);
//    println!(r#"test1.b points to "test1": {:?}..."#, test1.b);

//    let mut test2 = Test::new("test2");
//    mem::swap(&mut test1, &mut test2);
//    println!("... and now it points nowhere: {:?}", test1.b);
// }
```
与栈不同，将一个`!Unpin`类型的值固定到堆上会给予该值一个稳定的内存地址，它指向的堆中的值在`Pin`后是无法被移动的，并且堆上的值在整个生命周期内都会被稳稳地固定住。
```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

#[derive(Debug)]
struct Test {
    a: String,
    b: *const String,
    _marker: PhantomPinned,
}

impl Test {
    fn new(txt: &str) -> Pin<Box<Self>> {
        let t = Test {
            a: String::from(txt),
            b: std::ptr::null(),
            _marker: PhantomPinned,
        };
        // 使用 Box::pin 将结构体固定在堆上
        let mut boxed = Box::pin(t);
        let self_ptr: *const String = &boxed.as_ref().a;
        unsafe { boxed.as_mut().get_unchecked_mut().b = self_ptr };

        boxed
    }

    fn a(self: Pin<&Self>) -> &str {
        &self.get_ref().a
    }

    fn b(self: Pin<&Self>) -> &String {
        unsafe { &*(self.b) }
    }
}

pub fn main() {
    let test1 = Test::new("test1");
    let test2 = Test::new("test2");

    println!("a: {}, b: {}", test1.as_ref().a(), test1.as_ref().b());
    println!("a: {}, b: {}", test2.as_ref().a(), test2.as_ref().b());
    // // 输出
    // a: test1, b: test1
    // a: test2, b: test2
}
```

### 解冻 Future
`async`函数返回的`Future`默认就是`!Unpin`的。在实际应用中，一些函数会要求它们处理的`Future`是`Unpin`的，此时需要这样：
- `Box::pin`， 创建一个`Pin<Box<T>>`
- `pin_utils::pin_mut!`， 创建一个`Pin<&mut T>`
```rust
use pin_utils::pin_mut; // `pin_utils` 可以在crates.io中找到

// 函数的参数是一个`Future`，但是要求该`Future`实现`Unpin`
fn execute_unpin_future(x: impl Future<Output = ()> + Unpin) { /* ... */ }

let fut = async { /* ... */ };
// 下面代码报错: 默认情况下，`fut` 实现的是`!Unpin`，并没有实现`Unpin`
// execute_unpin_future(fut);

// 使用`Box`进行固定
let fut = async { /* ... */ };
let fut = Box::pin(fut);
execute_unpin_future(fut); // OK

// 使用`pin_mut!`进行固定
let fut = async { /* ... */ };
pin_mut!(fut);
execute_unpin_future(fut); // OK
```
固定后获得的`Pin<Box<T>>`和`Pin<&mut T>`既可以用于`Future`，又会自动实现`Unpin`。

### 总结
- 若`T: Unpin`（Rust 类型的默认实现），那么`Pin<'a, T>`跟`&'a mut T`完全相同，也就是`Pin`将没有任何效果, 该移动还是照常移动
- 绝大多数标准库类型都实现了`Unpin`，事实上，对于 Rust 中能遇到的绝大多数类型，该结论依然成立。其中一个例外就是：`async/await`生成的`Future`没有实现`Unpin`
- 可以通过以下方法为自己的类型添加 !Unpin 约束：
  - 使用 std::marker::PhantomPinned
  - 不稳定版有`impl !Unpin`的功能
- 可以将值固定到栈上，也可以固定到堆上
  - 将`!Unpin`值固定到栈上需要使用`unsafe`
  - 将`!Unpin`值固定到堆上无需`unsafe`，可以通过`Box`来简单的实现
- 当固定类型`T: !Unpin`时，你需要保证数据从被固定到被 drop 这段时期内，其内存不会变得非法或者被重用

## 异步的所有权和流处理
`async/await`在遇到阻塞操作时会让出当前线程的所有权而不是阻塞当前线程，这样就允许当前线程继续去执行其它代码，最终实现并发。

### 生命周期
有两种方式可以使用`async`：`async fn`用于声明函数，`async { ... }`用于声明语句块，它们会返回一个实现`Future`特质的值。

`async fn`函数如果拥有引用类型的参数，那它返回的`Future`的生命周期就会被这些参数的生命周期所限制。也就是说，被引用的数据生命周期至少与引用它的Future一样长。
```rust
// 当 x 依然有效时， 该 Future 就必须继续等待
// 也就是说 x 必须比 Future 活得更久
async fn foo(x: &u8) -> u8 { *x }

// 上面的函数跟下面的函数是等价的:
fn foo_expanded<'a>(x: &'a u8) -> impl Future<Output = u8> + 'a {
    async move { *x }
}
```
若`Future`被先存起来或发送到另一个任务或者线程，而不是调用`await`，就可能存在问题了：
```rust
use std::future::Future;
fn bad() -> impl Future<Output = u8> {
    let x = 5;
    // x 的生命周期只到 bad 函数的结尾，但是 Future 显然会活得更久
    // 此处只创建局部变量的引用显然是不够的
    borrow_x(&x) // ERROR: `x` does not live long enough
}

async fn borrow_x(x: &u8) -> u8 { *x }
```
常用的解决方法是使用具有静态生命周期的块：
```rust
use std::future::Future;

async fn borrow_x(x: &u8) -> u8 { *x }

fn good() -> impl Future<Output = u8> {
    async {
        // 生命周期扩展到 'static，并且和 Future 保持一致
        let x = 5;
        // 在正确的作用域内等待 Future 完成
        borrow_x(&x).await
    }
}
```

### 所有权移动
允许使用`async move`关键字来将环境中变量的所有权转移到语句块内，就像闭包那样。
```rust
// 多个不同的 `async` 语句块可以访问同一个本地变量，只要它们在该变量的作用域内执行
async fn blocks() {
    let my_string = "foo".to_string();

    let future_one = async {
        // ...
        println!("{my_string}");
    };

    let future_two = async {
        // ...
        println!("{my_string}");
    };

    // 运行两个 Future 直到完成
    let ((), ()) = futures::join!(future_one, future_two);
}

// 由于 `async move` 会捕获环境中的变量，因此只有一个 `async move` 语句块可以访问该变量，无法跟其它代码实现对变量的共享
// 但是它也有非常明显的好处：变量可以转移到返回的 Future 中，不再受借用生命周期的限制
fn move_block() -> impl Future<Output = ()> {
    let my_string = "foo".to_string();
    async move {
        // ...
        println!("{my_string}");
    }
}
```

### 多线程执行器
如果执行器是多线程的，`Future`内部的任何`.await`都可能导致它被切换到一个新线程上去执行。这就是说，`Future`可能会在线程间移动，因此`async`块中的变量必须要能在线程间传递。这会导致`Rc`、`RefCell`、没有实现`Send`的所有权类型、没有实现`Sync`的引用类型都不安全，无法在`.await`调用期间使用（不在作用域内还是可以用的）。同样的，普通的锁如`Mutex`也不可用（线程池会死锁），需要使用`futures::lock`来替代`Mutex`完成任务。

### 流处理
`Stream`特质类似于`Future`特质，但是在完成前可以生成多个值，这种行为跟标准库中的`Iterator`特质颇为相似。
```rust
trait Stream {
    // Stream 生成的值的类型
    type Item;

    // 尝试去解析 Stream 中的下一个值,
    // 若无数据，返回`Poll::Pending`, 若有数据，返回 `Poll::Ready(Some(x))`, `Stream`完成则返回 `Poll::Ready(None)`
    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<Option<Self::Item>>;
}
```
消息通道的接收者经常能够用到`Stream`：
```rust
async fn send_recv() {
    const BUFFER_SIZE: usize = 10;
    let (mut tx, mut rx) = mpsc::channel::<i32>(BUFFER_SIZE);

    tx.send(1).await.unwrap();
    tx.send(2).await.unwrap();
    drop(tx);

    // `StreamExt::next` 类似于 `Iterator::next`, 但是返回的不是值，而是一个 `Future<Output = Option<T>>`
    // 因此还需要使用`.await`来获取具体的值
    assert_eq!(Some(1), rx.next().await);
    assert_eq!(Some(2), rx.next().await);
    assert_eq!(None, rx.next().await);
}
```
当然也可以像`Iterator`那样迭代一个`Stream`，比如他支持`map`，`filter`，`fold`方法，以及它们对应的`try_map`，`try_filter`，`try_fold`。此外还有`next`和`try_next`：
```rust
async fn sum_with_next(mut stream: Pin<&mut dyn Stream<Item = i32>>) -> i32 {
    use futures::stream::StreamExt; // 引入 next
    let mut sum = 0;
    while let Some(item) = stream.next().await {
        sum += item;
    }
    sum
}

async fn sum_with_try_next(
    mut stream: Pin<&mut dyn Stream<Item = Result<i32, io::Error>>>,
) -> Result<i32, io::Error> {
    use futures::stream::TryStreamExt; // 引入 try_next
    let mut sum = 0;
    while let Some(item) = stream.try_next().await? {
        sum += item;
    }
    Ok(sum)
}
```
但是一次处理一个值还叫什么并发。于是还有并发方法`for_each_concurrent`和它对应的`try_for_each_concurrent`：
```rust
async fn jump_around(
    mut stream: Pin<&mut dyn Stream<Item = Result<u8, io::Error>>>,
) -> Result<(), io::Error> {
    use futures::stream::TryStreamExt; // 引入 `try_for_each_concurrent`，从一个 Stream 并发处理多个值
    const MAX_CONCURRENT_JUMPERS: usize = 100;

    stream.try_for_each_concurrent(MAX_CONCURRENT_JUMPERS, |num| async move {
        jump_n_times(num).await?;
        report_n_jumps(num).await?;
        Ok(())
    }).await?;

    Ok(())
}
```

## 多 Future 处理
`await`只能排队完成`Future`，如果希望同时运行多个任务，就要考虑其他方法。

### join!
在上面的例子中已经见过了 futures 包中的`join!`。`join!`宏允许同时等待多个不同`Future`的完成，并且可以让他们并发运行。
```rust
// Rust 中的 Future 是惰性的，直到调用 .await 时，才会开始运行
// 而两个 await 由于在代码中有先后顺序，因此是顺序运行的。
// async fn enjoy_book_and_music() -> (Book, Music) {
//     let book_future = enjoy_book();
//     let music_future = enjoy_music();
//     (book_future.await, music_future.await)
// }

use futures::join;

async fn enjoy_book_and_music() -> (Book, Music) {
    let book_fut = enjoy_book();
    let music_fut = enjoy_music();
    join!(book_fut, music_fut)
}
```
`join!`会返回一个元组，里面的值是对应的`Future`执行结束后输出的值。

如果希望同时运行一个数组里的多个异步任务，可以使用`futures::future::join_all`方法。

### try_join!
有了`join!`自然就有`try_join!`。如果希望在某一个`Future`报错后就立即停止所有`Future`的执行，就可以使用`try_join!`，特别是当`Future`返回`Result`时:
```rust
use futures::try_join;

async fn get_book() -> Result<Book, String> { /* ... */ Ok(Book) }
async fn get_music() -> Result<Music, String> { /* ... */ Ok(Music) }

async fn get_book_and_music() -> Result<(Book, Music), String> {
    let book_fut = get_book();
    let music_fut = get_music();
    try_join!(book_fut, music_fut)
}
```
注意，所有`Future`都必须拥有相同的错误类型。如果错误类型不同，可以考虑使用来自`futures::future::TryFutureExt`模块的`map_err`和`err_info`方法将错误转换成相同类型。
```rust
use futures::{
    future::TryFutureExt,
    try_join,
};

async fn get_book() -> Result<Book, ()> { /* ... */ Ok(Book) }
async fn get_music() -> Result<Music, String> { /* ... */ Ok(Music) }

async fn get_book_and_music() -> Result<(Book, Music), String> {
    let book_fut = get_book().map_err(|()| "Unable to get book".to_string());
    let music_fut = get_music();
    try_join!(book_fut, music_fut)
}
```

### select!
`join!`只有等所有`Future`结束后，才能集中处理结果，如果想同时等待多个`Future`，且任何一个`Future`结束后，都可以立即被处理，可以考虑使用`futures::select!`。
```rust
use futures::{
    future::FutureExt, // for `.fuse()`
    pin_mut,
    select,
};

async fn task_one() { /* ... */ }
async fn task_two() { /* ... */ }

async fn race_tasks() {
    // fuse 将一个 Future 转换为 FusedFuture
    // 防止在完成后再被 poll 导致错误，确保 select! 宏安全使用
    let t1 = task_one().fuse();
    let t2 = task_two().fuse();

    pin_mut!(t1, t2);

    select! {
        () = t1 => println!("任务1率先完成"),
        () = t2 => println!("任务2率先完成"),
    }
}
```
`select!`在“选择”一个分支调用后会立即结束，不会等待其他任务的完成。

`select!`也支持 complete 和 default 分支：
```rust
use futures::future;
use futures::select;
pub fn main() {
    // ready 的 future 会立即完成
    let mut a_fut = future::ready(4);
    let mut b_fut = future::ready(6);
    let mut total = 0;

    loop {
        select! {
            a = a_fut => total += a,
            b = b_fut => total += b,
            // 当所有的 Future 和 Stream 完成后才会被执行
            complete => break,
            // 没有任何 Future 或 Stream 处于 Ready 状态， 则该分支会被立即执行
            default => panic!(), // 该分支永远不会运行，因为 `Future` 会先运行，然后是 `complete`
            // 如果 complete 不 break，default 分支就能触发
        };
    }
    assert_eq!(total, 10);
}
```

### FuseFuture
之前的例子中出现过 fuse，这里将其展开。

`.fuse`方法可以让`Future`实现`FusedFuture`特质，而`pin_mut!`宏会为`Future`实现`Unpin`特质，这两个特质恰恰是使用`select!`所必须的：
- Unpin：由于 select 不会通过拿走所有权的方式使用`Future`，而是通过可变引用的方式去使用，这样当 select 结束后，该`Future`若没有被完成，它的所有权还可以继续被其它代码使用。注意，对于来自表达式的 future 可以放宽`Unpin`的限制（会自动变换）。
- FusedFuture：`Future`一旦完成后，那 select 就不能再对其进行轮询使用。Fuse 意味着熔断，相当于`Future`一旦完成，再次调用 poll 会直接返回 Poll::Pending。

只有实现了`FusedFuture`，select 才能配合 loop 一起使用。假如没有实现，就算一个 Future 已经完成了，它依然会被 select 不停的轮询执行。

`Stream` 稍有不同，它们使用的特质是`FusedStream`。通过 fuse 实现了该特质的 Stream，对其调用 next 或 try_next 方法可以获取实现了`FusedFuture`特质的`Future`:
```rust
use futures::{
    stream::{Stream, StreamExt, FusedStream},
    select,
};

async fn add_two_streams(
    mut s1: impl Stream<Item = u8> + FusedStream + Unpin,
    mut s2: impl Stream<Item = u8> + FusedStream + Unpin,
) -> u8 {
    let mut total = 0;

    loop {
        let item = select! {
            x = s1.next() => x,
            x = s2.next() => x,
            complete => break,
        };
        if let Some(next_num) = item {
            total += next_num;
        }
    }

    total
}
```

### select 时并发
`Fuse::terminated()`函数可以构建一个空的`Future`。

考虑以下场景：在 select 循环中运行一个任务，但是该任务却也是在 select 循环内部创建的。
```rust
use futures::{
    future::{Fuse, FusedFuture, FutureExt},
    stream::{FusedStream, Stream, StreamExt},
    pin_mut,
    select,
};

async fn get_new_num() -> u8 { /* ... */ 5 }

async fn run_on_new_num(_: u8) { /* ... */ }

async fn run_loop(
    mut interval_timer: impl Stream<Item = ()> + FusedStream + Unpin,
    starting_num: u8,
) {
    let run_on_new_num_fut = run_on_new_num(starting_num).fuse();
    let get_new_num_fut = Fuse::terminated();
    pin_mut!(run_on_new_num_fut, get_new_num_fut);
    loop {
        select! {
            () = interval_timer.select_next_some() => {
                // 定时器已结束，若`get_new_num_fut`没有在运行，就创建一个新的
                if get_new_num_fut.is_terminated() {
                    get_new_num_fut.set(get_new_num().fuse());
                }
            },
            new_num = get_new_num_fut => {
                // 收到新的数字 -- 创建一个新的`run_on_new_num_fut`并丢弃掉旧的
                run_on_new_num_fut.set(run_on_new_num(new_num).fuse());
            },
            // 运行 `run_on_new_num_fut`
            () = run_on_new_num_fut => {},
            // 若所有任务都完成，直接 `panic`， 原因是 `interval_timer` 应该连续不断的产生值，而不是结束
            //后，执行到 `complete` 分支
            complete => panic!("`interval_timer` completed unexpectedly"),
        }
    }
}
```
当某个 future 有多个拷贝都需要同时运行时，可以使用`FuturesUnordered`类型。下面的例子会将`run_on_new_num_fut`的每一个拷贝都运行到完成，而不是像之前那样一旦创建新的就终止旧的。
```rust
use futures::{
    future::{Fuse, FusedFuture, FutureExt},
    stream::{FusedStream, FuturesUnordered, Stream, StreamExt},
    pin_mut,
    select,
};

async fn get_new_num() -> u8 { /* ... */ 5 }

async fn run_on_new_num(_: u8) -> u8 { /* ... */ 5 }

// 使用从 `get_new_num` 获取的最新数字 来运行 `run_on_new_num`
//
// 每当计时器结束后，`get_new_num` 就会运行一次，它会立即取消当前正在运行的`run_on_new_num` ,
// 并且使用新返回的值来替换
async fn run_loop(
    mut interval_timer: impl Stream<Item = ()> + FusedStream + Unpin,
    starting_num: u8,
) {
    let mut run_on_new_num_futs = FuturesUnordered::new();
    run_on_new_num_futs.push(run_on_new_num(starting_num));
    let get_new_num_fut = Fuse::terminated();
    pin_mut!(get_new_num_fut);
    loop {
        select! {
            () = interval_timer.select_next_some() => {
                 // 定时器已结束，若 `get_new_num_fut` 没有在运行，就创建一个新的
                if get_new_num_fut.is_terminated() {
                    get_new_num_fut.set(get_new_num().fuse());
                }
            },
            new_num = get_new_num_fut => {
                 // 收到新的数字 -- 创建一个新的 `run_on_new_num_fut` (并没有像之前的例子那样丢弃掉旧值)
                run_on_new_num_futs.push(run_on_new_num(new_num));
            },
            // 运行 `run_on_new_num_futs`, 并检查是否有已经完成的
            res = run_on_new_num_futs.select_next_some() => {
                println!("run_on_new_num_fut returned {:?}", res);
            },
            // 若所有任务都完成，直接 `panic`， 原因是 `interval_timer` 应该连续不断的产生值，而不是结束
            //后，执行到 `complete` 分支
            complete => panic!("`interval_timer` completed unexpectedly"),
        }
    }
}
```

## 异步疑难杂症
原文还有一节 [Rust 异步疑难杂症](https://course.rs/advance/async/pain-points-and-workarounds.html)，里面都是异步编程的限制。按我目前浅薄的见识根本理解不了。俺不中嘞，感兴趣的直接看原文吧。

## 阅读材料
[Asynchronous Programming in Rust](https://rust-lang.github.io/async-book/intro.html)

***
[^1]: 异步编程的模型有一大堆，包括线程、协程、actor模型等等，具体可以看原文
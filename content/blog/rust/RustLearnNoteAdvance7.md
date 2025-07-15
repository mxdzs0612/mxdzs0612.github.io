+++
title = "Rust 进阶学习笔记（七）：多线程"
slug = "rust_learn_note_adv_7"
date = 2025-07-15
updated = 2025-07-15
description = "并发和并行，多线程创建与线程同步，线程安全"
# draft = true
[taxonomies]
tags = ["Rust", "Learn"]
[extra]
pinned = false
post_listing_date = "both"
+++

本节出处：[圣经 多线程](https://course.rs/advance/concurrency-with-threads/intro.html)

***

Rust 选择使用复杂度换取可控性和性能，其多线程是与`async`/`await`相结合的。

## 并发和并行
![并发和并行](https://pic1.zhimg.com/80/f37dd89173715d0e21546ea171c8a915_1440w.png)[^1]

从图中可以看出：
- 并发（Concurrent）是多个队列使用同一个咖啡机，然后两个队列轮换着使用（未必是 1:1 轮换，也可能是其它轮换规则），最终每个人都能接到咖啡。
- 并行（Parallel）是每个队列都拥有一个咖啡机，最终也是每个人都能接到咖啡，但是效率更高，因为同时可以有两个人在接咖啡。

总之，并发和并行都是对“多任务”处理的描述，其中并发是轮流处理，而并行是同时处理。

在只有 1 核 CPU 时，我们就需要并发来处理多个线程的任务队列。并发的关键在于：快速轮换处理不同的任务，给用户带来所有任务同时在运行的假象。当 CPU 核心增多时，同一时间就能有多个任务并行得到处理。并发与并行是同时在发生的，所有用户任务从表面来看都在并发的运行，但实际上，同一时刻只有 CPU 核数个任务能被同时并行的处理。

如果某个系统支持两个或者多个动作的同时**存在**，那么这个系统就是一个并发系统。如果某个系统支持两个或者多个动作同时**执行**，那么这个系统就是一个并行系统。可以认为“并行”概念是“并发”概念的一个子集。

不同语言对于线程的实现可能大相径庭：
- 由于操作系统提供了创建线程的 API，因此部分语言会直接调用该 API 来创建线程，因此最终程序内的线程数和该程序占用的操作系统线程数相等，一般称之为1:1 线程模型，例如 Rust。
- 还有些语言在内部实现了自己的线程模型（绿色线程、协程），程序内部的 M 个线程最后会以某种映射方式使用 N 个操作系统线程去运行，因此称之为M:N 线程模型，其中 M 和 N 并没有特定的彼此限制关系。一个典型的代表就是 Go 语言。
- 还有些语言使用了 Actor 模型，基于消息传递进行并发，例如 Erlang 语言。

最终选择了尽量小的运行时（runtime）实现。

> 运行时是那些会被打包到所有程序可执行文件中的 Rust 代码。小运行时的其中一个好处在于最终编译出的可执行文件会相对较小，同时也让该语言更容易被其它语言引入使用。

Rust 中也存在 M:N 模型，这些模型由三方库提供了实现，例如大名鼎鼎的 [tokio](https://github.com/tokio-rs/tokio)。

## 多线程
多线程的代码是同时运行的，难以保证线程间执行的顺序，因此会导致一些问题：
- 竞态条件（race conditions），多个线程以非一致性的顺序同时访问数据资源
- 死锁（deadlocks），两个线程都想使用某个资源，但是又都在等待对方释放资源后才能使用，结果最终都无法继续执行

使用多线程时，务必格外小心。

### 创建线程
```rust
use std::thread;
use std::time::Duration;

fn main() {
    thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {} from the spawned thread!", i);
            thread::sleep(Duration::from_millis(1));
        }
    });
    // 每次输出都不太一样
    for i in 1..5 {
        println!("hi number {} from the main thread!", i);
        thread::sleep(Duration::from_millis(1));
    }
}
```
使用标准库中的`thread::spawn`可以创建线程。注意：
- 线程内部的代码使用闭包来执行
- main 线程一旦结束，程序就立刻结束，因此需要保持它的存活，直到其它子线程完成自己的任务
- `thread::sleep`会让当前线程休眠指定的时间，此时如有其他线程，CPU 会调度到其他线程上运行。

### 等待子线程结束
在多线程程序中，主线程会在子线程完成前提前结束，导致子线程也随之结束。更过分的是，如果当前系统繁忙，甚至该子线程可能还没被创建，主线程就已经结束了。因此需要一个方法，让主线程安全、可靠地等所有子线程完成任务后再停止。
```rust
use std::thread;
use std::time::Duration;

fn main() {
    let handle = thread::spawn(|| {
        for i in 1..5 {
            println!("hi number {} from the spawned thread!", i);
            thread::sleep(Duration::from_millis(1));
        }
    });
    // `handle.join` 可以让当前线程阻塞，直到它等待的子线程的结束。
    handle.join().unwrap();

    for i in 1..5 {
        println!("hi number {} from the main thread!", i);
        thread::sleep(Duration::from_millis(1));
    }
    // 如果 `handle.join` 在这里调用，两个线程会交替输出
    // handle.join() 在主线程循环结束后才调用，所以主线程不会等待子线程完成就会执行自己的循环，但是会等待子线程完成才退出
    // 而两个线程几乎同时启动，并且睡眠时间相同，所以大致会交替打印
    // handle.join().unwrap();
}
```

### 所有权转移
默认情况下，子线程的闭包中不能捕获了环境中的变量，因为多个线程的结束顺序并不是固定的，Rust 无法确定新线程能活多久，所以无法确定新线程引用的变量在使用过程中是否一直合法。

多线程的闭包中，`move`可将所有权从一个线程转移到另外一个线程。Rust 的所有权机制保证了数据使用上的安全：所有权被转移给新的线程后，原来的线程将无法继续使用该变量。
```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];
    // 如果不加 move，会报错
    // closure may outlive the current function, but it borrows `v`, which is owned by the current function
    let handle = thread::spawn(move || {
        println!("Here's a vector: {:?}", v);
    });

    handle.join().unwrap();

    // 下面代码会报错borrow of moved value: `v`
    // println!("{:?}",v);
}
```

### 线程结束
Rust 没有直接提供结束线程的接口，因为直接终止线程可能导致其持有的资源没有释放，状态出现混乱，和 Rust 的设计理念相悖。

实际上，Rust 线程的代码执行完，线程就会自动结束。如果线程中的代码不执行完：
- 线程的任务是一个循环 IO 读取，大多数时候都处于阻塞状态，等待消息传入，只会占用少量 CPU。
- 线程的任务是一个循环，里面没有任何阻塞，包括休眠这种操作也没有，此时 CPU 很不幸的会被跑满，而且你如果没有设置终止条件，该线程将持续跑满一个 CPU 核心，并且不会被终止，直到`main`线程的结束，如下例：
```rust
use std::thread;
use std::time::Duration;
fn main() {
    // 创建一个线程A
    let new_thread = thread::spawn(move || {
        // 再创建一个线程B
        thread::spawn(move || {
            loop {
                println!("I am a new thread.");
            }
        })
    });

    // 等待新创建的线程执行完成
    new_thread.join().unwrap();
    println!("Child thread is finish!");

    // 睡眠一段时间，看子线程创建的子线程是否还在运行
    thread::sleep(Duration::from_millis(100));
    // 可以看到 A 线程在创建完 B 线程后就立即结束了，而 B 线程则在不停地循环输出。
}
```
可见，Rust 的子线程具有独立性。在 Rust 中，一旦线程被创建，它就作为一个独立的执行单元存在于进程中。父线程不是`main`线程时，其并不会在终止时自动终止其创建的子线程。

### 性能
创建线程的开销是不可忽略的。因此，只有当真的需要处理一个值得用线程去处理的任务时，才使用线程。

受 CPU 的核心数限制，当任务是 CPU 密集型时，就算线程数超过了 CPU 核心数，也并不能帮你获得更好的性能，因为每个线程的任务都可以轻松让 CPU 的某个核心跑满。既然如此，让线程数等于 CPU 核心数是最好的。

但是当你的任务大部分时间都处于阻塞状态时，就可以考虑增多线程数量，这样当某个线程处于阻塞状态时，会被切走，进而运行其它的线程，典型就是网络 IO 操作，我们可以为每一个进来的用户连接创建一个线程去处理，该连接绝大部分时间都是处于 IO 读取阻塞状态，因此有限的 CPU 核心完全可以处理成百上千的用户连接线程，但是事实上，对于这种网络 IO 情况，一般都不再使用多线程的方式了，毕竟操作系统的线程数是有限的，意味着并发数也很容易达到上限，而且过多的线程也会导致线程上下文切换的代价过大，使用 async/await 的 M:N 并发模型，就没有这个烦恼。

多线程的开销往往是在锁、数据竞争、缓存失效上，这些限制了现代化软件系统随着 CPU 核心的增多性能也线性增加的野心。

### 线程屏障
在 Rust 中，可以使用线程屏障（Barrier）让多个线程都执行到某个点后，才继续一起往后执行。
```rust
use std::sync::{Arc, Barrier};
use std::thread;

fn main() {
    let mut handles = Vec::with_capacity(6);
    let barrier = Arc::new(Barrier::new(6));

    for _ in 0..6 {
        let b = barrier.clone();
        handles.push(thread::spawn(move|| {
            println!("before wait");
            // 所有线程会先打印 before wait，再打印 after wait
            b.wait();
            println!("after wait");
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }
}
```

### 局部变量
对于多线程编程，线程局部变量（Thread Local Variable）在一些场景下非常有用。

#### thread_local!
使用`thread_local`宏可以初始化线程局部变量，然后在线程内部使用该变量的`with`方法获取变量值：
```rust
use std::cell::RefCell;
use std::thread;

// 创建线程局部变量 FOO，每个新的线程访问它时，都会使用它的初始值作为开始，各个线程中的 FOO 值彼此互不干扰。
// FOO 使用 static 声明为生命周期为 'static 的静态变量。
thread_local!(static FOO: RefCell<u32> = RefCell::new(1));

FOO.with(|f| {
    // 线程中对 FOO 的使用是通过借用的方式
    assert_eq!(*f.borrow(), 1);
    *f.borrow_mut() = 2;
});

// 每个线程开始时都会拿到线程局部变量的FOO的初始值
let t = thread::spawn(move|| {
    FOO.with(|f| {
        assert_eq!(*f.borrow(), 1);
        *f.borrow_mut() = 3;
    });
});

// 等待线程完成
t.join().unwrap();

// 尽管子线程中修改为了 3，我们在这里依然拥有main线程中的局部值：2
FOO.with(|f| {
    assert_eq!(*f.borrow(), 2);
});
```
结构体中：
```rust
use std::cell::RefCell;
use std::thread::LocalKey;

thread_local! {
    static FOO: RefCell<usize> = RefCell::new(0);
}
struct Bar {
    foo: &'static LocalKey<RefCell<usize>>,
}
impl Bar {
    fn constructor() -> Self {
        Self {
            foo: &FOO,
        }
    }
}
```
另一种方式：
```rust
use std::cell::RefCell;
use std::thread::LocalKey;

thread_local! {
    static FOO: RefCell<usize> = RefCell::new(0);
}
struct Bar {
    foo: &'static LocalKey<RefCell<usize>>,
}
impl Bar {
    fn constructor() -> Self {
        Self {
            foo: &FOO,
        }
    }
}
```

### thread-local-rs 库
这是一个[第三方库](https://github.com/Amanieu/thread_local-rs)，允许每个线程持有值的独立拷贝。
```rust
use thread_local::ThreadLocal;
use std::sync::Arc;
use std::cell::Cell;
use std::thread;

let tls = Arc::new(ThreadLocal::new());
let mut v = vec![];
// 创建多个线程
for _ in 0..5 {
    let tls2 = tls.clone();
    let handle = thread::spawn(move || {
        // 将计数器加 1
        // 请注意，由于线程 ID 在线程退出时会被回收，因此一个线程有可能回收另一个线程的对象
        // 这只能在线程退出后发生，因此不会导致任何竞争条件
        let cell = tls2.get_or(|| Cell::new(0));
        cell.set(cell.get() + 1);
    });
    v.push(handle);
}
for handle in v {
    handle.join().unwrap();
}
// 一旦所有子线程结束，收集它们的线程局部变量中的计数器值，然后进行求和
let tls = Arc::try_unwrap(tls).unwrap();
let total = tls.into_iter().fold(0, |x, y| {
    // 打印每个线程局部变量中的计数器值，发现不一定有 5 个线程，
    // 因为一些线程已退出，并且其他线程会回收退出线程的对象
    println!("x: {}, y: {}", x, y.get());
    x + y.get()
});

// 和为5
assert_eq!(total, 5);
```
该库不仅仅使用了值的拷贝，而且还能自动把多个拷贝汇总到一个迭代器中，最后进行求和，非常好用。

### Call Once
我们会需要某个函数在多线程环境下只被调用一次（Call Once），例如初始化全局变量，无论是哪个线程先调用函数来初始化，都会保证全局变量只会被初始化一次，随后的其它线程调用就会忽略该函数。

`Once::call_once`方法就能够做到执行且仅执行一次初始化。如果当前有另一个初始化过程正在运行，线程将阻止该方法被调用。

当这个函数返回时，保证一些初始化已经运行并完成，它还保证由执行的闭包所执行的任何内存写入都能被其他线程在这时可靠地观察到。
```rust
use std::thread;
use std::sync::Once;

static mut VAL: usize = 0;
static INIT: Once = Once::new();

fn main() {
    let handle1 = thread::spawn(move || {
        INIT.call_once(|| {
            unsafe {
                VAL = 1;
            }
        });
    });

    let handle2 = thread::spawn(move || {
        INIT.call_once(|| {
            unsafe {
                VAL = 2;
            }
        });
    });

    handle1.join().unwrap();
    handle2.join().unwrap();
    // 线程初始化是异步的，且耗时较久
    // 代码运行的结果取决于哪个线程先调用 INIT.call_once，可能是 1 或 2
    println!("{}", unsafe { VAL });
}
```

### 小结
Rust 的线程模型是 1:1 模型，因为 Rust 要保持尽量小的运行时。

我们可以使用`thread::spawn`来创建线程，创建出的多个线程之间并不存在执行顺序关系，因此代码逻辑千万不要依赖于线程间的执行顺序。

`main`线程若是结束，则所有子线程都将被终止，如果希望等待子线程结束后，再结束 `main`线程，你需要使用创建线程时返回的句柄的`join`方法。

在线程中无法直接借用外部环境中的变量值，因为新线程的启动时间点和结束时间点是不确定的，所以 Rust 无法保证该线程中借用的变量在使用过程中依然是合法的。你可以使用`move`关键字将变量的所有权转移给新的线程，来解决此问题。

父线程结束后，子线程仍在持续运行，直到子线程的代码运行完成或者`main`线程的结束。

## 消息传递
在多线程间有多种方式可以共享、传递数据，最常用的方式就是消息传递。

### 消息通道
Rust 在标准库里提供了消息通道（channel），一个通道支持多个发送者和接收者。

标准库提供了通道`std::sync::mpsc`，其中`mpsc`是“multiple producer, single consumer”的缩写，代表了该通道支持（一或）多个发送者，但是只支持唯一的接收者。如果需要多个接收者，可以考虑第三方库 [crossbeam-channel](https://github.com/crossbeam-rs/crossbeam/tree/master/crossbeam-channel) 或 [flume](https://github.com/zesterer/flume)。

#### 单发送者
消息收发使用`send`和`recv`方法，当然还存在一个`try_recv`。如下例：
```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    // 创建一个消息通道, 返回一个元组：(发送者，接收者)
    let (tx, rx) = mpsc::channel();

    // 创建线程，并发送消息
    // 需要使用 move 将 tx 的所有权转移到子线程的闭包中
    thread::spawn(move || {
        // 发送一个数字 1, send 方法返回 Result<T,E>，通过unwrap进行快速错误处理
        tx.send(1).unwrap();
        // 发送者和接收者的类型由编译器自动推导，发送过什么就推导成什么
        // 下面代码将报错，因为编译器自动推导出通道传递的值是 i32 类型，那么 Option<i32> 类型将产生不匹配错误
        // tx.send(Some(1)).unwrap()
    });

    // 在主线程中接收子线程发送的消息并输出
    // 接收消息的操作 rx.recv() 会阻塞当前线程，直到读取到值，或者通道被关闭
    println!("receive {}", rx.recv().unwrap());
    // try_recv 方法并不会阻塞线程，当通道中没有消息时，它会立刻返回一个错误并结束主线程。大部分情况下，子线程是来不及创建的。
    // println!("receive {:?}", rx.try_recv());
    // 如果多次打印，可能会看到不同的原因：
    // receive Err(Empty)
    // receive Ok(1)
    // receive Err(Disconnected)
}
```
`send`和`recv`方法都可能返回错误。比如另一方被 drop 导致不会有内容发送或发送的内容不会被接收，此时就应该返回一个错误。实际生产过程中应该使用错误处理来替代`unwrap`。

如果连续发送多条消息，可以循环接收：
```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let vals = vec![
            String::from("hi"),
            String::from("from"),
            String::from("the"),
            String::from("thread"),
        ];

        for val in vals {
            tx.send(val).unwrap();
            thread::sleep(Duration::from_secs(1));
        }
    });

    // 子线程运行完成时，发送者 tx 会随之被 drop
    // 此时通道也会被关闭，接收方顺利退出
    for received in rx {
        println!("Got: {}", received);
    }
}
```

#### 所有权
消息通道也要遵守所有权规则：
- 若值的类型实现了 Copy 特质，则直接复制一份该值，然后传输过去。如上例中的 i32
- 若值没有实现 Copy，则它的所有权会被转移给接收端，在发送端继续使用该值将报错
```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let s = String::from("我，飞走咯!");
        tx.send(s).unwrap();
        // value borrowed here after move
        // println!("val is {}", s);
    });

    let received = rx.recv().unwrap();
    println!("Got: {}", received);
}
```

#### 多发送者
由于子线程会拿走发送者的所有权，因此在有多发送者的时候，我们必须对发送者进行克隆，然后让每个线程拿走它的一份拷贝：
```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();
    // clone 的开销和创建线程相比并不大
    let tx1 = tx.clone();
    thread::spawn(move || {
        tx.send(String::from("hi from raw tx")).unwrap();
    });

    thread::spawn(move || {
        tx1.send(String::from("hi from cloned tx")).unwrap();
    });
    // 最终主线程的输出顺序是不确定的，但并不是无序的，会遵循先进先出（FIFO）原则
    // 需要所有的发送者都被 drop 掉后，接收者 rx 才会收到错误，进而跳出 for 循环，最终结束主线程
    for received in rx {
        println!("Got: {}", received);
    }
}
```
注意这里是有坑的。
```rust
use std::sync::mpsc;
fn main() {
    use std::thread;

    let (send, recv) = mpsc::channel();
    let num_threads = 3;
    for i in 0..num_threads {
        // 子线程的每一个发送者拿到的都是 clone 的 send
        let thread_send = send.clone();
        thread::spawn(move || {
            thread_send.send(i).unwrap();
            println!("thread {:?} finished", i);
        });
    }
    // 不加这一行，主线程会一直阻塞
    // drop(send);

    // send 本身被未使用，循环不会结束，需要手动 drop
    for x in recv {
        println!("Got: {}", x);
    }
    println!("finished iterating");
}
```

#### 关闭通道
从上面的例子中可以看出，当所有发送者被 drop 或者所有接收者被 drop 后，通道会自动关闭。这里的`Drop`是一个特质，能够在编译期实现，不会造成性能损耗。

### 同步通道
上文的消息通道是异步通道，也就是无论接收者是否正在接收消息，消息发送者在发送消息时都不会阻塞。
```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;
fn main() {
    let (tx, rx)= mpsc::channel();

    let handle = thread::spawn(move || {
        println!("发送之前");
        tx.send(1).unwrap();
        println!("发送之后");
    });

    println!("睡眠之前");
    thread::sleep(Duration::from_secs(3));
    println!("睡眠之后");

    println!("receive {}", rx.recv().unwrap());
    handle.join().unwrap();
    // // 输出：
    // 睡眠之前
    // 发送之前
    // 发送之后
    // // ···睡眠3秒
    // 睡眠之后
    // receive 1
}
```
而同步通道发送消息是阻塞的，只有在消息被接收后才解除阻塞。同步通道通过`sync_channel`创建。
```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;
fn main() {
    let (tx, rx)= mpsc::sync_channel(0);

    let handle = thread::spawn(move || {
        println!("发送之前");
        tx.send(1).unwrap();
        println!("发送之后");
    });

    println!("睡眠之前");
    thread::sleep(Duration::from_secs(3));
    println!("睡眠之后");

    println!("receive {}", rx.recv().unwrap());
    handle.join().unwrap();
    // // 输出：
    // 睡眠之前
    // 发送之前
    // // ···睡眠3秒
    // 睡眠之后
    // receive 1
    // 发送之后
}
```
可以看到，只有当接收消息彻底成功后，发送消息才算完成。

#### 消息缓存
眼尖的读者可能能够发现，`sync_channel`方法传入了一个数字参数。该值可以用来指定同步通道的消息缓存条数，设定为 N 时，发送者就可以无阻塞地往通道中发送 N 条消息，当消息缓冲队列满了后，新的消息发送将被阻塞。如果没有接收者消费缓冲队列中的消息，那么第 N+1 条消息就将触发发送阻塞。
```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;
fn main() {
    // 缓存了 1 条消息
    let (tx, rx)= mpsc::sync_channel(1);

    let handle = thread::spawn(move || {
        println!("首次发送之前");
        tx.send(1).unwrap();
        println!("首次发送之后");
        tx.send(1).unwrap();
        println!("再次发送之后");
    });

    println!("睡眠之前");
    thread::sleep(Duration::from_secs(3));
    println!("睡眠之后");

    println!("receive {}", rx.recv().unwrap());
    handle.join().unwrap();
    // 输出：
    // 睡眠之前
    // 首次发送之前
    // 首次发送之后
    // //···睡眠3秒
    // 睡眠之后
    // receive 1
    // 再次发送之后
}
```
使用异步消息虽然能非常高效且不会造成发送线程的阻塞（可以认为容量是内存大小），但是存在消息未及时消费，最终内存过大的问题。在实际项目中，可以考虑使用一个带缓冲值的同步通道来避免这种风险。

### 多数据类型
想要让通道传递多种数据类型，当然可以实现多个通道，但也可以使用枚举类型来实现。
```rust
use std::sync::mpsc::{self, Receiver, Sender};

enum Fruit {
    Apple(u8),
    Orange(String)
}

fn main() {
    let (tx, rx): (Sender<Fruit>, Receiver<Fruit>) = mpsc::channel();

    tx.send(Fruit::Orange("sweet".to_string())).unwrap();
    tx.send(Fruit::Apple(2)).unwrap();

    for _ in 0..2 {
        match rx.recv().unwrap() {
            Fruit::Apple(count) => println!("received {} apples", count),
            Fruit::Orange(flavor) => println!("received {} oranges", flavor),
        }
    }
}
```
Rust 会按照枚举中占用内存最大的那个成员进行内存对齐，这意味着就算你传输的是枚举中占用内存最小的成员，它占用的内存依然和最大的成员相同, 因此会造成内存上的浪费。

## 共享内存
除消息传递外，共享内存也是实现同步性的方式。通过锁或原子操作等并发原语，能够实现多个线程安全地访问同一个资源。消息传递的底层实际上也是通过共享内存来实现，但二者又有区别：
| 共享内存 | 消息传递 |
| --- | --- |
| 节省多次内存拷贝的成本 | 模拟现实世界，发送消息去通知某个目标执行相应的操作 |
| 实现简洁性能更高 | 可靠和简单 |
| 锁竞争更多 | 需要任务处理流水线/管道 |

总之，消息传递类似一个单所有权的系统：一个值同时只能有一个所有者，如果另一个线程需要该值的所有权，需要将所有权通过消息传递进行转移；共享内存类似于一个多所有权的系统：多个线程可以同时访问同一个值。

### 互斥锁
互斥锁Mutex（mutual exclusion 的缩写）让多个线程并发的访问同一个值变成了排队访问：同一时间，只允许一个线程 A 访问该值，其它线程需要等待 A 访问完成后才能继续。

#### 单线程中的互斥锁
```rust
use std::sync::Mutex;

fn main() {
    // 使用`Mutex`结构体的关联函数创建新的互斥锁实例
    let m = Mutex::new(5);

    {
        // 获取锁，然后 deref 为 `m` 的引用
        // lock 返回的是 Result，unwrap 得到 MutexGuard
        let mut num = m.lock().unwrap();
        // MutexGuard 实现了 Deref 特质，会被自动解引用后获得一个引用类型，该引用指向 Mutex 内部的数据
        *num = 6;
        // MutexGuard 还实现了 Drop 特质，在超出作用域后，自动释放锁，以便其它线程能继续获取锁
        // 如果没有这个代码块限制作用域，就需要手动释放锁
    }
    
    println!("m = {:?}", m); // 输出 m = Mutex { data: 6, poisoned: false, .. }
}
```
和`Box`类似，数据被`Mutex`所拥有，要访问内部的数据，需要使用方法`m.lock()`向 m 申请一个锁, 该方法会阻塞当前线程，直到获取到锁，因此当多个线程同时访问该数据时，只有一个线程能获取到锁，其它线程只能阻塞着等待，这样就保证了数据能被安全的修改。如果获取不到锁（例如当前持有锁的线程 panic! 了），lock 方法会返回一个错误。

正因为`Mutex`是一个智能指针，我们无需任何操作就能获取其中的数据，释放锁时也只需要做好作用域管理（这是说，如果同一线程中锁没有 drop 就再次尝试申请，线程就会阻塞）。

#### 多线程中的互斥锁
显然，正常情况下锁是不会在单线程程序中使用的。假设我们需要实现一个计数器，由于多个线程都需要去修改该计数器，因此我们需要使用锁来保证同一时间只有一个线程可以修改计数器，否则会导致脏数据。智能指针一节中，我们已经知道`Rc`无法在线程中传输（因为它没有实现`Send`特质），而`Arc`的是线程安全的：
```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();

            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap()); // 输出 10
}
```
我们已经知道，`Rc`和`RefCell`的结合，可以实现单线程的内部可变性。由于`Mutex`可以支持修改内部数据，当结合`Arc`一起使用时，可以实现多线程的内部可变性。

#### 死锁
互斥锁在使用时需时刻注意：
- 在使用数据前必须先获取锁
- 在数据使用完成后，必须及时的释放锁

尽管 Rust 通过智能指针的 drop 机制一定程度上解决了忘记释放锁的问题，但在多线程场景下，难免会遇到死锁（deadlock）：当一个操作试图锁住两个资源，然后两个线程各自获取其中一个锁，并试图获取另一个锁。

先来看一个单线程的死锁：
```rust
use std::sync::Mutex;

fn main() {
    let data = Mutex::new(0);
    let d1 = data.lock();
    let d2 = data.lock();
} // d1锁在此处释放
```
实际上，当代码量变得很大时，死锁可能就没这么容易看出来了。

多线程下，当我们拥有两个锁，且两个线程各自使用了其中一个锁，然后试图去访问另一个锁时，就**可能**发生死锁。之所以是可能，是因为子线程的初始化顺序和执行速度并不确定，我们无法确定哪个线程中的锁先被执行，因此也无法确定两个线程对锁的具体使用顺序。如果另一个线程获取锁之前，就已经执行完了，就不会造成死锁。增加 sleep 时间可以加大死锁发生的概率。
```rust
use std::{sync::{Mutex, MutexGuard}, thread};
use std::thread::sleep;
use std::time::Duration;

// lazy_static 是一个知名第三方库，用于动态生成静态变量
use lazy_static::lazy_static;
lazy_static! {
    static ref MUTEX1: Mutex<i64> = Mutex::new(0);
    static ref MUTEX2: Mutex<i64> = Mutex::new(0);
}

fn main() {
    // 存放子线程的句柄
    let mut children = vec![];
    for i_thread in 0..2 {
        children.push(thread::spawn(move || {
            for _ in 0..1 {
                // 线程1
                if i_thread % 2 == 0 {
                    // 锁住MUTEX1
                    let guard: MutexGuard<i64> = MUTEX1.lock().unwrap();

                    println!("线程 {} 锁住了MUTEX1，接着准备去锁MUTEX2 !", i_thread);

                    // 当前线程睡眠一小会儿，等待线程2锁住MUTEX2
                    sleep(Duration::from_millis(10));

                    // 去锁MUTEX2
                    let guard = MUTEX2.lock().unwrap();
                    // let guard = MUTEX2.try_lock();
                // 线程2
                } else {
                    // 锁住MUTEX2
                    let _guard = MUTEX2.lock().unwrap();

                    println!("线程 {} 锁住了MUTEX2, 准备去锁MUTEX1", i_thread);

                    let _guard = MUTEX1.lock().unwrap();
                    // let _guard = MUTEX1.try_lock();
                }
            }
        }));
    }

    // 等子线程完成
    for child in children {
        let _ = child.join();
    }

    println!("死锁没有发生");
}
```
为避免死锁，可考虑使用`try_lock`。此方法会尝试去获取一次锁，如果无法获取会返回一个错误 WouldBlock，因此不会发生阻塞。

> 一个有趣的命名规则：在 Rust 标准库中，使用try_xxx都会尝试进行一次操作，如果无法完成，就立即返回，不会发生阻塞。例如消息传递章节中的try_recv以及本章节中的try_lock

### 读写锁
互斥锁对每次读写操作都会加锁。对于读多写少的场景，互斥锁的性能就无法满足要求，此时就可以使用读写锁`RwLock`:
```rust
use std::sync::RwLock;

fn main() {
    let lock = RwLock::new(5);

    // 同一时间允许多个读
    {
        let r1 = lock.read().unwrap();
        let r2 = lock.read().unwrap();
        assert_eq!(*r1, 5);
        assert_eq!(*r2, 5);
    } // 读锁在此处被 drop

    // 同一时间只允许一个写
    {
        let mut w = lock.write().unwrap();
        *w += 1;
        assert_eq!(*w, 6);

        // 以下代码会阻塞发生死锁，因为读和写不允许同时存在
        // 写锁 w 直到该语句块结束才被释放，因此下面的读锁依然处于`w`的作用域中
        // let r1 = lock.read();
        // println!("{:?}",r1);
    }// 写锁在此处被drop
}
```
读写锁在使用上和互斥锁区别不大，只有在多个读的情况下不阻塞程序，只要有写操作，也是会阻塞的。

同样也可以使用`try_write`和`try_read`来尝试进行一次写/读，若失败则返回错误 WouldBlock。

读写锁有如下特点：
- 读和写不能同时发生，如果使用try_xxx解决，就必须做大量的错误处理和失败重试机制
- 当读多写少时，写操作可能会因为一直无法获得锁导致连续多次失败（即出现“写饥饿”）
- 锁本身的性能不如`Mutex`

因此如果要使用要确保满足以下两个条件：并发读，且需要对读到的资源进行"长时间"的操作（例如读写 HashMap 就不属于长时间操作）。其他情况一律应使用互斥锁。

### 更多锁
[parking_lot](https://github.com/Amanieu/parking_lot) 和 [spin](https://github.com/zesterer/spin-rs) 是两个较为知名的第三方锁实现库。

### 条件变量
锁并没有解决线程的访问顺序问题，因此 Rust 提供了条件变量（Condition Variables）。条件变量经常和互斥锁一起使用，可以让线程挂起，直到某个条件满足后再继续执行。

条件变量在 Rust 中通过`Condvar`创建。
```rust
use std::thread;
use std::sync::{Arc, Mutex, Condvar};

fn main() {
    let pair = Arc::new((Mutex::new(false), Condvar::new()));
    let pair2 = pair.clone();

    thread::spawn(move|| {
        let (lock, cvar) = &*pair2;
        // 子线程获取锁
        let mut started = lock.lock().unwrap();
        println!("changing started");
        // 将其修改为 true
        *started = true;
        // 调用条件变量的 notify_one 方法来通知主线程继续执行
        cvar.notify_one();
    });

    let (lock, cvar) = &*pair;
    let mut started = lock.lock().unwrap();
    // main 线程首先进入 while 循环
    while !*started {
        // 调用 wait 方法挂起等待子线程的通知，并释放了锁 started
        started = cvar.wait(started).unwrap();
    }

    println!("started changed");
}
```

### 信号量
信号量（Semaphore）可以让我们精准的控制当前正在运行的任务最大数量。

Rust 标准库中的信号量目前已废弃，Rust 官方更推荐直接使用 Mutex 和 Condvar 组合。如果要使用信号量，现在一般直接用 [tokio](https://github.com/tokio-rs/tokio/blob/master/tokio/src/sync/semaphore.rs)：
```rust
use std::sync::Arc;
use tokio::sync::Semaphore;

#[tokio::main]
async fn main() {
    // 创建了一个容量为 3 的信号量
    // 当正在执行的任务超过 3 时，剩下的任务需要等待正在执行任务完成并减少信号量后到 3 以内时，才能继续执行。
    let semaphore = Arc::new(Semaphore::new(3));
    let mut join_handles = Vec::new();

    for _ in 0..5 {
        let permit = semaphore.clone().acquire_owned().await.unwrap();
        join_handles.push(tokio::spawn(async move {
            //
            // 在这里执行任务...
            //
            drop(permit);
        }));
    }

    for handle in join_handles {
        handle.await.unwrap();
    }
}
```
信号量的关键点是，使用前需要申请，如果容量满了，就需要等待；使用后需要释放，以便其它等待者可以继续。

### 原子类型
原子指的是一系列不可被 CPU 上下文交换的机器指令，这些指令组合在一起就形成了原子操作。在多核 CPU 下，当某个 CPU 核心开始运行原子操作时，会先暂停其它 CPU 内核对内存的操作，以保证原子操作不会被其它 CPU 内核所干扰。

由于原子操作是通过指令提供的支持，因此它的性能相比锁和消息传递会好很多。相比较于锁而言，原子类型不需要开发者处理加锁和释放锁的问题，同时支持修改，读取等操作，还具备较高的并发性能，几乎所有的语言都支持原子类型。

原子类型是无锁类型，但是无锁不代表无需等待，因为原子类型内部使用了 CAS（Compare and swap, 它通过一条指令读取指定的内存地址，然后判断其中的值是否等于给定的前置值，如果相等，则将其修改为新的值）循环，既然存在循环，那就也是需要等待的。

#### 原子全局变量
原子类型最典型的场景就是作为一个全局的静态变量使用：
```rust
use std::ops::Sub;
use std::sync::atomic::{AtomicU64, Ordering};
use std::thread::{self, JoinHandle};
use std::time::Instant;

const N_TIMES: u64 = 10000000;
const N_THREADS: usize = 10;

// AtomicU64 就是一个原子类型
static R: AtomicU64 = AtomicU64::new(0);

fn add_n_times(n: u64) -> JoinHandle<()> {
    thread::spawn(move || {
        for _ in 0..n {
            R.fetch_add(1, Ordering::Relaxed);
        }
    })
}

fn main() {
    let s = Instant::now();
    let mut threads = Vec::with_capacity(N_THREADS);

    for _ in 0..N_THREADS {
        threads.push(add_n_times(N_TIMES));
    }

    for thread in threads {
        thread.join().unwrap();
    }
    // 内存顺序
    assert_eq!(N_TIMES * N_THREADS as u64, R.load(Ordering::Relaxed));

    println!("{:?}", Instant::now().sub(s));
}
```
和互斥锁一样，Atomic 的值具有内部可变性，无需将其声明为 mut。

多线程中的原子操作往往需要配合`Arc`：
```rust
use std::sync::Arc;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::{hint, thread};

fn main() {
    let spinlock = Arc::new(AtomicUsize::new(1));

    let spinlock_clone = Arc::clone(&spinlock);
    let thread = thread::spawn(move|| {
        spinlock_clone.store(0, Ordering::SeqCst);
    });

    // 等待其它线程释放锁
    while spinlock.load(Ordering::SeqCst) != 0 {
        hint::spin_loop();
    }

    if let Err(panic) = thread.join() {
        println!("Thread had an error: {:?}", panic);
    }
}
```

#### 内存顺序
上面实例代码中有一个奇怪的枚举`Ordering`，它用于控制原子操作使用的内存顺序。

内存顺序是指 CPU 在访问内存时的顺序，该顺序可能受以下因素的影响：
- 代码中的先后顺序
- 编译优化的重排序
- CPU 缓存导致的执行过程乱序

`Ordering`有如下成员：
- Relaxed：这是最宽松的规则，它对编译器和 CPU 不做任何限制，可以乱序
- Release：设定内存屏障(Memory barrier)，保证它之前的操作永远在它之前，但是它后面的操作可能被重排到它前面
- Acquire：, 设定内存屏障，保证在它之后的访问永远在它之后，但是它之前的操作却有可能被重排到它后面，往往和 Release 在不同线程中联合使用
- AcqRel：是 Acquire 和 Release 的结合，同时拥有它们俩提供的保证。比如要对一个 atomic 自增 1，同时希望该操作之前和之后的读取或写入操作不会被重新排序
- SeqCst：SeqCst 叫做顺序一致性，就像是 AcqRel 的加强版，它不管原子操作是属于读取还是写入的操作，只要某个线程有用到 SeqCst 的原子操作，线程中该 SeqCst 操作前的数据操作绝对不会被重新排在该 SeqCst 操作之后，且该 SeqCst 操作后的数据操作也绝对不会被重新排在 SeqCst 操作前。

来看一下什么叫内存屏障：
```rust
use std::thread::{self, JoinHandle};
use std::sync::atomic::{Ordering, AtomicBool};

static mut DATA: u64 = 0;
static READY: AtomicBool = AtomicBool::new(false);

fn reset() {
    unsafe {
        DATA = 0;
    }
    READY.store(false, Ordering::Relaxed);
}

fn producer() -> JoinHandle<()> {
    thread::spawn(move || {
        unsafe {
            DATA = 100;                                 // A
        }
        READY.store(true, Ordering::Release);           // B: 内存屏障 ↑
    })
}

fn consumer() -> JoinHandle<()> {
    thread::spawn(move || {
        while !READY.load(Ordering::Acquire) {}         // C: 内存屏障 ↓

        assert_eq!(100, unsafe { DATA });               // D
    })
}


fn main() {
    loop {
        reset();

        let t_producer = producer();
        let t_consumer = consumer();

        t_producer.join().unwrap();
        t_consumer.join().unwrap();
    }
}
```
原则上，`Acquire`用于读取，而`Release`用于写入，二者同时需要用`AcqRel`。不知道用什么就选`SeqCst`。`Relaxed`一般只出现在一些简单场景，如前面的多线程计数（只有 fetch_add）。

#### 原子 vs 锁
- `std::sync::atomic`包中仅提供了数值类型的原子操作：AtomicBool, AtomicIsize, AtomicUsize, AtomicI8, AtomicU16 等，而锁可以应用于各种类型
- 一些情况必须用锁（如条件变量）
- 锁的使用更加简单

原子操作经常用于：
- 标准库和高性能库
- 无锁（lock free）数据结构
- 全局变量，例如全局自增 ID
- 跨线程计数器

## 线程安全的特质

`Rc`无法在线程间安全转移，当尝试在多线程中调用时，会报错“the trait \`Send\` is not implemented for \`Rc\<i32\>\`”。而换成`Arc`就可以。来看一下二者的源码，能够发现如下片段：
```rust
// Rc源码片段
// `!`代表移除实现
impl<T: ?Sized> !marker::Send for Rc<T> {}
impl<T: ?Sized> !marker::Sync for Rc<T> {}

// Arc源码片段
unsafe impl<T: ?Sized + Sync + Send> Send for Arc<T> {}
unsafe impl<T: ?Sized + Sync + Send> Sync for Arc<T> {}
```

### Send 和 Sync
`Send`和`Sync`实际上只是标记特质，未定义任何行为。
- 实现`Send`的类型可以在线程间安全地传递其所有权
- 实现`Sync`的类型可以在线程间通过引用安全共享

引用必须能传递才能让类型在线程间共享。也就是说，如果一个类型 T 的引用 &T 是 Send，则 T 是 Sync。

来看看锁：
```rust
// unsafe impl 是因为 Sync（也包括Send）是 Unsafe 特质
// T: ?Sized 表示类型 T 可以是动态大小类型，如 dyn Trait 或切片
unsafe impl<T: ?Sized + Send + Sync> Sync for RwLock<T> {}

unsafe impl<T: ?Sized + Send> Sync for Mutex<T> {}
```
锁本身肯定都是实现了`Sync`的。此外可以发现，由于`RwLock`是需要支持并发读的，其中的值必须能够在线程间共享，所以值也需要实现`Sync`，而`Mutex`就没有这个需求，所以实现的特质少了一个。

### 没有实现 Send 和 Sync 的类型
几乎所有类型都默认实现了`Send`和`Sync`，而且由于这两个特质都是可通过`derive`自动派生（auto trait）的，意味着结构体等复合类型, 只要它内部的所有成员都实现了`Send`或者`Sync`，那么它就自动实现了`Send`或`Sync`。

因此，只需要看看哪些类型默认没有实现`Send`和`Sync`。以下是一些常见的：
- 裸指针：没有任何安全保证，所以二者都没有实现
- `Cell`和`RefCell`：二者底层都是`UnsafeCell`，`UnsafeCell`并非`Sync`的
- `Rc`：内部引用计数器不是线程安全的，所以两者都没实现

此外，手动实现`Send`和`Sync`是不安全的，几乎可以肯定需要`unsafe`。通常情况下无需开发者自行实现。

### 为裸指针实现 Send 和 Sync
这里用到了`newtype`类型 MyBox。复合类型中有一个成员没实现特质，该复合类型就也未实现特质，因此我们需要手动为它实现。先看一个只有`Send`的例子：
```rust
use std::thread;

#[derive(Debug)]
struct MyBox(*mut u8);
unsafe impl Send for MyBox {}
fn main() {
    let p = MyBox(5 as *mut u8);
    let t = thread::spawn(move || {
        println!("{:?}",p);
    });

    t.join().unwrap();
}
```
可以看到，只要实现了`Send`，变量就可以在线程间共享。但此时访问的引用实际上还是对主线程中的数据的借用，为了能够共享值，还是需要`Sync`：
```rust
use std::thread;
use std::sync::Arc;
use std::sync::Mutex;

#[derive(Debug)]
struct MyBox(*const u8);
unsafe impl Send for MyBox {}
unsafe impl Sync for MyBox {}

fn main() {
    let b = &MyBox(5 as *const u8);
    let v = Arc::new(Mutex::new(b));
    // 将智能指针 v 的所有权转移给新线程，同时 v 包含了一个引用类型 b
    let t = thread::spawn(move || {
        let _v1 =  v.lock().unwrap();
    });

    t.join().unwrap();
}
```
这里需要配合`Arc`，才能进行线程间的同步。

### 总结
- 实现`Send`的类型可以在线程间安全的传递其所有权, 实现`Sync`的类型可以在线程间安全的共享（通过引用）
- 绝大部分类型都实现了`Send`和`Sync`，常见的未实现的有：裸指针、`Cell`、`RefCell`、`Rc`等
- 可以为自定义类型实现`Send`和`Sync`，但是需要`unsafe`代码块
- 可以为部分 Rust 中的类型实现`Send`、`Sync`，但是需要使用`newtype`

***
[^1]: 出处：Joe Armstrong
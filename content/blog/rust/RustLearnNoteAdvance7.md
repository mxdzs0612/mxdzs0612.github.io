+++
title = "Rust 进阶学习笔记（七）：多线程"
slug = "rust_learn_note_adv_7"
date = 2025-07-15
updated = 2025-07-15
description = "并发和并行，多线程创建与线程同步，线程安全"
draft = true
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
    // `handle.join`可以让当前线程阻塞，直到它等待的子线程的结束。
    handle.join().unwrap();

    for i in 1..5 {
        println!("hi number {} from the main thread!", i);
        thread::sleep(Duration::from_millis(1));
    }
    // 如果 `handle.join` 在这里调用，两个线程会交替输出
    // handle.join()在主线程循环结束后才调用，所以主线程不会等待子线程完成就会执行自己的循环，但是会等待子线程完成才退出
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

// 尽管子线程中修改为了3，我们在这里依然拥有main线程中的局部值：2
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
        // 将计数器加1
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
    // 打印每个线程局部变量中的计数器值，发现不一定有5个线程，
    // 因为一些线程已退出，并且其他线程会回收退出线程的对象
    println!("x: {}, y: {}", x, y.get());
    x + y.get()
});

// 和为5
assert_eq!(total, 5);
```
该库不仅仅使用了值的拷贝，而且还能自动把多个拷贝汇总到一个迭代器中，最后进行求和，非常好用。

### 条件变量


### Call Once

{{ admonition(type="warning", title="注意", text="施工中") }}













***
[^1]: 出处：Joe Armstrong
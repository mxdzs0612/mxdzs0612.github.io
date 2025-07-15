+++
title = "Rust 进阶学习笔记（八）：异步编程"
slug = "rust_learn_note_adv_8"
date = 2025-07-16
updated = 2025-07-16
description = ""
# draft = true
[taxonomies]
tags = ["Rust", "Learn"]
[extra]
pinned = false
post_listing_date = "both"
+++

本节出处：[圣经 异步编程](https://course.rs/advance/async/intro.html)

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






































{{ admonition(type="warning", title="注意", text="施工中") }}

***
[^1]: 异步编程的模型有一大堆，包括线程、协程、actor模型等等，具体可以看原文
+++
title = "Rust 入门学习笔记（完）：闭包"
slug = "rust_learn_note_9"
date = 2025-07-03
updated = 2025-07-04
description = "闭包，闭包获取外部参数，闭包原理，闭包类型实例"
# draft = true
[taxonomies]
tags = ["Rust", "Learn"]
[extra]
pinned = false
post_listing_date = "both"
+++

## 基础概念
闭包是一种可以捕获其环境中变量的匿名函数。

闭包的语法相对比较灵活，同时也具有强大的功能。闭包在 Rust 中被广泛用于函数式编程、并发编程和代码简化。

### 使用
定义
- 在`| |`内定义参数
- （可选）指定参数与返回类型
- 在`{ }`内定义闭包体
可以将闭包分配给一个变量，然后使用该变量，就像它是一个函数名一样。

***
### 例子
```rust
#[derive(Debug)]
struct User {
    name: String,
    score: u64,
}
// 不用闭包
// sort_by_key 是一个内置原地排序方法
fn sort_score(users: &mut Vec<User>) {
    users.sort_by_key(sort_helper);
}
fn sort_helper(u: &User) -> u64 {
    u.score
}
// 使用闭包效率更高，一个方法解决
fn sort_score_closure(users: &mut Vec<User>) {
    users.sort_by_key(|u| u.score); // u 的类型能够自动推导
}

fn main() {
    let f = |a, b| a + b;
    println!("{}", f(1.0, 2.0)); // 自动推导

    let a = User {
        name: "U1".to_owned(),
        score: 100,
    };
    let b = User {
        name: "U2".to_owned(),
        score: 80,
    };
    let c = User {
        name: "U3".to_owned(),
        score: 40,
    };
    let d = User {
        name: "U4".to_owned(),
        score: 90,
    };
    let mut users = vec![a, b, c, d];
    // sort_score(&mut users); // 能够排序，但不建议
    sort_score_closure(&mut users);
    println!("{:?}", users);
}
```

## 闭包参数获取
闭包获取外部参数的方式共有三种：
- 不可变引用`Fn`
- 可变引用`FnMut`
- 转移所有权`FnOnce`
默认情况下是由 Rust 编译器决定用哪种方式获取外部参数。

闭包是有层级的：`Fn`能被当作`FnMut`和`FnOnce`使用；`FnMut`能被当作`FnOnce`；`FnOnce`只能当作自己。

### 所有权转移
如果想要进行 Move 操作，需要使用`move`关键字强制将所有权转移到闭包。

另一种方式是在闭包中主动调用类似 drop 参数的方法将值消耗掉，此时能够自动推断。

***
### 例子
```rust
fn main() {
    // Fn 不可变引用获取外部参数
    let s1 = String::from("1111111111111111111");
    let s2 = String::from("2222222222222222222");
    let fn_func = |s| { // 闭包中捕获了外部值，没有修改，没有销毁，会自动实现 impl Fn(String)
        println!("{s1}");
        println!("I am {s}");
        println!("{s2}");
    };
    fn_func("yz".to_owned());
    fn_func("原子".to_owned());
    println!("{s1} {s2}");

    // FnMut 可变引用获取外部参数
    let mut s1 = String::from("1111111111111111111");
    let mut s2 = String::from("2222222222222222222");
    // fn_func 闭包变量本质上是结构体的一个匿名实例，也需要声明为 mut
    let mut fn_func = |s| { // impl FnMut(String)
        s1.push_str("😀");
        s2.push_str("😀");
        println!("{s1}");
        println!("I am {s}");
        println!("{s2}");
    };
    fn_func("yz".to_owned());
    fn_func("原子".to_owned());
    println!("{s1} {s2}");

    // 所有权转移 由编译器根据代码来判断
    let s1 = String::from("1111");
    let fn_once_func = || { // impl FnOnce()
        println!("{s1}");
        std::mem::drop(s1); // 主动销毁
    };
    fn_once_func(); // 只能调用一次
    // println!("{s1}"); // 已被销毁
    // 所有权转移 强制转移
    let s1 = String::from("1111");
    let move_fn = move || { // 会自动推断为 Fn，因为 Fn 也包含了 FnOnce
        println!("{s1}");
    };
    move_fn();
    move_fn(); // 还是可以多次调用的
    // println!("{s1}"); // s1 不能打印了

    let s1 = String::from("1111");
    // 必须加 move 转移所有权给线程
    // 否则不能保证线程运行过程中 s1 还存在
    std::thread::spawn(move || println!("d  {s1}"));
}
```

### 工作原理
- Rust 会将闭包放入一个匿名结构体
- 结构体会声明一个 call 函数，由于闭包本身也是函数，所以该函数可以包含闭包的所有代码
- 结构体会生产一些属性去捕获闭包外的参数
- 结构体会根据上下文实现一些特质（`Fn` `FnMut` `FnOnce`）

***
### 例子
```rust
// F 是一个泛型
fn apply_closure<F: Fn(i32, i32) -> i32>(closure: F, x: i32, y: i32) -> i32 {
    closure(x, y)
}

fn main() {
    let x = 5;
    let add_closure = |a, b| {
        println!("x is: {}", x);
        a + b + x
    };
    let result = apply_closure(add_closure, 5, 6);
    println!("{}", result);
}
```

## 闭包特质和函数参数
```rust
fn closure_fn<F>(func: F)
where
    F: Fn(),
{
    func();
    func();
}
fn closure_fn_mut<F>(mut func: F)
where
    F: FnMut(),
{
    func();
    func();
}

fn closure_fn_once<F>(func: F)
where
    F: FnOnce(),
{
    func();
}

fn main() {
    // Fn 只能传不可变引用一种闭包
    let s1 = String::from("11111");
    closure_fn(|| println!("{}", s1));

    // FnMut 可以传可变或不可变两种
    let s1 = String::from("11111");
    closure_fn_mut(|| println!("{}", s1));
    // println!("{}", s1);
    let mut s2 = String::from("22222");
    closure_fn_mut(|| {
        s2.push_str("😀");
        println!("{}", s2);
    });
    println!("{s2}");

    // FnOnce 什么都能调用，具体怎么做由代码决定
    let s1 = String::from("11111");
    closure_fn_once(|| println!("{}", s1));
    let mut s2 = String::from("22222");
    closure_fn_once(|| {
        s2.push_str("😀");
        println!("{}", s2);
    });
    println!("{s2}");
    let s3 = " ff".to_owned();
    closure_fn_once(move || println!("{s3}"));
    // println!("{s3}")
}
```

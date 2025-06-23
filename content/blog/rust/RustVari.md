+++
title = "Rust 学习笔记（一）"
slug = "rust_note_1"
date = 2025-06-21T15:00:07Z
updated = 2025-06-23T20:00:07Z
[taxonomies]
tags = ["Rust", "Learn"]
[extra]
summary = "Rust入门学习笔记"
pinned = false
post_listing_date = "both"
+++
<!-- 抄了一份 Rust 的新手教程，先丢个链接方便自己找，有空再来整理吧。 -->
<!-- [地址](https://juejin.cn/post/7150187051621548046) -->

学习笔记。来源：原子之音
> [视频](https://www.bilibili.com/video/BV15y421h7j7/)
 [代码](https://gitlab.com/yzzy/rust_project)

# 变量与数据类型

## 变量与不可变性

### 变量基础
1. Rust 使用`let`关键字声明变量。
2. Rust 支持类型推导，如果推导不出来会在编译时报错。也可显式指定。
```rust
let x: i32 = 5;
```
3. 变量一般使用蛇形命名法（即下划线），枚举和结构体使用帕斯卡命名法（全大写）。
    - 如果变量声明了但是没有用，可以在变量名前面添加一个下划线，这样就不会报错。
4. 支持强制类型转换。
```rust
let a = 3.1; let b = a as i32;
```
5. 打印变量，基础类型默认都支持了，其他类型需要实现特质。
```rust
println!("val: {}", x);
println!("val: {x}");
```

### 不可变变量
rust 的变量默认是不可变的！

不可变性是 rust 实现可靠性和安全性的关键，有效防止数据竞争和并发问题。

如果希望可变，需要使用`mut`关键字。

### Shawowing
Rust 允许隐藏变量。意思是可以声明一个与前一个变量同名的新变量（值、类型、可变性均可改变）。

这个变量是隐藏了，而不是消失了，具体可以看下面的例子。

***
**例子**
```rust
fn main() {
    // 不可变与命名
    let _nice_count = 100; // 自动推导i32
    let _nice_number: i64 = 54;
    // nice_count = 23; // 不可直接修改
    // 声明可变
    let mut _count = 3;
    _count = 4;
    // Shadowing
    let x = 5;
    {
        // 命名空间
        let x = 10;
        println!("inner x : {}", x); // 10
    } // 内部的x被销毁了
    println!("Outer x: {x}"); // 5
    let x = "hello"; // 在同一作用域下重新声明了x，最终覆盖了之前的x
    println!("New x : {x}"); // hello
    let mut x = "this";
    println!("x: {x}"); // this
    x = "that";
    println!("Mut x: {x}"); // that
    // 可以重定义类型 可变性
}

```

## const 与 static

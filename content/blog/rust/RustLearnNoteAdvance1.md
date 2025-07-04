+++
title = "Rust 进阶学习笔记（一）：智能指针"
slug = "rust_learn_note_adv_1"
date = 2025-07-04
updated = 2025-07-04
description = "智能指针与常见智能指针"
# draft = true
[taxonomies]
tags = ["Rust", "Learn"]
[extra]
pinned = false
post_listing_date = "both"
+++

本节出处：原子之音 [视频](https://www.bilibili.com/video/BV1Lg4y1w7aL/)

其实感觉进阶学习的第一篇不应该是智能指针，毕竟集合、IO 什么的都还没看。不过没办法，谁让我对这个最感兴趣呢，就先学它了。

## 智能指针

### 指针
Rust 最常见的指针是引用，裸指针被 Rust 判定为不安全，需要加上`unsafe`才能使用。

智能指针的概念起源于 C++，它们提供了额外的功能和安全性保证，以帮助管理内存和数据。

在 Rust 中，智能指针是一种封装了对动态分配内存的所有权和生命周期管理的数据类型。

智能指针通常封装了一个原始指针，并提供了一些额外的功能，比如引用计数、所有权转移、生命周期管理等。

### 原始指针（裸指针）
由于 Rust 的所有权机制，使用原始指针是有风险的。

使用原始指针时，需要用`as`来标注类型以及可变/不可变。解引用则需要`unsafe`。

> 也不需要太过恐惧要`unsafe`，如果确实有需要，该用就用。

```rust
fn main() {
    // 指向不可变
    let x: usize = 1;
    let raw1: *const usize = &x as *const usize;
    let raw2: *const usize = &x;
    let some_usize1 = unsafe { *raw1 };
    println!("{some_usize1}");
    let some_usize2 = unsafe { *raw2 };
    println!("{some_usize2}");

    // 指向可变
    let mut y: usize = 2;
    let raw_mut1: *mut usize = &mut y as *mut usize;
    let raw_mut2: *mut usize = &mut y;
    let some_mut_usize1 = unsafe { *raw_mut1 };
    println!("{some_mut_usize1}");
    let some_mut_usize2 = unsafe { *raw_mut2 };
    println!("{some_mut_usize2}");
}
```

### 常见智能指针
- `Box`是最简单的智能指针，只是将数据存储在堆上。

- `Rc`指针是智能指针的核心，完成共享所有权的功能
  - `Weak`：弱版的`Rc`
  - `Arc`：多线程的`Rc`
  - `Mutex`：提供可变性的`Rc`

- `Cell`和`RefCell`用于内部可变性。

## Box 指针

### 数据存储在堆上的情况
- 编译期数据大小无法确定
- 数据太大，copy 浪费资源
- 需要 dyn Trait Object 而非具体类型

***
### 例子
```rust
trait Animal {
    fn eat(&self);
}

#[derive(Debug)]
struct Cat {
    children: Option<Box<Cat>>,
}

impl Animal for Cat {
    fn eat(&self) {
        println!("cat is eating");
    }
}

fn main() {
    let cat = Box::new(Cat { children: None });
    println!("{:?}", cat); // Cat { children: None }
    let t: Box<dyn Animal>;
    t = Box::new(Cat {
        children: Some(cat),
    });
    // trait object 有两个指向：
    // 1、具体的类型 Cat
    // 2、虚表，多态相关
    t.eat();
}
```

## Rc 指针
{{ admonition(type="warning", icon="tip", title="注意", text="施工中") }}

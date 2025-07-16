+++
title = "Rust 进阶学习笔记（一）：智能指针"
slug = "rust_learn_note_adv_1"
date = 2025-07-04
updated = 2025-07-04
description = "Rust指针，智能指针与常见智能指针"
# draft = true
[taxonomies]
tags = ["Rust", "Learn"]
[extra]
pinned = false
post_listing_date = "both"
+++

本节出处：原子之音 [视频](https://www.bilibili.com/video/BV1Lg4y1w7aL/) 代码纯手打，不过都跑通验证过，应该没什么问题。

其实感觉进阶学习的第一篇不应该是智能指针，毕竟集合、IO 什么的都还没看。不过没办法，谁让我对这个最感兴趣呢，就先学它了。

<s>结果看完没有特别理解，主要大部分时候根本想不起来要用智能指针。还是要找点题做做。</s>

学完多线程后回来看，会容易理解得多。

## 智能指针

### 指针
Rust 最常见的指针是引用，裸指针被 Rust 判定为不安全，需要加上`unsafe`才能使用。

智能指针的概念起源于 C++，它们提供了额外的功能和安全性保证，以帮助管理内存和数据。

在 Rust 中，智能指针是一种封装了对动态分配内存的所有权和生命周期管理的数据类型。

智能指针通常封装了一个原始指针，并提供了一些额外的功能，比如引用计数、所有权转移、生命周期管理等。

### 原始指针（裸指针）
由于 Rust 的所有权机制，使用原始指针是有风险的。

使用原始指针时，需要用`as`来标注类型以及可变/不可变。解引用则需要`unsafe`。

> 也不需要太过恐惧`unsafe`，如果确实有需要，该用就用。后续会有[一节](../rust-learn-note-adv-6)专门讲`unsafe`。

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

### 智能指针
智能指针往往是基于结构体实现，它与我们自定义的结构体最大的区别在于它实现了`Deref`和`Drop`特质：
- Deref：可以让智能指针像引用那样工作，这样就可以写出同时支持智能指针和引用的代码，例如 *T
- Drop：允许指定智能指针超出作用域后自动执行的代码，例如做一些数据清除等收尾工作

智能指针究竟智能在何处？答：
- 自动解引用
- 自动化内存管理

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
// 本例主要演示：
// 如何循环调用自己
// 构造 trait object
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
Rc 指针也叫计数指针，位于`std::rc::Rc`下，是一个存储引用计数的指针。

Rc 指针主要追踪两个方向：
- 对单个值的多次引用
- 何时销毁变量

`Rc::clone`会创建一个新的引用，让 Rc 计数加一。

```rust
// 本例主要演示：
// 所有权 move
// “强”计数
use std::rc::Rc;

#[derive(Debug)]
struct Cat {}

fn main() {
    let cat1 = Cat {};
    let cat2 = Cat {};
    let cat3 = Cat {};

    // 如果 cat 是一个原始类型，这么写是可以的
    // let cat_vec1 = vec![cat1, cat2];
    // let cat_vec2 = vec![cat2, cat3];
    // println!("{:?}", cat_vec1);

    let cat1 = Rc::new(Cat {});
    let cat2 = Rc::new(Cat {});
    let cat3 = Rc::new(Cat {});

    let cat_vec1 = vec![cat1, Rc::clone(&cat2)];
    // cat2 count++
    let cat_vec2 = vec![cat2, cat3];
    // cat2 销毁
    println!("{:?}", cat_vec1);
    println!("{:?}", cat_vec2);
    // println!("{:?}", cat2);

        let cat1 = Rc::new(Cat {});
    let cat2 = Rc::new(Cat {});
    let cat3 = Rc::new(Cat {});

    let cat_vec1 = vec![cat1, Rc::clone(&cat2)];
    // cat2 count++
    let cat_vec2 = vec![Rc::clone(&cat2), cat3];
    // cat2 销毁
    println!("{:?}", cat_vec1);
    println!("{:?}", cat_vec2);
    println!("{}", Rc::strong_count(&cat2)); // 3， let 一次，两个 clone 各一次
    std::mem::drop(cat_vec2);
    println!("{}", Rc::strong_count(&cat2)); // 2
}
```

## Cell 与 RefCell
Cell 与 RefCell 用于内部可变性，说人话就是能对一个不可变的值进行可变借用。它们带来了灵活性但同时也造成了一些安全隐患。

Cell 只适用于 Copy 类型，用于提供值，而 RefCell 用于提供引用。

### RefCell
RefCell 会造成 panic。

RefCell 在编译期不会报错，应慎重使用。

***
### 例子
```rust
// 本例主要演示：
// 两种指针区别
// RefCell 编译时不报错
use std::cell::{Cell, RefCell};

fn main(){
    // Cell
    let c = Cell::new("old");
    let c1: &str = c.get();
    println!("{c1}"); // old
    c.set("new");
    let c2: &str = c.get();
    println!("{c1} {c2}"); // old new
    let c = Cell::new(String::from("rust"));
    // println!("{:?}", c); // 不能打印
    // let c3 = c.get(); // 不能 get

    // RefCell
    let rc = RefCell::new(String::from("rust"));
    let r1 = rc.borrow();
    println!("{}", r1); // rust
    // 注意必须重新 new 一个，同一个 rc 不能同时有不可变和可变引用
    // 但是编译时不会报错，在运行时才会报出来
    let rc = RefCell::new(String::from("rust"));
    let mut r2 = rc.borrow_mut();
    r2.push_str(" new");
    println!("{}", r2); // rust new
}
```

## Weak 指针
Weak 指针是指持有对分配数据的**非拥有**引用的指针。

与之相对，Rc 指针是拥有所有权的。

Weak 指针主要用于解决 Rc 指针的循环引用（Reference Cycle）问题。

### 相互转换
- Rc -> Weak：`Rc::upgrade`
- Weak -> Rc：`Rc::downgrade`

Weak 同样也有引用计数，可以通过`weak_count()`查看。

***
### 例子
```rust
// 本例主要演示：
// Weak 和 Rc 配合
// 什么叫解决循环引用
// 循环引用就是：比如说 A 是 B 的父母，B 是 A 的孩子，为了表示这一关系，就会存在 A 调用 B，B 调用 A。
use std::{cell::RefCell, rc::{Rc, Weak}};

#[derive(Debug)]
struct People {
    age: usize,
    parent: RefCell<Weak<People>>,
    children: RefCell<Vec<Rc<People>>>,
}

fn main(){
    let son = Rc::new(People{
        age: 10,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![]),
    });
    println!("son parent = {:?}", son.parent.borrow()); // (Weak)
    println!("son parent = {:?}", son.parent.borrow().upgrade()); // None，因为没有设置双亲
    println!("{} {}", Rc::strong_count(&son), Rc::weak_count(&son)); // 1 0

    let father = Rc::new(People{
        age: 40,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![Rc::clone(&son)]),
    });
    // son 被克隆所以强引用 + 1
    println!("{} {}", Rc::strong_count(&son), Rc::weak_count(&son)); // 2 0

    // 设置父母，因为 parent 是 Weak 所以需要降级
    *son.parent.borrow_mut() = Rc::downgrade(&father);
    // 如果 parent 是 Rc，这里会无限循环
    // Weak 就能够防止再被打开
    println!("son parent = {:?}", son.parent.borrow().upgrade()); // Some(People { age: 40, parent: RefCell { value: (Weak) }, children: RefCell { value: [People { age: 10, parent: RefCell { value: (Weak) }, children: RefCell { value: [] } }] } })

    println!("{} {}", Rc::strong_count(&son), Rc::weak_count(&son)); // 2 0
    println!("{} {}", Rc::strong_count(&father), Rc::weak_count(&father)); // 1 1
}
```

## Arc 指针
Rc 指针不支持 Send 和 Sync 特质，在多线程环境下会产生数据竞争（Data Race）。

Arc 实现了引用计数的原子化操作，是线程安全的，但会带来较大性能损耗。

Arc 位于`std::sync::Arc`下，注意这个包里同样存在`std::sync::Weak`，作用和 Rc 的 Weak 完全相同。

***
### 例子
```rust
// 把 rc 移植到多线程上
use std::{clone, sync::{Arc, Weak}, thread::{self, Thread}};

#[derive(Debug)]
struct Owner {
    name: String,
}

#[derive(Debug)]
struct Dog {
    owner: Arc<Owner>,
}

fn main() {
    let someone = Arc::new(Owner{
        name: "tom".to_string(),
    });
    for i in 0..10 {
        let someone = Arc::clone(&someone);
        let join_handle = thread::spawn(move || {
            let yellow = Arc::new(Dog{
                owner: Arc::clone(&someone),
            });
            let black = Arc::new(Dog{
                owner: Arc::clone(&someone),
            });
            println!("yellow owner {}", yellow.owner.name);
            println!("black owner {}", black.owner.name);
            println!("Thread {i} end");
        });
    }
}
```

## Mutex
想要加入可变的功能就需要 Mutex。

可变不可变是编译器对引用的附加限制，使用可变会增加性能损耗。

### 互斥锁
Mutex（是mutual exclusion 的缩写）意思是互斥锁，其主要作用是让多个线程并发的访问同一个值变成了排队访问：同一时间，只允许一个线程 A 访问该值，其它线程需要等待 A 访问完成后才能继续。

这里就是先提一下，后续[多线程](../rust-learn-note-adv-7#hu-chi-suo)一节中会更详细描述 Mutex。

***
### 例子
```rust
use std::{clone, sync::{Arc, Mutex, Weak}, thread::{self, Thread}};

#[derive(Debug)]
struct Owner {
    name: String,
    dogs: Mutex<Vec<Weak<Dog>>>,
}

#[derive(Debug)]
struct Dog {
    name: String,
    owner: Arc<Owner>,
}

fn main() {
    let someone = Arc::new(Owner{
        name: "tom".to_string(),
        dogs: Mutex::new(vec![]),
    });
    let yellow = Arc::new(Dog{
        name: "yel".to_string(),
        owner: Arc::clone(&someone),
    });
    let black = Arc::new(Dog{
        name: "blc".to_string(),
        owner: Arc::clone(&someone),
    });
    for i in 0..10 {
        let someone = Arc::clone(&someone);
        let yellow = Arc::clone(&yellow);
        let black = Arc::clone(&black);
        let join_handle = thread::spawn(move || {
            let mut guard = someone.dogs.lock().unwrap();
            guard.push(Arc::downgrade(&yellow));
            guard.push(Arc::downgrade(&black));
            println!("{} {}", guard[0].upgrade().unwrap().name, guard[1].upgrade().unwrap().name);
            println!("Thread {i} end");
        });
    }
}
```

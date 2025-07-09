+++
title = "Rust 入门学习笔记（八）：迭代器"
slug = "rust_learn_note_8"
date = 2025-07-02
updated = 2025-07-03
description = "迭代器，迭代与循环，迭代器特质，迭代方法，自定义迭代"
# draft = true
[taxonomies]
tags = ["Rust", "Learn"]
[extra]
pinned = false
post_listing_date = "both"
+++

## 迭代与循环

### 循环
- 定义：循环（Loop）是一种控制流结构，会反复执行一组语句直到满足某个条件。
- 控制条件：循环常常包含一个条件表达式，只有在条件为真时循环体中的语句才会执行。
- 退出条件：循环执行直到条件不再满足，或者遇到`break`语句显式推出
- 使用场景：反复执行某组语句直到满足某种条件

### 迭代
- 定义：迭代（Iteration）是指对序列中的**元素**进行逐个访问
- 控制条件：使用迭代器（Iterator）实现，迭代器提供对序列元素的访问和操作
- 退出条件：不需要，处理完全部元素后自动停止
- 使用场景：需要遍历数据结构中的数据

迭代器提供了一种更抽象的方式来处理序列。Rust 的迭代通常能达到或接近循环的最优性能水平。

***
### 例子
```rust
// &[i32] &Vec
// loop
fn sum_with_loop(arr: &[i32]) -> i32 {
    let mut sum = 0;
    for &item in arr {
        sum += item;
    }
    sum
}

// iter
fn sum_with_iter(arr: &[i32]) -> i32 {
    // 迭代方法
    arr.iter().sum()
}

fn main() {
    const ARRAY_SIZE: usize = 10000;
    // 消耗函数
    let array: Vec<i32> = (1..=ARRAY_SIZE as i32).collect();
    let sum1 = sum_with_loop(&array);
    println!("sum loop {}", sum1);
    let sum2 = sum_with_iter(&array);
    println!("sum loop {}", sum2);
}
```

## 迭代器与迭代器特质

### 特质 IntoIterator
`IntoIterator`特质定义了一种将类型转换为迭代器的能力。

该特质包含一个`into_iter()`方法，该方法返回了一个实现了`Iterator`特质的迭代器。

通常希望能够对一个类型进行迭代时，会实现`IntoIterator`特质来提供将该类型转换为迭代器的方法。通常同时会实现所有权转移。

并不是所有类型都使用 IntoIterator。

### 特质 Iterator
`Iterator`特质定义了一种访问序列元素的方式。

该特质包含一系列方法，如`next`、`map`、`filter`、`sum`等，用于对序列进行不同类型的操作。操作迭代器时通常就是操作`Iterator`。

可以通过实现`Iterator`特质来创建自定义迭代器，以定义如何迭代类型中的元素。通常没有所有权转移。
```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

### Iter
`Iter`是`Iterator`特质的**一个**具体实现，通常用于对集合中的元素进行迭代。

`Iter`常见于对数组、切片等集合类型进行迭代时。

通过`IntoIterator`特质能够获取到一个特定类型的迭代器，如`Iter`，然后可以使用`Iterator`特质的方法进行操作。

***
### 例子
```rust
fn main() {
    // vec
    let v = vec![1, 2, 3, 4, 5]; // vec 实现了 intoIterator 特质
    // 转换为迭代器
    let iter = v.into_iter(); // 返回的类型是一个 IntoIter，类似 Iter，是 Iterator 的特质对象，支持 sum 等方法
    let sum: i32 = iter.sum();
    println!("sum: {}", sum);
    // println!("{:?}", v) // into_iter 默认实现了 move 所有权转移，这里无法打印了
    // array
    let array = [1, 2, 3, 4, 5];
    let iter: std::slice::Iter<'_, i32> = array.iter(); // 另外一种实现，返回的类型是一个 Iter，也是 Iterator 的特质对象，支持 sum 等方法
    let sum: i32 = iter.sum();
    println!("sum: {}", sum);
    println!("{:?}", array); // 不可变引用，没有所有权转移
    // chars
    let text = "hello, world!";
    let iter = text.chars(); // 返回类型是一个 Chars，Chars 是一个结构体封装的迭代器，它并不直接遍历字符串，而是内部持有 &str 的字节切片（u8）的迭代器（slice::Iter<'a, u8>）。
    let uppercase = iter.map(|c| c.to_ascii_uppercase()).collect::<String>(); // collect 消耗函数，相当于不停调用 next
    // 另一种写法
    // let uppercase: String = iter.map(|c| c.to_ascii_uppercase()).collect();
    println!("uppercase: {}", uppercase);
    println!("{:?}", text); // 不可变引用，没有所有权转移
}
```

## 获取迭代器

### iter() 方法
`iter()`方法返回一个不可变引用的迭代器，用于只读访问集合的元素。

该方法适用于希望在不修改集合的情况下迭代集合元素的场景。

### iter_mut() 方法
`iter_mut()`方法返回一个可变引用的迭代器，允许修改集合的元素。

该方法适用于希望在迭代过程中修改集合元素的场景。

该方法性能较差，而且会导致现有的不可变引用全部失效，所以非必要不要使用。

### into_iter() 方法
`into_iter()`方法返回一个拥有所有权的迭代器，该迭代器会消耗集合本身，将所有权转移到迭代器。

该方法适用于希望迭代过程中拥有集合的所有权，以便进行消耗性的操作的场景，如移除元素。

***
### 例子
```rust
fn main() {
    let vec = vec![1, 2, 3, 4, 5];
    // 不可变引用
    for &item in vec.iter() {
        println!("{}", item);
    }
    println!("{:?}", vec); // [1, 2, 3, 4, 5]
    // 可变引用
    let mut vec = vec![1, 2, 3, 4, 5];
    for item in vec.iter_mut() {
        *item *= 2;
    }
    println!("{:?}", vec); // [2, 4, 6, 8, 10]
    // 所有权转移
    let vec = vec![1, 2, 3, 4, 5];
    for item in vec.into_iter() {
        println!("{}", item);
    }
    // println!("{:?}", vec);
}
```

## 自定义类型实现迭代器
```rust
#[derive(Debug)]
struct Stack<T> {
    // 自定义类型最底层肯定也是内置的类型
    // 这里使用泛型可变数组
    items: Vec<T>,
}

impl<T> Stack<T> {
    // 创建方法，创建时可不指定泛型
    fn new() -> Self {
        Stack { items: Vec::new() }
    }
    // 入栈
    fn push(&mut self, item: T) {
        self.items.push(item);
    }
    // 出栈
    fn pop(&mut self) -> Option<T> {
        self.items.pop()
    }
    // 自己实现迭代器时可以改变命名，如之前用过的 chars
    fn iter(&self) -> std::slice::Iter<T> {
        self.items.iter()
    }
    fn iter_mut(&mut self) -> std::slice::IterMut<T> {
        self.items.iter_mut()
    }
    // 注意返回值，另外两个是 slice，这个是 vec
    // 不建议修改 into_iter 函数的命名，因为它会转移所有权，容易让使用者迷茫
    fn into_iter(self) -> std::vec::IntoIter<T> {
        self.items.into_iter()
    }
}

fn main() {
    let mut my_stack = Stack::new();
    my_stack.push(1);
    my_stack.push(2);
    my_stack.push(3);

    for item in my_stack.iter() { // item： &i32
        println!("Item {}", item);
    }
    println!("{:?}", my_stack);

    for item in my_stack.iter_mut() { // item： &mut i32
        *item *= 2;
    }
    println!("{:?}", my_stack);
    for item in my_stack.into_iter() { // item： i32
        println!("{}", item);
    }
    // println!("{:?}", my_stack);
}
```

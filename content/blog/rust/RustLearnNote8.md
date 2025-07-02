+++
title = "Rust 入门学习笔记（八）：迭代器"
slug = "rust_learn_note_8"
date = 2025-07-02
updated = 2025-07-03
description = "迭代器，迭代与循环，自定义迭代"
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

## 迭代器与迭代的关系
{{ admonition(type="warning", icon="tip", title="注意", text="施工中") }}

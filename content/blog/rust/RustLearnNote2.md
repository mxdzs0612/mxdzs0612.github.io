+++
title = "Rust 学习笔记（二）：所有权与复杂类型"
slug = "rust_learn_note_2"
date = 2025-06-24T20:50:07Z
updated = 2025-06-25T20:45:07Z
[taxonomies]
tags = ["Rust", "Learn"]
[extra]
summary = "所有权与复杂类型，结构体，枚举"
pinned = false
post_listing_date = "both"
+++

## Rust 内存管理模型
常见的内存管理模型有三种：
- C/C++：手动管理，效率高但易错
- Java/C#/Python：交给 GC，安全但 STW 伤害性能
- Rust：Onwership、Borrow Checker、Lifetime。编译期做检查，如果发现内存有问题会直接编译不通过，通过所有权机制避免一些问题产生。安全且理论性能接近C，但更难

### Stop the world
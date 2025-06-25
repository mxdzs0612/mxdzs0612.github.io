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
Stop the world（STW）与垃圾回收（GC）相关，是指在 GC 进行时系统暂停程序的运行。

STW 主要用于描述一种全局性的暂停，即应用所有线程都会停止，以便垃圾回收器安全工作。对于需要低延迟高性能的程序，这种全局性的停止会导致一些潜在的性能问题。

并非所有 GC 算法都会导致 STW，一些现代的算法采用并发或者增量的 GC，减少全局停顿带来的影响。

### C 内存错误大全
1. 内存泄漏 
```C
int* ptr = new int;
// delete ptr;
```
2. 悬空指针
```C
int* ptr = new int;
delete ptr;
```
3. 重复释放
```C
int* ptr = new int;
delete ptr;
delete ptr;
```
4. 数组越界
```C
int arr[5];
arr[5] = 5;
```
5. 野指针
```C
int* ptr;
ptr = 10;
```
6. Use after free
```C
int* ptr = new int;
delete ptr;
*ptr = 10;
```
7. Stack Overflow
```
递归溢出
```
8. 不匹配
```
new/delete malloc/free
```

### Rust 内存管理模型
```rust
fn get_length(s: String) -> usize {
    println!("String: {}", s);
    s.len()
}

fn main() {
    // copy  move
    // copy
    let c1 = 1;
    let c2 = c1; // 基础类型，此处执行 copy 操作
    println!("{}", c1);
    let s1 = String::from("value");
    // let s2 = s1 // s1 的所有权转移给s2
    //  println!("{s1}"); //  value borrowed here after move
    let s2 = s1.clone(); // 深拷贝
    println!("{}", s1);

    println!("{}", s1.len()); // 所有权默认会交给函数
    // println!("{}", s1); // 函数结束之后 s1 也销毁了

    let len = get_length(s2);
    println!("{}", len);

    let back = first_word("hello world");
    println!("{}", back);
    let back = first_word("we are the world");
    println!("{}", back);
}

fn dangle() -> String {
    // 直接传实体 String，而非 &str
    "hello".to_owned()
}

// 静态的生命周期，会污染全局作用域，不推荐
fn dangle_static() -> &'static str {
    "jdkfj"
}

// String 与 &str vec u8 的借用
// 这种情况生命周期不会变化
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();
    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }
    &s[..]
}
```

## 字符串数据类型
String：堆分配的可变字符串类型
```rust
pub struct String {
    vec: Vec<u8>,
}
```

&str：字符串字面量，字符串的不可变切片引用，栈上分配，UTF-8 编码，由指针和长度构成
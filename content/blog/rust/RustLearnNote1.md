+++
title = "Rust 入门学习笔记（一）：变量与基础数据类型"
slug = "rust_learn_note_1"
date = 2025-06-21
updated = 2025-06-24
description = "变量，基础数据类型，元组和数组"
[taxonomies]
tags = ["Rust", "Learn"]
[extra]
pinned = false
post_listing_date = "both"
+++
<!-- 抄了一份 Rust 的新手教程，先丢个链接方便自己找，有空再来整理吧。 -->
<!-- [地址](https://juejin.cn/post/7150187051621548046) -->

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
Rust 的变量默认是不可变的！

不可变性是 Rust 实现可靠性和安全性的关键，有效防止数据竞争和并发问题。

如果希望可变，需要使用`mut`关键字。

### Shawowing
Rust 允许隐藏变量。意思是可以声明一个与前一个变量同名的新变量（值、类型、可变性均可改变）。

这个变量是隐藏了，而不是消失了，具体可以看下面的例子。

***
### 例子
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
常量和静态变量的名称通常应全部大写，且在单词间加下划线。它们的区别如下：

### 常量 const
常量的值必须是编译时已知的常量表达式，必须制定类型和值。

常量被直接作用于底层编译结果，而不是简单的字符替换。

作用域是**块**级，只在声明的作用域内可见。

### 静态变量 static
静态变量是在运行时分配内存的。

静态变量是**变量**，并非不可变的，可在`unsafe`代码段中修改。

> 应尽量避免使用`unsafe`。

生命周期是整个程序的运行时间。

***
### 例子
```rust
static MY_STATIC: i32 = 42;
static mut MY_MUT_STATIC: i32 = 42;

fn main() {
    // const
    const SECOND_HOUR: usize = 3_600;
    const SECOND_DAY: usize = 24 * SECOND_HOUR; // compile-time constant

    {
        const SE: usize = 1_000;
        println!("{SE}");
    }

    // println!("{SE}");  // 无法打印
    println!("{}", SECOND_DAY);
    // MY_STATIC = 2; // 不可修改
    println!("{MY_STATIC}");
    unsafe {
        MY_MUT_STATIC = 32;
        println!("{MY_MUT_STATIC}");
    }
    // println!("{MY_MUT_STATIC}"); // 不可打印，只能在`unsafe`里打印
}
```

## 基础数据类型
| 类型 | 描述 | 子类型 |
| --- | --- | --- |
| 整型 | 有符号整型 | i8，i16，i32（默认推断），i64，i128 |
|  | 无符号整型 | u8，u16，u32，u64，u128 |
|  | 平台决定大小整型 | usize，isize |
| 浮点型 | 浮点数 | f32，f64（强烈建议使用） |
| 布尔型 | true/false | bool |
| 字符型 | 使用单引号表示，可以表示Unicode表情 | char  |
***

### 例子
```rust
fn main() {
    // 进制的字面量
    let a1 = -125;
    let a2 = 0xFF;
    let a3 = 0o13;
    let a4 = 0b10;
    println!("{a1} {a2} {a3} {a4}"); // -125 255 11 2
    // Max Min
    println!("u32 max: {}", u32::MAX); // 4294967295
    println!("u32 min: {}", u32::MIN); // 0
    println!("i32 max: {}", i32::MAX); // 2147483647
    println!("i32 min: {}", i32::MIN); // -2147483648
    println!("usize max: {}", usize::MAX); // 2^64 - 1 = 18,446,744,073,709,551,615
    // 基础数据类型占用的空间大小
    println!("isize is {} bytes", std::mem::size_of::<isize>()); // 8
    println!("usize is {} bytes", std::mem::size_of::<usize>()); // 8
    println!("u64 is {} bytes", std::mem::size_of::<u64>()); // 8
    println!("i64 is {} bytes", std::mem::size_of::<i64>()); // 8
    println!("i32 is {} bytes", std::mem::size_of::<i32>()); // 4

    // float
    let f1: f32 = 1.23234;
    let f2: f64 = 9.88888;
    // 四舍五入
    println!("Float are {:.2} {:.2}", f1, f2); // 1.23 9.89

    // bool
    let is_ok = true;
    let can_ok: bool = false;
    println!("is ok? {is_ok} can ok? {can_ok}");  // true false
    println!(
        "is ok or can ok ?{}, can ok and is ok? {}",
        is_ok || can_ok,
        is_ok && can_ok
    ); // true false
    // char
    let char_c = 'C';
    let emo_char = '😀';
    println!("You Get {char_c} feel {emo_char}");
    println!("{}", emo_char as usize); // 128512
    println!("{}", emo_char as i32); // 128512
}

```

## 元组和数组
相同点：
- 都是复合类型（Compound Types，Vec Map这些是集合类型 Collection Types）
- 长度固定
- 都可以设置成可变的（mut）
不同点：
- 元组（Tuples）是不同数据类型的复合数据
- 数组（Arrays）是同一类型的复合数据

### 数组
数组是固定长度的同构集合。

创建方式:
- [a, b, c]
- [value: size] 长度为 size 初始化值为 value 的数组

获取元素: arr[index]

获取长度: arr.len()

可以放在 for 循环中遍历。

### 元组
元组是固定长度的异构集合。

空元组是函数的默认返回值。

获取元素: tup.index

没有 length。

***
### 例子
```rust
fn main() {
    // tuple
    let tup = (0, "hi", 3.4);
    // 不支持 {tup.0} 这种形式
    println!("tup elements {} {} {}", tup.0, tup.1, tup.2);

    let mut tup2 = (0, "hi", 3.4);
    println!("tup2 elements {} {} {}", tup2.0, tup2.1, tup2.2);
    tup2.1 = "f"; // 必须是同一类型，比如 tup2.0 = "f"; 是不行的
    println!("tup2 elements {} {} {}", tup2.0, tup2.1, tup2.2);

    // :?表示全部打印，这里只会打印 `()`
    let tup3 = ();
    println!("tup3 {:?}", tup3);
    // println!("tup3 {}", tup3); // 报错没有实现特质

    println!("Array");
    let mut arr = [11, 12, 34];
    arr[0] = 999;
    println!("arr len {} first element is {}", arr.len(), arr[0]);

    for elem in arr {
        println!(" {}", elem);
    }
    let ar = [2; 3];
    for i in ar {
        println!("{}", i); // 2 2 2
    }

    // ownership
    // 基础数据类型赋值是 copy
    let arr_item = [1, 2, 3];
    let tup_item = (2, "ff");
    println!("arr: {:?}", arr_item);
    println!("tup: {:?}", tup_item); // Clone
    let arr_ownership = arr_item;
    let tup_ownership = tup_item;
    println!("arr: {:?}", arr_item);
    println!("tup: {:?}", tup_item);
    let a = 3;
    let _a = a;
    println!("{a}");
    // copy
    // move ownership
    // 大部分复杂数据类型赋值默认都会执行 move
    let string_item = String::from("aa");
    let _string_item = string_item; // String 类型就把Ownership 进行move操作
    // println!("{string_item}"); // 无法打印，报错value borrowed here after move
}
```

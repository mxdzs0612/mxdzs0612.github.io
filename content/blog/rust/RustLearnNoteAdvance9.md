+++
title = "Rust 进阶学习笔记（九）：强类型"
slug = "rust_learn_note_adv_9"
date = 2025-07-20
updated = 2025-07-20
description = ""
# draft = true
[taxonomies]
tags = ["Rust", "Learn"]
[extra]
pinned = false
post_listing_date = "both"
+++

本节出处：[圣经-4.3深入类型](https://course.rs/advance/into-types/intro.html)

没想到吧，都到高级九了，类型还能回归。因为这节还是有一点难的，值得单开一篇。比如说之前遇到过很多自动解引用的情况，这其实就是一种类型转换。

***

## 类型转换
Rust 是强类型的语言，也是类型安全的语言，因此在 Rust 中进行类型转换并不容易。

### as 转换
Rust 不允许两种不同的类型进行比较。最简单的解决办法就是用`as`操作符来进行转换。
```rust
fn main() {
    let a: i32 = 10;
    let b: u16 = 100;
    // 报错， 无法直接比较
    // if a < b {
    if a < (b as i32) {
      println!("Ten is less than one hundred.");
    }
    // 常见转换
    let a = 3.1 as i8;
    let b = 100_i8 as i32;
    let c = 'a' as u8; // 将字符'a'转换为整数，97

   println!("{}, {}, {}", a, b, c)
}
```
每个类型能表达的数据范围不同，如果把范围较大的类型转换成较小的类型，会造成数据失准（实际上就是粗暴地把二进制高位截断了只保留地位），因此只能把范围较小的类型转换成较大的类型。可以这样查看数据类型的最大值：
```rust
let a = i8::MAX;
println!("{}",a);
```
除了类型的转换，`as`的另一个应用是内存地址和指针的转换：
```rust
let mut values: [i32; 2] = [1, 2];
let p1: *mut i32 = values.as_mut_ptr();
let first_address = p1 as usize; // 将p1内存地址转换为一个整数
let second_address = first_address + 4; // 4 == std::mem::size_of::<i32>()，i32类型占用4个字节，因此将内存地址 + 4
let p2 = second_address as *mut i32; // 访问该地址指向的下一个整数p2
unsafe {
    *p2 += 1;
}
assert_eq!(values[1], 3);
```

### TryInto 转换
`TryInto`是一种内置转换的替代方案，显然这是一个特质。因此，它更灵活更自由，能够让开发者拥有对类型转换的完全控制。
```rust
// 要使用一个特质的方法的时候，也需要把特质引入到当前的作用域中
// 实际上，这个方法可以不 use，因为会被 std::prelude 自动引入
// 这是预引入列表 https://doc.rust-lang.org/std/prelude/index.html
use std::convert::TryInto;

fn main() {
    let a: u8 = 10;
    let b: u16 = 1500;
    // unwrap 方法在发现错误时，会直接调用 panic 导致程序的崩溃退出
    // 这里只是演示，实际项目中应该进行处理
    let b_: u8 = b.try_into().unwrap();

    if a < b_ {
        println!("Ten is less than one hundred.");
    }
}
```
`try_into`会尝试进行一次转换，并返回一个`Result`，此时就可以对其进行相应的错误处理。`try_into`转换最常见的应用就是捕获大类型向小类型转换时导致的溢出错误：
```rust
fn main() {
    let b: i16 = 1500;

    let b_: u8 = match b.try_into() {
        Ok(b1) => b1,
        Err(e) => {
            println!("{:?}", e.to_string()); // out of range integral type conversion attempted
            0
        }
    };
}
```

### 通用转换
`as`和`try_into`只能用于数值类型的转换。对于一些复杂类型，如结构体，当然可以循环转换其中的所有字段。当然，Rust 还有一些办法进行通用转换。

#### 强制类型转换
Rust 不提供原生类型的隐式转换，但部分类型可以进行一些隐式强制转换。这往往需要一些自动强转点（coercion sites），最典型的就是所需类型已给出或可以自动推导的（不是推断）的场景。例如：
- let 语句显式指定类型：`let _: &i8 = &mut 42;`
- 静态和常量项的声明
- 函数的参数，实参可以被自动强制转换成形参
- 实例化结构体、枚举或联合体
- 函数 return 的结果

自动强转允许发生在下列类型之间：
- 反射性场景，如 T 到 U，如果 T 是 U 的一个子类型
- 传递性场景，如 T_1 到 T_3，当 T_1 可自动强转到 T_2、同时 T_2 又能自动强转到 T_3（注意这个还没有得到完全支持）
- &mut T 到 &T
- *mut T 到 *const T
- &T 到 *const T
- &mut T 到 *mut T
- &T 或 &mut T 到 &U（如果 T 实现了 Deref<Target = U>）
- &mut T 到 &mut U（如果 T 实现了 DerefMut<Target = U>）
- 类型构造器 TyCtor(T) 到 TyCtor(U)（TyCtor(T) 是`&T/&mut T/*const T/*mut T/Box<T>`）
- ! 到任意 T
- 非捕获闭包到函数指针

还存在非固定 size 的自动转换：
- [T; n] 到 [T]
- T 到 dyn U，（当 T 实现 U + Sized, 并且 U 是对象安全的时）
- Foo<..., T, ...> 到 Foo<..., U, ...> 要求
  - Foo 是一个结构体。
  - T 实现了`Unsize<U>`。
  - Foo 的最后一个字段是和 T 相关的类型。
  - 如果这最后一个字段是类型`Bar<T>`，那么`Bar<T>`实现了`Unsized<Bar<U>>`。
  - T 不是任何其他字段的类型的一部分。
  - T 实现了`Unsize<U>`或`CoerceUnsized<Foo<U>>`，且类型`Foo<T>`可以实现`CoerceUnsized<Foo<U>>`

注意匹配特质时不会做除方法外的强制转换（T 到 U 不代表 impl T 到 impl U）。

在某些上下文中，编译器必须将多个类型强制在一起，以尝试找到最通用的类型，这就是最小上界自动强转（Least Upper Bound, LUB）。下面是一个 LUB 的例子：
```rust
#![allow(unused)]
fn main() {
    // LUB 共支持五种情况：
    let (a, b, c) = (0, 1, 2);
    // if分支的情况
    let bar = if true {
        a
    } else if false {
        b
    } else {
        c
    };

    // 匹配臂的情况
    let baw = match 42 {
        0 => a,
        1 => b,
        _ => c,
    };

    // 数组元素的情况
    let bax = [a, b, c];

    // 多个返回项语句的闭包的情况
    let clo = || {
        if true {
            a
        } else if false {
            b
        } else {
            c
        }
    };
    let baz = clo();

    // 检查带有多个返回语句的函数的情况
    fn foo() -> i32 {
        let (a, b, c) = (0, 1, 2);
        match 42 {
            0 => a,
            1 => b,
            _ => c,
        }
    }
}
```

#### 点操作符
方法调用会用到点操作符，这个过程中会发生大量类型转换，比如自动引用、自动解引用、强制类型转换等。

假设一个 value 拥有类型 T，然后有一个方法 foo，编译器会按如下顺序尝试进行类型转换：
1. 值方法调用：编译器检查它是否可以直接调用 T::foo(value)
2. 引用方法调用：如果方法类型错误或者特质没有针对 Self 进行实现（再次强调，特质不能强制转换）编译器会尝试增加自动引用，如`<&T>::foo(value)`和`<&mut T>::foo(value)`
3. 解引用方法调用：编译器会使用`Deref`特质解引用 T ，然后再进行尝试。例如`T: Deref<Target = U>`，编译器会用 U 尝试
4. 如果 T 是定长类型，编译器会尝试将 T 转为不定长类型，如 [i32; 2] 到 [i32]
5. 如果以上都不行，编译器会报错。





































{{ admonition(type="warning", title="注意", text="施工中") }}
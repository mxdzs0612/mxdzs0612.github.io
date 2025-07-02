+++
title = "Rust 入门学习笔记（六）：泛型"
slug = "rust_learn_note_6"
date = 2025-07-01T10:30:10Z
updated = 2025-07-01
description = "泛型结构体与泛型函数"
[taxonomies]
tags = ["Rust", "Learn"]
[extra]
pinned = false
post_listing_date = "both"
+++

<!-- {{ dimmable_image(src="images/avatar.jpg", alt="x") }} -->
<!-- 绝对路径 -->
<!-- ![这是图片](/images/avatar.jpg "Magic Gardens") -->
## 泛型

泛型（Generic）是一种编程语言特性，允许代码中使用参数化类型，以便在不同地方使用相同的代码逻辑处理多种数据类型，无需为每种类型编写单独的代码。

作用：
- 提高代码重用性
- 提高代码可读性
- 提高代码抽象度

泛型常用于定义结构体/枚举、定义函数，还经常和特质有关。

### 泛型结构体
```Rust
#[derive(Debug)]
// T 可以改成任意字母，但常用 T/E/K/V/N
struct Point<T> {
    x: T,
    y: T,
}

#[derive(Debug)]
// 可以设置不同类型的泛型
struct PointTwo<T, E> {
    x: T,
    y: E,
}

fn main() {
    let c1 = Point { x: 1.0, y: 2.0 };
    let c2 = Point { x: 'x', y: 'y' };
    println!("c1 {:?} c2{:?}", c1, c2);
    let c = PointTwo { x: 1.0, y: 'z' };
    println!("{:?}", c);
    // Rust 能做到零成本抽象（Zero-Cost Abstraction），即抽象没有性能损耗，所以使用泛型收益大
}
```

### 泛型与函数
在 Rust 中泛型也可以用于函数，使得函数能够处理多种类型的参数，提高代码的重用性和灵活性。

```rust
// 交换
// 一般泛型会和特质一起使用，所以能够限定参数范围。这里仅作为演示。
fn swap<T>(a: T, b: T) -> (T, T) {
    (b, a)
}

struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {
    // 在 impl 后面声明了 T，impl 中的函数就不需要再声明一遍，可以直接用了（当然再声明一次也不算错）
    fn new(x: T, y: T) -> Self {
        Point { x, y }
    }
    // 方法最好返回泛型的引用，否则如果 T 没实现 Copy，易出问题
    fn get_coordinates(&self) -> (&T, &T) {
        (&self.x, &self.y)
    }
}

fn main() {
    let result = swap(0.1, 1.0); // 这个是省略写法，省略部分均可自动推断
    // 不省略的写法，也可主动声明为其他类型（如 f32）
    let result: (f64, f64) = swap::<f64>(0.1, 1.0);
    println!("{:?}", result);
    let str2 = swap("hh", "tt");
    println!("str2.0 {} str2.1 {}", str2.0, str2.1);
    let str2 = swap(str2.0, str2.1);
    println!("str2.0 {} str2.1 {}", str2.0, str2.1);

    let i32_point = Point::new(2, 3);
    let f64_point = Point::new(2.0, 3.0);
    let (x1, y1) = i32_point.get_coordinates();
    let (x2, y2) = f64_point.get_coordinates();
    println!("i32 point: x= {} y= {}", x1, y1);
    println!("f64 point: x= {} y= {}", x2, y2);
    // 传 &str 程序也能正常执行，但可能会有生命周期管理问题
    let string_point = Point::new("xxx".to_owned(), "yyyy".to_owned());
    println!("string point x = {} y = {}", string_point.x, string_point.y);
}
```

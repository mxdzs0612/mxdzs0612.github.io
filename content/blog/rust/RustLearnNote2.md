+++
title = "Rust 入门学习笔记（二）：所有权与复杂类型"
slug = "rust_learn_note_2"
date = 2025-06-24
updated = 2025-06-27
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

&str：字符串**字面量**，字符串的不可变切片引用，栈上分配，UTF-8 编码，由指针和长度构成

区别：String 有所有权，字面量没有。
- 结构体属性尽量使用 String。
  - 如果不使用显式声明生命周期，就无法使用 &str。
  - 可能有隐患。
- 函数参数在不想交出所有权的情况下，建议使用 &str
  - &str 参数可以传递 &str 和 &String。
  - &String 参数只能传递 &String。

***
### 例子
```rust
struct Person<'a> {
    // 标注生命周期，代表字面量和结构体拥有相同的生命周期 'a
    name: &'a str,
    color: String,
    age: i32,
}

// 传 &String &str 均可
fn print(data: &str) {
    println!("{}", data);
}

// 只能 &String
fn print_string_borrow(data: &String) {
    println!("{}", data);
}

fn main() {
    // String &str
    let name = String::from("Value C++");
    // 三种方式：
    // String::from
    // to_string()
    // to_owned()
    let course = "Rust".to_string();
    let new_name = name.replace("C++", "CPP");
    println!("{name} {course} {new_name}");
    let rust = "\x52\x75\x73\x74"; // ascii
    println!("{rust}");

    // struct
    // &str
    let color = "green".to_string();
    // String
    let name = "John";
    let people = Person {
        name: name,
        color: color,
        age: 89,
    };
    // func
    let value = "value".to_owned();
    print(&value);
    print("value");
    // print_string_borrow("value"); // 不能传字面量
    print_string_borrow(&value);
}
```

## 枚举与匹配

### 枚举
枚举（enum）是自定义的数据类型，表示一组具有离散可能值的变量。
- 每种可能值都成为变体（variant）
- 用法：{枚举名}::{变体名}

枚举可以让代码更严谨易读安全。

枚举支持内嵌类型，使得 rust 表达能力非常强，抽象度非常高。
```rust
enum Shape {
  Circle(f64),
  Rectangle(f64, f64),
  Square(f64),
}
```

常用枚举类型
```rust
pub enum Option<T> {
    None,
    Some<T>,
}

pub enum Result<T, E> {
    Ok(T),
    Err<E>,
}
```

### 匹配模式 match
匹配模式必须覆盖所有变体，可以使用`_`、`..=`、`if`等来实现。
```rust
match number {}
    0 => println!("zero");
    1 | 2 => println!("one or two");
    3..=9 => println!("three to nine");
    n if n % 2 == 0  => println!("even");
    _  => println!("others");
```

***
### 例子
```rust
use std::collections::btree_set::Union;

// 简单枚举类型
enum Color {
    Red,
    Yellow,
    Blue,
}

fn print_color(my_color: Color) {
    match my_color {
        Color::Red => println!("Red"),
        Color::Yellow => println!("Yellow"),
        Color::Blue => println!("Blue"),
        // 如果没有覆盖所有枚举值，这个地方就需要加一个下划线表示 default
    }
}

// 复杂枚举类型
enum BuildingLocation {
    Number(i32),
    Name(String), // 不要用 &str，否则所有权会出现问题
    Unknown,
}

// 关联函数，结构体/枚举都可以用
// 用法： impl + 类型名
impl BuildingLocation {
    fn print_location(&self) {
        match self {
            // BuildingLocation::Number(44)
            BuildingLocation::Number(c) => println!("building number {}", c),
            // BuildingLocation::Name("ok".to_string())
            BuildingLocation::Name(s) => println!("building name {}", *s),
            BuildingLocation::Unknown => println!("unknown"),
        }
    }
}

fn main() {
    let a = Color::Red;
    print_color(a);
    // let b = a; // 报错。此时 a 所有权已经被交给函数了

    let house = BuildingLocation::Name("fdfd".to_string());
    let house = BuildingLocation::Number(1);
    let house = BuildingLocation::Unknown;
    house.print_location();
}
```

## 结构体
结构体是用户自定义的数据类型，可以用于创建各种自定义的数据结构。
```rust
struct Point {
    x: i32,
    y: i32,
}
```
结构体中的每条数据（如上述 x,y）称为属性（field），可以通过`.`运算符来访问结构体中的属性。

下面会通过结构体引入一些概念。这些概念通常会和结构体/特质/枚举一起使用，且定义在类型的`impl`中，和类型本身定义分离。

### 方法
严格来说 Rust 中并不存在方法，只有关联函数。本章概念是为了方便有面向对象语言基础的初学者理解 Rust。

这里的方法是指通过实例调用的关联函数，参数中往往包含 &self（实例引用）、&mut self（可变实例引用）、self（所有权移动）。

```rust
impl Point {
    fn distance(&self, other: &Other) -> f64 {
        let dx = (self.x - other.x) as f64;
        let dy = (self.y - other.y) as f64;
        (dx * dx + dy * dy).sqrt()
    }
}
```

### 关联函数
关联函数是与类型相关联的函数，调用时使用`类型名::函数名`。
```rust
impl Point {
    // 函数名可为任意
    // Self 是指结构体的名字
    fn new(x: u32, y: u32) -> Self {
        Point { x, y }
    }
}
```

###  关联变量
这里的关联变量是指和类型相关的变量。调用时使用`类型名::变量名`。
```rust
impl Point {
    const PI: f64 = 3.14;
}
```

***
### 例子
```rust
enum Flavor {
    Spicy,
    Sweet,
    Fruity,
}

struct Drink {
    flavor: Flavor,
    price: f64,
}

impl Drink {
    // 关联变量
    const MAX_PRICE: f64 = 10.0;
    // 方法
    fn buy(&self) {
        // 也可以写成 Self::MAX_PRICE
        if self.price > Drink::MAX_PRICE {
            println!("I am poor");
            return;
        }
        println!("buy it");
    }
    // 关联函数
    fn new(price: f64) -> Self {
        Drink {
            flavor: Flavor::Fruity,
            price,
        }
    }
}

fn print_drink(drink: Drink) {
    match drink.flavor {
        Flavor::Fruity => println!("fruity"),
        Flavor::Spicy => println!("spicy"),
        Flavor::Sweet => println!("sweet"),
    }
    println!("{}", drink.price);
}

fn main() {
    let sweet = Drink {
        flavor: Flavor::Sweet,
        price: 6.0,
    };
    println!("{}", sweet.price);
    print_drink(sweet); // 注意这里会交出所有权
    let sweet = Drink::new(12.0);
    sweet.buy();
}
```

## 所有权
Rust 所有权总是会涉及到`self`，但这个`self`和面向对象的`this`完全不一样。

### 所有权规则
1. Rust 中每个 value 都有一个所有者。
2. 同时只能有一个所有者。
3. value 超出作用域时会自动销毁。

因此，每次将值从一个位置传到另一个位置时，borrow checker 都会重新评估所有权。

### 值传递语义
1. 不可变借用（immutable borrow）：值的所有权归发送方所有，接收方直接接收对该值的引用而不是副本。接收方不能通过引用来修改它指向的值。释放资源的行为由发送方负责。（&self或self: &Self）
2. 可变借用（mutable borrow）：值的所有权和释放责任依然由发送方承担，但接收方可以通过接收的引用来修改值。在同一时刻只能有一个可变借用。（&mut self或self: &mut Self）
3. 所有权转移（move）：值的所有权会直接移交给接收方，发送方在引用移动的上下文之后就不能再使用该引用（报错 borrow of moved value）。（self或self: Self）

> 上述括号中或后面的`self`可以起任意变量名字。

***
### 例子
```rust
struct Counter {
    number: i32,
}

impl Counter {
    fn new(number: i32) -> Self {
        Self { number }
    }

    // 不可变借用
    fn get_number(&self) -> i32 {
        self.number
    } // 相当于 Counter::get_number(self: &Self)
    // 可变借用
    fn add(&mut self, increment: i32) {
        self.number += increment;
    } // 相当于 Counter::add(self: &mut Self, increment:)
    // move
    fn give_up(self) {
        println!("free {}", self.number);
    }
    // 两个入参所有权都释放了
    fn combine(c1: Self, c2: Self) -> Self {
        Self {
            number: c1.number + c2.number,
        }
    }
}

fn main() {
    let mut c1 = Counter::new(0);
    println!("c1 number {}", c1.get_number());
    println!("c1 number {}", c1.get_number());
    c1.add(2);
    println!("c1 number {}", c1.get_number());
    println!("c1 number {}", c1.get_number());
    c1.give_up();
    // println!("c1 number {}", c1.get_number()); // 无法打印了

    let c1 = Counter::new(2);
    let c2 = Counter::new(1);
    let c3 = Counter::combine(c1, c2);
    // println!("c1 number {}", c1.get_number());
    // println!("c2 number {}", c2.get_number());

    println!("c3 number {}", c3.get_number());
}
```

## 堆栈

### 堆和栈
- 栈（stack）：先入后出
  - 按照获取值的顺序存储值，并按照相反顺序删除值
  - 高效，函数作用域就在栈上
  - 栈上存储的所有数据的大小都必须已知
- 堆（heap）：
  - 堆是无人管理的自由内存区，规律性较差，需要时手动申请，不需要时手动释放（当然在 Rust 中是自动管理的）
  - 长度不固定

基础类型、元组和数组、结构体与枚举都会存储在栈上；Box、Rc、String、集合（Vec等）会指向堆。注意如果结构体、枚举中包含堆上的类型，也会指向堆。

### Box
Box 是一个智能指针，提供堆上分配内存的所有权（强制入堆），并且在复制和移动的时候保持数据的**唯一**所有权。能够有效避免一些内存管理问题。

Box 的用途主要包括所有权转移、释放内存、解引用、构建递归数据结构。

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let boxed_point = Box::new(Point { x: 10, y: 20 });
    println!("x:{}, y:{}", boxed_point.x, boxed_point.y);

    let mut boxed_point = Box::new(32); // 自动推断成Box<i32>
    println!("{}", boxed_point); // 32
    println!("{}", *boxed_point); // 32
    // boxed_point += 10; // box 是一个指针，无法直接修改
    *boxed_point += 10; // *解引用
    println!("{}", *boxed_point); // 42
}
```

### 拷贝和克隆
- move 所有权转移
- clone 深拷贝，需显式调用clone()方法
- copy 基于 clone 的标记特质
> Rust 中特质（trait）是定义共享行为的机制，clone就是特质。而标记特质（marker trait）是一个没有任何方法的特质，表面上没有定义任何行为，实际上是用于向编译器传递的**附加信息**，以改变类型的默认行为。

一般栈上的类型都默认实现了 copy，但结构体等默认是 move，实现 copy 需要实现 copy 特质，或者实现 clone 特质并调用 clone。

***
例子
```rust
// 宏，自动为结构体生成一些特质
#[derive(Debug, Clone, Copy)]
struct Book {
    page: i32,
    rating: f64,
    // 如果结构体有一个 String，就无法自动生成 Copy，需要自己实现
    // name: String
}

fn main() {
    let x = vec![1, 2, 3, 4];
    // let y = x; // 所有权转移给 y
    let y = x.clone();
    println!("{:?}", y);
    println!("{:?}", x);

    let x = "ss".to_string();
    // let y = x;
    let _x = x.clone();
    println!("{:?}", x);
    // vec 和 string 都有 clone 的实现

    let b1 = Book {
        page: 1,
        rating: 0.1,
    };
    let b2 = b1; // 默认是 move
    // 增加 #[derive(Debug, Clone, Copy)] 后，由于结构体中都是基础类型，可以通过宏为结构体自动生成 copy
    println!("{:?}", b1);
}
```

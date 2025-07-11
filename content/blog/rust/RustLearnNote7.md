+++
title = "Rust 入门学习笔记（七）：特质"
slug = "rust_learn_note_7"
date = 2025-07-01T19:30:00Z
updated = 2025-07-01
description = "特质，Box，特质与泛型，重载，多态继承"
# draft = true
[taxonomies]
tags = ["Rust", "Learn"]
[extra]
pinned = false
post_listing_date = "both"
+++

## 特质
特质（Traits，有的地方也翻译成*特征*）是一种定义方法签名的机制。

特质允许提供一组方法的签名，但不提供具体实现。这些方法签名可以包括参数和返回类型，可选择提供或不提供默认实现（通常不提供）。

任何类型都可以实现特质，只要提供特质中定义的所有方法，使得我们可以为不同类型提供相同的行为。此时需要显式声明特质中定义的方法。

### 特质的特点
- 内置常量：可以内置常量（const），其生命周期是静态的
- 默认实现：可以提供默认方法实现，如果类型没有提供自定义实现，就会使用默认实现
- 多重实现：类型可以实现多个特质，将不同行为组合在一起。
- 特质边界：特质可以在泛型中作为类型约束。也就是说，特质可用于限定泛型类型的范围，泛型类型必须实现了这些特质。
- 特质别名：特质支持别名（alias），为复杂的特质组合创建简洁的别名，方便引用。

***
### 例子
```rust
trait Greeter {
    fn greet(&self);
    // 默认实现
    fn hello() {
        println!("hello");
    }
}

struct Person {
    name: String,
}

// 实现特质的语法： impl 特质名 for 结构名
impl Greeter for Person {
    fn greet(&self) {
        println!("greet {}", self.name);
    }
}

fn main() {
    let person = Person {
        name: "xxx".to_owned(),
    };
    person.greet();
    // 声明了特质，就可以使用特质的默认方法
    Person::hello();
}
```

## 特质对象和 Box

### 特质对象
特质对象（Trait Object）是实现了特定特质的类型的实例。

和面向对象语言的对象不同，特质对象是在运行时动态分配的“对象”。也称作“运行时泛型”（普通的泛型类型在编译期就分配了），比泛型更灵活。

在集合中可以混用一些不同的类型对象，方便处理相似数据。

虽然可能存在一定性能损耗，但一般还是建议使用特质对象而非泛型。

### dyn 关键字
Rust 中，`dyn`关键字用于声明特质对象的类型。特质对象的类型在编译时是未知的，为了让编译器知道正在处理的是特质对象，需要在特质名称前面添加`dyn`关键字。

也就是说，`dyn`关键字的作用是指示编译器处理特质对象。

### 特质对象的数据传输
- 不可变引用 `&dyn Trait`
- 可变引用 `&mut dyn Trait` （很少用到）
- 所有权转移 `Box<dyn Trait>`

### Box
特质需要用`Box<dyn Trait>`实现所有权转移（即 Move）。如果需要在函数调用之间传递特质的所有权，并且希望避免在栈上分配大量的内存，可以使用`Box<dyn Trait>`。

***
### 例子
```rust
struct Obj {}
trait Overview {
    fn overview(&self) -> String {
        String::from("overview")
    }
}

impl Overview for Obj {
    // 重新定义，覆盖默认
    fn overview(&self) -> String {
        String::from("Obj")
    }
}
// 不可变引用的传参方式
fn call_obj(item: &impl Overview) {
    println!("Overview {}", item.overview());
}
// Move
fn call_obj_box(item: Box<dyn Overview>) {
    println!("Overview {}", item.overview());
}

trait Sale {
    fn amount(&self) -> f64;
}

// 元组结构体
struct Common(f64);
impl Sale for Common {
    fn amount(&self) -> f64 {
        self.0
    }
}

struct TenDiscount(f64);
impl Sale for TenDiscount {
    fn amount(&self) -> f64 {
        self.0 - 10.0
    }
}

struct TenPercentDiscount(f64);
impl Sale for TenPercentDiscount {
    fn amount(&self) -> f64 {
        self.0 * 0.9
    }
}

fn calculate(sales: &Vec<Box<dyn Sale>>) -> f64 {
    // 对集合所有元素应用 amount 函数，并将结果集合的值相加
    sales.iter().map(|sale| sale.amount()).sum()
}

fn main() {
    let a = Obj {};
    call_obj(&a);
    println!("{}", a.overview());
    let b_a = Box::new(Obj {});
    call_obj_box(b_a);
    // println!("{}", b_a.overview()); // call_obj_box 使用 move 传参，所有权转移了，所以此处无法打印
    // 集合
    let c: Box<dyn Sale> = Box::new(Common(100.0));
    let t1: Box<dyn Sale> = Box::new(TenDiscount(100.0));
    let t2: Box<dyn Sale> = Box::new(TenPercentDiscount(100.0));

    let sales: Vec<Box<dyn Sale>> = vec![c, t1, t2]; // 如果变量没有显式声明 Sale（即写成了 let c = xxx），此处不声明 Vec<Box<dyn Sale>> 会报错 mismatched types
    // 一般情况下，变量后续还要用则需显式声明，不再使用可以在集合处声明，让变量自动推断

    println!("pay {}", calculate(&sales)); // 280
}
```

## 特质对象和泛型
泛型能够提供特质对象的另一种写法。

### 写法
- 单个特质
  - impl 写法：可以是不同类型：`fn call(item1: &impl Trait, item2: &impl Trait);`
  - 泛型写法：同一泛型必须是相同类型：`fn call_generic<T: Trait>(item1: &T, item2: &T);`
> impl 写法是一种语法糖，实际上的完整写法是泛型写法，也常被称作**特质约束（Trait Bound）**
- 多个特质
  - 语法糖形式：`fn call(item1: &(impl Trait + AnotherTrait));`
  - 特质约束（推荐）：`fn call_generic<T: Trait + AnotherTrait>(item1: &T);`
  - where 约束（更推荐，清晰）：`fn call_generic<T>(item: &T) where T: Trait + AnotherTrait,`

***
### 例子
```rust
trait Overview {
    fn overview(&self) -> String {
        String::from("Course")
    }
}

trait Another {
    fn hell(&self) {
        println!("welcome to hell");
    }
}

struct Course {
    headline: String,
    author: String,
}

impl Overview for Course {}
impl Another for Course {}

struct AnotherCourse {
    headline: String,
    author: String,
}

impl Overview for AnotherCourse {}

// impl 的写法
fn call_overview(item: &impl Overview) {
    println!("Overview {}", item.overview());
}
// 泛型写法，限定泛型需实现 Overview
fn call_overview_generic<T: Overview>(item: &T) {
    println!("Overview {}", item.overview());
}

fn call_overviewT(item: &impl Overview, item1: &impl Overview) {
    println!("Overview {}", item.overview());
    println!("Overview {}", item1.overview());
}

fn call_overviewTT<T: Overview>(item: &T, item1: &T) {
    println!("Overview {}", item.overview());
    println!("Overview {}", item1.overview());
}

// 多绑定
fn call_mul_bind(item: &(impl Overview + Another)) {
    println!("Overview {}", item.overview());
    item.hell();
}
// where 的写法
fn call_mul_bind_generic<T>(item: &T)
where
    T: Overview + Another,
{
    println!("Overview {}", item.overview());
    item.hell();
}

fn main() {
    let c0 = Course {
        headline: "xx".to_owned(),
        author: "yy".to_owned(),
    };
    let c1 = Course {
        headline: "ff".to_owned(),
        author: "yy".to_owned(),
    };

    let c2 = AnotherCourse {
        headline: "ff".to_owned(),
        author: "yz".to_owned(),
    };
    // 两种写法调用时是一样的
    call_overview(&c1);
    call_overview_generic(&c1);

    call_overviewT(&c1, &c2);
    // call_overviewTT(&c1, &c2); // 类型不同不能调用
    // 两种写法调用时是一样的
    call_overviewTT(&c1, &c0);
    call_overviewT(&c1, &c0);

    call_mul_bind(&c1);
    // call_mul_bind(&c2); // 报错 c2 没有满足 Another 特质
    call_mul_bind_generic(&c1);
}
```

## 重载操作符
Rust 重载只需要实现相应的特质。

### 示例：为结构体实现加号
```rust
use std::ops::Add; // 这就是一个特质

// 泛型在编译时确定，性能好
#[derive(Debug)]
struct Point<T> {
    x: T,
    y: T,
}

// T的这样类型 它可以执行相加的操作
impl<T> Add for Point<T>
where
    T: Add<Output = T>, // 指定 Add 特质的关联类型 Output 为 T。这意味着当两个 T 类型的值相加时，结果类型必须也是 T。从 Add 的源码中可以看到 Output 是 Add 的返回值
{
    type Output = Self;
    fn add(self, rhs: Self) -> Self::Output {
        Point {
            x: self.x + rhs.x,
            y: self.y + rhs.y,
        }
    }
}

fn main() {
    let i1 = Point { x: 1, y: 2 };
    let i2 = Point { x: 1, y: 3 };
    let sum = i1 + i2;
    println!("{:?}", sum); // Point { x: 2, y: 5 }
    let f1 = Point { x: 1.0, y: 2.2 };
    let f2 = Point { x: 1.0, y: 3.0 };
    let sum = f1 + f2; // Point { x: 2.0, y: 5.2 }
    println!("{:?}", sum);
}
```

## 多态和继承

### 继承
Rust 不支持面向对象，因此也不支持传统的继承概念，只是在思想上可以使用特质通过**层级化**的方式来完成继承的需求。

Rust 选择了函数化的编程方式，即通过组合和委托来平替继承。

### 多态
多态并非面向对象独有的概念，它通常是指同一个方法可以根据对象的不同类型表现出不同的行为。

多态允许一个接口或方法在不同的上下文中表现出不同的行为，这样做的好处是可以提高代码的灵活性和可扩展性，使得代码易于维护和理解。

Rust 中的多态无处不在。

***
### 例子
```rust
use std::collections::VecDeque;
// 多态
trait Driver {
    fn drive(&self);
}
struct Car;
impl Driver for Car {
    fn drive(&self) {
        println!("Car is driving");
    }
}

struct SUV;
impl Driver for SUV {
    fn drive(&self) {
        println!("SUV is driving");
    }
}

fn road(vehicle: &dyn Driver) {
    vehicle.drive();
}

// 继承思想：层级性特质
// 单向队列特质
trait Queue {
    fn len(&self) -> usize;
    fn push_back(&mut self, n: i32);
    fn pop_front(&mut self) -> Option<i32>;
}

// 双向队列特质
// 加了`:`后就会自动实现 Queue 里的东西，有点像“继承”
trait Deque: Queue {
    fn push_front(&mut self, n: i32);
    fn pop_back(&mut self) -> Option<i32>;
}

#[derive(Debug)]
struct List {
    // 偷懒的写法，直接调用 Rust 中的双向链表
    data: VecDeque<i32>,
}
// 非要实现继承，就会有很多无效代码
impl List {
    fn new() -> Self {
        let data = VecDeque::<i32>::new();
        Self { data }
    }
}

impl Deque for List {
    fn push_front(&mut self, n: i32) {
        self.data.push_front(n)
    }

    fn pop_back(&mut self) -> Option<i32> {
        self.data.pop_back()
    }
}

impl Queue for List {
    fn len(&self) -> usize {
        self.data.len()
    }

    fn push_back(&mut self, n: i32) {
        self.data.push_back(n)
    }

    fn pop_front(&mut self) -> Option<i32> {
        self.data.pop_front()
    }
}

fn main() {
    // 只要用了特质，基本肯定会有多态
    // 函数式编程到处都是多态
    road(&Car);
    road(&SUV);

    let mut l = List::new();
    l.push_back(1);
    l.push_front(0);
    println!("{:?}", l);
    l.push_front(2);
    println!("{:?}", l);
    l.push_back(2);
    println!("{:?}", l);
    println!("{}", l.pop_back().unwrap());
    println!("{:?}", l);
}
```

## 常见特质示例
```rust
// Debug Clone Copy PartialEq
// 注意层级，也就是结构体每层都要实现
#[derive(Debug, Clone, Copy)]
enum Race {
    White,
    Yellow,
    Black,
}

impl PartialEq for Race {
    fn eq(&self, other: &Self) -> bool {
        match (self, other) {
            (Race::White, Race::White) => true,
            (Race::Yellow, Race::Yellow) => true,
            (Race::Black, Race::Black) => true,
            _ => false,
        }
    }
}

#[derive(Debug, Clone)]
struct User {
    id: u32,
    name: String, // 因为存在 String，User 没有 Copy 的默认实现
    race: Race,
}

impl PartialEq for User {
    fn eq(&self, other: &Self) -> bool {
        self.id == other.id && self.name == other.name && self.race == other.race
    }
}

fn main() {
    let user = User {
        id: 3,
        name: "John".to_owned(),
        race: Race::Yellow,
    };
    println!("{:?}", user); // debug 特质，注意 User 和 Race 都要实现
    println!("{:#?}", user); // 更美观一些
    let user2 = user.clone(); // clone 特质，是实例的方法，调用方式存在区别。同样 User 和 Race 都要实现
    println!("{:#?}", user2);
    // println!("{:#?}", user); // 没有实现 Copy，所有权没了。想要打印需把 String 注释掉并 derive
    println!("{}", user == user2); // PartialEq 特质，需要 Race 实现等号判定
}
```

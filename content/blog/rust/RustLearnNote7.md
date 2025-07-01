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
特质（Traits）是一种定义方法签名的机制。

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

## Trait Object 和 Box

### 特质对象
特质对象（Trait Object）是实现了特定特质的类型的实例。

和面向对象语言的对象不同，特质对象是在运行时动态分配的“对象”。也称作“运行时泛型”（普通的泛型类型在编译期就分配了），比泛型更灵活。

在集合中可以混用一些不同的类型对象，方便处理相似数据。

虽然可能存在一定性能损耗，但一般还是建议使用特质对象而非泛型。

### dyn 关键字
Rust 中，`dyn`关键字用于声明特质对象的类型。特质对象的类型在编译时是位置的，为了让编译器知道正在处理的是特质对象，需要在特质名称前面添加`dyn`关键字。

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
// trait 不可变引用 \ Move
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
    // 对集合所有元素做一个映射，取到值后相加
    sales.iter().map(|sale| sale.amount()).sum()
}

fn main() {
    let a = Obj {};
    call_obj(&a);
    println!("{}", a.overview());
    let b_a = Box::new(Obj {});
    call_obj_box(b_a);
    // println!("{}", b_a.overview()); // 所有权转移了
    // 集合
    let c: Box<dyn Sale> = Box::new(Common(100.0));
    let t1: Box<dyn Sale> = Box::new(TenDiscount(100.0));
    let t2: Box<dyn Sale> = Box::new(TenPercentDiscount(100.0));

    let sales: Vec<Box<dyn Sale>> = vec![c, t1, t2]; // 如果变量没有显式声明 box（即直接 let c = xxx），此处不声明 Vec<Box<dyn Sale>> 会报错 mismatched types

    println!("pay {}", calculate(&sales)); // 280
}
```

## 特质和泛型
{{ admonition(type="warning", icon="tip", title="注意", text="WIP") }}

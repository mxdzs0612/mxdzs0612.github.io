+++
title = "Rust 入门学习笔记（五）：生命周期"
slug = "rust_learn_note_5"
date = 2025-06-30
updated = 2025-06-30
[taxonomies]
tags = ["Rust", "Learn"]
[extra]
summary = "借用，生命周期（函数，结构体）"
pinned = false
post_listing_date = "both"
+++

## 借用和生命周期

### 借用
可以认为，借用和引用是一个东西的两种描述。
- 引用（Reference）：更看重对象
  - 引用是一种变量的别名，通过`&`符号来创建，不会获取所有权
  - 引用既可以是不可变的`&T`，也可以是可变的`mut &T`
  - 引用可以在不传递所有权的情况下访问数据，安全且低开销
- 借用（Borrowing）：更看重行为
  - 通过引用来借用数据，从而在一段时间内访问而不拥有数据
  - 可变借用`&mut`允许修改数据，但在生命周期内只能同时存在一个可变借用
  - 不可变借用`&`不允许修改数据

### 借用检查规则
Rust 存在一个借用检查器（Borrow Checker）。它有以下规则：
1. 不可变引用规则：在任意时间，要么有一个可变引用，要么有一或多个不可变引用，二者不能同时存在。
2. 可变引用规则：在任意时间，只能有一个可变引用访问数据，以防止并发修改导致的数据竞争。
3. 生命周期规则：引用的生命周期必须在被引用的数据有效的时间范围内，以防止悬垂引用（即：引用的数据已经被销毁但引用还存在）。

### 生命周期
一般情况下 Borrow Checker 能够自行推断生命周期。如果不行，就需要在函数/结构体的签名中指定。

> 生命周期只需要在定义声明，调用的时候是不需要的。

代码块内部的生命周期不会销毁其外部变量的生命周期。

***
### 例子
```rust
fn main() {
    let mut s = String::from("Hello");
    // 不可变引用，可以同时有多个不可变引用
    let r1 = &s;
    let r2 = &s;
    println!("{} {}", r1, r2);

    let r3 = &mut s;
    println!("{}", r3);
    // println!("{} {}", r1, r2); // 存在可变引用，不可变引用就不能用了

    let result: &str;
    {
        // result = "ff"; // 定义初始化可以分离。也就是说可以在块中初始化，此时外面能正常打印 ff
        let r4 = &s;
        result = ff(r4);
    }
    // println!("r4 {}", r4); // 生命周期在代码块内结束了，无法打印
    println!("{}", result); // 正常打印 hello
}

// 函数生命周期会自动推导为`'0`，也可以手动定义，一般会用 a b out...
fn ff<'a>(s: &'a str) -> &'a str {
    s
}
```

## 函数生命周期
任何引用都有生命周期。
- 大多数情况下，生命周期都是隐式推断的。
- 生命周期的主要目的就是防止悬垂引用。
- Rust 能够在编译时检查代码，确保引用的有效性，而不是在运行时出现悬垂引用的错误。

### 推断规则
1. 每个作为引用的参数都会得到它自己的生命周期参数。
2. 如果只有一个输入生命周期参数，该参数会被分配给所有返回值。
3. 如果存在多个输入生命周期参数，其中一个是对`&self`或`&mut self`的引用，`self`的生命周期会被分配给所有输出生命周期参数（因为这种情况下函数是`self`的一个方法，所以生命周期需要是一样的）。

***
### 例子
```rust
// 如果没有返回，是不需要标注生命周期的
// 这里有多个入参，且返回了一个字面量的引用，就需要标注生命周期
// s1 s2 默认具有不同的生命周期（'0 '1），可以写成相同的
fn longest<'a>(s1: &'a str, s2: &'a str) -> &'a str {
    if s1.len() > s2.len() {
        s1
    } else {
        s2
    }
}

// 为每一个参数和返回值标注生命周期
// 这样的好处是灵活，且性能好一些
fn longest_str<'a, 'b, 'out>(s1: &'a str, s2: &'b str) -> &'out str
// 如果 out 写成 a，会在 else b 报错生命周期不够长，反之在if a 报错
// 所以需要一个 where 语句进行限定
// 这种写法是取了 a b 的交集
where
    'a: 'out,
    'b: 'out,
{
    if s1.len() > s2.len() {
        s1
    } else {
        s2
    }
}

// &'static 生命周期是程序结束，一般仅用于测试场景
fn no_need(s: &'static str, s1: &str) -> &'static str {
    s
}

fn main() {
    println!("no need {}", no_need("hh", "")); // no need hh

    let s1 = "hello world";
    let s2 = "hello";
    println!("longest {}", longest(s1, s2)); // longest hello world

    let result: &str;
    {
        let r2 = "world";
        result = longest_str(r2, s1); // 显然，r2 和 s1 生命周期不一样
        println!("Longest string: {}", result);
    }
}
```

## 结构体生命周期
结构体中的引用必须标注生命周期。但结构体的方法（&self等）不需要标注生命周期。

应尽量避免自己标注生命周期。

***
### 例子
```rust
// 这里只是为了演示，真正代码开发时结构体应该避免使用字面量，而是直接用 String
struct MyString<'a> {
    text: &'a str,
}

// 有生命周期的结构体的 impl 后面需要标注生命周期，但方法上不用标注
impl<'a> MyString<'a> {
    fn get_length(&self) -> usize {
        self.text.len()
    }

    fn modify_data(&mut self) {
        self.text = "world";
    }
}

struct StringHolder {
    data: String,
}

impl StringHolder {
    fn get_length(&self) -> usize {
        self.data.len()
    }
    // 也可以声明生命周期，但其实是能自动推断的
    fn get_reference<'a>(&'a self) -> &'a String {
        &self.data
    }
    fn get_ref(&self) -> &String {
        &self.data
    }
}

fn main() {
    let str1 = String::from("value");
    let mut x = MyString {
        text: str1.as_str(),
    };
    x.modify_data();
    println!("{}", x.text); // 输出 world

    let holder = StringHolder {
        data: String::from("Hello"),
    };
    println!("{}", holder.get_reference());
    println!("{}", holder.get_ref());
}
```
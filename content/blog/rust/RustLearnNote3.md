+++
title = "Rust 学习笔记（三）：流程控制与函数"
slug = "rust_learn_note_3"
date = 2025-06-28T15:50:00Z
updated = 2025-06-29T21:50:00Z
[taxonomies]
tags = ["Rust", "Learn"]
[extra]
summary = "流程控制，循环，函数，参数传递，函数所有权，高级函数"
pinned = false
post_listing_date = "both"
+++

## 流程控制与模式匹配

### 流程
正常情况下，代码是从上到下一行一行执行的，但执行一些操作会导致流程控制改变。

主流流程控制结构有：
- 顺序结构：程序按代码顺序一步一步执行
- 选择结构：根据条件选择不同的路径执行
  - if 语句：根据条件执行不同代码块
  - switch 语句：根据不同条件执行不同代码块
- 循环结构：重复执行一段代码直到满足某个条件为止
  - for
  - while
  - do/while
- 跳转结构：跳转到指定位置
  - break
  - continue
  - goto

### if
代码逐行执行，执行流程可被`if`改变。

应尽量避免过多嵌套使用，会导致可读性问题。

基本语法：
```rust
if condition {
    // do sth
} else {
    // do sth
}
```

### match
Rust 中，`match`用于模式匹配，允许更复杂的条件和分支，可处理多个模式，且可以返回值。

基本语法：
```rust
match value {
    pattern1 => // 模式1
    pattern2 if condition => // 模式2且 condition 为真
    _ => // 其他情况
}
```
相比之下 match 更灵活和清晰，可用于更复杂的场景。

***
### 例子
```rust
fn main() {
    let age = 50;
    if age < 50 {
        println!("You are young");
    } else {
        println!("You are old");
    }
    // if 的表达能力很弱，不直观
    let scores = 70;
    if scores > 90 {
        println!("Good!!!");
    } else if scores > 60 {
        println!("You are OK!");
    } else {
        println!("Bad!!!");
    }
    let msg = if age > 50 { "old" } else { "young" };
    println!("You are {msg}");

    // match，函数式编程思想
    let num = 90;
    // 精准匹配
    match num {
        80 => println!("80"),
        90 => println!("90"),
        _ => println!("Some else"),
    }
    // 范围匹配
    match num {
        25..=50 => println!("25 ... 50"),
        51..=100 => println!("51 ... 100"),
        _ => println!("Some else"),
    }
    // 或匹配
    match num {
        25 | 50 | 75 => print!("25 or 50 or 75"),
        100 | 200 => println!("100 or 200"),
        _ => println!("Some else"),
    }
    // 和 if 一起使用
    match num {
        x if x < 60 => println!("bad"),
        x if x == 60 => println!("luck"),
        _ => println!("Some else"),
    }
    // 赋值
    let num = 60;
    let res = match num {
        x if x < 60 => "bad".to_owned(),
        x if x == 60 => "luck".to_owned(),
        _ => "Some else".to_owned(),
    };
    println!("res value : {res}");
}
```

## 循环

### 循环结构
Rust 提供了如下几种循环结构：
- loop：一个无限循环，通过`break`中断 
- while：每次循环检查条件，条件为真时执行循环体
- for：迭代集合或范围，执行代码处理每个元素
```rust
for item in iterable {
    // do sth
}
```

### 跳出关键字
- break：立即终止循环并跳出循环体，可以通过标签的方式在内层循环体中跳出外层循环
- continue：立即跳过当前循环并开始执行下一次循环

### 迭代
Rust 的迭代器是一个抽象，通过实现一个 Iterator 特质，提供统一的访问集合元素的方式。
```rust
pub trait Iterator {
    type Item;
    fn next (&mut self) -> Option<Self::Item>;
}
```
迭代器提供了一系列用于遍历集合元素的方法，如`next()`、`map()`、`filter()`。`for`循环就依赖了迭代器的`next()`。

循环更适合明确控制循环流程的情况，而迭代器则提供了一种抽象的方式来处理集合元素。

***
### 例子
```rust
fn main() {
    // 无限循环
    // loop {
    //     println!("Ctrl+C");
    //     std::thread::sleep(std::time::Duration::from_secs(1));
    // }
    let mut i = 0;
    while i < 10 {
        println!("{}", i);
        i += 1;
    }
    println!("for");
    // vec 也同理
    let arr = [0, 1, 2, 3, 4, 5, 6, 7, 8];
    for element in arr {
        println!("{}", element);
    }
    // 默认左闭右开
    for i in 0..10 {
        println!("{}", i);
    }
    // ..= 表示右闭
    for i in 0..=10 {
        println!("{}", i);
    }
    // break
    let arr = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11];
    for element in arr {
        if element == 10 {
            break;
        }
        println!("{element}");
    }
    let arr = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11];
    for element in arr {
        // 11 会打印，不打印 10
        if element == 10 {
            continue;
        }
        println!("{element}");
    }
    'outer: loop {
        println!("outer");
        loop {
            println!("inner");
            // break 一个标签
            break 'outer;
        }
    }

    // 循环的写法
    let numbers = [1, 2, 3, 4, 5];
    let mut for_numbers = Vec::new();
    for &number in numbers.iter() {
        let item = number * number;
        for_numbers.push(item);
    }
    println!("for : {:?}", for_numbers);
    // 迭代的写法
    let numbers = [1, 2, 3, 4, 5].to_vec();
    // 迭代器的消耗函数，后续会展开
    // 完整写法 let iter_number: Vec<i32> = numbers.iter().map(|&x: i32| x * x).collect();
    let iter_number: Vec<_> = numbers.iter().map(|&x| x * x).collect();
    println!("iter : {:?}", iter_number);
}
```

## 函数
函数由`fn`关键字声明和定义。函数可以接受 0 或多个参数，每个参数都要指定类型。

函数可以有或没有返回值，有返回值时，通过`->`指定返回类型，否则省略或指定空（`-> ()`）。函数的最后一行如果不写分号，其结果就会被当作返回值，否则需要用`return`。

调用函数需要使用函数名，并传递给函数具体的参数。

`main`函数是一个特殊函数，是程序的入口。

### copy 特质的函数
如果数据类型实现了`copy`特质，则在函数传参时会实现 copy by value 操作。此时会将实参拷贝为形参，形参改变（需要加`mut`才能改变）不会影响实参。

***
### 例子
```rust
fn add(x: i32, y: i32) -> i32 {
    x + y
}

fn change_i32(mut x: i32) {
    x = x + 4;
    println!("fn {x}"); // 5
}

fn modify_i32(x: &mut i32) {
    // 借用必须解引用
    *x += 4;
}

#[derive(Copy, Clone)]
struct Point {
    x: i32,
    y: i32,
}

fn print_point(point: Point) {
    println!("point x {}", point.x);
}

fn main() {
    let a = 1;
    let b = 2;
    let c = add(a, b);
    println!("c: {c}"); // 3
    let mut x = 1;
    change_i32(x); // 不会修改实参
    println!("x {x}"); // 1
    modify_i32(&mut x); // 借用
    println!("x {x}"); // 5
    let s = Point { x: 1, y: 2 };
    print_point(s); // 所有权已经消失
    // 结构体默认是 move ，如果不加 #[derive(Copy, Clone)]，就无法打印了
    println!("{}", s.x);
}
```

## 参数传递

### 函数值参数传递 move
函数调用时会在栈上开辟一个新的栈帧，用于存储函数的局部变量、参数和返回地址等信息，函数结束后会释放该空间。

当传入 non-copy value（如 Vec String）时，传入函数的实参值的所有权会转移到形参，函数结束时就会释放。说人话就是，`move`只能用一次。

### 不可变借用
借用其实就是引用。不过在 rust 中一般叫借用。

如果不想失去值的所有权，又没有修改需求，就可以使用不可变借用。

不可变引用可以作为函数的参数，从而在函数内部访问参数值，同时不能修改。这这有助于确保数据的安全性，防止在多处同时对数据进行写操作，从而避免数据竞争。

使用不可变借用需要使用`*`解引用（deference），来获取值。

### 可变借用
可变借用允许在函数内部修改参数的值，执行写操作。同一时间只能有一个可变借用。

需要手动在形参前加`&mut`，同样需要使用`*`解引用。

***
### 例子
```rust
fn move_func(p1: i32, p2: String) {
    println!("p1 is {}", p1);
    println!("p2 is {}", p2);
}

// borrow
fn print_value(value: &i32) {
    // 不加 * 也可以，默认解引用
    println!("{}", value);
}

fn string_func_borrow(s: &String) {
    // println!("{}", s); // 这么写也是可以的
    println!("{}", (*s).to_uppercase());
}

// 结构体没有实现 display 特质，所以需要一个自动实现的 debug 特质
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

fn modify_point(point: &mut Point) {
    (*point).x += 2;
    point.y += 2;
}

fn main() {
    let n = 12;
    let s = String::from("oo");
    move_func(n, s);
    // n 做了 copy, s 做了 move
    println!("n is {}", n);
    // println!("s is {}", s);
    let s = String::from("oo");
    // 保留所有权，需要不可变借用
    print_value(&n);
    print_value(&n);
    string_func_borrow(&s);
    println!("n is {} s is {}", n, s);
    let mut p = Point { x: 0, y: 0 };
    // :? 是 debug 特质的打印方式
    println!("{:?}", p);
    modify_point(&mut p);
    println!("{:?}", p);
}
```
+++
title = "Rust 入门学习笔记（四）：错误处理"
slug = "rust_learn_note_4"
date = 2025-06-30
updated = 2025-06-30
[taxonomies]
tags = ["Rust", "Learn"]
[extra]
summary = "常见错误处理和自定义错误处理"
pinned = false
post_listing_date = "both"
+++

## 常见错误

### 错误
Rust 中的错误分为可恢复和不可恢复。
- 可恢复：有返回类型，可以为 Result 或 Option
- 不可恢复：panic! 终止当前线程

### Result
Result 是一个枚举类型，通常用于表示函数执行的结果。
```rust
pub enum Result<T, E> {
    Ok(T), // 成功的结果
    Err(E), // 出现了错误
}
```

### Option
Option 也是一个枚举类型，通常用于表示一个可能为空的值。
```rust
pub enum Option<T> {
    None, // 空
    Some(T), // 其他返回值
}
```

### panic 宏
当程序遇到无法继续执行的错误时，可以使用`panic!`宏来引发“恐慌”，导致程序立即终止并显示一条错误信息。

通常用于程序不能继续处理的情况下。

***
### 例子
```rust
fn divide(a: i32, b: i32) -> Result<f64, String> {
    if b == 0 {
        // 一般都要实现 std:error:Error 特质，否则将难以串联业务逻辑。
        // 这里是为了方便才用的 String
        return Err(String::from("cannot be zero"));
    }
    let a = a as f64;
    let b = b as f64;
    Ok(a / b)
}

fn find_element(array: &[i32], target: i32) -> Option<usize> {
    for (index, element) in array.iter().enumerate() {
        if (*element) == target {
            return Some(index);
        }
    }
    None
}

fn main() {
    // result
    match divide(1, 2) {
        Ok(number) => println!("{}", number),
        Err(err) => println!("{}", err),
    }

    match divide(1, 0) {
        Ok(number) => println!("{}", number),
        Err(err) => println!("{}", err),
    }

    // option
    let arr = [1, 2, 3, 4, 5];
    match find_element(&arr, 4) {
        Some(index) => println!("found in {}", index),
        None => println!("None"),
    }
    match find_element(&arr, 7) {
        Some(index) => println!("found in {}", index),
        None => println!("None"),
    }
    // panic
    let vec = vec![1, 2, 3, 4, 5]; // array 在编译时确定长度，而 vec 则没有，如用 array 出现的将是编译错误而非运行时
    vec[43];
    // 会输出 process didn't exit successfully
    // 使用 `RUST_BACKTRACE=1` 环境变量能够打印更详细的栈
}
```

## 错误结果处理

### unwrap()
`unwrap`并不是一个安全的方法。它是`Result`和`Option`类型提供的方法之一，可用于获取`OK`或`Some`的值，如果是`Err`或`None`会引发`panic`。

一般只用于确定不会出现错误或错误不需要处理的情况。

### `?` 运算符
`?`用于简化`Result`和`Option`类型的错误传播，表示有预期之外的结果出现。它只能用于返回`Result`或`Option`的函数中，并且在函数内部也可以用于获取`OK`或`Some`的值，如果是`Err`或`None`则会**立即**提前返回。

***
### 例子
```rust
use std::num::ParseIntError;

fn find_first_even(numbers: Vec<i32>) -> Option<i32> {
    // 这里 find 的返回值是一个 Option<&i32> 类型
    // 用了 ? 之后，在出现 None 或者 Err 时会立即返回
    let first_even = numbers.iter().find(|&num| num % 2 == 0)?;
    print!("Option");
    // 必须解引用
    Some(*first_even)
}

// 传递错误的处理模式，可以把错误作为一种选择传到外部
fn parse_numbers(input: &str) -> Result<i32, ParseIntError> {
    let val = input.parse::<i32>()?;
    Ok(val)
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let result_ok: Result<i32, &str> = Ok(32);
    let value = result_ok.unwrap();
    println!("{}", value); // 32
    // let result_ok: Result<i32, &str> = Err("ff");
    // let value = result_ok.unwrap();
    // println!("{}", value); // panic
    let result_ok: Result<i32, &str> = Ok(32);
    // main 函数中想要使用 ?，必须修改返回值为 Result
    let value = result_ok?;
    println!("{}", value); // 32

    let numbers = vec![1, 3, 5];
    match find_first_even(numbers) {
        Some(number) => println!("first even {}", number),
        None => println!("no such number"), // 此时并不会输出 find_first_even 里打印的 `Option`
    }

    match parse_numbers("d") {
        Ok(i) => println!("parsed {}", i),
        Err(err) => println!("failed to parse: {}", err), // 会报错`invalid digit found in string`，该信息来自 ParseIntError
    }

    Ok(())
}
```

## 自定义错误类型
1. 定义错误类型结构体。需要先创建一个结构体来表示错误类型，其中通常会包含一些字段来描述详细错误信息。
2. 实现 Display 特质。`std::fmt::Display`特质可以定义如何展示错误信息，方便使错误能够以人类可读的方式打印出来。
3. 实现 Error 特质。`std::error::Error`特质用于满足 Rust 错误处理机制的要求，以便在一层一层的错误之中互相传递，

***
### 例子
```rust
#[derive(Debug)]
struct MyError {
    // 不要用字面量，否则要自己定义生命周期
    detail: String,
}

// 这是实现特质的语法
impl std::fmt::Display for MyError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "Custom Error: {}", self.detail)
    }
}

// 可以不重写，有默认的
impl std::error::Error for MyError {
    fn description(&self) -> &str {
        &self.detail
    }
    // 字符串引用会自动被转成字符串字面量 &String => &str
}

fn func() -> Result<(), MyError> {
    Err(MyError {
        detail: "CustomError".to_owned(),
    })
    // Ok(())
}

fn main() -> Result<(), MyError> {
    match func() {
        Ok(_) => println!("func ok"),
        Err(err) => println!("Error: {}", err),
    }
    func()?;
    println!("oo"); // 错误时不会打印，如果想要完成传递，必须返回 Result<(), Box<dyn std::error::Error>>
    Ok(())
}
```
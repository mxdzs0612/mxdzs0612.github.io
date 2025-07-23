+++
title = "Rust 进阶学习笔记（十）：高级错误处理"
slug = "rust_learn_note_adv_10"
date = 2025-07-21
updated = 2025-07-22
description = "组合错误方法，自定义错误处理，归一化错误类型"
# draft = true
[taxonomies]
tags = ["Rust", "Learn"]
[extra]
pinned = false
post_listing_date = "both"
+++

本节出处：[圣经-4.8错误处理](https://course.rs/advance/errors.html)

入门篇学过基础错误处理，有基础当然就有进阶。在本章节中一起来看看如何对 Result/Option 做进一步的处理。

艰难险阻都走过了，最后终于又简单了起来。

***

## 组合器
组合模式是一种设计模式，可以将对象组合成树形结构以表示“部分整体”的层次结构。

在 Rust 中，组合器更多的是用于对返回结果的类型进行变换：例如使用 ok_or 将一个 Option 类型转换成 Result 类型。

### 与和或
跟布尔关系的与/或很像，and 和 or 两个方法会对两个表达式做逻辑组合，最终返回 Option / Result。
- and()：若两个表达式的结果都是 Some 或 Ok，则第二个表达式中的值被返回。若任何一个的结果是 None 或 Err ，则立刻返回。
- or()：表达式按照顺序求值，若任何一个表达式的结果是 Some 或 Ok，则该值会立刻返回
```rust
fn main() {
  let s1 = Some("some1");
  let s2 = Some("some2");
  let n: Option<&str> = None;

  let o1: Result<&str, &str> = Ok("ok1");
  let o2: Result<&str, &str> = Ok("ok2");
  let e1: Result<&str, &str> = Err("error1");
  let e2: Result<&str, &str> = Err("error2");

  assert_eq!(s1.or(s2), s1); // Some1 or Some2 = Some1
  assert_eq!(s1.or(n), s1);  // Some or None = Some
  assert_eq!(n.or(s1), s1);  // None or Some = Some
  assert_eq!(n.or(n), n);    // None1 or None2 = None2

  assert_eq!(o1.or(o2), o1); // Ok1 or Ok2 = Ok1
  assert_eq!(o1.or(e1), o1); // Ok or Err = Ok
  assert_eq!(e1.or(o1), o1); // Err or Ok = Ok
  assert_eq!(e1.or(e2), e2); // Err1 or Err2 = Err2

  assert_eq!(s1.and(s2), s2); // Some1 and Some2 = Some2
  assert_eq!(s1.and(n), n);   // Some and None = None
  assert_eq!(n.and(s1), n);   // None and Some = None
  assert_eq!(n.and(n), n);    // None1 and None2 = None1

  assert_eq!(o1.and(o2), o2); // Ok1 and Ok2 = Ok2
  assert_eq!(o1.and(e1), e1); // Ok and Err = Err
  assert_eq!(e1.and(o1), e1); // Err and Ok = Err
  assert_eq!(e1.and(e2), e1); // Err1 and Err2 = Err1
}
```
Rust 还提供了 xor ，但是它只能应用在 Option 上，因为对 Result 是没办法进行异或的。

### 可选与和或
or_else() 和 and_then() 与普通的与和或的唯一区别在于，它们的第二个表达式是一个闭包。
```rust
fn main() {
    // or_else with Option
    let s1 = Some("some1");
    let s2 = Some("some2");
    let fn_some = || Some("some2"); // 类似于: let fn_some = || -> Option<&str> { Some("some2") };

    let n: Option<&str> = None;
    let fn_none = || None;

    assert_eq!(s1.or_else(fn_some), s1);  // Some1 or_else Some2 = Some1
    assert_eq!(s1.or_else(fn_none), s1);  // Some or_else None = Some
    assert_eq!(n.or_else(fn_some), s2);   // None or_else Some = Some
    assert_eq!(n.or_else(fn_none), None); // None1 or_else None2 = None2

    // or_else with Result
    let o1: Result<&str, &str> = Ok("ok1");
    let o2: Result<&str, &str> = Ok("ok2");
    let fn_ok = |_| Ok("ok2"); // 类似于: let fn_ok = |_| -> Result<&str, &str> { Ok("ok2") };

    let e1: Result<&str, &str> = Err("error1");
    let e2: Result<&str, &str> = Err("error2");
    let fn_err = |_| Err("error2");

    assert_eq!(o1.or_else(fn_ok), o1);  // Ok1 or_else Ok2 = Ok1
    assert_eq!(o1.or_else(fn_err), o1); // Ok or_else Err = Ok
    assert_eq!(e1.or_else(fn_ok), o2);  // Err or_else Ok = Ok
    assert_eq!(e1.or_else(fn_err), e2); // Err1 or_else Err2 = Err2
}
```
```rust
fn main() {
    // and_then with Option
    let s1 = Some("some1");
    let s2 = Some("some2");
    let fn_some = |_| Some("some2"); // 类似于: let fn_some = |_| -> Option<&str> { Some("some2") };

    let n: Option<&str> = None;
    let fn_none = |_| None;

    assert_eq!(s1.and_then(fn_some), s2); // Some1 and_then Some2 = Some2
    assert_eq!(s1.and_then(fn_none), n);  // Some and_then None = None
    assert_eq!(n.and_then(fn_some), n);   // None and_then Some = None
    assert_eq!(n.and_then(fn_none), n);   // None1 and_then None2 = None1

    // and_then with Result
    let o1: Result<&str, &str> = Ok("ok1");
    let o2: Result<&str, &str> = Ok("ok2");
    let fn_ok = |_| Ok("ok2"); // 类似于: let fn_ok = |_| -> Result<&str, &str> { Ok("ok2") };

    let e1: Result<&str, &str> = Err("error1");
    let e2: Result<&str, &str> = Err("error2");
    let fn_err = |_| Err("error2");

    assert_eq!(o1.and_then(fn_ok), o2);  // Ok1 and_then Ok2 = Ok2
    assert_eq!(o1.and_then(fn_err), e2); // Ok and_then Err = Err
    assert_eq!(e1.and_then(fn_ok), e1);  // Err and_then Ok = Err
    assert_eq!(e1.and_then(fn_err), e1); // Err1 and_then Err2 = Err1
}
```

### 过滤器
filter() 用于对 Option 进行过滤：
```rust
fn main() {
    let s1 = Some(3);
    let s2 = Some(6);
    let n = None;

    let fn_is_even = |x: &i8| x % 2 == 0;

    assert_eq!(s1.filter(fn_is_even), n);  // Some(3) -> 3 is not even -> None
    assert_eq!(s2.filter(fn_is_even), s2); // Some(6) -> 6 is even -> Some(6)
    assert_eq!(n.filter(fn_is_even), n);   // None -> no value -> None
}
```

### 映射
map() 可以将 Some 或 Ok 中的值映射为另一个，但不会处理 Err。相对的，map_err() 则是能够改变 Err 的值，但是不会处理 Ok。
```rust
fn main() {
    let s1 = Some("abcde");
    let s2 = Some(5);

    let n1: Option<&str> = None;
    let n2: Option<usize> = None;

    let o1: Result<&str, &str> = Ok("abcde");
    let o2: Result<usize, &str> = Ok(5);

    let e1: Result<&str, &str> = Err("abcde");
    let e2: Result<usize, &str> = Err("abcde");

    let fn_character_count = |s: &str| s.chars().count();

    assert_eq!(s1.map(fn_character_count), s2); // Some1 map = Some2
    assert_eq!(n1.map(fn_character_count), n2); // None1 map = None2

    assert_eq!(o1.map(fn_character_count), o2); // Ok1 map = Ok2
    assert_eq!(e1.map(fn_character_count), e2); // Err1 map = Err2
}
```
```rust
fn main() {
    let o1: Result<&str, &str> = Ok("abcde");
    let o2: Result<&str, isize> = Ok("abcde");

    let e1: Result<&str, &str> = Err("404");
    let e2: Result<&str, isize> = Err(404);

    let fn_character_count = |s: &str| -> isize { s.parse().unwrap() }; // 该函数返回一个 isize

    assert_eq!(o1.map_err(fn_character_count), o2); // Ok1 map = Ok2
    assert_eq!(e1.map_err(fn_character_count), e2); // Err1 map = Err2
}
```

### 带默认值的映射
map_or() 在 map 的基础上提供了一个默认值，在遇到 None 的时候会直接返回默认值。而 map_or_else() 通过一个闭包来提供默认值。
```rust
fn main() {
    const V_DEFAULT: u32 = 1;

    let s: Result<u32, ()> = Ok(10);
    let n: Option<u32> = None;
    let fn_closure = |v: u32| v + 2;

    assert_eq!(s.map_or(V_DEFAULT, fn_closure), 12);
    assert_eq!(n.map_or(V_DEFAULT, fn_closure), V_DEFAULT);
}
```
```rust
fn main() {
    let s = Some(10);
    let n: Option<i8> = None;

    let fn_closure = |v: i8| v + 2;
    let fn_default = || 1;

    assert_eq!(s.map_or_else(fn_default, fn_closure), 12);
    assert_eq!(n.map_or_else(fn_default, fn_closure), 1);

    let o = Ok(10);
    let e = Err(5);
    let fn_default_for_result = |v: i8| v + 1; // 闭包可以对 Err 中的值进行处理，并返回一个新值

    assert_eq!(o.map_or_else(fn_default_for_result, fn_closure), 12);
    assert_eq!(e.map_or_else(fn_default_for_result, fn_closure), 6);
}
```

### Option 转 Result
ok_or() 接收一个默认的 Err 参数，ok_or_else() 接收一个闭包作为 Err 参数。
```rust
fn main() {
    const ERR_DEFAULT: &str = "error message";

    let s = Some("abcde");
    let n: Option<&str> = None;

    let o: Result<&str, &str> = Ok("abcde");
    let e: Result<&str, &str> = Err(ERR_DEFAULT);

    assert_eq!(s.ok_or(ERR_DEFAULT), o); // Some(T) -> Ok(T)
    assert_eq!(n.ok_or(ERR_DEFAULT), e); // None -> Err(default)
}
```
```rust
fn main() {
    let s = Some("abcde");
    let n: Option<&str> = None;
    let fn_err_message = || "error message";

    let o: Result<&str, &str> = Ok("abcde");
    let e: Result<&str, &str> = Err("error message");

    assert_eq!(s.ok_or_else(fn_err_message), o); // Some(T) -> Ok(T)
    assert_eq!(n.ok_or_else(fn_err_message), e); // None -> Err(default)
}
```

## 自定义错误类型
入门时候学习过[自定义错误类型](../rust-learn-note-4/#zi-ding-yi-cuo-wu-lei-xing)，这里会更深入更现代一点。Rust 标准库中提供了可复用的 Error 特质，最常用的是`std::error::Error`。
```rust
use std::fmt::{Debug, Display};

pub trait Error: Debug + Display {
    fn source(&self) -> Option<&(Error + 'static)> { ... }
}
```
当然我们从前面的章节知道还有其他错误，如`std::io::Error`，我们先不管他。

### 最简单的错误
自定义错误类型只需要派生 Debug，然后实现 Display 特质即可。
```rust
use std::fmt;

// AppError 是自定义错误类型，它可以是当前包中定义的任何类型，在这里为了简化，我们使用了单元结构体作为例子。
// 为 AppError 自动派生 Debug 特质
#[derive(Debug)]
struct AppError;

// 为 AppError 实现 std::fmt::Display 特质
impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "An Error Occurred, Please Try Again!") // user-facing output
    }
}

// 一个示例函数用于产生 AppError 错误
fn produce_error() -> Result<(), AppError> {
    Err(AppError)
}

fn main(){
    match produce_error() {
        Err(e) => eprintln!("{}", e),
        _ => println!("No error"),
    }

    eprintln!("{:?}", produce_error()); // Err({ file: src/main.rs, line: 17 })
}
```
实际上，实现 Debug 和 Display 特质并不是作为 Err 使用的必要条件，但一般都会这么用。原因是：
- 错误肯定是要打印输出后，才能有实际用处，而打印输出就需要实现这两个特质
- 可以将自定义错误转换成`Box<dyn std::error:Error>`特质对象

### 详细的错误
再来定义一个具有错误码和信息的错误:
```rust
use std::fmt;

struct AppError {
    code: usize,
    message: String,
}

// 根据错误码显示不同的错误信息
impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        let err_msg = match self.code {
            404 => "Sorry, Can not find the Page!",
            _ => "Sorry, something is wrong! Please Try Again!",
        };

        write!(f, "{}", err_msg)
    }
}

impl fmt::Debug for AppError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(
            f,
            "AppError {{ code: {}, message: {} }}",
            self.code, self.message
        )
    }
}

fn produce_error() -> Result<(), AppError> {
    Err(AppError {
        code: 404,
        message: String::from("Page not found"),
    })
}

fn main() {
    match produce_error() {
        Err(e) => eprintln!("{}", e), // 抱歉，未找到指定的页面!
        _ => println!("No error"),
    }

    eprintln!("{:?}", produce_error()); // Err(AppError { code: 404, message: Page not found })

    eprintln!("{:#?}", produce_error());
    // Err(
    //     AppError { code: 404, message: Page not found }
    // )
}
```
这里手动实现了 Debug 特质，用于自定义 Debug 的输出内容，而不是使用派生后系统提供的默认输出形式。

### 错误转换
Rust 为我们提供了`std::convert::From`特质:
```rust
pub trait From<T>: Sized {
  fn from(_: T) -> Self;
}
// String::from 函数也是这个特质提供的
```
为自定义类型实现 From 的方式是：
```rust
use std::fs::File;
use std::io;

#[derive(Debug)]
struct AppError {
    kind: String,    // 错误类型
    message: String, // 错误信息
}

// 为 AppError 实现 std::convert::From 特质，由于 From 包含在 std::prelude 中，因此可以直接简化引入。
// 实现 From<io::Error> 意味着我们可以将 io::Error 错误转换成自定义的 AppError 错误
impl From<io::Error> for AppError {
    fn from(error: io::Error) -> Self {
        AppError {
            kind: String::from("io"),
            message: error.to_string(),
        }
    }
}

fn main() -> Result<(), AppError> {
    // ? 可以将错误进行隐式的强制转换
    // File::open 返回的是 std::io::Error，能自动变成 AppError
    let _file = File::open("nonexistent_file.txt")?;

    Ok(())
}

// // 输出
// Error: AppError { kind: "io", message: "No such file or directory (os error 2)" }
```

## 错误归一化
实际项目中往往会为不同的错误定义不同的类型，此时往往会遇到一个函数里需要返回若干个不同错误的情况。此时就需要将多个错误归一化成相同的类型。于是就有本节的方式。

### 特质对象 Box\<dyn Error\>
自定义类型实现 Debug + Display 特质的主要原因就是为了能转换成 Error 的特质对象，而特质对象恰恰是在同一个地方使用不同类型的关键:
```rust
use std::error::Error;
use std::fs::read_to_string;
fn main() -> Result<(), Box<dyn Error>> {
    let html = render()?;
    println!("{}", html);
    Ok(())
}

fn render() -> Result<String, Box<dyn Error>> {
    // 返回 std::env::VarError
    let file = std::env::var("MARKDOWN")?;
    // 返回 std::io::Error
    let source = read_to_string(file)?;
    Ok(source)
}
```
这样很好，但有一个问题：Result 实际上不会限制错误的类型，也就是一个类型就算不实现 Error 特质，它依然可以在`Result<T, E>`中作为 E 来使用，此时这种特质对象的解决方案就无能为力了。

### 自定义错误类型
自定义错误类型麻烦但灵活，也是一个常用手段。
```rust
use std::fs::read_to_string;

fn main() -> Result<(), MyError> {
  let html = render()?;
  println!("{}", html);
  Ok(())
}

fn render() -> Result<String, MyError> {
  let file = std::env::var("MARKDOWN")?;
  let source = read_to_string(file)?;
  Ok(source)
}

#[derive(Debug)]
enum MyError {
  EnvironmentVariableNotFound,
  IOError(std::io::Error),
}

impl From<std::env::VarError> for MyError {
  fn from(_: std::env::VarError) -> Self {
    Self::EnvironmentVariableNotFound
  }
}

impl From<std::io::Error> for MyError {
  fn from(value: std::io::Error) -> Self {
    Self::IOError(value)
  }
}
// 只有为自定义错误类型实现 Error 特质后，才能转换成相应的特质对象。
impl std::error::Error for MyError {}

impl std::fmt::Display for MyError {
  fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
    match self {
      MyError::EnvironmentVariableNotFound => write!(f, "Environment variable not found"),
      MyError::IOError(err) => write!(f, "IO Error: {}", err.to_string()),
    }
  }
}
```

### 简化错误处理
很明显，前两种方式一个使用受限，一个写起来太麻烦。于是就有了一些第三方库。

#### thiserror
[thiserror](https://github.com/dtolnay/thiserror) 是一个能够简化自定义错误类型的库，只要简单写写注释，就可以实现错误处理了：
```rust
use std::fs::read_to_string;

fn main() -> Result<(), MyError> {
  let html = render()?;
  println!("{}", html);
  Ok(())
}

fn render() -> Result<String, MyError> {
  let file = std::env::var("MARKDOWN")?;
  let source = read_to_string(file)?;
  Ok(source)
}

#[derive(thiserror::Error, Debug)]
enum MyError {
  #[error("Environment variable not found")]
  EnvironmentVariableNotFound(#[from] std::env::VarError),
  #[error(transparent)]
  IOError(#[from] std::io::Error),
}
```

#### anyhow
[anyhow](https://github.com/dtolnay/anyhow) 自定义了一个 Result，非常省事。

如果想要设计自己的错误类型，同时给调用者提供具体的信息时，就使用 thiserror，例如在开发一个三方库代码时；如果只想要简单，就使用 anyhow，例如在自己的应用服务中。
> 这两个库作者是同一个人，这个人也是宏 workshop 的作者，真大佬！
```rust
use std::fs::read_to_string;

use anyhow::Result;

fn main() -> Result<()> {
    let html = render()?;
    println!("{}", html);
    Ok(())
}

fn render() -> Result<String> {
    let file = std::env::var("MARKDOWN")?;
    let source = read_to_string(file)?;
    Ok(source)
}
```

#### error-chain
[error-chain](https://github.com/rust-lang-deprecated/error-chain) 是一个 Rust 早期的错误处理库，已多年不维护，虽然好用但也不建议再使用，因此这里也不写例子了。现在可以考虑下面这个。

#### snafu
[snafu](https://github.com/shepmaster/snafu) 是一个非常强大的错误处理库，提供了宏来自动生成错误类型和上下文信息。更适合需要详细错误信息和结构的库或复杂项目。
```rust
use snafu::{Snafu, ResultExt};

#[derive(Debug, Snafu)]
enum Error {
    #[snafu(display("IO error: {}", source))]
    Io { source: std::io::Error },
}

fn read_file() -> Result<(), Error> {
    std::fs::read("file.txt").context(Io)?;
    Ok(())
}
```
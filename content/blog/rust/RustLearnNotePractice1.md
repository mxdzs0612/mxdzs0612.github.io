+++
title = "Rust 项目实战（一）：minigrep"
slug = "rust_learn_note_prac_1"
date = 2025-08-01
updated = 2025-08-08
description = "实现一个搜索文件内容的命令行工具"
# draft = true
[taxonomies]
tags = ["Rust", "Learn", "Project"]
[extra]
pinned = false
post_listing_date = "both"
+++

本节出处：[圣经 文件搜索工具](https://course.rs/basic-practice/intro.html) [The Book 构建一个命令行程序](https://rustwiki.org/zh-CN/book/ch12-00-an-io-project.html)，主要是前者。

学完了，该整项目了。先来个入门的命令行工具吧。本节会尝试构建一个命令行程序，从命令行参数中读取指定的文件名和字符串，然后在相应的文件中找到包含该字符串的内容，最终打印出来。这个项目肯定是要单独起一个工程敲一遍的。我直接 init 一个库吧，反正 github 不要钱（

## 基本功能
通过 cargo 创建一个项目 [minigrep](https://github.com/mxdzs0612/minigrep)。相关内容略。
<!-- {{ admonition(type="warning", title="注意", text="施工中") }} -->

### 接受参数
如果要传入文件路径和待搜索的字符串，那这个命令大概率是这样
```bash
cargo run -- searchstring example-filename.txt
```
`--` 告诉 cargo 后面的参数是给我们的程序使用的，而不是给 cargo 自己使用。

rust 标准库中有一个 env，负责在程序中读取传入的参数。
```rust,name=main.rs
use std::env;

fn main() {
    // collect 是迭代器的方法，会将迭代器消费后转换成集合类型
    let args: Vec<String> = env::args().collect();
    // 输出读取到的数组内容
    dbg!(args);
}
```

### 存储参数
需要两个变量来存储文件路径和待搜索的字符串。
```rust,name=main.rs
use std::env;

fn main() {
    let args: Vec<String> = env::args().collect();

    let query = &args[1];
    let file_path = &args[2];

    println!("Searching for {}", query);
    println!("In file {}", file_path);
}
```

### 文件读取
在相同路径下创建一个 `poem.txt` 作为待读取的文件，然后完善读取文件的代码。
```rust,name=main.rs
use std::env;
use std::fs; // 引入文件操作模块

// cargo run -- the poem.txt
fn main() {
    // --省略之前的内容--
    println!("In file {}", file_path);

    let contents = fs::read_to_string(file_path) // 读取指定的文件内容
        .expect("Should have been able to read the file");

    println!("With text:\n{contents}");
}
```

## 模块化和错误处理
上面的基础代码有如下改进点：
1. 单一且庞大的函数。对于 minigrep 程序而言， main 函数当前执行两个任务：解析命令行参数和读取文件。但随着代码的增加，main 函数承载的功能也将快速增加。从软件工程角度来看，一个函数具有的功能越多，越是难以阅读和维护。因此最好的办法是将大的函数拆分成更小的功能单元。
2. 配置变量散乱在各处。还有一点要考虑的是，当前 main 函数中的变量都是独立存在的，这些变量很可能被整个程序所访问，在这个背景下，独立的变量越多，越是难以维护，因此我们还可以将这些用于配置的变量整合到一个结构体中。
3. 细化错误提示。 目前的实现中，我们使用 expect 方法来输出文件读取失败时的错误信息，这个没问题，但是无论任何情况下，都只输出 Should have been able to read the file 这条错误提示信息，显然是有问题的，毕竟文件不存在、无权限等等都是可能的错误，一条大一统的消息无法给予用户更多的提示。
4. 使用错误而不是异常。 假如用户不给任何命令行参数，那程序显然会无情崩溃，原因很简单：index out of bounds，一个数组访问越界的 panic，但问题来了，用户能看懂吗？甚至于未来接手的维护者能看懂吗？因此需要增加合适的错误处理代码，来给予使用者详细、友善的提示。还有就是需要在一个统一的位置来处理所有错误，利人利己！

### 分离 main
Rust 社区给出了统一的指导方案:
- 将程序分割为 main.rs 和 lib.rs，并将程序的逻辑代码移动到后者内
- 命令行解析属于非常基础的功能，严格来说不算是逻辑代码的一部分，因此还可以放在 main.rs 中
  
因此 main 函数应该包含的功能有：
- 解析命令行参数
- 初始化其它配置
- 调用 lib.rs 中的 run 函数，以启动逻辑代码的运行
- 如果 run 返回一个错误，需要对该错误进行处理

main.rs 负责启动程序，lib.rs 负责逻辑代码的运行，这种做法叫**关注点分离**（Separation of Concerns）。
```rust,name=main.rs
fn main() {
    let args: Vec<String> = std::env::args().collect();

    let config = Config::new(&args);

    println!("Searching for {}", config.query);
    println!("In file {}", config.file_path);

    let contents = std::fs::read_to_string(config.file_path)
        .expect("Should have been able to read the file");

    println!("With text:\n{contents}");
}

// 聚合配置变量为一个结构体，避免配置分散不好维护
struct Config {
    query: String,
    file_path: String,
}

impl Config {
    fn new(args: &[String]) -> Config {
        // 结构体拥有内部字符串的所有权
        // clone 整个程序只执行一次，性能消耗可忽略不计
        let query = args[1].clone();
        let file_path = args[2].clone();

        Config { query, file_path }
    }
}
// 一般对于初始化，较少使用函数
// fn parse_config(args: &[String]) -> Config {
//     let query = args[1].clone();
//     let file_path = args[2].clone();

//     Config { query, file_path }
// }
```

### 错误处理
在未传入任何参数的情况下，上面的程序会报错。这样主动调用就可以解决：
```rust
// in main.rs
 // --snip--
    fn new(args: &[String]) -> Config {
        if args.len() < 3 {
            panic!("not enough arguments");
        }
        // --snip--
```
用户看到了更为明确的提示，但是还是有一大堆我们希望屏蔽掉的 debug 输出。此时，panic 就不好使了，应考虑返回一个 Result：成功时包含 Config 实例，失败时包含一条错误信息。
```rust
impl Config {
    // new 一般不会失败，用在这里不合适
    fn build(args: &[String]) -> Result<Config, &'static str> {
        if args.len() < 3 {
            return Err("not enough arguments");
        }

        let query = args[1].clone();
        let file_path = args[2].clone();

        Ok(Config { query, file_path })
    }
}
```

### 分离主体逻辑到库中
继续精简 main 函数。
```rust
fn run(config: Config) -> Result<(), Box<dyn Error>> {
    let contents = fs::read_to_string(config.file_path)?;

    println!("With text:\n{contents}");

    Ok(())
}
```
注意这个返回值是需要处理的，否则编译的时候会有 warn。
```rust
    if let Err(e) = run(config) {
        println!("Application error: {e}");
        process::exit(1);
    }
```
将所有的非 main 函数都移动到 lib.rs 中，然后在 main.rs 中引入 lib.rs 中定义的 Config 类型。不要忘记加 pub 修饰！
```rust,name=lib.rs
// 为了满足 Result<T,E> 的要求，使用了 Ok(()) 返回一个单元类型 ()。
// Box<dyn Error> 是一个特质对象，它表示函数返回一个类型，该类型实现了 Error 特质，这样我们就无需指定具体的错误类型
fn run(config: Config) -> Result<(), Box<dyn std::error::Error>> {
    // 错误转换的关键就是 ?
    let contents = std::fs::read_to_string(config.file_path)?;

    println!("With text:\n{contents}");

    Ok(())
}

// 聚合配置变量为一个结构体，避免配置分散不好维护
pub struct Config {
    pub query: String,
    pub file_path: String,
}

impl Config {
    // new 一般不会失败，用在这里不合适
    pub fn build(args: &[String]) -> Result<Config, &'static str> {
        if args.len() < 3 {
            return Err("not enough arguments");
        }

        let query = args[1].clone();
        let file_path = args[2].clone();

        Ok(Config { query, file_path })
    }
}

// 一般对于初始化，较少使用函数
// fn parse_config(args: &[String]) -> Config {
//     let query = args[1].clone();
//     let file_path = args[2].clone();

//     Config { query, file_path }
// }
```

## 测试驱动开发
进入逻辑代码编程的环节之前，我们需要先编写一些测试代码，也就是最近颇为流行的测试驱动开发模式（TDD, Test Driven Development）。先写测试再写代码，这样的好处是只要测试通过，代码在功能层面就没什么问题了。
```rust,name=lib.rs
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn one_result() {
        let query = "duct";
        let contents = "\
Rust:
safe, fast, productive.
Pick three.";

        assert_eq!(vec!["safe, fast, productive."], search(query, contents));
    }
}
```
现在测试肯定是跑不过的，于是需要增加代码。
```rust,name=lib.rs
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    let mut results = Vec::new();
    // 遍历迭代每一行
    for line in contents.lines() {
        // 在每一行中查询目标字符串
        if line.contains(query) {
            // 存储匹配到的结果
            results.push(line);
        }
    }

    results
}
```
这样测试就能够通过。最后需要在程序中调用 search：
```rust,name=lib.rs
// in src/lib.rs
pub fn run(config: Config) -> Result<(), Box<dyn Error>> {
    let contents = fs::read_to_string(config.file_path)?;

    for line in search(&config.query, &contents) {
        println!("{line}");
    }

    Ok(())
}
```

## 环境变量
Rust 可以通过环境变量参数来控制 panic! 的栈是否展开。同样的，这种方式也可以用来实现更多操作，例如增加一个大小写敏感的开关。

首先编写一个注定失败的用例：
```rust,name=lib.rs
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn case_sensitive() {
        let query = "duct";
        let contents = "\
Rust:
safe, fast, productive.
Pick three.
Duct tape.";

        assert_eq!(vec!["safe, fast, productive."], search(query, contents));
    }

    #[test]
    fn case_insensitive() {
        let query = "rUsT";
        let contents = "\
Rust:
safe, fast, productive.
Pick three.
Trust me.";

        assert_eq!(
            vec!["Rust:", "Trust me."],
            search_case_insensitive(query, contents)
        );
    }
}
```
接着来实现这个大小写不敏感的搜索函数:
```rust,name=lib.rs
// 引入了一个新的方法 to_lowercase，它会将 line 转换成全小写的字符串
pub fn search_case_insensitive<'a>(
    query: &str,
    contents: &'a str,
) -> Vec<&'a str> {
    let query = query.to_lowercase();
    let mut results = Vec::new();

    for line in contents.lines() {
        if line.to_lowercase().contains(&query) {
            results.push(line);
        }
    }

    results
}
```
新增并检查一个配置项，用于控制是否开启大小写敏感。
```rust,name=lib.rs
pub struct Config {
    pub query: String,
    pub file_path: String,
    pub ignore_case: bool,
}

pub fn run(config: Config) -> Result<(), Box<dyn Error>> {
    let contents = fs::read_to_string(config.file_path)?;

    let results = if config.ignore_case {
        search_case_insensitive(&config.query, &contents)
    } else {
        search(&config.query, &contents)
    };

    for line in results {
        println!("{line}");
    }

    Ok(())
}
```
Rust 的 env 包提供了相应的方法来控制环境变量：
```rust,name=lib.rs
use std::env;
// --snip--

impl Config {
    pub fn build(args: &[String]) -> Result<Config, &'static str> {
        if args.len() < 3 {
            return Err("not enough arguments");
        }

        let query = args[1].clone();
        let file_path = args[2].clone();
        // is_ok 方法是 Result 提供的，用于检查是否有值，有就返回 true，没有则返回 false
        let ignore_case = env::var("IGNORE_CASE").is_ok();

        Ok(Config {
            query,
            file_path,
            ignore_case,
        })
    }
}
```
这样就可以了。测试一下：
```sh
$ IGNORE_CASE=1 cargo run -- to poem.txt
Are you nobody, too?
How dreary to be somebody!
To tell your name the livelong day
To an admiring bog!
```

## 附加

### 重定向错误
所有的输出信息，无论 debug 还是 error 类型，都是通过 println! 宏输出到终端的标准输出 stdout，但是对于程序来说，错误信息更适合输出到标准错误输出 stderr。

之前在[宏](../rust-learn-note-adv-4/)那一章里学过，将错误信息重定向到 stderr 很简单，只需在打印错误的地方，将 println! 宏替换为 eprintln!即可。

这样，运行如下命令的时候，错误就会显示在终端上而非文件里。
```sh
cargo run > output.txt
```

### 迭代器
可以直接传入迭代器，而非迭代器的消耗函数。
```rust,name=main.rs
fn main() {
    // env::args 可以直接返回一个迭代器
    let config = Config::build(env::args()).unwrap_or_else(|err| {
        eprintln!("Problem parsing arguments: {err}");
        process::exit(1);
    });

    // --snip--
}
```
再来改写 build 方法适配迭代器写法。
```rust,name=lib.rs
impl Config {
    pub fn build(
        mut args: impl Iterator<Item = String>,
    ) -> Result<Config, &'static str> {
        // --snip--
```
为了可读性和更好的通用性，这里的 args 类型并没有使用本身的 std::env::Args ，而是使用了特质约束的方式来描述 impl Iterator<Item = String>，这样意味着 arg 可以是任何实现了 String 迭代器的类型。还有一点值得注意，由于迭代器的所有权已经转移到 build 内，因此可以直接对其进行修改，这里加上了 mut 关键字。

数组索引会越界，为了安全性和简洁性，使用 Iterator 特质自带的 next 方法是一个更好的选择。
```rust,name=lib.rs
impl Config {
    pub fn build(
        mut args: impl Iterator<Item = String>,
    ) -> Result<Config, &'static str> {
        // 第一个参数是程序名，由于无需使用，因此这里直接空调用一次
        args.next();

        let query = match args.next() {
            Some(arg) => arg,
            None => return Err("Didn't get a query string"),
        };

        let file_path = match args.next() {
            Some(arg) => arg,
            None => return Err("Didn't get a file path"),
        };

        let ignore_case = env::var("IGNORE_CASE").is_ok();

        Ok(Config {
            query,
            file_path,
            ignore_case,
        })
    }
}
```
使用迭代器适配器让搜索的代码更简洁：
```rust,name=lib.rs```
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    contents
        .lines()
        .filter(|line| line.contains(query))
        .collect()
}
```
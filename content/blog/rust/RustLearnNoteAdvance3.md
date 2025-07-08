+++
title = "Rust 进阶学习笔记（三）：文件操作"
slug = "rust_learn_note_adv_3"
date = 2025-07-07
updated = 2025-07-08
description = "IO，文件的增删，读写，流操作"
# draft = true
[taxonomies]
tags = ["Rust", "Learn"]
[extra]
pinned = false
post_listing_date = "both"
+++

本节出处：原子之音 [视频](https://www.bilibili.com/video/BV1jf4y1p7BV/)

## 基本操作
Rust 主要通过`std::fs::File`来实现文件操作。

结构体`File`可用于描述或操作一个文件。

`File`的所有方法都会返回一个`Result`枚举。

[官方文档](https://doc.rust-lang.org/std/fs/struct.File.html) [中文文档](https://rustwiki.org/zh-CN/std/fs/struct.File.html)。

### 文件的打开、创建、删除
`File::open`使用只读模式打开一个文件。
- 返回文件句柄
- 文件不存在则抛出错误
`File::create`使用只写模式打开一个文件。
- 文件存在则清空
- 文件不存在则新建
- 返回文件句柄
- `std::fs::OpenOptions`可设置为追加模式
`std::fs::remove_file`从文件系统删除一个文件，注意删除文件的方法在 fs 下面，和 File 是没有关系的！

***
### 例子
```rust
use std::io::Write;

fn main() -> std::io::Result<()> {
    // 打开
    let f = std::fs::File::open("README.md");
    println!("{:?}", f); // Ok(File { fd: 3, path: "/media/jmy/code/rust-by-practice/README.md", read: true, write: false })
    // 若不存在，会返回 Err(Os { code: 2, kind: NotFound, message: "No such file or directory" })
    // 创建，注意是覆盖式的，会清空已存在文件
    let mut f = std::fs::File::create("foo.txt")?;
    println!("{:?}", f);
    // 必须要 use std::io::Write; 将 Write 特质引入当前作用域
    // ? 用于消除警告
    f.write("中文".as_bytes())?; // 返回值是写入的 usize，可能部分写入
    f.write_all(b"hello!")?; // 返回值是写入结果

    std::fs::remove_file("foo.txt")?;
    Ok(())
}
```

## 读写操作

### 读取文件的方式
- 读取成一个数组
- 读取成一个完整字符串
- 逐行读取
```rust
use std::io::{BufRead, BufReader, Read, Write};

fn main() -> std::io::Result<()> {
    let mut file = std::fs::File::open("README.md")?;
    let mut data = Vec::new();
    // u8 数组
    file.read_to_end(&mut data)?;
    println!("{:?}", data);
    // 字符串
    let mut file = std::fs::File::open("README.md")?;
    let mut content = String::new();
    file.read_to_string(&mut content)?;
    println!("{:?}", content);
    // 一行
    let mut file = std::fs::File::open("README.md")?;
    // 流式
    let reader = BufReader::new(file);
    // 注意 line 是一个 result
    // lines() 需要 BufRead 特质
    for (index, line) in reader.lines().enumerate() {
        let line = line?;
        println!("{}. {}", index + 1, line);
    }
    Ok(())
}
```

### 写文件的方式
- 新建
  - 写字符串
  - 写比特
- 追加：`std::fs::OpenOptions::new().append(true).open("foo.txt");`
```rust
use std::env;
use std::io::Write;

fn main() -> std::io::Result<()> {
    let temp_dir = env::temp_dir();
    println!("{:?}", temp_dir);
    let temp_file = temp_dir.join("temp.txt");
    let mut file = std::fs::File::create(&temp_file)?; // 不加 ? 返回值会是一个 Result
    // 写一行
    writeln!(&mut file, "字符串")?;
    // 写 byte
    file.write(b"111111111111111111111111111111111111111111111111111111111111111111111111111111")?;
    // 追加
    let mut file = std::fs::OpenOptions::new().append(true).open(temp_file)?;
    // 注意此处只是为了演示，实际上不能这么写，字符串会被截断
    file.write_all("\n1111111111111111111111111".as_bytes())?;
    Ok(())
}
// 文件内容
// cat /tmp/temp.txt
// 字符串
// 111111111111111111111111111111111111111111111111111111111111111111111111111111
// 1111111111111111111111111%  
```

### 路径操作
rust 路径操作都在`std::fs`下，包括`create_dir`、`remove_dir`等。

下面例子[出处](https://doc.rust-lang.org/nightly/rust-by-example/zh/std_misc/fs.html)。
```rust
use std::fs;
use std::fs::{File, OpenOptions};
use std::io;
use std::io::prelude::*;
#[cfg(target_family = "unix")]
use std::os::unix;
#[cfg(target_family = "windows")]
use std::os::windows;
use std::path::Path;

// `% cat path` 命令的简单实现
fn cat(path: &Path) -> io::Result<String> {
    let mut f = File::open(path)?;
    let mut s = String::new();
    match f.read_to_string(&mut s) {
        Ok(_) => Ok(s),
        Err(e) => Err(e),
    }
}

// `% echo s > path` 命令的简单实现
fn echo(s: &str, path: &Path) -> io::Result<()> {
    let mut f = File::create(path)?;

    f.write_all(s.as_bytes())
}

// `% touch path` 命令的简单实现（忽略已存在的文件）
fn touch(path: &Path) -> io::Result<()> {
    match OpenOptions::new().create(true).write(true).open(path) {
        Ok(_) => Ok(()),
        Err(e) => Err(e),
    }
}

fn main() {
    println!("`mkdir a`");
    // 创建目录，返回 `io::Result<()>`
    match fs::create_dir("a") {
        Err(why) => println!("! {:?}", why.kind()),
        Ok(_) => {},
    }

    println!("`echo hello > a/b.txt`");
    // 可以使用 `unwrap_or_else` 方法简化之前的匹配
    echo("hello", &Path::new("a/b.txt")).unwrap_or_else(|why| {
        println!("! {:?}", why.kind());
    });

    println!("`mkdir -p a/c/d`");
    // 递归创建目录，返回 `io::Result<()>`
    fs::create_dir_all("a/c/d").unwrap_or_else(|why| {
        println!("! {:?}", why.kind());
    });

    println!("`touch a/c/e.txt`");
    touch(&Path::new("a/c/e.txt")).unwrap_or_else(|why| {
        println!("! {:?}", why.kind());
    });

    println!("`ln -s ../b.txt a/c/b.txt`");
    // 创建符号链接，返回 `io::Result<()>`
    #[cfg(target_family = "unix")] {
        unix::fs::symlink("../b.txt", "a/c/b.txt").unwrap_or_else(|why| {
            println!("! {:?}", why.kind());
        });
    }
    #[cfg(target_family = "windows")] {
        windows::fs::symlink_file("../b.txt", "a/c/b.txt").unwrap_or_else(|why| {
            println!("! {:?}", why.to_string());
        });
    }

    println!("`cat a/c/b.txt`");
    match cat(&Path::new("a/c/b.txt")) {
        Err(why) => println!("! {:?}", why.kind()),
        Ok(s) => println!("> {}", s),
    }

    println!("`ls a`");
    // 读取目录内容，返回 `io::Result<Vec<Path>>`
    match fs::read_dir("a") {
        Err(why) => println!("! {:?}", why.kind()),
        Ok(paths) => for path in paths {
            println!("> {:?}", path.unwrap().path());
        },
    }

    println!("`rm a/c/e.txt`");
    // 删除文件，返回 `io::Result<()>`
    fs::remove_file("a/c/e.txt").unwrap_or_else(|why| {
        println!("! {:?}", why.kind());
    });

    println!("`rmdir a/c/d`");
    // 删除空目录，返回 `io::Result<()>`
    fs::remove_dir("a/c/d").unwrap_or_else(|why| {
        println!("! {:?}", why.kind());
    });
}
```
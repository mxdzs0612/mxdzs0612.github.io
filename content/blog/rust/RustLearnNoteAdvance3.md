+++
title = "Rust 进阶学习笔记（三）：文件操作"
slug = "rust_learn_note_adv_3"
date = 2025-07-07
updated = 2025-07-07
description = "IO，文件的增删，读写，路径的操作"
# draft = true
[taxonomies]
tags = ["Rust", "Learn"]
[extra]
pinned = false
post_listing_date = "both"
+++

出处：原子之音 [视频](https://www.bilibili.com/video/BV1jf4y1p7BV/)

## 文件操作
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
`std::fs::remove_file`从文件系统删除一个文件，注意删除文件只涉及到 fs，和 File 是没有关系的！

<!-- {{ admonition(type="warning", icon="tip", title="注意", text="施工中") }} -->
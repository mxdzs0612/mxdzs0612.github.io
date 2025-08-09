+++
title = "Rust 项目实战（四）：写个链表"
slug = "rust_learn_note_prac_4"
date = 2025-07-30
updated = 2025-07-30
description = ""
draft = true
[taxonomies]
tags = ["Rust", "Learn", "Project"]
[extra]
pinned = false
post_listing_date = "both"
+++

本节出处：[圣经 文件搜索工具](https://course.rs/basic-practice/intro.html) [The Book 构建一个命令行程序](https://rustwiki.org/zh-CN/book/ch12-00-an-io-project.html)，主要是前者

学完了，该整项目了。先来个入门的命令行工具吧。本节会尝试构建一个命令行程序，从命令行参数中读取指定的文件名和字符串，然后在相应的文件中找到包含该字符串的内容，最终打印出来。这个项目肯定是要单独起一个工程敲一遍的。我直接 init 一个库吧，反正 github 不要钱（

{{ admonition(type="warning", title="注意", text="施工中") }}

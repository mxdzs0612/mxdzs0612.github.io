+++
title = "Rust 进阶学习笔记（五）：Cargo指南"
slug = "rust_learn_note_adv_5"
date = 2025-07-09
updated = 2025-07-10
description = "项目及其目录结构，包，模块，模块的引入与模块可见性，包的构建，依赖的添加，cargo配置和清单，工作空间，条件编译，发布配置"
# draft = true
[taxonomies]
tags = ["Rust", "Learn"]
[extra]
pinned = false
post_listing_date = "both"
+++

本节出处：[圣经：包和模块](https://course.rs/basic/crate-module/intro.html) [圣经：Cargo](https://course.rs/cargo/intro.html)

当工程规模变大时，把代码写到一个甚至几个文件中，都是不太聪明的做法。反之，将大的代码文件拆分成包和模块，还允许我们实现代码抽象和复用。因此，跟其它语言一样，Rust 也提供了代码的组织管理的方式。

Cargo 是包管理工具，可以用于依赖包的下载、编译、更新、分发等。Cargo 依赖 [crates.io](https://crates.io/)，它是社区提供的包注册中心：用户可以将自己的包发布到该注册中心，然后其它用户通过注册中心引入该包。可以理解成 maven 中央仓库。

本节中，我们会仔细学习这些概念，目的是搞清楚一个项目结构的方方面面。

## 项目，包和模块
- 项目（Packages）：一个 Cargo 提供的 feature，可以用来构建、测试和分享包
- 包（Crate）：一个由多个模块组成的树形结构，可以作为三方库进行分发，也可以生成可执行文件进行运行
- 模块（Module）：可以一个文件多个模块，也可以一个文件一个模块，模块可以被认为是真实项目中的代码组织单元

### 包和项目

#### 包
包（Crate）是 Rust 的一个独立的可编译单元，它编译后会生成一个可执行文件或者一个库。

一个包会将相关联的功能打包在一起，使得该功能可以很方便的在多个项目中分享。我们只需要将该包通过`use xxx;`引入到当前项目的作用域中，就可以在项目中使用 xxx 的功能。

同一个包中不能有同名的类型，但是在不同包中就可以。在代码中通过不同包名头引用，编译器是不会产生歧义的。

多个包联合在一起，组织成工作空间（WorkSpace）。

#### 项目
这里的项目（Package）可以理解为工程、软件包。它包含有独立的`Cargo.toml`文件，以及因为功能性被组织在一起的一个或多个包。

{{ admonition(type="warning", icon="tip", title="注意", text="施工中") }}
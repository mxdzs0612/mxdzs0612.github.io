+++
title = "Rust 进阶学习笔记（五）：包，模块与Cargo指南"
slug = "rust_learn_note_adv_5"
date = 2025-07-09
updated = 2025-07-12
description = "项目及其目录结构，包，模块，模块的引入与模块可见性，包的构建，依赖的添加，cargo配置和清单，工作空间，条件编译，发布配置"
# draft = true
[taxonomies]
tags = ["Rust", "Learn"]
[extra]
pinned = false
post_listing_date = "both"
+++

本节出处：[圣经：包和模块](https://course.rs/basic/crate-module/intro.html) [圣经：Cargo](https://course.rs/cargo/intro.html)

***

当工程规模变大时，把代码写到一个甚至几个文件中，都是不太聪明的做法。反之，将大的代码文件拆分成包和模块，还允许我们实现代码抽象和复用。因此，跟其它语言一样，Rust 也提供了代码的组织管理的方式。

Cargo 是包管理工具，可以用于依赖包的下载、编译、更新、分发等。Cargo 依赖 [crates.io](https://crates.io/)，它是社区提供的包注册中心：用户可以将自己的包发布到该注册中心，然后其它用户通过注册中心引入该包。可以理解成 maven 中央仓库。另有一个网站 [lib.rs](https://lib.rs/) 非常适合用来查找包。

本节中，我们会仔细学习这些概念，目的是搞清楚一个项目结构的方方面面。这一节乍一看十分基础，其他语言也都有类似的东西，实际上非常重要，能让我们从代码之外进一步理解 Rust。

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
这里的项目（Package）可以理解为工程、软件包。它包含有独立的`Cargo.toml`文件，以及因为功能性被组织在一起的一个或多个包。一个项目只能包含一个库（library）包，但可以包含多个可执行的二进制包。

下面的命令将会创建一个项目。
```bash
cargo new my-project
```
此时会出现一个名称是 my-project 的 package，其中包含一个`Cargo.toml`文件，以及一个`src/main.rs`。`src/main.rs`会被 Rust 默认作为二进制包的根文件，该二进制包的包名跟所属 Package 相同，所有的代码执行都从该文件中的`fn main()`函数开始。

使用`cargo run`可以运行该项目，输出：*Hello, world!*。

再来创建一个库包。
```bash
cargo new my-lib --lib
```
库类型的包只能作为三方库被其它项目引用，无法独立运行。同样道理，`src/lib.rs`会被 Rust 默认当作库包的根文件。

如此一来，一个典型项目会拥有如下的结构。其中可能会包含多个二进制包，这些包文件被放在`src/bin`目录下，每一个文件都是独立的二进制包；同时也会包含一个库包，该包只能存在一个`src/lib.rs`：
```bash
.
├── Cargo.toml
├── Cargo.lock
├── src
│   ├── main.rs # 主二进制包，编译后生成和项目同名的可执行文件
│   ├── lib.rs # 唯一库包
│   └── bin
│       └── main1.rs # 副二进制包，编译后会生成和文件同名的二进制可执行文件
│       └── main2.rs
├── tests
│   └── some_integration_tests.rs # 集成测试文件
├── benches
│   └── simple_bench.rs # 基准测试
└── examples
    └── simple_example.rs # 示例
```

### 模块
模块（module）是 Rust 的代码构成单元。模块可以将包中的代码按照功能进行重组，最终实现更好的可读性及易用性。同时，还能让开发人员非常灵活地去控制代码的可见性。

#### 模块树
模块之间的关系可以用一棵树表示，类似文件系统的树。从一个包根（crate root）形成的模块出发，可以嵌套若干个模块。模块有如下特点：
- 使用`mod`关键字来创建新模块，后面紧跟着模块名称
- 模块可以嵌套
- 模块中可以定义各种 Rust 类型，例如函数、结构体、枚举、特质等
- 所有模块均定义在同一个文件中

因此，就像树的节点一样，模块之间也存在父子关系。

#### 引用模块

##### 路径引用
Rust 中的路径有两种形式：
- 绝对路径：从包根开始，路径名以包名（或`crate`）开头。
- 相对路径：从当前模块开始，以当前模块的标识符（`self`、`super`）开头。

当然，相对路径什么都不加也会视为`self`。可以看如下例子：
```rust
// 假设函数和模块都定义在包根下
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

pub fn eat_at_restaurant() {
    // 绝对路径
    // 绝对路径引用时，可以直接以 crate 开头，然后逐层引用，每一层之间使用`::`分隔：
    crate::front_of_house::hosting::add_to_waitlist();

    // 相对路径
    // 模块和函数位于同一个包根下，可以直接从模块名开头
    front_of_house::hosting::add_to_waitlist();
}
```
实际开发中，调用的地方和定义的地方往往是分离的，而定义的地方较少会变动。所以可以优先考虑使用绝对路径。

Rust 出于安全的考虑，默认情况下，所有的类型都是私有化的，包括函数、方法、结构体、枚举、常量甚至模块本身。父模块完全无法访问子模块中的私有项，但是子模块却可以访问父模块、父父..模块的私有项。

Rust 提供了`pub`关键字，通过它你可以控制模块和模块中指定项的**可见性**。模块可见性不代表模块内部项的可见性，模块的可见性仅仅是允许其它模块去引用它，但是想要引用它内部的项，还得继续将对应的项标记为`pub`。上例中，子模块和里面的函数都需要标注。

##### 父引用
`super`代表的是父模块为开始的引用方式，非常类似于文件系统中的`..`语法
```rust
fn serve_order() {}

// 厨房模块
mod back_of_house {
    fn fix_incorrect_order() {
        cook_order();
        super::serve_order();
    }

    fn cook_order() {}
}
```
这样的好处是，只要相对关系不变，未来就算它们都不在包根了，依然无需修改引用路径。

##### 自引用
`self`其实就是引用自身模块中的项。
```rust
fn serve_order() {
    self::back_of_house::cook_order()
}

mod back_of_house {
    fn fix_incorrect_order() {
        cook_order();
        crate::serve_order();
    }

    pub fn cook_order() {}
}
```
表面看起来，加不加`self`没有区别，似乎多此一举，本例中确实如此。实际上，self 通常起到在嵌套场景下指明路径起点的作用，防止在某些上下文中无法推断路径。

#### 模块分离
当模块变多或者变大时，需要将模块放入一个单独的文件中，让代码更好维护。
```rust,name=src/front_of_house.rs
pub mod hosting {
    pub fn add_to_waitlist() {}
}
```
```rust,name=src/lib.rs
// 从另一个和模块 front_of_house 同名的文件中加载该模块的内容
mod front_of_house;
// 使用绝对路径的方式来引用模块
pub use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
}
```
这种情况下，模块的声明和实现是分离的，声明语句可以把模块的内容从声明对应的文件中加载进来。`use`关键字能够将外部模块中的项引入到当前作用域中来，避免冗长的调用前缀。

当一个模块有许多子模块时，也可以通过文件夹的方式来组织这些子模块。需要显示指定暴露哪些子模块。假设此时创建一个目录 front_of_house，然后在文件夹里创建一个 hosting.rs 文件，内容是
```rust,name=hosting.rs
pub fn add_to_waitlist() {}
```
指定方法是创建一个**新的文件**来定义子模块，子模块名与文件名需要相同。
```rust
pub mod hosting;
```
这个新的文件可以是如下两种：
- 在 front_of_house 目录里创建一个`mod.rs`，如果 rustc 版本在 1.30 之前，这是唯一的方法。
- 在 front_of_house 同级目录里创建一个与模块（目录）同名的 rs 文件`front_of_house.rs`，在新版本里，更建议使用这样的命名方式来避免项目中存在大量同名的 mod.rs 文件。

### 可见性
我们已经知道，模块默认是私有的。可以添加`pub`关键字，使其变成公有的。模块上的`pub`关键字只允许其父模块引用它，而不允许访问内部代码。因为模块是一个容器，只是将模块变为公有能做的其实并不太多；同时需要更深入地选择将一个或多个项变为公有。

#### 结构体和枚举的可见性
结构体和枚举的字段的可见性完全不同：
- 将结构体设置为`pub`，但它的所有字段依然是私有的
- 将枚举设置为`pub`，它的所有字段也将对外可见

原因在于，枚举和结构体的使用方式不一样。如果枚举的成员对外不可见，那该枚举将一点用都没有，因此枚举成员的可见性自动跟枚举可见性保持一致，这样可以简化用户的使用。

而结构体的应用场景比较复杂，其中的字段可能一部分在这里被使用，以部分在那里被使用，因此无法确定成员的可见性，那索性就设置为全部不可见，将选择权交给程序员。

#### use 与 pub
在 Rust 中，可以使用`use`关键字把路径提前引入到当前作用域中，随后的调用就可以省略该路径，极大地简化了代码。

##### 基本方式
引入模块中的项有两种方式：绝对路径和相对路径。也可以选择引入模块或者函数。
```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }

    pub mod serving {
        pub fn from_waitlist() {}
    }
}
// 绝对路径 + 引入模块
use crate::front_of_house::hosting;
// 相对路径 + 引入函数
use front_of_house::serving::from_waitlist;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
    from_waitlist();
}
```
从使用简洁性来说，引入函数自然是更甚一筹，但是在某些时候，引入模块会更好：
- 需要引入同一个模块的多个函数
- 作用域中存在同名函数

一般建议优先使用最细粒度（引入函数、结构体等）的引用方式，如果引起了某种麻烦（例如前面两种情况），再使用引入模块的方式。

##### 避免同名引用
我们只要保证同一个模块中不存在同名项就行，模块之间、包之间可以存在同名。引入时避免同名的方式有：
- 模块::函数，即通过父模块来调用
```rust
use std::fmt;
use std::io;

fn function1() -> fmt::Result {
    // --snip--
}

fn function2() -> io::Result<()> {
    // --snip--
}
```
- 别名：使用`as`关键字赋予引入项一个全新的名称
```rust
use std::fmt::Result;
use std::io::Result as IoResult;

fn function1() -> Result {
    // --snip--
}

fn function2() -> IoResult<()> {
    // --snip--
}
```

##### 引入项导出
当外部的模块项 A 被引入到当前模块中时，它的可见性自动被设置为私有的。使用`pub use`可实现允许其它外部代码引用我们的模块项。
```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

pub use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
}
```
这常用于统一使用一个模块来提供对外的 API的场景。此时可以引入其它模块中的 API，然后进行再导出，最终对于用户来说，所有的 API 都是由一个模块统一提供的。

##### 第三方包
修改 Cargo.toml 文件，在`[dependencies]`区域添加一行：{包名} = "{版本号}"。等下载完成后，使用`use`即可添加第三方库。

第三方包都可在 [crates.io](https://crates.io/) 和 [lib.rs](https://lib.rs/) 中下载和查找。

##### 简化引入
可以使用`{}`来一起引入具有相同前缀的模块，大量减少`use`的使用。
```rust
// 前
use std::collections::HashMap;
use std::collections::BTreeMap;
use std::collections::HashSet;

use std::io;
use std::io::Write;
//后，注意这个 self，可用来替代模块自身
use std::collections::{HashMap,BTreeMap,HashSet};
use std::io::{self, Write};
```

##### 全部引入
还可以使用
```rust
use std::collections::*;
```
引入一个模块中的所有公共项。此时，由于我们根本无法判断引入了哪些东西，容易引发冲突，一般不建议这么使用，但是可以用于快速编写测试代码。

#### 受限可见性
Rust 还提供了受限可见性，即可以控制模块中的公开内容能被哪些人看见。

`pub(crate)` 或 `pub(in crate::a)` 就是限制可见性语法，前者是限制在整个包内可见，后者是通过绝对路径，限制在包内的某个模块内可见。
```rust
pub mod a {
    pub const I: i32 = 3;

    fn semisecret(x: i32) -> i32 {
        use self::b::c::J;
        x + J
    }

    pub fn bar(z: i32) -> i32 {
        semisecret(I) * z
    }
    pub fn foo(y: i32) -> i32 {
        semisecret(I) + y
    }

    mod b {
        // 指定了模块 c 和常量 J 的可见范围都只是 a 模块中，a 之外的模块是完全访问不到它们的。
        pub(in crate::a) mod c {
            pub(in crate::a) const J: i32 = 4;
        }
    }
}
```

### pub 总结
- `pub` 意味着可见性无任何限制
- `pub(crate)` 表示在当前包可见
- `pub(self)` 在当前模块可见
- `pub(super)` 在父模块可见
- `pub(in <path>)` 表示在某个路径代表的模块中可见，其中 path 必须是父模块或者祖先模块

## Cargo

{{ admonition(type="warning", icon="warning", title="注意", text="内容太多信息密度太低，看完不想写了，已 TJ") }}
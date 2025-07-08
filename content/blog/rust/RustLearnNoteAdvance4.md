+++
title = "Rust 进阶学习笔记（四）：宏"
slug = "rust_learn_note_adv_4"
date = 2025-07-08
updated = 2025-07-08
description = "声明宏，过程宏"
# draft = true
[taxonomies]
tags = ["Rust", "Learn"]
[extra]
pinned = false
post_listing_date = "both"
+++

本节出处：[Databend 分享](https://www.bilibili.com/video/BV1Yb4y1U7r1)

[Rust 程序设计语言-20.5宏](https://kaisery.github.io/trpl-zh-cn/ch20-05-macros.html)

[Rust语言圣经-4.10宏编程](https://course.rs/advance/macro.html)
[^1]

## 宏
之前已经遇到过很多带有`!`的东西，比如`println!`、`vec!`等，以及一些`#[derive]`。这些东西就叫做宏。

宏（Macro）指的是 Rust 中一系列的功能：使用`macro_rules!`的声明宏（declarative macro），和三种过程宏（procedural macro）：派生宏、属性宏、函数宏。

所谓宏，其实就是一种通过代码生成代码的方式，即“元编程（metaprogramming）”。宏以**展开**的方式来生成比宏本身的代码更多的代码。

和函数相比，宏不需要声明参数的数量和类型，并且会在编译器解析代码前展开，而非运行时调用。因此，宏的定义更复杂，更难维护。

## Rust 编译过程
![编译过程](/pictures/compile-process.png) [^2]

Rust 整体编译过程就是图中间“编译前端”`Rustc`到“编译后端”`LLVM`这一部分。我们的文本代码会经过分词形成词条流（即词法分析），这一过程中就会解析宏。词法分析后会形成一个抽象语法树（AST），然后进行语义分析。HIR 是 AST 简化后的降级，进行语法糖的“脱糖”，做类型推断。MIR 是进一步降级形成中级中间语言，会进行借用检查。最后生成一个 LLVM 的中间语言。

可以从 [Rust 官方Playground](https://play.rust-lang.org/)的左上角清晰看到整个过程生成的代码是什么样的。

宏的解析器是主线之外单独的，并且声明宏和过程宏也是分开的。声明宏实际上是一个**替换**，替换会发生在声明层面，然后混入普通的 token stream 里面，这个过程可以理解为和正则表达式是一样的，没有计算和语义分析。过程宏的工作机制也类似，但是其出入参都是 token stream，在 token stream 上通过第三方库 Syn 构造了一个自己的 AST，其目的是方便根据类型信息进行一些计算，最后会转换为普通的 token stream。

## 声明宏
声明宏（declarative macros）允许编写一些类似`match`表达式的控制结构，将一个值和包含相关代码的**模式**进行比较，一旦匹配成功，每个模式的相关代码会替换传递给宏的代码。所有这一切都发生于编译时。

举个例子，假设我们创建了一个数组，rust 会通过`std::prelude`自动引入宏。
```rust
let v: Vec<u32> = vec![1, 2, 3];
```
`vec!`创建的动态数组支持任何元素类型，也并没有限制数组的长度。

来看`!vec`的简化版本定义，这里省略了预分配空间的代码。
```rust,name=lib.rs
#[macro_export]
macro_rules! vec {
    ( $( $x:expr ),* ) => {
        {
            let mut temp_vec = Vec::new();
            $(
                temp_vec.push($x);
            )*
            temp_vec
        }
    };
}
```
可以看到，`!vec`的定义看起来就是只有一个分支、一个模式的`match`表达式。

### 定义方式
`#[macro_export]`注解表明只要导入了定义这个宏的`crate`，该宏就应该是可用的。如果没有该注解，这个宏不能被引入作用域。

接着使用`macro_rules!`和宏名称开始宏定义，且所定义的宏并**不带**感叹号。名字后跟大括号表示宏定义体，在该例中宏名称是`vec`。

`vec!`宏的结构和`match`表达式的结构类似。此处有一个分支模式`( $( $x:expr ),* )`，后跟`=>`以及和模式相关的代码块。如果模式匹配，该相关代码块将被**展开**。鉴于这个宏只有一个模式，那就只有一个有效匹配方式，其他任何不匹配这个模式都会导致错误。更复杂的宏会有不止一个分支。

### 模式解析
首先，我们使用圆括号`()`将整个宏模式包裹其中。紧随其后的是`$()`，与`$`后的括号中的模式相匹配的值(传入的 Rust 源代码)会被捕获，然后用于代码替换。在本例中，模式`$x:expr`会匹配任何 Rust 表达式，并把他叫做`$x`。`$()`后面的`,`表示所匹配的代码使用逗号分隔符分割，`*`表示`*`之前的模式，即`$()`中的部分将会被匹配任意次（零到无限次，类似正则表达式）。

我们创建的数组中，这个模式就是被匹配并循环了 3 次。

> 注意这种写法最后不能再跟`,`。想要能够匹配这种可有可无的逗号，需要这样写：
```rust
($($x:expr),+ $(,)?) => (
    <[_]>::into_vec(
        // This rustc_box is not required, but it produces a dramatic improvement in compile
        // time when constructing arrays with many elements.
        #[rustc_box]
        $crate::boxed::Box::new([$($x),+])
    )
);
```
接下来，我们再来看看与模式相关联、在`=>`之后的代码。`$()`中的部分将根据模式匹配的次数生成对应的代码，本例中就是循环执行了 3 次`temp_vec.push($x);`。当调用`vec![1, 2, 3]`时，下面这段生成的代码将替代传入的源代码：
```rust
{
    let mut temp_vec = Vec::new();
    temp_vec.push(1);
    temp_vec.push(2);
    temp_vec.push(3);
    temp_vec
}
```
至此，我们定义了一个宏，它可以接受任意类型和数量的参数，并且理解了其语法的含义。

### 扩展
[The Little Book of Rust Macros](https://lukaswirth.dev/tlborm/)


## 过程宏


{{ admonition(type="warning", icon="tip", title="注意", text="施工中") }}
***

[^1]: 从这里可以看出来，圣经就是 rust book 的说人话版本
[^2]: 图源 [张汉东](https://github.com/ZhangHanDong/inviting-rust)
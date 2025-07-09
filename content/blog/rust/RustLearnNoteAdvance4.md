+++
title = "Rust 进阶学习笔记（四）：宏"
slug = "rust_learn_note_adv_4"
date = 2025-07-08
updated = 2025-07-08
description = "Rust编译过程和宏，声明宏，过程宏：派生宏、属性宏、函数宏"
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

这一节写的比较粗糙，等我学完 Workshop 后会重新整理的。

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
首先，我们使用圆括号`()`将整个宏模式包裹其中。紧随其后的是`$()`，与`$`后的括号中的模式相匹配的值(传入的 Rust 源代码)会被捕获，然后用于代码替换。在本例中，模式`$x:expr`会匹配任何 Rust 表达式，并把它叫做`$x`。`$()`后面的`,`表示所匹配的代码使用逗号分隔符分割，`*`表示`*`之前的模式，即`$()`中的部分将会被匹配任意次（零到无限次，类似正则表达式）。

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
过程宏的本质是编译过程中的一个过滤器或中间件，它接收一段用户编写的源代码，返回给编译器一段经过修改后的代码。

过程宏必须定义在一个单独的 crate 中。这一点需要回到编译过程来理解：Rust 的最小编译单元就是 crate，过程宏是在编译一个 crate 之前对 crate 的代码进行加工的一段程序，然而它本身也需要编译后才能执行。如果定义和使用过程宏的代码写在一个 crate 中，编译过程就死锁了。

得益于此，在一个项目中定义过程宏的代码往往会位于一个单独的 crate，我们很容易找到并学习。

### 定义过程宏
直接来看几个例子。

首先需要在`Cargo.toml`中定义`proc-macro=true`，然后添加三个依赖包，因为过程宏编译器依赖这三个包来进行转换。
```toml,name=my_macro/Cargo.toml
[package]
name = "my_macro"
version = "0.1.0"
edition = "2024"

[lib]
proc-macro=true

[dependencies]
proc-macro2 = "1.0.60"
quote = "1"
syn = { version = "2.0", features = ["full"] }
```
然后在主项目的`Cargo.toml`中使用`dependencies`或`workspace`引入过程宏的包。
```toml,name=Cargo.toml
[package]
name = "macro_test"
version = "0.1.0"
edition = "2024"

[dependencies]
my_macro = { path = "my_macro"}
```
接下来定义一个简单的过程宏。
```rust,name=my_macro/src/lib.rs
use proc_macro::TokenStream;

// proc_macro_attribute 表示定义的是一个属性宏，派生宏是 #[proc_macro_derive]，如果是函数宏，则是 #[proc_macro]
// 只有在 Cargo.toml 中设置了 proc-macro=true 的 crate 能够引入这一标注
#[proc_macro_attribute]
pub fn my_first_attribute_proc_macro(attr: TokenStream, input: TokenStream) -> TokenStream {
    // println! 在宏中无法 debug 和输出，只能使用用于输出错误的打印来调试
    eprintln!("attr: {:#?}", attr);
    eprintln!("input: {:#?}", input);
    input
}
```
这段过程宏定义的代码已经写完了，可以看到作为演示，它实际上什么都没做。

然后我们尝试进行调用。
```rust,name=src/main.rs
// 过程宏所在的包名叫 my_macro
// test 就是 attr
#[my_macro::my_first_attribute_proc_macro("test")]
fn add(a:i32, b:i32) -> i32 {
    a + b
}

fn main() {
    println!("{}", add(1, 2));
}
```
上文定义的这个宏不会改变代码，只在编译时可见。如要在运行时改变函数，可考虑这种宏
```rust
#[proc_macro_attribute]
pub fn trace(_attr: TokenStream, input: TokenStream) -> TokenStream {
    // 为函数添加执行日志
    let modified = quote! {
        println!("Function entered!");
        #input // 插入原函数代码
    };
    modified.into()
}
```
再来看一个简单的过程宏。这是 deno flask test 的 0.1.0 版本，[源码](https://github.com/denoland/flaky_test/blob/v0.1.0/src/lib.rs)。
```rust
// 这个宏创建了一个“不稳定测试”的包装器，会将一个普通函数转换为一个可以自动重试的测试函数，最多重试 3 次。
extern crate proc_macro;
extern crate syn;
use proc_macro::TokenStream;
use quote::quote;

#[proc_macro_attribute]
pub fn flaky_test(_attr: TokenStream, input: TokenStream) -> TokenStream {‘
  // ItemFn 是 syn 库中表示函数定义的类型
  // parse_macro_input 用于解析输入的 TokenStream
  let input_fn = syn::parse_macro_input!(input as syn::ItemFn);
  // sig.ident 是在访问函数标识符，也就是函数名
  let name = input_fn.sig.ident.clone();
  // quote! 的作用是生成 rust 代码
  // TokenStream::from 提供 rust 代码到 TokenStream 的转换
  TokenStream::from(quote! {
    // #[test]：标准Rust测试属性，使函数成为测试
    // 创建一个新的测试函数，使用与原始函数相同的名称（#name）
    // quote! 宏使用`#`符号作为插值标记，插值是指将一个变量的值嵌入或插入到另一个上下文中的过程。
    #[test]
    fn #name() {
      // 将原始函数的完整定义嵌入到新函数中，使其在函数体内可用
      #input_fn

      for i in 0..3 {
        println!("flaky_test retry {}", i);
        // catch_unwind 捕获闭包执行过程中的panic
        // 它接收一个闭包并执行，如果闭包panic，返回Result::Err，否则返回Result::Ok
        let r = std::panic::catch_unwind(|| {
            #name();
        });
        if r.is_ok() {
            return;
        }
        if i == 2 {
            // resume_unwind 重新抛出捕获的 panic，保持原始的 panic 信息
            std::panic::resume_unwind(r.unwrap_err());
        }
      }
    }
  })
}
```

### 理解过程宏
上一节中我们打印了 attr 和 input 的内容。其输出如下：
```text
attr: TokenStream [
    Literal {
        kind: Str,
        symbol: "test",
        suffix: None,
        span: #0 bytes(54..60),
    },
]
input: TokenStream [
    Ident {
        ident: "fn",
        span: #0 bytes(63..65),
    },
    Ident {
        ident: "add",
        span: #0 bytes(66..69),
    },
    Group {
        delimiter: Parenthesis,
        stream: TokenStream [
            Ident {
                ident: "a",
                span: #0 bytes(70..71),
            },
            Punct {
                ch: ':',
                spacing: Alone,
                span: #0 bytes(71..72),
            },
            Ident {
                ident: "i32",
                span: #0 bytes(72..75),
            },
            Punct {
                ch: ',',
                spacing: Alone,
                span: #0 bytes(75..76),
            },
            Ident {
                ident: "b",
                span: #0 bytes(77..78),
            },
            Punct {
                ch: ':',
                spacing: Alone,
                span: #0 bytes(78..79),
            },
            Ident {
                ident: "i32",
                span: #0 bytes(79..82),
            },
        ],
        span: #0 bytes(69..83),
    },
    Punct {
        ch: '-',
        spacing: Joint,
        span: #0 bytes(84..85),
    },
    Punct {
        ch: '>',
        spacing: Alone,
        span: #0 bytes(85..86),
    },
    Ident {
        ident: "i32",
        span: #0 bytes(87..90),
    },
    Group {
        delimiter: Brace,
        stream: TokenStream [
            Ident {
                ident: "a",
                span: #0 bytes(97..98),
            },
            Punct {
                ch: '+',
                spacing: Alone,
                span: #0 bytes(99..100),
            },
            Ident {
                ident: "b",
                span: #0 bytes(101..102),
            },
        ],
        span: #0 bytes(91..104),
    },
]
```
从上面打印的信息中，可以知道：
- TokenStream以树形结构的数据组织，表达了⽤户源代码中各个语⾔元素的类型以及相互之间的关系。
- 每个语⾔元素都有⼀个span属性，记录了这个元素在⽤户源代码中的位置。
- 不同类型的节点，有各⾃独有的属性。可以去 syn 包里查看各个类型的注释。
- Ident类型表示的是⼀个标识符，变量名、函数名等等都是标识符。
- TokenStream⾥⾯的信息，是没有语义信息的，⽐如在上⾯的例⼦中，路径表达式中的双冒号::被拆分为两个独⽴的冒号对待，TokenStream并没有把它们识别为路径表达式，同样，它也不区分这个冒号是出现在⼀个引⽤路径中，还是⽤来表示数据类型。
- 针对attr属性⽽⾔，其中不包括宏⾃⼰的名称的标识符，它包含的仅仅是传递给这个过程宏的参数的信息。

总结：所谓的过程宏，就是我们可以⾃⼰修改上⾯的 item 变量中的值，从⽽等价
于加⼯原始输⼊代码，最后将加⼯后的代码返回给编译器即可。

补充学习资料：[过程宏作者提供的 Workshop](https://github.com/dtolnay/proc-macro-workshop)

### 三种过程宏
可以使用 cargo-expand 展开一个宏，方便阅读和调试。
```bash
# 安装方法
cargo install cargo-expand
# 使用方法
cargo expand --bin hello_macro
```

#### 派生宏
假设我们有一个特征 HelloMacro，现在有两种方式让用户使用它：
1. 为每个类型手动实现该特征，就像之前特征章节所做的
2. 使用过程宏来统一实现该特征，这样用户只需要对类型进行标记即可：`#[derive(HelloMacro)]`

以上两种方式并没有孰优孰劣，主要在于不同的类型是否可以使用同样的默认特征实现，如果可以，那过程宏的方式可以帮我们减少很多代码实现。不过为了实现这种功能，我们还需要创建相应的过程宏才行。
```rust,name=lib.rs
extern crate proc_macro;

use proc_macro::TokenStream;
use quote::quote;
use syn;
use syn::DeriveInput;

#[proc_macro_derive(HelloMacro)]
pub fn hello_macro_derive(input: TokenStream) -> TokenStream {
    // 基于 input 构建 AST 语法树
    // DeriveInput 包含被注解项的所有信息（名称、泛型参数、字段等）
    let ast:DeriveInput = syn::parse(input).unwrap();
    // 构建特征实现代码
    impl_hello_macro(&ast)
}

fn impl_hello_macro(ast: &syn::DeriveInput) -> TokenStream {
    // 从AST中提取结构体或枚举的名称
    let name = &ast.ident;
    let gen = quote! {
        // 为 #name 实现 HelloMacro 特征
        impl HelloMacro for #name {
            fn hello_macro() {
                // stringify!(#name): 将标识符转换为字符串字面量，而不是求值
                println!("Hello, Macro! My name is {}!", stringify!(#name));
            }
        }
    };
    gen.into()
}
```
可以看到，他只有一个参数，是没有属性的。
```rust
// 使用示例
// 假设HelloMacro特征在某处定义
trait HelloMacro {
    fn hello_macro();
}
// 用户代码
#[derive(HelloMacro)]
struct MyType;
fn main() {
    MyType::hello_macro(); // 输出: "Hello, Macro! My name is MyType!"
}
```
总结：派生宏自动为类型实现特征，避免用户手动为每种类型编写重复代码。

派生宏只能用在 struct/enum/union 上，多数时候是用在结构体上。

#### 属性宏
和派生宏相比，属性宏允许我们定义自己的属性，并且可以用于包括函数在内的多种类型。

假设我们在开发一个 web 框架，当用户通过 HTTP GET 请求访问`/`根路径时，使用 index 函数为其提供服务:
```rust
#[route(GET, "/")]
fn index() {
```
这里的 #[route] 属性就是一个过程宏，它的定义函数大概如下：
```rust
#[proc_macro_attribute]
pub fn route(attr: TokenStream, item: TokenStream) -> TokenStream {
```
与 derive 宏不同，类属性宏的定义函数有两个参数：
- 第一个参数时用于说明属性包含的内容，即括号内的`Get, "/"`部分
- 第二个是属性所标注的类型项，在这里是`fn index() {...}`。注意，函数体也被包含其中。

除此之外，类属性宏跟 derive 宏的工作方式并无区别：创建一个包，类型是 proc-macro，接着实现一个函数用于生成想要的代码。

#### 函数宏
类函数宏可以让我们定义像函数那样调用的宏，和声明宏的区别在于，函数宏并不是模式匹配的形式，而是过程宏的形式。过程宏使用起来更加灵活。

假设我们需要对 SQL 语句进行解析并检查其正确性，就可以定义这样一个宏：
```rust
#[proc_macro]
pub fn sql(input: TokenStream) -> TokenStream {
```
而使用形式则类似于函数调用:
```rust
let sql = sql!(SELECT * FROM posts WHERE id=1);
```

## 总结
虽然 Rust 中的宏很强大，但是它并不应该成为我们的常规武器，原因是它会影响 Rust 代码的可读性和可维护性。

***

[^1]: 从这两篇文章可以看出来，圣经就是 rust book 的说人话版本
[^2]: 图源 [张汉东](https://github.com/ZhangHanDong/inviting-rust)
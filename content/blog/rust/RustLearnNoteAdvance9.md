+++
title = "Rust 进阶学习笔记（九）：强类型特性"
slug = "rust_learn_note_adv_9"
date = 2025-07-18
updated = 2025-07-21
description = "类型转换与自动类型转换，类型别名，不定长类型，全局变量"
# draft = true
[taxonomies]
tags = ["Rust", "Learn"]
[extra]
pinned = false
post_listing_date = "both"
+++

本节出处：[圣经-4.3深入类型](https://course.rs/advance/into-types/intro.html) [圣经-4.7全局变量](https://course.rs/advance/global-variable.html)

没想到吧，都到高级九了，类型还能回归。因为这节难度和异步有一拼的，值得为它单开一篇。比如说之前遇到过很多自动解引用的情况，这其实就是一种类型转换。

除类型转换外，本节还有很多之前没接触过的类型特性。

***

## 类型转换
Rust 是强类型的语言，也是类型安全的语言，因此在 Rust 中进行类型转换并不容易。

### as 转换
Rust 不允许两种不同的类型进行比较。最简单的解决办法就是用`as`操作符来进行转换。
```rust
fn main() {
    let a: i32 = 10;
    let b: u16 = 100;
    // 报错， 无法直接比较
    // if a < b {
    if a < (b as i32) {
      println!("Ten is less than one hundred.");
    }
    // 常见转换
    let a = 3.1 as i8;
    let b = 100_i8 as i32;
    let c = 'a' as u8; // 将字符'a'转换为整数，97

   println!("{}, {}, {}", a, b, c)
}
```
每个类型能表达的数据范围不同，如果把范围较大的类型转换成较小的类型，会造成数据失准（实际上就是粗暴地把二进制高位截断了只保留地位），因此只能把范围较小的类型转换成较大的类型。可以这样查看数据类型的最大值：
```rust
let a = i8::MAX;
println!("{}",a);
```
除了类型的转换，`as`的另一个应用是内存地址和指针的转换：
```rust
let mut values: [i32; 2] = [1, 2];
let p1: *mut i32 = values.as_mut_ptr();
let first_address = p1 as usize; // 将p1内存地址转换为一个整数
let second_address = first_address + 4; // 4 == std::mem::size_of::<i32>()，i32类型占用4个字节，因此将内存地址 + 4
let p2 = second_address as *mut i32; // 访问该地址指向的下一个整数p2
unsafe {
    *p2 += 1;
}
assert_eq!(values[1], 3);
```

### TryInto 转换
`TryInto`是一种内置转换的替代方案，显然这是一个特质。因此，它更灵活更自由，能够让开发者拥有对类型转换的完全控制。
```rust
// 要使用一个特质的方法的时候，也需要把特质引入到当前的作用域中
// 实际上，这个方法可以不 use，因为会被 std::prelude 自动引入
// 这是预引入列表 https://doc.rust-lang.org/std/prelude/index.html
use std::convert::TryInto;

fn main() {
    let a: u8 = 10;
    let b: u16 = 1500;
    // unwrap 方法在发现错误时，会直接调用 panic 导致程序的崩溃退出
    // 这里只是演示，实际项目中应该进行处理
    let b_: u8 = b.try_into().unwrap();

    if a < b_ {
        println!("Ten is less than one hundred.");
    }
}
```
`try_into`会尝试进行一次转换，并返回一个`Result`，此时就可以对其进行相应的错误处理。`try_into`转换最常见的应用就是捕获大类型向小类型转换时导致的溢出错误：
```rust
fn main() {
    let b: i16 = 1500;

    let b_: u8 = match b.try_into() {
        Ok(b1) => b1,
        Err(e) => {
            println!("{:?}", e.to_string()); // out of range integral type conversion attempted
            0
        }
    };
}
```

### 通用转换
`as`和`try_into`只能用于数值类型的转换。对于一些复杂类型，如结构体，当然可以循环转换其中的所有字段。当然，Rust 还有一些办法进行通用转换。

#### 强制类型转换
本小节随便看看就好，不需要记住。

Rust 不提供原生类型的隐式转换，但部分类型可以进行一些隐式强制转换。这往往需要一些自动强转点（coercion sites），最典型的就是所需类型已给出或可以自动推导的（不是推断）的场景。例如：
- let 语句显式指定类型：`let _: &i8 = &mut 42;`
- 静态和常量项的声明
- 函数的参数，实参可以被自动强制转换成形参
- 实例化结构体、枚举或联合体
- 函数 return 的结果

自动强转允许发生在下列类型之间：
- 反射性场景，如 T 到 U，如果 T 是 U 的一个子类型
- 传递性场景，如 T_1 到 T_3，当 T_1 可自动强转到 T_2、同时 T_2 又能自动强转到 T_3（注意这个还没有得到完全支持）
- &mut T 到 &T
- *mut T 到 *const T
- &T 到 *const T
- &mut T 到 *mut T
- &T 或 &mut T 到 &U（如果 T 实现了 Deref<Target = U>）
- &mut T 到 &mut U（如果 T 实现了 DerefMut<Target = U>）
- 类型构造器 TyCtor(T) 到 TyCtor(U)（TyCtor(T) 是`&T/&mut T/*const T/*mut T/Box<T>`）
- ! 到任意 T
- 非捕获闭包到函数指针

还存在非固定 size 的自动转换：
- [T; n] 到 [T]
- T 到 dyn U，（当 T 实现 U + Sized, 并且 U 是对象安全的时）
- Foo<..., T, ...> 到 Foo<..., U, ...> 要求
  - Foo 是一个结构体。
  - T 实现了`Unsize<U>`。
  - Foo 的最后一个字段是和 T 相关的类型。
  - 如果这最后一个字段是类型`Bar<T>`，那么`Bar<T>`实现了`Unsized<Bar<U>>`。
  - T 不是任何其他字段的类型的一部分。
  - T 实现了`Unsize<U>`或`CoerceUnsized<Foo<U>>`，且类型`Foo<T>`可以实现`CoerceUnsized<Foo<U>>`

注意匹配特质时不会做除方法外的强制转换（T 到 U 不代表 impl T 到 impl U）。

在某些上下文中，编译器必须将多个类型强制放在一起，以尝试找到最通用的类型，这就是最小上界自动强转（Least Upper Bound, LUB）。下面是一个 LUB 的例子：
```rust
#![allow(unused)]
fn main() {
    // LUB 共支持五种情况：
    let (a, b, c) = (0, 1, 2);
    // if分支的情况
    let bar = if true {
        a
    } else if false {
        b
    } else {
        c
    };

    // 匹配臂的情况
    let baw = match 42 {
        0 => a,
        1 => b,
        _ => c,
    };

    // 数组元素的情况
    let bax = [a, b, c];

    // 多个返回项语句的闭包的情况
    let clo = || {
        if true {
            a
        } else if false {
            b
        } else {
            c
        }
    };
    let baz = clo();

    // 检查带有多个返回语句的函数的情况
    fn foo() -> i32 {
        let (a, b, c) = (0, 1, 2);
        match 42 {
            0 => a,
            1 => b,
            _ => c,
        }
    }
}
```

#### 点操作符
方法调用会用到点操作符，这个过程中会发生大量类型转换，比如自动引用、自动解引用、强制类型转换等。

假设一个 value 拥有类型 T，然后有一个方法 foo，编译器会按如下顺序尝试进行类型转换：
1. 值方法调用：编译器检查它是否可以直接调用 T::foo(value)
2. 引用方法调用：如果方法类型错误或者特质没有针对 Self 进行实现（再次强调，特质不能强制转换）编译器会尝试增加自动引用，如`<&T>::foo(value)`和`<&mut T>::foo(value)`
3. 解引用方法调用：编译器会使用`Deref`特质解引用 T ，然后再进行尝试。例如`T: Deref<Target = U>`，编译器会用 U 尝试
4. 如果 T 是定长类型，编译器会尝试将 T 转为不定长类型，如 [i32; 2] 到 [i32]
5. 如果以上都不行，编译器会报错。

举个例子：
```rust
// 首先，array[0] 只是Index特质的语法糖：编译器会将 array[0] 转换为 array.index(0) 调用（当然在调用之前，编译器会先检查 array 是否实现了 Index 特质）
// 接着，编译器检查 Rc<Box<[T; 3]>> 是否有实现 Index 特质，结果是否，不仅如此，&Rc<Box<[T; 3]>> 与 &mut Rc<Box<[T; 3]>> 也没有实现。
// 上面的都不能工作，编译器开始对 Rc<Box<[T; 3]>> 进行解引用，把它转变成 Box<[T; 3]>
// 此时继续对 Box<[T; 3]> 进行上面的操作 ：Box<[T; 3]>， &Box<[T; 3]>，和 &mut Box<[T; 3]> 都没有实现 Index 特质，所以编译器开始对 Box<[T; 3]> 进行解引用，然后我们得到了 [T; 3]
// [T; 3] 以及它的各种引用都没有实现 Index 索引（只有数组切片才可以），它也不能再进行解引用，因此编译器只能将定长转为不定长，因此 [T; 3] 被转换成 [T]，也就是数组切片，它实现了 Index 特质，因此最终我们可以通过 index 方法访问到对应的元素。
let array: Rc<Box<[T; 3]>> = ...;
let first_entry = array[0];
```
再看另一个例子：
```rust
// fn do_stuff<T: Clone>(value: &T) { // 值引用直接生效
fn do_stuff<T>(value: &T) {
    // 首先通过值方法调用就不再可行，因为 T 没有实现 Clone 特质，也就无法调用 T 的 clone 方法。
    // 接着编译器尝试引用方法调用，此时 T 变成 &T，在这种情况下， clone 方法的签名如下： fn clone(&&T) -> &T，现在对 value 进行了引用。
    // 编译器发现 &T 实现了 Clone 类型（所有的引用类型都可以被复制，因为其实就是复制一份地址），因此可以推出 cloned 也是 &T 类型。
    // 最终复制出一份引用指针
    let cloned = value.clone();
}
```
再来看看同一个方法在参数 T 不同时候的行为：
```rust
// 一个复杂类型能否派生 Clone，需要它内部的所有子类型都能进行 Clone。
// 因此 Container<T>(Arc<T>) 是否实现 Clone 的关键在于 T 类型是否实现了 Clone 特质。
#[derive(Clone)]
// 派生宏可能会生成如下代码：
// impl<T> Clone for Container<T> where T: Clone {
//     fn clone(&self) -> Self {
//         Self(Arc::clone(&self.0))
//     }
// }
struct Container<T>(Arc<T>);

fn clone_containers<T>(foo: &Container<i32>, bar: &Container<T>) {
    // Container<i32> 实现了 Clone 特质，因此编译器可以直接进行值方法调用，此时相当于直接调用 foo.clone，其中 clone 的函数签名是 fn clone(&T) -> T，
    // 由此可以看出 foo_cloned 的类型是 Container<i32>。
    let foo_cloned = foo.clone();
    // bar_cloned 的类型是 &Container<T>
    // 派生 Clone 能实现的根本是 T 实现了 Clone 特质：where T: Clone，于是 Container<T> 就没有实现 Clone 特质。
    // 编译器接着会去尝试引用方法调用，此时 &Container<T> 引用实现了 Clone，最终可以得出 bar_cloned 的类型是 &Container<T>。
    let bar_cloned = bar.clone();

    // 可以为 Container<T> 手动实现 Clone 特质
    impl<T> Clone for Container<T> {
        fn clone(&self) -> Self {
            Self(Arc::clone(&self.0))
        }
    }
}
```
> 果然够难啊。。。好歹还看懂了，但是很难记住。后面还有根本看不懂的

#### 逆天的“变形”
这里的“变形”是指 transmute，这可以说是 Rust 最不安全的操作了。`mem::transmute<T, U>`将类型 T 直接转成类型 U，只要保证 T 和 U 在内存中的字节数大小相同即可。这很显然非常不安全：
- 转换后创建一个任意类型的实例会造成无法预测的混乱，例如把一个 i32 转换成 bool，他表示的内容根本没法知道是什么
- 变形后会有一个重载的返回类型，即使你没有指定返回类型，为了满足类型推导的需求，依然会产生千奇百怪的类型
- 将 & 变形为 &mut 是未定义的行为
- 变形为一个未指定生命周期的引用会导致无界生命周期[^1]
- 在复合类型之间互相变换时，需要保证它们的排列布局是一模一样的否则字段会得到不可预期的值

`mem::transmute_copy<T, U>`是一个更加离谱的操作：它从 T 类型中拷贝出 U 类型所需的字节数，然后转换成 U。如果 U 的尺寸比 T 大，会是一个未定义行为。可以发现，连内存大小检查都不要了。

为什么会有变形？来看看他们的应用：
```rust
// 裸指针转换为函数指针
fn foo() -> i32 {
    0
}

let pointer = foo as *const ();
let function = unsafe { 
    // 将裸指针转换为函数指针
    std::mem::transmute::<*const (), fn() -> i32>(pointer) 
};
assert_eq!(function(), 0);
```
```rust
// 延长生命周期，或者缩短一个静态生命周期寿命
struct R<'a>(&'a i32);

// 将 'b 生命周期延长至 'static 生命周期
unsafe fn extend_lifetime<'b>(r: R<'b>) -> R<'static> {
    std::mem::transmute::<R<'b>, R<'static>>(r)
}

// 将 'static 生命周期缩短至 'c 生命周期
unsafe fn shorten_invariant_lifetime<'b, 'c>(r: &'b mut R<'static>) -> &'b mut R<'c> {
    std::mem::transmute::<&'b mut R<'static>, &'b mut R<'c>>(r)
}
```
这是非常先进的用法，但也很不安全。

## 自定义类型

### 新类型
所谓新类型（newtype）就是使用元组结构体的方式将已有的类型包裹起来，比如`struct A(u32)`。自定义类型可以给出更有意义和可读性的类型名，并隐藏内部的类型细节。来看几个应用：
```rust
// 为外部类型实现外部特质
use std::fmt;

struct Wrapper(Vec<String>);

impl fmt::Display for Wrapper {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "[{}]", self.0.join(", "))
    }
}

fn main() {
    let w = Wrapper(vec![String::from("hello"), String::from("world")]);
    // 实现对 Vec 动态数组的格式化输出（替代 Debug）
    println!("w = {}", w);
}
```
```rust
// 更好的可读性及类型异化
use std::ops::Add;
use std::fmt;

// 为 u32 实现 Display 和 Add 特质
struct Meters(u32);
impl fmt::Display for Meters {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "目标地点距离你{}米", self.0)
    }
}

impl Add for Meters {
    type Output = Self;

    fn add(self, other: Meters) -> Self {
        Self(self.0 + other.0)
    }
}

fn main() {
    // 提高可读性（可能会为此增大代码量）
    // 另外的优势：传一个其它的类型，例如 struct MilliMeters(u32);，该代码将无法编译。
    // 尽管 Meters 和 MilliMeters 都是对 u32 类型的简单包装，但是它们是不同的类型
    let d = calculate_distance(Meters(10), Meters(20));
    println!("{}", d);
}

fn calculate_distance(d1: Meters, d2: Meters) -> Meters {
    d1 + d2
}
```
```rust
// 隐藏内部类型的细节
struct Meters(u32);

fn main() {
    let i: u32 = 2;
    assert_eq!(i.pow(2), 4);
    // 把类型传给了用户，但是又不想用户调用该类型的方法
    let n = Meters(i);
    // 下面的代码将报错，因为`Meters`类型上没有`pow`方法
    // assert_eq!(n.pow(2), 4);
    // 实际上用 n.0.pow(2) 是能调用的
}
```

### 类型别名
类型别名（Type Alias）形如`type A = u32`。和 newtype 不同，类型别名并不是一个独立的全新的类型，而是某一个类型的别名，只是起到增强可读性和减少模版代码使用的作用，并不会改变编译器的认知。
```rust
// 增强可读性
type Meters = u32;

let x: u32 = 5;
let y: Meters = 5;
// 能正常编译和输出
println!("x + y = {}", x + y);
```
```rust
type Thunk = Box<dyn Fn() + Send + 'static>;

// f 的类型太长，起个别名会更加优美
let f: Thunk = Box::new(|| println!("hi"));

fn takes_long_type(f: Thunk) {
    // --snip--
}

fn returns_long_type() -> Thunk {
    // --snip--
}
```
```rust
// 简化 Result 枚举
// std::io 库定义了自己的 Error 类型：std::io::Error
// 可以把该错误对用户隐藏起来，只在内部使用
type Result<T> = std::result::Result<T, std::io::Error>;
// 这是类型别名在标准库中最常见的作用
// 由于它只是别名，因此我们可以用它来调用真实类型的所有方法，甚至包括 ? 符号
```

### 永不返回
`!`用来说明一个函数永不返回任何值。当然也可以理解成所有类型的子类型。
```rust
fn main() {
    let i = 2;
    let v = match i {
       0..=3 => i,
       // _ => println!("不合规定的值:{}", i) // `match` arms have incompatible types
       _ => panic!("不合规定的值:{}", i)
       // panic 的返回值是 !，代表它决不会返回任何值
       // 既然没有任何返回值，那自然不会存在分支类型不匹配的情况。
    };
}
```

## 不定长类型
Rust 的类型从编译器何时能获知类型大小的角度出发，可以分成两类:
- 定长（sized）类型：这些类型的大小在编译时是已知的
- 不定长（unsized）类型：大小只有到了程序运行时才能动态获知，又被称之为动态大小类型（DST，dynamically sized types）

### 动态大小类型
动态大小类型就是指编译器无法在编译期得知该类型值的大小，只有到了程序运行时，才能动态获知的类型。

集合并不是动态大小类型。集合只是把底层数据保存在堆上，在栈中还存有一个引用类型，该引用包含了集合的内存地址、元素数目、分配空间信息，栈上的引用类型是固定大小的，因此它们依然是固定大小的类型。可以把集合类型理解成一种智能指针。

Rust 中常见的 DST 类型有: str、[T]、dyn Trait，它们都无法单独被使用，必须要通过引用或者 Box 来间接使用 。

#### 动态大小数组
```rust
fn my_function(n: usize) {
    let array = [123; n];
}
```
n 在编译期无法得知，而数组类型的一个组成部分就是长度，长度变为动态的，自然类型就变成了 unsized。因此这段代码无法通过编译

#### str 及其他切片
切片也无法直接创建。因为底层的切片长度是可以动态变化的，而编译器无法在编译期得知它的具体的长度，因此该类型无法被分配在栈上，只能分配在堆上。

而切片引用是固定大小的。将动态数据固定化的秘诀就是使用引用指向这些动态数据，然后在引用中存储相关的内存位置、长度等信息。

#### 特质对象
特质对象无法直接使用。
```rust
fn foobar_1(thing: &dyn MyThing) {}     // OK
fn foobar_2(thing: Box<dyn MyThing>) {} // OK
fn foobar_3(thing: MyThing) {}          // ERROR!
```

### 定长特质
所有在编译时就能知道其大小的类型，都会自动实现`Sized`特质。编译器也会给泛型类型 T 自动加上`Sized`特质约束。

每一个特质都是一个可以通过名称来引用的动态大小类型。因此如果想把特质作为具体的类型来传递给函数，必须将其转换成一个特质对象（`&dyn Trait`/` Box<dyn Trait>`/`Rc<dyn Trait>`...）。

如果想在泛型函数中使用动态数据类型，需要使用`?Sized`特质。`?Sized`特质用于表明类型 T 既有可能是固定大小的类型，也可能是动态大小的类型。此时函数参数类型从 T 变成了 &T，因为 T 可能是动态大小的，因此需要用一个固定大小的引用来包裹它。

### Box<str>
如何把一个动态大小类型转换成固定大小的类型：使用引用指向这些动态数据，然后在引用中存储相关的内存位置、长度等信息。
```rust
fn main() {
    // 主动转换成 str 的方式不可行
    // let s1: Box<str> = Box::new("Hello there!" as str); // the size for values of type `str` cannot be known at compilation time
    // 对于特质对象，编译器无需知道它具体是什么类型，只要知道它能调用哪几个方法即可，因此编译器帮我们实现了剩下的一切。
    // 可以让编译器来推断，只要告诉它我们需要的类型即可。
    let s1: Box<str> = "Hello there!".into();
}
```

## 整数到枚举的转换
从枚举到整数的转换很容易，反之则不然。但这个需求还是常见的。

假设有一个枚举类型，需要从外面传入一个整数，用于控制后续的流程走向，此时就需要用整数去匹配相应的枚举。这种使用方式在 C 语言中很常见，尤其是和 thrift 协议的交互时。但是 Rust 无法使用 C 语言的写法：
```rust
enum MyEnum {
    A = 1,
    B,
    C,
}

fn main() {
    // 将枚举转换成整数，顺利通过
    let x = MyEnum::C as i32;

    // 将整数转换为枚举，失败 MyEnum::A => {} mismatched types, expected i32, found enum MyEnum。
    match x {
        MyEnum::A => {}
        MyEnum::B => {}
        MyEnum::C => {}
        _ => {}
    }
}
```
所以需要想一些其他办法。

### 三方库
```toml,name=Cargo.toml
[dependencies]
num-traits = "0.2.14"
num-derive = "0.3.3"
# 还可以用 num_enums 库
```
```rust
use num_derive::FromPrimitive;
use num_traits::FromPrimitive;

#[derive(FromPrimitive)]
enum MyEnum {
    A = 1,
    B,
    C,
}

fn main() {
    let x = 2;

    match FromPrimitive::from_i32(x) {
        Some(MyEnum::A) => println!("Got A"),
        Some(MyEnum::B) => println!("Got B"),
        Some(MyEnum::C) => println!("Got C"),
        None            => println!("Couldn't convert {}", x),
    }
}
```

### TryFrom
```rust
use std::convert::TryFrom;

// TryFrom 特质可以来做转换
impl TryFrom<i32> for MyEnum {
    type Error = ();

    fn try_from(v: i32) -> Result<Self, Self::Error> {
        match v {
            x if x == MyEnum::A as i32 => Ok(MyEnum::A),
            x if x == MyEnum::B as i32 => Ok(MyEnum::B),
            x if x == MyEnum::C as i32 => Ok(MyEnum::C),
            _ => Err(()),
        }
    }
}

use std::convert::TryInto;

fn main() {
    let x = MyEnum::C as i32;

    // TryInto来实现转换
    match x.try_into() {
        Ok(MyEnum::A) => println!("a"),
        Ok(MyEnum::B) => println!("b"),
        Ok(MyEnum::C) => println!("c"),
        Err(_) => eprintln!("unknown number"),
    }
}
```
```rust
// 使用宏来自动根据枚举的定义实现 TryFrom 特质
#[macro_export]
macro_rules! back_to_enum {
    ($(#[$meta:meta])* $vis:vis enum $name:ident {
        $($(#[$vmeta:meta])* $vname:ident $(= $val:expr)?,)*
    }) => {
        $(#[$meta])*
        $vis enum $name {
            $($(#[$vmeta])* $vname $(= $val)?,)*
        }

        impl std::convert::TryFrom<i32> for $name {
            type Error = ();

            fn try_from(v: i32) -> Result<Self, Self::Error> {
                match v {
                    $(x if x == $name::$vname as i32 => Ok($name::$vname),)*
                    _ => Err(()),
                }
            }
        }
    }
}

back_to_enum! {
    enum MyEnum {
        A = 1,
        B,
        C,
    }
}
```

### transmute
transmute 显然也可以用来做这个。从下面的例子可以看到，transmute 如果真用好了还是很有存在价值的。
```rust
// #[repr(..)] 控制底层类型的大小，防止类型对不齐
#[repr(i32)]
enum MyEnum {
    A = 1, B, C
}

fn main() {
    let x = MyEnum::C;
    let y = x as i32;
    let z: MyEnum = unsafe { std::mem::transmute(y) };

    // match the enum that came from an int
    match z {
        MyEnum::A => { println!("Found A"); }
        MyEnum::B => { println!("Found B"); }
        MyEnum::C => { println!("Found C"); }
    }
}
```

## 全局变量
全局变量也是一个常见的需求。全局变量的生命周期肯定是`'static`，但是不代表它需要用 static 来声明。

### 编译期初始化











{{ admonition(type="warning", title="注意", text="施工中") }}

***
[^1]: 无界（unbound）生命周期通常是不安全代码凭空产生的引用或生命周期，常常是解引用一个裸指针时产生的，因为输入参数根本就没有这个生命周期
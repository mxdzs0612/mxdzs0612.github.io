+++
title = "Rust 进阶学习笔记（六）：Unsafe"
slug = "rust_learn_note_adv_6"
date = 2025-07-11
updated = 2025-07-12
description = "为什么要有unsafe，unsafe能做什么，怎么用unsafe"
# draft = true
[taxonomies]
tags = ["Rust", "Learn"]
[extra]
pinned = false
post_listing_date = "both"
+++

本节出处：[圣经 Unsafe](https://course.rs/advance/unsafe/intro.html)

***

几乎每个语言都有`unsafe`关键字，但 Rust 语言使用`unsafe`的原因可能与其它编程语言还有所不同。

`unsafe`存在的主要是 Rust 的静态检查太强且保守，这就会导致当编译器在分析代码时，一些正确代码会因为编译器无法分析出它的所有正确性，结果将这段代码拒绝，导致编译错误。尤其是涉及到所有权规则时，编译器检查是很难绕过的。这种时候就可以使用`unsafe`，此时需要程序员承担编译器的职责，对代码正确性负责。

此外，一些系统底层的编程本身就是不安全的，如果没有`unsafe`的能力，Rust 的性能会大受影响。

## 不安全？信任我！
使用`unsafe`非常简单，只需要将对应的代码块标记一下即可:
```rust
fn main() {
    let mut num = 5;
    // r1 是一个裸指针（raw pointer），由于它具有破坏 Rust 内存安全的潜力，因此只能在`unsafe`代码块中使用，如果去掉 unsafe {}，编译器会立刻报错。
    let r1 = &num as *const i32;
    unsafe {
        println!("r1 is: {}", *r1);
    }
}
```
虽然名字叫不安全，但是 Rust 依然提供了强大的安全支撑。`unsafe`并不能绕过 Rust 的借用检查，也不能关闭任何 Rust 的安全检查规则，只有使用`unsafe`提供的能力时，编译器才不会进行内存安全上的检查。`unsafe`在 Rust 的标准库中就大量存在，所以这个命名其实有待商榷，更应该叫“信任我”或“别搞了，让我通过！”😆

`unsafe`的使用原则是：没必要用时，就不要用，当有必要用时，就大胆用，但是尽量控制好边界，让`unsafe`的范围尽可能小。

`unsafe`能帮助我们大幅降低代码实现的成本。并且，一旦出现内存访问出错，我们很容易意识到是在`unsafe`代码块中出错了。这就要求我们控制好`unsafe`的边界，越小的`unsafe`代码块就越好调试。此外，另一个很常用的方式就是在`unsafe`代码块外包裹一层 safe 的 API。

## 五种兵器
本节介绍`unsafe`赋予的 5 种超能力，这些能力在安全的 Rust 代码中是无法获取的。

### 解引用裸指针
在智能指针一节中提到过裸指针(raw pointer，又称原生指针)。裸指针在功能上跟引用类似，也需要显式地注明可变性。但是又和引用有所不同。裸指针使用`*`标记: *const T 和 *mut T，它们分别代表了不可变和可变。我们已经知道`*`可以用于解引用，但是在裸指针中，`*`只是类型名称的一部分，并没有解引用的含义。

裸指针可以绕过 Rust 的借用规则，可以同时拥有一个数据的可变、不可变指针，甚至还能拥有多个可变的指针。因此，裸指针并不能保证指向合法的内存，甚至可以为 null，也没有实现任何自动的回收。这和 C 语言的指针是非常像的：以牺牲安全性为前提，换来更好的性能以及和底层硬件或其他语言代码直接的交互。此时会出现数据竞争问题，需要小心处理。

#### 基于引用创建裸指针
```rust
let mut num = 5;

let r1 = &num as *const i32;
let r2 = &mut num as *mut i32;
```
`as`可以用于强制类型转换，这里将引用强制转换为对应的裸指针。

基于现有的引用来创建裸指针并没有不安全，只有解引用才是不安全的行为。
```rust
fn main() {
    let mut num = 5;

    let r1 = &num as *const i32;

    unsafe {
        println!("r1 is: {}", *r1);
    }
}
```

#### 基于内存地址创建裸指针
```rust
let address = 0x012345usize;
let r = address as *const i32;
```
这里基于一个内存地址来创建裸指针，这是非常危险的行为，因为指定的内存地址有可能存在值，也有可能没有，就算有值，也大概率不是想要的值。虽然这么写是可行的，但很可能会导致实际没有出现内存访问（被编译器优化）甚至段错误，几乎没有这么做的理由。

需要内存地址时，可以考虑先取地址再使用。
```rust
use std::{slice::from_raw_parts, str::from_utf8_unchecked};

// 获取字符串的内存地址和长度
fn get_memory_location() -> (usize, usize) {
  let string = "Hello World!";
  let pointer = string.as_ptr() as usize;
  let length = string.len();
  (pointer, length)
}

// 在指定的内存地址读取字符串
fn get_str_at_location(pointer: usize, length: usize) -> &'static str {
  unsafe { from_utf8_unchecked(from_raw_parts(pointer as *const u8, length)) }
}

fn main() {
  let (pointer, length) = get_memory_location();
  let message = get_str_at_location(pointer, length);
  println!(
    "The {} bytes at 0x{:X} stored: {}",
    length, pointer, message
  );
  // 如果大家想知道为何处理裸指针需要 `unsafe`，可以试着反注释以下代码
  // let message = get_str_at_location(1000, 10);
  // 理论上应该出现段错误，不知道是不是新版优化了，程序会直接退出
}
```

#### 解引用
```rust
let a = 1;
// as 显式转换（建议使用，可以提示这是裸指针）
let b: *const i32 = &a as *const i32;
// 隐式转换
let c: *const i32 = &a;
unsafe {
    println!("{}", *c);
}
```
使用`*`可以对裸指针进行解引用，由于该指针的内存安全性并没有任何保证，因此我们需要使用`unsafe`来包裹解引用的逻辑。

#### 基于智能指针创建裸指针
```rust
let a: Box<i32> = Box::new(10);
// 需要先解引用a
let b: *const i32 = &*a;
// 使用 into_raw 来创建
let c: *const i32 = Box::into_raw(a);
```

### 调用一个 unsafe 或外部的函数
`unsafe`函数除了是使用`unsafe fn`定义的之外，从外表上来看跟普通函数并无区别。这种定义方式是为了告诉调用者：当调用此函数时，需要注意正在调用的是一个不安全的函数，因为 Rust 无法担保调用者在使用该函数时能满足它所需的一切需求。此时最好自行查阅文档，看看到底哪里不安全了。
```rust
unsafe fn dangerous() {}
fn main() {
    unsafe {
        dangerous();
    }
}
```
注意，在`unsafe`函数体中使用`unsafe`语句块是多余的行为。

实际上，标准库中有大量的安全函数，它们内部都包含了`unsafe`代码块。比如下面的`split_at_mut`:
```rust
use std::slice;

fn split_at_mut(slice: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    let len = slice.len();
    // 返回指向 slice 首地址的裸指针 *mut i32
    let ptr = slice.as_mut_ptr();
    // 通过这个断言，我们保证了裸指针一定指向了 slice 切片中的某个元素，而不是一个莫名其妙的内存地址。
    assert!(mid <= len);

    unsafe {
        (
            // 通过指针和长度来创建一个新的切片，简单来说，该切片的初始地址是 ptr，长度为 mid
            slice::from_raw_parts_mut(ptr, mid),
            // 获取第二个切片的初始地址。ptr.add(mid) 表示将指针向后移动 mid 个元素的位置。
            // 由于切片中的元素是 i32 类型，每个元素都占用了 4 个字节的内存大小，因此我们不能简单的用 ptr + mid 来作为初始地址，而应该使用 ptr + 4 * mid，这样麻烦且不安全，还得自己考虑类型的长度。
            // 所以建议使用 mid
            slice::from_raw_parts_mut(ptr.add(mid), len - mid),
        )
    }
}

fn main() {
    let mut v = vec![1, 2, 3, 4, 5, 6];

    let r = &mut v[..];

    let (a, b) = split_at_mut(r, 3);

    assert_eq!(a, &mut [1, 2, 3]);
    assert_eq!(b, &mut [4, 5, 6]);
}
```
想象一下这个场景：需要将一个数组分成两个切片，且每一个切片都要求是可变的。Rust 的借用检查器无法理解我们是分别借用了同一个切片的两个不同部分，但事实上，这种行为是没任何问题的，毕竟两个借用没有任何重叠之处。此时就只能使用`unsafe`了。

`slice::from_raw_parts_mut`使用裸指针作为参数，因此它是一个`unsafe fn`，我们在使用它时，就必须用`unsafe`语句块进行包裹，类似的，`add`也是如此。

此种情况就是使用**安全的抽象**包裹`unsafe`代码，这里的`unsafe`使用是非常安全的，因为我们从合法数据中创建了合法的指针。

如果像下面这么搞，直接会出现段错误。
```rust
use std::slice;

let address = 0x01234usize;
let r = address as *mut i32;

let slice: &[i32] = unsafe { slice::from_raw_parts_mut(r, 10000) };
println!("{:?}",slice);
```

### 访问或修改一个可变的静态变量
Rust 要求必须使用`unsafe`语句块才能访问和修改`static`变量，因为这种使用方式往往并不安全，其实编译器是对的，当在多线程中同时去修改时，会不可避免的遇到脏数据。只有在同一线程内或者不在乎数据的准确性时，才应该使用全局静态变量。对于多线程的情况，可以考虑使用原子类型。
```rust
static mut REQUEST_RECV: usize = 0;
fn main() {
   unsafe {
        REQUEST_RECV += 1;
        assert_eq!(REQUEST_RECV, 1);
   }
}
```
和常量相同，定义静态变量的时候必须赋值为在编译期就可以计算出的值（常量表达式/数学表达式），不能是运行时才能计算出的值（如函数）。

静态变量不会被内联，在整个程序中，静态变量只有一个实例，所有的引用都会指向同一个地址。存储在静态变量中的值必须要实现`Sync trait`。

### 实现一个 unsafe 特征
之所以会有`unsafe`的特征，是因为该特征至少有一个方法包含有编译器无法验证的内容。`unsafe`特征的声明很简单：
```rust
unsafe trait Foo {
    // 方法列表
}

unsafe impl Foo for i32 {
    // 实现相应的方法
}

fn main() {}
```
特征标记为`unsafe`是因为 Rust 无法验证我们的类型是否能在线程间安全的传递。`unsafe`就是用于告诉编译器，它无需操心，交给我们自己来处理即可。

### 访问 union 中的字段
`union`主要用于和 C 代码进行交互。访问`union`的字段是不安全的，因为 Rust 无法保证当前存储在`union`实例中的数据类型。
```rust
#[repr(C)]
union MyUnion {
    f1: u32,
    f2: f32,
}
```
`union`的使用方式跟结构体确实很相似，但是前者的所有字段都共享同一个存储空间，意味着往`union`的某个字段写入值，会导致其它字段的值被覆盖。

### FFI 与 extern
FFI（Foreign Function Interface）可是指和其它语言进行交互的方法，和 Java 的 JNI（Java Native Interface）是一个意思。

如果我们需要使用某个库，但是它是由其它语言编写的，那么往往只有两个选择：
- 对该库进行重写或者移植
- 使用 FFI

很多时候，并没有那么多时间去重写，因此 FFI 就成了最佳选择。(除此之外还有一个办法可以解决跨语言调用的问题，那就是将其作为一个独立的服务，然后使用网络调用的方式去访问，HTTP，gRPC 都可以。)

```rust
extern "C" {
    fn abs(input: i32) -> i32;
}

fn main() {
    unsafe {
        println!("Absolute value of -3 according to C: {}", abs(-3));
    }
}
```
C 语言的代码定义在了`extern`代码块中， 而`extern`必须使用`unsafe`才能进行进行调用，原因在于其它语言的代码并不会强制执行 Rust 的规则，因此 Rust 无法对这些代码进行检查，最终还是要靠开发者自己来保证代码的正确性和程序的安全性。

在`extern "C"`代码块中，我们列出了想要调用的外部函数的签名。其中 "C" 定义了外部函数所使用的应用二进制接口ABI (Application Binary Interface)：ABI 定义了如何在汇编层面来调用该函数。在所有 ABI 中，C 语言的是最常见的。

当然反过来也是可以的。我们可以使用`extern`来创建一个接口，其它语言可以通过该接口来调用相关的 Rust 函数。但是此处的语法与之前有所不同，之前用的是语句块，而这里是在函数定义时加上`extern`关键字，当然，别忘了指定相应的 ABI：
```rust
#[no_mangle]
pub extern "C" fn call_from_c() {
    println!("Just called a Rust function from C!");
}
```
上面的代码可以让`call_from_c`函数被 C 语言的代码调用，当然，前提是将其编译成一个共享库，然后链接到 C 语言中。

注解`#[no_mangle]`用于告诉 Rust 编译器：不要乱改函数的名称。“Mangling”的定义是：当 Rust 因为编译需要去修改函数的名称，例如为了让名称包含更多的信息，这样其它的编译部分就能从该名称获取相应的信息，这种修改会导致函数名变得相当不可读。
因此，为了让 Rust 函数能顺利被其它语言调用，我们必须要禁止掉该功能。

## 内联汇编
内联汇编（Inline assembly）的意思是将汇编语言内嵌在高级语言的代码中，Rust 是支持内联汇编的。

{{ admonition(type="danger", title="危！", text="汇编忘光了，这章看的一头雾水，哪天理解了再回来补吧") }}
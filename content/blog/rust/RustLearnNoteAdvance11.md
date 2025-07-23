+++
title = "Rust 进阶学习笔记（十一）：自动化测试"
slug = "rust_learn_note_adv_11"
date = 2025-07-23
updated = 2025-07-24
description = "单元测试，集成测试，CICD，基准测试，文档测试"
# draft = true
[taxonomies]
tags = ["Rust", "Learn"]
[extra]
pinned = false
post_listing_date = "both"
+++

本节出处：[圣经-8自动化测试](https://course.rs/test/intro.html)
 [圣经-2.13文档测试](https://course.rs/basic/comment.html#%E6%96%87%E6%A1%A3%E6%B5%8B%E8%AF%95doc-test)

测试可以有效的发现程序存在的缺陷，但是它却无法证明程序不存在缺陷。尽管如此，对于程序开发而言，测试可以说依然是至关重要的一环，虽然它无法完全消除所有的 Bug，但是依然可以在某种程度上保证程序的正确性。

## 编写测试
测试是通过函数的方式实现的，它可以用于验证被测试代码的正确性。测试函数往往依次执行以下三种行为：
1. 设置所需的数据或状态
2. 运行想要测试的代码
3. 判断（assert）返回的结果是否符合预期

### 测试函数
使用 Cargo 创建一个 lib 类型的包时，它会为我们自动生成一个测试模块。
```rust,name=src/lib.rs
#[cfg(test)]
mod tests {
    // 默认自带的测试
    #[test]
    fn it_works() {
        // 该宏用于对结果进行断言
        assert_eq!(2 + 2, 4);
    }
}
```
tests 就是一个测试模块，it_works 则是测试函数。当然，在测试模块中，还可以定义非测试函数，用于设置环境或执行一些通用操作。

测试函数需要使用 test 属性进行标注，以获取测试特性。经过 test 标记的函数就可以被测试执行器发现并运行。

### 运行测试
使用 cargo test 命令来运行项目中的所有测试。
```sh
$ cargo new adder --lib
     Created library `adder` project
$ cd adder
$ cargo test
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished test [unoptimized + debuginfo] target(s) in 0.57s
     Running unittests (target/debug/deps/adder-92948b65e88960b4)

running 1 test
test tests::it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```
注意到
- 测试用例是分批执行的，running 1 test 表示下面的输出 test result 来自一个测试用例的运行结果。
- test tests::it_works 中包含了测试用例的名称
- test result: ok 中的 ok 表示测试成功通过
- 1 passed 代表成功通过一个测试用例，0 failed 意味着没有测试用例失败，0 ignored 说明我们没有将任何测试函数标记为运行时可忽略，0 filtered 意味着没有对测试结果做任何过滤，0 measured 代表基准测试的结果
- Doc-tests 是文档测试，代码中没有任何文档测试的内容，因此这里的测试用例数为 0

再来看看会失败的测试：
```rust
#[cfg(test)]
mod tests {
    #[test]
    fn exploration() {
        assert_eq!(2 + 2, 4);
    }
    // 直接 panic 让测试失败
    #[test]
    fn another() {
        panic!("Make this test fail");
    }
}
```
输出为
```sh
running 2 tests
test tests::another ... FAILED
test tests::exploration ... ok

failures:

---- tests::another stdout ----
thread 'main' panicked at 'Make this test fail', src/lib.rs:10:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    tests::another

test result: FAILED. 1 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass '--lib'
```
输出中准确的告知了失败的函数名（failures: tests::another）和具体的失败原因（tests::another stdout），后者用于给出一眼可得结论的汇总信息。

Rust 在默认情况下会为每一个测试函数启动单独的线程去处理，当主线程发现有一个测试线程异常退出时，会将相应的测试标记为失败。当然，这样存在数据竞争的风险。

### 自定义失败信息
很多时候，断言默认的报错除了告诉我们错误发生的地方，并没有更多的信息，那再来看看该如何提供一些更有用的信息：
```rust
fn greeting_contains_name() {
    let result = greeting("mxdzs0612");
    let target = "XXX";
    assert!(
        result.contains(target),
        "你的问候中并没有包含目标姓名 {} ，你的问候是 `{}`",
        target,
        result
    );
}
```
一旦测试用例数量多起来，详细的报错信息更有利于 Debug。

### 测试 panic 
如果一个函数在某种情况下 panic 是符合预期的，想要测试这种情况，需要对目标函数用`should_panic`标注：
```rust
pub struct Guess {
    value: i32,
}

impl Guess {
    pub fn new(value: i32) -> Guess {
        if value < 1 || value > 100 {
            panic!("Guess value must be between 1 and 100, got {}.", value);
        }

        Guess { value }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[should_panic]
    fn greater_than_100() {
        Guess::new(200);
    }
    // 如果未发生 panic，会报错 test did not panic as expected 
}
```
`should_panic`可以增加`expected`参数，用于表示实际发生的 panic 正是测试希望看到的 panic，而非其他 panic。
```rust
// --snip--
impl Guess {
    pub fn new(value: i32) -> Guess {
        if value < 1 {
            panic!(
                "Guess value must be greater than or equal to 1, got {}.",
                value
            );
        } else if value > 100 {
            panic!(
                "Guess value must be less than or equal to 100, got {}.",
                value
            );
        }

        Guess { value }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    // 指定了期望的 panic 信息
    // expected 的内容只需要是实际发生的 panic 的前缀
    #[should_panic(expected = "Guess value must be less than or equal to 100")]
    fn greater_than_100() {
        Guess::new(200);
    }
}
```

### Result
如果希望在测试中使用链式调用，就需要`Result<T, E>`。
```rust
#[cfg(test)]
mod tests {
    #[test]
    fn it_works() -> Result<(), String> {
        // 手动进行逻辑判断，并返回一个 Result
        if 2 + 2 == 4 {
            Ok(())
        } else {
            Err(String::from("two plus two does not equal four"))
        }
    }
}
```

## 控制测试执行
现在测试已经写好了，接下来看看怎么跑。

`cargo build`可以将代码编译成一个可执行文件，`cargo run`和`cargo test`也一样，都是将代码编译成可执行文件然后运行，唯一的区别就在于可执行文件随后会被删除。因此，`cargo test`也可以通过命令行参数来控制测试的执行。

### 测试用例的并行或顺序执行
当运行多个测试函数时，默认情况下是为每个测试都生成一个线程，然后通过主线程来等待它们的完成和结果。这种模式的优点很明显，那就是并行运行会让整体测试时间变短很多。但是，并行测试最大的问题就在于共享状态的修改，因为难以控制测试的运行顺序，因此如果多个测试共享一个数据，那么对该数据的使用也将变得不可控制。

可以用如下命令让所有测试一个接着一个顺序运行：
```sh
$ cargo test -- --test-threads=1
```
> 提供给 cargo test 命令本身的参数在 -- 之前指定
> 
> 提供给编译后的可执行文件的参数在 -- 之后指定
线程数可以任意指定，当然，想要测试顺次执行，就只能是 1。

### 打印
默认情况下，如果测试通过，那写入标准输出的内容是不会显示在测试结果中的。
```rust
fn prints_and_returns_10(a: i32) -> i32 {
    // 如果测试顺利进行，这句话不会打印
    println!("I got the value {}", a);
    10
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn this_test_will_pass() {
        let value = prints_and_returns_10(4);
        assert_eq!(10, value);
    }

    #[test]
    fn this_test_will_fail() {
        let value = prints_and_returns_10(8);
        assert_eq!(5, value);
    }
}
```
想要打印所有结果需要这样：
```sh
$ cargo test -- --show-output
```

### 运行部分测试
如果直接使用`cargo test`运行测试，所有测试函数都会运行，对于的中大型项目，每次都运行全部测试是不可接受的。

以下面这段测试为例。
```rust
pub fn add_two(a: i32) -> i32 {
    a + 2
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn add_two_and_two() {
        assert_eq!(4, add_two(2));
    }

    #[test]
    fn add_three_and_two() {
        assert_eq!(5, add_two(3));
    }

    #[test]
    fn one_hundred() {
        assert_eq!(102, add_two(100));
    }
}
```
注意本小节的方式是可以组合使用的。

#### 运行单个测试
指定的测试函数名作为参数即可。
```sh
# 此时，只有测试函数 one_hundred 会被运行，其它两个由于名称不匹配，会被直接忽略。
$ cargo test one_hundred
running 1 test
test tests::one_hundred ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 2 filtered out; finished in 0.00s
```

#### 通过名称来过滤测试
参数无法同时指定多个名称，但可以通过指定部分名称的方式来过滤运行相应的测试。

可以指定测试方法的任意一段，甚至模块名称也是可以的。
```sh
# 前缀
$ cargo test add
running 2 tests
test tests::add_three_and_two ... ok
test tests::add_two_and_two ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 1 filtered out; finished in 0.00s

# 任意一段
$ cargo test and
running 2 tests
test tests::add_two_and_two ... ok
test tests::add_three_and_two ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 1 filtered out; finished in 0.00s

# 通过模块名称来过滤测试
$ cargo test tests

running 3 tests
test tests::add_two_and_two ... ok
test tests::add_three_and_two ... ok
test tests::one_hundred ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```
#### 忽略部分测试
Rust 允许通过 ignore 关键字来忽略特定的测试用例：
```rust
#[test]
fn it_works() {
    assert_eq!(2 + 2, 4);
}

#[test]
#[ignore]
fn expensive_test() {
    // 假设这里的代码很慢
}
```
也可以通过以下方式运行被忽略的测试函数
```sh
$ cargo test -- --ignored
running 1 test
test expensive_test ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 1 filtered out; finished in 0.00s

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

### 仅测试可用的依赖
可以通过`[dev-dependencies]`引入仅测试时可见的依赖。它们在非测试代码中可能会报错。
```toml
# 用来扩展标准库中的 assert_eq! 和 assert_ne!
[dev-dependencies]
pretty_assertions = "1"
```
```rust
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;
    use pretty_assertions::assert_eq; // 该包仅能用于测试

    #[test]
    fn test_add() {
        assert_eq!(add(2, 3), 5);
    }
}
```

### 生成测试二进制文件
在 cargo test 运行的时候，系统会自动生成一个可运行测试的二进制可执行文件，通常在 target 路径下，可以与他人分享。直接运行该文件就可以执行编译好的测试。

如果你只想生成编译生成文件，不想看`cargo test`的输出结果，可以使用`cargo test --no-run`。

## 实现测试

### 单元测试
单元测试目标是测试某一个代码单元（一般都是函数），验证该单元是否能按照预期进行工作。在 Rust 中，单元测试的惯例是将测试代码的模块跟待测试的正常代码放入同一个文件中。
```rust
pub fn add_two(a: i32) -> i32 {
    a + 2
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_works() {
        assert_eq!(add_two(2), 4);
    }
}
```

#### 条件编译
`#[cfg(test)]`标注告诉编译器只有在 cargo test 时才编译和运行模块 tests，其它时候直接忽略。`cfg`是 configuration 的缩写，它告诉 Rust ：当 test 配置项存在时，才运行下面的代码，而`cargo test`在运行时，就会将 test 这个配置项传入进来，因此后面的 tests 模块会被包含进来。这是典型的**条件编译**（坏了，这提醒我该回去填坑了）。

这么做有几个好处：
- 节省构建代码时的编译时间
- 减小编译出的可执行文件的体积

作为单元测试，测试跟正常的逻辑代码在同一个文件中，因此必须对其进行特殊的标注，以便 Rust 编译器识别。

#### 私有函数的测试
Rust 是支持对私有函数进行测试的。
```rust
pub fn add_two(a: i32) -> i32 {
    internal_adder(a, 2)
}

// 没有 pub
fn internal_adder(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn internal() {
        assert_eq!(4, internal_adder(2, 2));
    }
}
```
子模块的项可以使用其上级模块的项。通过`use super::*`将 tests 模块的父模块的所有项引入了作用域，就可以实现对私有函数的测试。

### 集成测试
集成测试的代码是在一个单独的目录下的。只能调用通过`pub`定义的 API。

如果说单元测试是对代码单元进行测试，那集成测试则是对某一个功能或者接口进行测试，因此单元测试的通过，并不意味着集成测试就能通过：局部上反映不出的问题，在全局上很可能会暴露出来。

#### tests 目录
一个标准的 Rust 项目，在它的根目录下会有一个 tests 目录（需要手动创建）。它就是用来存放集成测试的，Cargo 会自动来此目录下寻找集成测试文件，Cargo 会对里面每个文件都进行自动编译。
```rust,name=tests/integration_test.rs
use adder;

#[test]
fn it_adds_two() {
    assert_eq!(4, adder::add_two(2));
}
```
此时无需再使用`#[cfg(test)]`。

与单元测试类似，可以通过指定名称的方式来运行特定的集成测试用例。
```sh
$ cargo test --test integration_test
```

#### 共享模块
在集成测试的 tests 目录下，每一个文件都是一个独立的包，这种组织方式可以很好的帮助我们理清测试代码的关系，但是如果大家想要在多个文件中共享同一个功能该怎么办？

这种时候可以创建一个类似`tests/common/mod.rs`的文件。子文件夹能够让 Rust 在测试时忽略掉这个模块。也就是说，目录下的子目录中的文件不会被当作独立的包，也不会有测试输出。
```rust
use adder;

mod common;

#[test]
fn it_adds_two() {
    common::setup();
    assert_eq!(4, adder::add_two(2));
}
```
此时，就可以在测试中调用 common 中的共享函数了，不过还有一点值得注意，为了使用 common，这里使用了`mod common`的方式来声明该模块。

#### 二进制包的集成测试
目前来说，Rust 只支持对 lib 类型的包进行集成测试，对于二进制包例如 main.rs 是无能为力的。原因在于，我们无法在其它包中使用 use 引入二进制包，而只有 lib 类型的包才能被引入。

所以需要将代码逻辑从 main.rs 剥离出去放入 lib 包中，例如很多 Rust 项目中都同时有 main.rs 和 lib.rs ，前者中只保留代码的主体脉络部分，而具体的实现通通放在类似后者的 lib 包中。这样，我们就可以对 lib 包中的具体实现进行集成测试。只要 main.rs 中的主体脉络足够简单，当集成测试通过时，意味着 main.rs 中相应的调用代码也将正常运行。

### 文档测试
随着代码的进化，单元测试用例经常会失效，过段时间后就需要连续修改不少处代码，才能让测试重新工作起来。

而 Rust 允许在文档注释中写单元测试用例。
> 这是真的离谱，第一次知道这回事的时候把我惊掉了下巴。
```rust
/// 下面的注释不仅仅是文档，还可以作为单元测试的用例运行
/// 可能需要使用完整路径来调用函数，因为测试是在另外一个独立的线程中运行的
///
/// # Examples11
///
/// ```
/// let arg = 5;
/// let answer = world_hello::compute::add_one(arg);
///
/// assert_eq!(6, answer);
/// ```
pub fn add_one(x: i32) -> i32 {
    x + 1
}
```

#### 造成 panic
对于可能出现 panic 的用例，文档测试同样可以 should_panic，就是位置不太一样：
```rust
/// # Panics
///
/// The function panics if the second argument is zero.
///
/// ```rust,should_panic
/// // panics on division by zero
/// world_hello::compute::div(10, 0);
/// ```
pub fn div(a: i32, b: i32) -> i32 {
    if b == 0 {
        panic!("Divide-by-zero error");
    }

    a / b
}
```

#### 保留测试，隐藏文档
在某些时候，我们希望保留文档测试的功能，但是又要将某些测试用例的内容从文档中隐藏起来。使用`#`开头的行会在文档中被隐藏起来，但是依然会在文档测试中运行
```rust
/// ```
/// # // 使用#开头的行会在文档中被隐藏起来，但是依然会在文档测试中运行
/// # fn try_main() -> Result<(), String> {
/// let res = world_hello::compute::try_div(10, 0)?;
/// # Ok(()) // returning from try_main
/// # }
/// # fn main() {
/// #    try_main().unwrap();
/// #
/// # }
/// ```
pub fn try_div(a: i32, b: i32) -> Result<i32, String> {
    if b == 0 {
        Err(String::from("Divide-by-zero"))
    } else {
        Ok(a / b)
    }
}
```
> 还没有好好学习过 Rust 的注释和文档，要补课的更多了……

## 断言
在编写测试函数时，断言决定了我们的测试是通过还是失败。本节来看看 Rust 提供的断言都有什么。

{{ admonition(type="warning", title="注意", text="施工中") }}
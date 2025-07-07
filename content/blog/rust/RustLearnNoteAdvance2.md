+++
title = "Rust 进阶学习笔记（二）：复合数据类型"
slug = "rust_learn_note_adv_2"
date = 2025-07-05
updated = 2025-07-07
description = "字符串与切片，元组数组，数组、双向数组和链表，键值对和Map"
# draft = true
[taxonomies]
tags = ["Rust", "Learn"]
[extra]
pinned = false
post_listing_date = "both"
+++

本节出处：原子之音 [视频](https://www.bilibili.com/video/BV1r34y1G7b7/) [代码](https://gitlab.com/yzzy/compound_types)

## 概述
复合数据类型（Compound types）通过组合基础数据类型，来表达更复杂的数据结构。也就是使用其他类型定义的类型。

也被称为派生类型。

### 内置复合类型
- 结构体 struct
- 数组 array
- 元组 tuple
- 动态数组 vector

### 标准库中的复合类型
来自`std::collections`：
- VecDeque
- LinkedList
- HashMap, BTreeMap
- HashSet, BTreeSet

## 字符串与字符串切片

### 字符串
Rust 中 String 的源码如下：
```rust,name=string.rs
#[derive(PartialEq, PartialOrd, Eq, Ord)]
#[stable(feature = "rust1", since = "1.0.0")]
#[lang = "String"]
pub struct String {
    vec: Vec<u8>,
}
```
可以看到，rust 中的字符串被设计成一个内容为 u8 可变数组的结构体，其中 u8 是一个无符号整型，可以表示 0～255 之间的数字。

#### 二进制到字符串
从二进制数组到字符串的过程，需要解决两个问题：
1. 字符串什么时候结束
   1. 存储结束符号（C 语言，非常不安全）
   2. 在开头存储字符串长度（Rust）
2. 采用什么编码
   1. ASCII 一个字节对应一个字符，可以表示常用的 256 个字符
   2. UTF-8 一到四个四届对应一个字符，可以辨识所有语言的字符，且可以和 ASCII 兼容。

因为一个字节就是一个 8 位二进制数，所以 String 才使用了 u8。

#### Rust 字符串
- 在开头存储字符串长度
- 所有都是 UTF-8
- 默认不可变

#### string_item
存储了（详见下文 Vec 源码）：
- pointer：字符串头一个字符的地址
- length：具体字符个数
- capacity：容量，扩容一般会翻倍

### 字符串切片
str 就是字符串切片，实际上是 [u8] 数组切片，也就是动态大小的 UTF-8 字节。

str 在 rust 中无法直接使用(str 是不安全的)，经常用的是`&str`，以及`Box<str>`、`Rc<str>`等智能指针。

#### 区别
String 具有所有权，&str 是切片引用。

String 的数据必须存储在堆上，&str 则根据引用指向，可能存储在堆上、栈上甚至数据段上。

在创建需要所有权的字符串、有修改需求时，使用 String；在查找、只读，以及不需要关心生命周期时，使用 &str。

***
### 例子
```rust
// String 和 &str 的区别与相互转换
// 如何通过 &[u8] 转换为 String
// &str 与生命周期

fn rust_say() -> &'static str {
    // 不加转义的写法
    r#"Rust said "str" is danger"#
}

// 调用第三库经常遇到 &[u8] -> String
fn u8_to_string(string_data: &[u8]) -> String {
    // String::from_utf8_lossy(string_data).to_string()
    // u8 可以强制转换成 char
    string_data.iter().map(|&s|s as char).collect()
}

fn main() {
    // 创建空字符串
    let mut item = String::new();
    println!("String {}: cap {}", item, item.capacity()); // "" 0
    item.push('c');
    println!("String {}: cap {}", item, item.capacity()); // "c" 8
    // 创建带容量的字符串
    let item = String::with_capacity(10);
    println!("String {}: cap {}", item, item.capacity()); // "" 10 

    // &str -> String 
    let item = String::from("hello world");
    println!("String {}: cap {}", item, item.capacity());
    let item = "hello world".to_string();
    println!("String {}: cap {}", item, item.capacity());
    // String -> &str
    let i = &item;
    // str 声明生命周期，一般都是静态的
    println!("{}", rust_say());
    // &str 引用的创建方式
    const C_S: &str = "";
    let yzzy = "yzzy";
    // &str => &u8 => String
    println!("u8 to String: {}", u8_to_string(yzzy.as_bytes()));

    // 尝试直接打印 &u8
    for item in yzzy.as_bytes() {
        println!("item {}", item); // 打印的是数字 121 122 122 121
    }

}
```

## 元组和数组
元组可以包含各种类型值的组合，数组只能包含统一类型数据的组合。

默认不可变。

```rust
fn u8_to_string(string_data: &[u8]) -> String {
    string_data.iter().map(|&s|s as char).collect()
}

#[derive(Debug)]
struct User{
    name: String,
    age: i32,
}

fn main() {
    // 创建元组
    let tup = (1,2,"hello", User{name: "y".to_owned(), age: 45});
    // 取元组中的值
    println!("tup {} {} {} {:?}",tup.0, tup.1, tup.2, tup.3 );
    // 元组中值可以命名
    let (a, b, c, d) = tup;
    println!("a {} b {} c {} d {:#?}", a, b, c, d);

    // 创建数组
    let my_array = [1,2,3,4];
    // let my_array: [i32;4] = [1,2,3,4]; // 类型;容量
    // let my_array: [i32;3] = [1, 2, 3];
    // 数组切片只能传引用，否则报错 doesn't have a size known at compile-time
    let my_slice = &my_array[1..3];
    // 没有容量，只能获得长度
    println!("my_array len {}", my_array.len());
    println!("my_slice len {}", my_slice.len());
    
    // u8;4
    let u8_array = [121u8, 122u8, 122u8, 121u8];
    // &u8
    let u8_slice_ref = &u8_array[0..4]; //&[u8]
    // u8_slice_ref 和 &u8_array是没有区别的，不需要区分数组引用和切片引用
    println!("{:#?}", u8_to_string(&u8_array));
    println!("{:#?}", u8_to_string(u8_slice_ref));
}
```

## 序列类型

### Vec
可变数组（vector，又是译作“矢量”）存储在堆上，在运行时可以增加或减少数组长度。

数组默认是不可变的。

#### Vec 堆栈存储
Vec 的源码如下。其中，`buf`在堆上，其他在栈上。
```rust
pub struct Vec<T, #[unstable(feature = "allocator_api", issue = "32838")] A: Allocator = Global> {
    buf: RawVec<T, A>,
    len: usize,
}

pub(crate) struct RawVec<T, A: Allocator = Global> {
    inner: RawVecInner<A>,
    _marker: PhantomData<T>,
}

struct RawVecInner<A: Allocator = Global> {
    ptr: Unique<u8>,
    /// Never used for ZSTs; it's `capacity()`'s responsibility to return usize::MAX in that case.
    ///
    /// # Safety
    ///
    /// `cap` must be in the `0..=isize::MAX` range.
    cap: Cap,
    alloc: A,
}
```

#### 用途
- 在不知道应该选择哪种集合结构时，首选就是 Vec。
- 需要一个长度实时变化的数据存储单元时
- 希望存储单元有序时
- 希望内存中各个元素的存储地址连续时

***
#### 例子
```rust
#[derive(Debug, Clone)]
struct Car {
    id: i32,
    name: String
}

fn main() {
    // 初始化方法
    let my_vec = vec![2, 3, 4];
    println!("{:#?}: {}", my_vec, my_vec.capacity()); // [2,3,4] 3
    // 初始值;长度
    let my_vec = vec![2;3];
    println!("{:#?}: {}", my_vec, my_vec.capacity()); // [2,2,2] 3
    // 如果后续进行了操作，可以自动推断，否则需要指定类型
    let my_vec:Vec<i32> = Vec::new();
    println!("{:#?}: {}", my_vec, my_vec.capacity()); // [] 0
    // 初始化容量
    let my_vec:Vec<i32> = Vec::with_capacity(5);
    println!("{:#?}: {}", my_vec, my_vec.capacity()); // [] 5

    // 增：push
    let mut cars = Vec::new();
    for i in 1..11 {
        // rust 没有三元表达式
        let car_type = if i % 2 == 0 {"car"} else {"bus"};
        cars.push(Car {
            id: i,
            name: format!("{}-{}", car_type, i)
        });
    }
    println!("{:?}", cars);
    // 删：pop（末尾删除）
    let car = cars.pop().unwrap();
    println!("{:?}", cars);
    println!("{:?}", car); // 10
    // remove(i)：固定index删除
    let car = cars.remove(0);
    println!("{:?}", cars);
    println!("{:?}", car); // 1
    // push 尾插
    cars.push(car.clone());
    println!("{:?}", cars);
    // insert 任意位置插入
    cars.insert(0, car.clone());
    println!("{:?}", cars);

    // 读取和修改
    // 不推荐，因为可能会超出索引，不安全
    cars[9] = Car{
        id: 11,
        name: "car-11".to_string()
    };
    println!("{:?}", cars);
    // 推荐：get 和 set
    let item = cars.get(0);
    println!("{:?}", item);
    let item = cars.get_mut(9).unwrap();
    // item 是 &mut Car 类型，需要解引用才能修改
    // item 不需要加 mut，因为这里没有修改 item 本身，只修改了他指向的内容
    *item = Car{id:10, name: "car-10".to_owned()};
    // 如果这么写就需要加 mut 了
    // item.name = xxxx
    println!("{:?}", cars);
    let mut cars2 = vec![car.clone(), car.clone()];
    // 末尾追加，必须可变
    cars.append(&mut cars2);
    println!("{:?}", cars);
    println!("{:?}", cars2);
    // 转换成 slice
    let car_slice = cars.as_slice();
    println!("{:?}", car_slice);

    // 清空
    cars.clear();
    println!("{:?}", cars);
    println!("{:?}", cars.is_empty());
}
```

### VecDeque
`VecDeque`是一个双向动态数组，两端都可以以O(1)的时间复杂度插入和移除。

`VecDeque`在内存中并不一定是连续的。

#### 用途
有双向增删需求的，如队列。

#### 方法
VecDeque 继承了 Vec 的所有方法。此外还新增了如下方法：
- push_back/push_front
- pop_back/pop_front
- contains
- front/front_mut
- back/back_mut

***
#### 例子
```rust
use std::collections::VecDeque;
#[derive(Debug, Clone)]
struct Car {
    id: i32,
    name: String
}

fn main() {
    let mut int_queue: VecDeque<i32> = VecDeque::new();
    println!("{:?} cap {}", int_queue, int_queue.capacity()); // [] 0
    int_queue.push_back(3);
    // 写入一个数，容量会变成 4
    println!("{:?} cap {}", int_queue, int_queue.capacity()); // [3] 4
    let mut int_queue: VecDeque<i32> = VecDeque::with_capacity(20);
    println!("{:?} cap {}", int_queue, int_queue.capacity());

    let mut cars = VecDeque::from([
        Car{id: 1, name: String::from("Car-1")},
        Car{id: 2, name: String::from("Car-2")},
        Car{id: 3, name: String::from("Car-3")},
    ]);
    println!("{:?} cap {}", cars, cars.capacity());
    // push
    cars.push_back(Car { id: 4, name: "Car-4".to_string() });
    println!("{:?} cap {}", cars, cars.capacity());
    cars.push_front(Car { id: 0, name: "Car-0".to_string() });
    println!("{:?} cap {}", cars, cars.capacity());
    // pop
    let back_car = cars.pop_back();
    println!("{:?}", back_car);
    println!("{:?} cap {}", cars, cars.capacity());
    let front_car = cars.pop_front();
    println!("{:?} ", front_car);
    println!("{:?} cap {}", cars, cars.capacity());

    // front & back
    println!("{:?}", cars.front());
    println!("{:?}", cars.back());
    // get
    println!("car{:?}", cars[1]);
    let car = cars.get(1).unwrap();
    println!("car{:?}", car);

    let car = cars.get_mut(1).unwrap();
    *car = Car{id: 10, name: "Car-n".to_owned()};
    println!("car{:?}", cars);

    cars.clear();
    println!("{:?}", cars.is_empty());
}
```

### LinkedList
LinkedList 是一种通过链式存储数据的序列数据结构，是双向的，并且具有所有权。

#### 用途
LinkedList 在分割和追加上较为高效。

如果对分配多少内存和何时分配内存有细粒度的控制要求时，可以考虑使用。

绝大多数情况下更应该直接使用 Vec 和 VecDeque。

#### 方法
LinkedList 支持大多数 VecDeque 的方法，不支持 Vec 的方法。

***
#### 例子
```rust
use std::collections::LinkedList;

#[derive(Debug, Clone)]
struct Car {
    id: i32,
    name: String
}
// vecque 有
// vec 无
fn main() {
    let mut int_link: LinkedList<i32> = LinkedList::new();
    // 注意 LinkedList 需要所有权，所以不支持 with_capacity 等方法
    let mut cars = LinkedList::from([
            Car{id: 1, name: "Car-1".to_string()},
            Car{id: 2, name: "Car-2".to_string()},
            Car{id: 3, name: "Car-3".to_string()},
        ]);
        cars.push_back(Car{id: 4, name: "Car-4".to_string()});
        println!("back: {:?}", cars.back());
        cars.push_front(Car{id: 0, name: "Car-0".to_string()});
        println!("front: {:?}", cars.front());
        println!("{:?}", cars);
        let car = cars.pop_back().unwrap();
        println!("{:?}", car); // 4
        let car = cars.pop_front().unwrap();
        println!("{:?}", car); // 0

        // 从指定位置分割链表的方法
        let mut car_list = cars.split_off(cars.len() -2);
        println!("{:?}", car_list); // 2 3
        println!("{:?}", cars); // 1
        cars.append(&mut car_list);
        println!("{:?}", cars);
        println!("{:?}", car_list);
        cars.clear();
        println!("{}", cars.is_empty());
}
```

## 键值对 Map
键值对又称为字典数据类型，主要是为了查找创建的数据结构。

### HashMap
HashMap<K, V> 类型存储了一个键类型 K 和一个值类型 V 的映射。他通过一个哈希函数（Hash Function，也叫散列函数）来决定如何存储键和值，实现映射。

HashMap 和 Vec 一样，数据也在堆上。

HashMap 无序的。

#### 用途
- 仅次于 Vec 的第二选择
- 存储映射关系
- 要求较快查找速度（O(1)）
- 需要内存缓存

#### 键
HashMap 对键有特殊要求，即必须满足哈希函数。

所以需要实现`Hash`、`Eq`、`PartialEq`等特质。

对于普通的结构体，可以通过增加`#[derive(Debug, Hash, Eq, PartialEq)]`自动实现。

***
#### 例子
```rust
use std::collections::HashMap;

// Hash Eq PartialEq
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
struct Car {
    id: i32,
    price: i32
}

fn main() {
    // 创建需要指定键值的类型
    let _int_map:HashMap<i32, i32> = HashMap::new();
    let _int_map:HashMap<i32, i32> = HashMap::with_capacity(10);
    
    // 通过数组来创建map
    let mut car_map = HashMap::from([
        ("Car1", Car{id: 1, price: 10000}),
        ("Car2", Car{id: 2, price: 4000}),
        ("Car3", Car{id: 3, price: 890000}),
    ]);
    // 无序，每次打印结果不一定相同
    for (k, v) in &car_map {
        println!("{k}:{:?}", v);
    }
    // 查找值 get，返回的是 Some 或者 None
    println!("Some {:?}", car_map.get("Car1"));
    println!("None {:?}", car_map.get("Car6"));
    // 覆盖型插入 insert
    car_map.insert("Car4", Car { id: 4, price: 80000 });
    println!("{:?}", car_map);
    car_map.insert("Car4", Car { id: 5, price: 300000 });
    println!("{:?}", car_map);
    // 只在键没有时才能插入 entry or_insert
    car_map.entry("Car4").or_insert(Car{id: 9, price: 9000});
    println!("{:?}", car_map); //没写成功
    // 删除 remove
    car_map.remove("Car4");
    println!("{:?}", car_map);
    car_map.entry("Car4").or_insert(Car{id: 9, price: 9000});
    println!("{:?}", car_map); //写成功

    // 将结构体作为键存储
    let mut car_map = HashMap::from([
        (Car{id: 1, price: 10000}, "Car1"),
        (Car{id: 2, price: 4000}, "Car2"),
        (Car{id: 3, price: 890000}, "Car3"),
    ]);

    println!("Car2: {:?}\n", car_map.get(&Car{id: 1, price:10000}));

    for (car, name) in &car_map {
        println!("{:?}: {name}", car)
    }

    // Filter
    car_map.retain(|c, _| c.price < 5000);
    println!("< 5000  {:?}", car_map);
}
```

### BTreeMap
BTreeMap 是一种有序形式的键值对，内部基于 BTree 构建。

#### 用途
- 需要有序 map
- 查找时可以通过二分提高性能
- 有序的代价：缓存效率和查找性能的折中

#### 键
因为需要排序，所以 BTreeMap 需要实现`Ord`、`PartialOrd`。

***
#### 例子
```rust
use std::collections::BTreeMap;

// Hash Eq PartialEq
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
struct Car {
    id: i32,
    price: i32
}
// 手动实现比较特质
impl Ord for Car {
    fn cmp(&self, other: &Self) -> std::cmp::Ordering {
        self.price.cmp(&other.price)
    }
}
// 返回的是 Some
impl PartialOrd for Car {
    fn partial_cmp(&self, other: &Self) -> Option<std::cmp::Ordering> {
        Some(self.price.cmp(&other.price))
    }
}

fn main() {
    // 创建
    let _int_map:BTreeMap<i32, i32> = BTreeMap::new();
    // 树没有办法预留空间
    // let _int_map:BTreeMap<i32, i32> = BTreeMap::with_capacity(10);
    
    // 通过数组来创建map
    let mut car_map = BTreeMap::from([
        ("Car1", Car{id: 1, price: 10000}),
        ("Car2", Car{id: 2, price: 4000}),
        ("Car3", Car{id: 3, price: 890000}),
    ]);
    println!("{:#?}", car_map); // 有序，id 越小越靠前

    let mut car_map = BTreeMap::from([
        (Car{id: 1, price: 10000}, 1),
        (Car{id: 2, price: 4000}, 2),
        (Car{id: 3, price: 890000}, 3),
    ]);
    for (k, v) in &car_map {
        println!("{:?}: {v}", k); // 有序，price 越小越靠前
    }
    // 插入，动态有序
    car_map.insert(Car{id: 4, price: 90000}, 4);
    for (k, v) in &car_map {
        println!("{:?}: {v}", k);
    }
    // 查找
    println!("{:?}", car_map.get(&Car{id: 1, price: 10000}));
    // 可以查找第一个或最后一个键
    println!("{:?}", car_map.first_key_value());
    println!("{:?}", car_map.last_key_value());

    // 删除，注意返回值是 Option 元组，即完整键值
    let car = car_map.pop_first().unwrap();
    println!("{:?}", car);
    let car = car_map.pop_last().unwrap();
    println!("{:?}", car);
    for (k, v) in &car_map {
        println!("{:?}: {v}", k);
    }
    // 删除固定 index，性能较差不建议用
    car_map.remove(&Car{id: 1, price: 10000});
    for (k, v) in &car_map {
        println!("{:?}: {v}", k);
    }
    // 清空，并判断是否为空
    car_map.clear();
    println!("{}", car_map.is_empty());
}
````
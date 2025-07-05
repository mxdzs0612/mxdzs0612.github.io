+++
title = "Rust 进阶学习笔记（二）：复合数据类型"
slug = "rust_learn_note_adv_2"
date = 2025-07-05
updated = 2025-07-05
description = "字符串与切片，元组数组，数组、双向数组和链表，键值对和Map"
# draft = true
[taxonomies]
tags = ["Rust", "Learn"]
[extra]
pinned = false
post_listing_date = "both"
+++

出处：原子之音 [视频](https://www.bilibili.com/video/BV1r34y1G7b7/) [代码](https://gitlab.com/yzzy/compound_types)

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
（WIP）

## 元组和数组
（WIP）

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

## 字典型 Map
{{ admonition(type="warning", icon="tip", title="注意", text="施工中") }}

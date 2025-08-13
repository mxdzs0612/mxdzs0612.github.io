+++
title = "Rust 项目实战（二）：Mini-LSM"
slug = "rust_learn_note_prac_2"
date = 2025-07-30
updated = 2025-07-30
description = "迟策大佬的 Mini-LSM 项目通关笔记"
# draft = true
[taxonomies]
tags = ["Rust", "Learn", "Project"]
[extra]
pinned = false
post_listing_date = "both"
+++

本节出处：迟神的 [LSM in a Week](https://skyzh.github.io/mini-lsm/00-preface.html) 项目。

搞了这么久 Paimon，不研究 LSM Tree 的代码怎么行。久仰迟神大名，正好借学 Rust 的机会来整一波。

我的 [fork](https://github.com/mxdzs0612/mini-lsm)  参考答案 [官方](https://github.com/skyzh/mini-lsm-solution-checkpoint) [民间](https://github.com/skyzh/mini-lsm/blob/main/SOLUTIONS.md)

不过这玩意的官方答案基本上就不是给人看的，尤其是对于 Rust 新手来说，谁会能想到这样写啊！民间版就易读很多。其实不看答案也没问题，迟神给每个子任务都写了测试，只要测试能跑过就说明（至少在当前阶段）代码没问题。

我会尽量把每个子任务的 commit 拆开，方便回溯。

***
![All](https://skyzh.github.io/mini-lsm/lsm-tutorial/00-full-overview.svg)

这是项目的整体结构，共三周，每周 7 小节，可以看到还是非常强大和全面的。

本文暂时省略 LSM 的介绍，详细信息可以参考 [RocksDB Wiki](https://github.com/facebook/rocksdb/wiki/RocksDB-Overview)。

本文不能代替项目文档，读者如有疑问还需优先查看项目文档。项目的环境安装、测试准备等在本文中也不再赘述，请看项目文档。

## Week 1：Mini-LSM
![week1](https://skyzh.github.io/mini-lsm/lsm-tutorial/week1-overview.svg)

在 Week 1，我们将构建存储格式，系统的读写路径，并实现一个可用的基于 LSM 树的键值存储。本章结束时，引擎理应具备除持久化外，一个 LSM 树的全部必备功能。

### Day1：Memtables
本节是内存表读写路径的实现。

#### Task1：基于跳表的内存表
第一个任务的内容很简单，只需要修改`mem_table.rs`，补全创建内存表的方法，然后实现内存表基础的 get 和 put。

其实第一步就卡住我了，主要原因是一上来就引入了一大堆没见过的第三方库的结构。不过目前用到的不算多。我们查看 MemTable 的定义
```rust,name=mem_table.rs
pub struct MemTable {
    map: Arc<SkipMap<Bytes, Bytes>>,
    wal: Option<Wal>,
    id: usize,
    approximate_size: Arc<AtomicUsize>,
}
```
文档说这个跳表提供了和 Rust 标准库中 BTreeMap 类似的 insert、get 和 iter 接口，那就好说了。先写初始化，既然是初始化，那就给结构体的所有字段赋空值就行。如果不知道默认值或者构造方法，可以直接点进源码去查看。这是我的实现：
```rust,name=mem_table.rs
/// Create a new mem-table.
pub fn create(_id: usize) -> Self {
    Self {
        map: Arc::new(SkipMap::new()),
        wal: None,
        id: _id,
        approximate_size: Arc::new(std::sync::atomic::AtomicUsize::new(0)),
    }
}
```
接下来写 get。get 肯定就是根据键去取结构体中 map 属性对应的值了。
```rust,name=mem_table.rs
/// Get a value by key.
pub fn get(&self, _key: &[u8]) -> Option<Bytes> {
    self.map.get(_key).map(|f| f.value().clone())
}
```
我这里用了闭包，当然也可以不用，也就是多几个 if/else 的事。同理写 put，put 不需要返回 Option，写起来就更简单了，直接 insert 就行：
```rust,name=mem_table.rs
pub fn put(&self, _key: &[u8], _value: &[u8]) -> Result<()> {
    self.map
        .insert(Bytes::copy_from_slice(_key), Bytes::copy_from_slice(_value));
    Ok(())
}
```
运行测试，完美！task1 的两个都通过了。

#### Task2：单个内存表
第二个任务也很简单，需要在存储引擎中进行内存表的读写。显然，这个任务需要我们调用第一个任务实现的方法。本节要求我们实现`lsm_storage.rs`的LsmStorageInner::get 、 LsmStorageInner::put 和 LsmStorageInner::delete。

注意文档中有句关键的提示：
>  As our memtable implementation only requires an immutable reference for put, you ONLY need to take the read lock on state in order to modify the memtable.

MiniLsm 这个结构体默认已经实现了一些方法，来看这个方法：
```rust,name=lsm_storage.rs
pub fn force_flush(&self) -> Result<()> {
    if !self.inner.state.read().memtable.is_empty() {
        self.inner
            .force_freeze_memtable(&self.inner.state_lock.lock())?;
    }
    if !self.inner.state.read().imm_memtables.is_empty() {
        self.inner.force_flush_next_imm_memtable()?;
    }
    Ok(())
}
```
我们只需要知道它调用的`self.inner`就是我们正在开发的 LsmStorageInner。这样就有一个大概思路了。如法炮制，还是先来实现 get。这里会比任务 1 中复杂一些，主要在于这里需要获取锁，然后返回的结果又用 Result 包了一下。其实也没复杂多少，顺手的事：
```rust,name=lsm_storage.rs
/// Get a key from the storage. In day 7, this can be further optimized by using a bloom filter.
pub fn get(&self, _key: &[u8]) -> Result<Option<Bytes>> {
    Ok(self.state.read().memtable.get(_key).and_then(|bytes| {
        if bytes.is_empty() {
            None
        } else {
            Some(bytes.clone())
        }
    }))
}
```
这里我依然用了闭包。不用闭包的话这块写出来好丑。然后写 put，根据提示，我们依然只需要获取读锁，那就非常简单了：
```rust,name=lsm_storage.rs
/// Put a key-value pair into the storage by writing into the current memtable.
pub fn put(&self, _key: &[u8], _value: &[u8]) -> Result<()> {
    self.state.read().memtable.put(_key, _value)
}
```
delete 更简单，根据提示，本课程中的删除就是写一个空值。那就正好利用上了我们刚实现好的 put 方法了：
```rust,name=lsm_storage.rs
/// Remove a key from the storage by writing an empty value.
pub fn delete(&self, _key: &[u8]) -> Result<()> {
    self.put(_key, &[])
}
```
可以发现并不难嘛。写到这里，我就渐渐有了信心，不慌了。

{{ admonition(type="warning", title="注意", text="施工中") }}
```rust,name=

```
+++
title = "Rust 项目实战（二）：Mini-LSM（更新至1-1）"
slug = "rust_learn_note_prac_2"
date = 2025-08-13
updated = 2025-08-21
description = "迟策大佬的 Mini-LSM 项目的通关笔记"
# draft = true
[taxonomies]
tags = ["Rust", "Learn", "Project"]
[extra]
pinned = false
post_listing_date = "both"
+++

本节出处：迟神的 [LSM in a Week](https://skyzh.github.io/mini-lsm/00-preface.html) 项目。

搞了这么久 Paimon，不研究 LSM Tree 的具体代码实现怎么行。久仰迟神大名，正好借学 Rust 的机会来整一波。想知道这是个什么，可以看[知乎上的项目简介](https://zhuanlan.zhihu.com/p/680608573)。

参考答案: [官方](https://github.com/skyzh/mini-lsm-solution-checkpoint) [民间](https://github.com/skyzh/mini-lsm/blob/main/SOLUTIONS.md)
> 这玩意的官方答案基本上就不是给人看的，尤其是对于 Rust 新手来说，谁会能想到这样写啊！民间版就易读很多。其实不看答案也没问题，迟神给每个子任务都写了测试，只要测试能跑过就说明（至少在当前阶段）代码没什么大问题。

[我的代码实现](https://github.com/mxdzs0612/mini-lsm)，我会尽量把每个子任务的 commit 拆开，方便回溯。

***
![All](https://skyzh.github.io/mini-lsm/lsm-tutorial/00-full-overview.svg)

这是项目的整体结构，共三周，每周 7 小节，可以看到还是非常强大和全面的。

本文暂时省略 LSM 的介绍，详细信息可以参考 [RocksDB Wiki](https://github.com/facebook/rocksdb/wiki/RocksDB-Overview)。

本文不会复述题目内容，因此不能代替项目文档。我默认读者阅读本文前已经看过对应章节的项目文档。此外，环境安装、测试怎么跑等基础内容在本文中也不再赘述，请自行从项目文档里面找。

## Week 1：Mini-LSM
![week1](https://skyzh.github.io/mini-lsm/lsm-tutorial/week1-overview.svg)

在 Week 1，我们将构建存储格式，系统的读写路径，并实现一个可用的基于 LSM 树的键值存储。本章结束时，引擎理应具备除持久化外，一个 LSM 树的全部必备功能。

### Day1：Memtables
本节是内存表读写的实现。这是一个 LSM 树的最基本内容，放在最前面理所应当。

#### Task1：基于跳表的内存表
第一个任务的内容很简单，只需要修改`mem_table.rs`，补全创建内存表的方法，然后实现内存表基础的 get 和 put。

其实第一步就卡住我了，主要原因是一上来就引入了一大堆没见过的第三方库的结构，不过目前用到的不算多所以没卡太久。我们查看 MemTable 的定义
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
>其实理论上应该把变量名开头的下划线去掉，因为这时候这些变量已经是使用了的，并非未使用的。不过我知道写完 day1 才意识到这一点，算了就这样吧，以后再说。

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
&[u8] 转 Bytes 的写法可以从 map_bound 方法抄。运行测试，完美！task1 的两个都通过了。

#### Task2：单个内存表
第二个任务也很简单，需要在存储引擎中进行内存表的读写。本节要求我们实现`lsm_storage.rs`的LsmStorageInner::get 、 LsmStorageInner::put 和 LsmStorageInner::delete。显然，这个任务需要我们调用第一个任务实现的方法。

注意文档中有句关键的提示：
>  As our memtable implementation only requires an immutable reference for put, you ONLY need to take the read lock on state in order to modify the memtable.

什么锁？怎么获取锁？什么都不知道怎么办？一翻上下翻找后，我发现内置的 MiniLsm 这个结构体默认已经实现了一些方法，主要来看这个方法：
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
具体在做什么不用管（应该能猜到和落盘有关），我们只需要知道它调用的`self.inner`就是我们正在开发的 LsmStorageInner。这样就有一个大概思路了。如法炮制，还是先来实现 get。这里会比任务 1 中复杂一些，主要在于需要获取锁，然后返回的结果又用 Result 包了一下。其实也没复杂多少，顺手的事：
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
这里我依然用了闭包，不用闭包的话这块写出来好丑。然后写 put，根据提示，我们依然只需要获取读锁，那就非常简单了：
```rust,name=lsm_storage.rs
/// Put a key-value pair into the storage by writing into the current memtable.
pub fn put(&self, _key: &[u8], _value: &[u8]) -> Result<()> {
    self.state.read().memtable.put(_key, _value)
}
```
delete 更简单，根据提示，本课程中的删除就是写一个空值覆盖掉原本的值。那就正好利用上了我们刚实现好的 put 方法了：
```rust,name=lsm_storage.rs
/// Remove a key from the storage by writing an empty value.
pub fn delete(&self, _key: &[u8]) -> Result<()> {
    self.put(_key, &[])
}
```
可以发现并不难嘛。写到这里，我就渐渐有了信心，不慌了。

#### Task3：冻结内存表
好家伙，刚说完不难，就给我来了个下马威：这么长的说明文字！

其实仔细看下来也还好。本节就是要实现 force_freeze_memtable 方法。正当一头雾水无从下手的时候，我突然从上一个 practice 中得到了一个思路：完全可以面向测试用例编程嘛！看看测试是怎么做的就好了！
```rust,name=week1_day1.rs
#[test]
fn test_task3_storage_integration() {
    // ... 
    storage
        .force_freeze_memtable(&storage.state_lock.lock())
        .unwrap();
    assert_eq!(storage.state.read().imm_memtables.len(), 1);
    let previous_approximate_size = storage.state.read().imm_memtables[0].approximate_size();
    assert!(previous_approximate_size >= 15);
    storage.put(b"1", b"2333").unwrap();
    storage.put(b"2", b"23333").unwrap();
    storage.put(b"3", b"233333").unwrap();
    storage
        .force_freeze_memtable(&storage.state_lock.lock())
        .unwrap();
    assert_eq!(storage.state.read().imm_memtables.len(), 2);
    assert!(
        storage.state.read().imm_memtables[1].approximate_size() == previous_approximate_size,
        "wrong order of memtables?"
    );
    assert!(storage.state.read().imm_memtables[0].approximate_size() > previous_approximate_size);
}

#[test]
fn test_task3_freeze_on_capacity() {
    let dir = tempdir().unwrap();
    let mut options = LsmStorageOptions::default_for_week1_test();
    options.target_sst_size = 1024;
    options.num_memtable_limit = 1000;
    let storage = Arc::new(LsmStorageInner::open(dir.path(), options).unwrap());
    for _ in 0..1000 {
        storage.put(b"1", b"2333").unwrap();
    }
    let num_imm_memtables = storage.state.read().imm_memtables.len();
    assert!(num_imm_memtables >= 1, "no memtable frozen?");
    for _ in 0..1000 {
        storage.delete(b"1").unwrap();
    }
    assert!(
        storage.state.read().imm_memtables.len() > num_imm_memtables,
        "no more memtable frozen?"
    );
}
```
可以看到，这两个测试分别是主动和被动调用了 force_freeze_memtable。并且这里涉及到一个变量 approximate_size 和两个配置 target_sst_size、num_memtable_limit。也就是说还需要估算一下 memtable 的大小。

先来写 force_freeze_memtable。这个方法要做的事情，就是在它无论因为什么而被调用到了的时候，就把当前 memtable 写到 imm_memtables 里面。注意到文档中还有句话
>You can simply assign the next memtable id as self.next_sst_id(). Note that the imm_memtables stores the memtables from the latest one to the earliest one. That is to say, imm_memtables.first() should be the last frozen memtable.

我们先来看看这个 next_sst_id：
```rust,name=lsm_storage.rs
pub(crate) fn next_sst_id(&self) -> usize {
    self.next_sst_id
        .fetch_add(1, std::sync::atomic::Ordering::SeqCst)
}
```
next_sst_id 是一个 AtomicUsize。注意到 approximate_size 也是一个 AtomicUsize，这就是在提示我们 approximate_size 仿照这个改就可以了。此外，这段话还告诉我们 imm_memtables 需要头插，那没办法了，只能这么写：
```rust,name=lsm_storage.rs
// _state_lock_observer 暂时不知道是做什么用的，先不管它
pub fn force_freeze_memtable(&self, _state_lock_observer: &MutexGuard<'_, ()>) -> Result<()> {
    // 获取写锁的可变引用
    let mut state = self.state.write();
    let state = Arc::make_mut(&mut state);
    // replace 的作用是获取第一个参数的值并拿到所有权，然后将原来的参数替换成第二个参数中的值。
    // 我承认这个地方看了答案并问了 AI，我自己是不知道这个 API 的
    let old_mem = std::mem::replace(
        &mut state.memtable,
        Arc::new(MemTable::create(self.next_sst_id())),
    );
    state.imm_memtables.insert(0, old_mem);
    Ok(())
}
```
然后我们发现，主动调用的测试是要验证 approximate_size 的，那行吧，开抄。仿照 next_sst_id 在 mem_table.rs 的 put 方法里加一行就行：
```rust,name=mem_table.rs
pub fn put(&self, _key: &[u8], _value: &[u8]) -> Result<()> {
    self.approximate_size.fetch_add(
        _key.len() + _value.len(),
        // 这里的顺序无所谓，完全可以用 Relaxed，不必死板地照抄
        // 具体不展开解释了，点开看看源码里的注释即可
        std::sync::atomic::Ordering::Relaxed,
    );
    self.map
        .insert(Bytes::copy_from_slice(_key), Bytes::copy_from_slice(_value));
    Ok(())
}
```
这个时候，主动调用的测试就通过了。

然后再来搞被动调用的。这需要我们处理增/删内存表元素的代码，每修改一个，就要做一次判断，看有没有达到配置的上限。
```rust,name=lsm_storage.rs
/// Put a key-value pair into the storage by writing into the current memtable.
pub fn put(&self, _key: &[u8], _value: &[u8]) -> Result<()> {
    // 不能再简化成一行，需要把读锁拆出来了
    let guard = self.state.read();
    let res = guard.memtable.put(_key, _value);
    if res.is_ok() {
        // 从 UT case 中得知需要做一下判断
        if guard.memtable.approximate_size() > self.options.target_sst_size {
            // 参数不会传？直接照抄 force_flush 方法
            return self.force_freeze_memtable(&self.state_lock.lock());
        }
    }
    res
}
```
运行测试——坏了，怎么卡住了？卡住肯定是死锁了。为什么会卡住？还记不记得 force_freeze_memtable 中拿了写锁，但是这个方法里调用 force_freeze_memtable 的时候，读锁还在作用域没释放呢。那咋办呢？只能手动 drop 了。我们把变量也改个名字吧，看的更清楚：
```rust,name=lsm_storage.rs
/// Put a key-value pair into the storage by writing into the current memtable.
pub fn put(&self, _key: &[u8], _value: &[u8]) -> Result<()> {
    let read_lock = self.state.read();
    let res = read_lock.memtable.put(_key, _value);
    if res.is_ok() {
        if read_lock.memtable.approximate_size() > self.options.target_sst_size {
            // 在拿写锁前释放读锁
            drop(read_lock);
            return self.force_freeze_memtable(&self.state_lock.lock());
        }
    }
    res
}
```
好！这时候运行测试，一切正常了！

**补充**：仔细看看文档，能够看到了这段
>Because there could be multiple threads getting data into the storage engine, force_freeze_memtable might be called concurrently from multiple threads. You will need to think about how to avoid race conditions in this case.

原来还需要我们处理并发问题，并且测试测不出来。好吧，我就勉为其难地先加一个双重检查锁：
```rust,name=lsm_storage.rs
/// Put a key-value pair into the storage by writing into the current memtable.
pub fn put(&self, _key: &[u8], _value: &[u8]) -> Result<()> {
    let read_lock = self.state.read();
    let res = read_lock.memtable.put(_key, _value);
    if res.is_ok() {
        if read_lock.memtable.approximate_size() > self.options.target_sst_size {
            let state_lock = self.state_lock.lock();
            if read_lock.memtable.approximate_size() > self.options.target_sst_size {
                drop(read_lock);
                return self.force_freeze_memtable(&state_lock);
            }
        }
    }
    res
}
```
*此处有个问题，如果相同的键多次写入，approximate_size 会一直增加，但实际 map 里的内容并没增加，内存表大小的估值就不准了。从文档看这是符合预期的，不过我没太理解，等我以后想通了再回来修改这一段。*

#### Task4：获取路径
最后一个 Task 相对就比较简单了，其实就是说得能拿到历史数据。这是显然的：当前的 get 方法只从 memtable 里查找键，完全没考虑 immemtables，这哪儿行呢？写进去的数一旦满了落盘了就不作数了是吧？

所以就修改 get 吧。原来的结构肯定不能要了，还是优先闭包，这里显然要用 Some。把 memtable 的改造好后，如法炮制，写个遍历就好了。注意，我们之前向 imm_memtables 写的时候就是从前往后写的，所以读的时候也只需按顺序读，只要读到键，就可以根据他的值是不是空来返回结果，不需要继续读剩下的 imm_memtable 了。所以这么写完全 OK：
```rust,name=lsm_storage.rs
/// Get a key from the storage. In day 7, this can be further optimized by using a bloom filter.
pub fn get(&self, _key: &[u8]) -> Result<Option<Bytes>> {
    let read_lock = self.state.read();
    if let Some(bytes) = read_lock.memtable.get(_key) {
        return Ok((!bytes.is_empty()).then(|| bytes.clone()));
    }
    for imm_memtable in &read_lock.imm_memtables {
        if let Some(bytes) = imm_memtable.get(_key) {
            return Ok((!bytes.is_empty()).then(|| bytes.clone()));
        }
    }
    Ok(None)
}
```
很好，最后一个测试也通过了！

***
连做题带写博客，截至目前已经花了我 5 个小时，这才只是 Day1！没办法，毕竟还不熟悉，慢慢来吧。这篇年内能更新完就不错了……毕竟我还得干活，还得打游戏呢。我在考虑把这篇文章拆开，每周甚至每天单独作一篇文章，但为了看着方便，先不拆了，除非以后打开文章特别卡，那到时再说。

文档最后还有几个思考题，我顶不住了，先下线了。以后再补充吧。最下面甚至还有个 Bonus Tasks，讲道理这东西适合二刷的时候再搞，跳过跳过。

### Day2：Merge Iterator
Day2 还是内存表。没办法，谁让这是基础呢。这波需要结合迭代器，开始上强度了！

#### Task1：内存表迭代器
这个任务需要实现 mem_table.rs 中 StorageIterator 的四个方法，以及 scan 方法。

一上来就喂了一大堆新东西进来，光一个自引用的宏就看了半天。此时我有一句 xxx 不知当讲不当讲。
![啥啥啥](https://www.diydoutu.com/bq/2032.gif)

先挑软柿子来。很明显这个 value 最简单，直接返回 item 的值就行。
```rust,name=mem_table.rs
fn value(&self) -> &[u8] {
    self.borrow_item().1.as_ref()
}
```
你可能想问 borrow_item 这个方法是哪里来的，其实我也不知道，打了`self.`后编译器弹出的提示里面第一个方法就是这个。不过猜也能猜到，这个方法肯定是`#[self_referencing]`这个宏，也就是文档中提到的 ouroboros 库带进来的。与之相对 borrow_map 这个方法也是有的。我贴一段 Kimi 生成的回答：
>在 ouroboros（或类似库）里，只要字段被标记为 #[borrows(...)] 或 #[covariant] / #[not_covariant]，宏就会为 每一个普通字段（即没有被 #[borrows] 的字段）额外生成一组访问器：  
>borrow_<字段名>() —— 只读借用  
>borrow_<字段名>_mut() —— 可变借用  
>into_<字段名>() —— 所有权转移  
>笔者：那也就是除了 iter，另两个都会有

同理，key 和 is_valid 也非常简单：
```rust,name=mem_table.rs
fn key(&self) -> KeySlice {
    KeySlice::from_slice(self.borrow_item().0.as_ref())
}

fn is_valid(&self) -> bool {
    // 直接判断 self.borrow_item().0 是否为空也是一样的
    !self.key().is_empty()
}
```
next 就麻烦了，怎么让他走到下一位啊？其实思路还是很简单的，难点主要在 rust 语言上了，这写的太费劲了！
```rust,name=mem_table.rs
fn next(&mut self) -> Result<()> {
    let next = self.with_iter_mut(|iter| {
        // 先把当前的迭代器取出来，并让它往后走一步
        iter.next()
            // 把 key/value 拿出来
            .map(|e| (e.key().clone(), e.value().clone()))
            // 没有更多元素了，把 item 设成空 Bytes，以便能够让 is_valid 返回 false
            .unwrap_or_else(|| (Bytes::new(), Bytes::new()))
    });
    // 更新 item
    self.with_mut(|iter| *iter.item = next);
    Ok(())
}
```
最后还得写 scan。首先映入眼帘的就是这个方法的入参：Bound 是什么玩意？仔细看了下测试 case 才明白，原来 scan 的时候是要获取一个范围内的键值对的，这个 Bound 就是范围的上下界。然后这个方法返回的就是我们刚刚写的 MemTableIterator，检索一番后发现——坏了，这回没得抄了！

那就直接写吧，先不管边界，建一个结构体出来——当我这么想的时候，怎么写怎么不对劲，结构体怎么创建不出来了？actual_data 是个什么玩意？仔细看了下编译器的提示，原来又是 ouroboros 在搞鬼！ouroboros 的过程宏会自动给结构体生成一个 Builder，这时候必须用 Builder 了。先搞个框架出来。
```rust,name=mem_table.rs
pub fn scan(&self, _lower: Bound<&[u8]>, _upper: Bound<&[u8]>) -> MemTableIterator {
    let builder = MemTableIteratorBuilder { 
        map: self.map,
        iter_builder: todo!(),
        item: (Bytes::new(), Bytes::new()),
    };
    let iter = builder.build();
    iter
}
```
不出意外，iter_builder 的构造也在搞事情。这个自引用确实太烦了。那到底怎么实现呢？我们知道 SkipMapRangeIter 是一个 Range
```rust,name=mem_table.rs
type SkipMapRangeIter<'a> =
    crossbeam_skiplist::map::Range<'a, Bytes, (Bound<Bytes>, Bound<Bytes>), Bytes, Bytes>;
```
那我们也写一个 Range 就行了，参数就是传进来的上下界。
```rust,name=mem_table.rs
iter_builder: |map| {
    map.range((map_bound(_lower), map_bound(_upper)))
},
```
此外还得把 map 改成 clone 的。

能直接返回吗？显然不行！item 表示当前键值对，现在还是空的呢。写吧！这段很简单，和 next 方法完全一样。完整方法如下：
```rust,name=mem_table.rs
/// Get an iterator over a range of keys.
pub fn scan(&self, _lower: Bound<&[u8]>, _upper: Bound<&[u8]>) -> MemTableIterator {
    let builder = MemTableIteratorBuilder {
        map: self.map.clone(),
        iter_builder: |map| map.range((map_bound(_lower), map_bound(_upper))),
        item: (Bytes::new(), Bytes::new()),
    };
    let mut iter = builder.build();
    let first = iter.with_iter_mut(|iter| {
        iter.next()
            .map(|e| (e.key().clone(), e.value().clone()))
            .unwrap_or_else(|| (Bytes::new(), Bytes::new()))
    });
    iter.with_mut(|iter| *iter.item = first);
    iter
}
```
好！两个测试都过了。Day2 难度猛增，写到这我都有点想放弃了，哎，太难了！万恶的 ouroboros！要是以后每节都来个新的第三方库，我可受不了啊！

{{ admonition(type="warning", title="注意", text="施工中") }}
```rust,name=

```
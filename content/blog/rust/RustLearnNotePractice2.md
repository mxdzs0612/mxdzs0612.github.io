+++
title = "Rust 项目实战（二）：Mini-LSM（更新至1-2）"
slug = "rust_learn_note_prac_2"
date = 2025-08-13
updated = 2025-08-25
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
> 这玩意的官方答案基本上就不是给人看的，尤其是对于 Rust 新手来说，里面存在各种令人头晕目眩的操作，谁会能想到这样写啊！民间版就易读很多。其实不对答案也没问题，迟神给每个子任务都写了测试，只要测试能跑过就说明（至少在当前阶段）代码是 ok 的，不过可能通不过后续测试，需要还债。
> 
> 个人推荐的几个社区实现：[redixhumayun](https://github.com/redixhumayun/mini-lsm/) [Jerry-GK](https://github.com/Jerry-GK/MyLSM) [ping-jz](https://github.com/ping-jz/mini-lsm-solution)

[我的代码实现](https://github.com/mxdzs0612/mini-lsm)，我会尽量把每个子任务的 commit 拆开，方便回溯。

***
![All](https://skyzh.github.io/mini-lsm/lsm-tutorial/00-full-overview.svg)

这是项目的整体结构，共三周，每周 7 小节，分为 6 个 大活 + 最后的 1 个甜点时间，可以看到还是非常强大和全面的。正因如此，我更新速度预计会很慢，年内能写完就不错了。

本文暂时不会对 LSM 进行介绍，基本概念和术语可以参考万恶之源 [RocksDB Wiki](https://github.com/facebook/rocksdb/wiki/RocksDB-Overview)。

本文也不会复述题目内容，因此不能代替项目文档。我默认读者阅读本文前已经看过对应章节的项目文档（下文简称“文档”）。包括环境安装、测试怎么跑等基础内容在本文中也不再赘述，请自行从文档里面找。

每节文档下面都会有个思考题，我姑且都会做一下，但由于没有参考答案我不确定做的对不对，这部分内容均会折叠起来。欢迎留言交流探讨。

## Week 1：Mini-LSM
![week1](https://skyzh.github.io/mini-lsm/lsm-tutorial/week1-overview.svg)

在 Week 1，我们将构建存储格式，系统的读写路径，并实现一个可用的基于 LSM 树的键值存储。本章结束时，项目理应具备除真正在磁盘上的持久化外，一个 LSM 树的全部必备功能。

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
>其实理论上应该把变量名开头的下划线去掉，因为 Rust 中变量以下划线开头是为了消除变量未使用的告警，但给方法添加内容后这些变量已经是使用了的，而非未使用的。不过我直到写完 day1 才意识到这一点，算了就这样吧，不改问题也不大。

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

什么锁？怎么获取锁？什么都不知道怎么办？一番上下翻找后，我发现内置的 MiniLsm 这个结构体默认已经实现了一些方法，主要来看 force_flush 这个方法：
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
具体在做什么不用管（应该能猜到和落盘有关），我们只需要知道它调用的`self.inner`就是我们正在开发的 LsmStorageInner。哦，他就是在获取锁，这样就有一个大概思路了。如法炮制，还是先来实现 get。这里会比任务 1 中复杂一些，主要在于需要获取锁，然后返回的结果又用 Result 包了一下。其实也没复杂多少，顺手的事：
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

我们先来看看他说的这个 next_sst_id：
```rust,name=lsm_storage.rs
pub(crate) fn next_sst_id(&self) -> usize {
    self.next_sst_id
        .fetch_add(1, std::sync::atomic::Ordering::SeqCst)
}
```
next_sst_id 是一个 AtomicUsize。注意到 approximate_size 也是一个 AtomicUsize，这就是在提示我们 approximate_size 仿照这个改就可以了。此外，这段话还告诉我们 imm_memtables 需要头插，那没办法了，只能用 insert 了：
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

然后再来搞被动调用的，也就是满了自动冻结。这需要我们处理增/删内存表元素的代码，每修改一个，就要做一次判断，看有没有达到配置的上限。
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
运行测试——坏了，怎么卡住了？卡住肯定是死锁了。为什么会卡住？还记不记得 force_freeze_memtable 中拿了写锁，但是这个方法里调用 force_freeze_memtable 的时候，读锁还在作用域没释放呢。那咋办呢？只能手动 drop 了（如果想不到要 drop，看一眼本文最后的思考题，里面有提示）。我们把变量也改个名字吧，看的更清楚：
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

原来还需要我们处理并发问题，并且测试测不出来。好吧，我就勉为其难地先加一个最简单的双重检查锁：
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
此外还有个问题，如果相同的键多次写入，approximate_size 会一直增加，但实际 map 里的内容并没增加，内存表大小的估值就不准了。从文档看这是符合预期的，不过这也还算合理，毕竟内存表存的东西实实在在增加了。

#### Task4：读内存表
最后一个 Task 相对就比较简单了，其实就是说得能拿到之前批次的数据。这是显然的：当前的 get 方法只从 memtable 里查找键，完全没考虑 immemtables，这哪儿行呢？写进去的数一旦满了落盘了就不作数了是吧？

所以就修改 get 吧。原来的结构肯定不能要了，还是优先闭包，这里显然要用 Some。把 memtable 的改造好后，如法炮制，针对 Vector 写个遍历就好了。注意，我们之前向 imm_memtables 写的时候就是从前往后写的，所以读的时候也只需按顺序读，只要读到键，就可以根据他的值是不是空来返回结果，不需要继续读剩下的 imm_memtable 了（因为就算读到读的也是过期数据，没有任何价值）。所以这么写完全 OK：
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
<details>
<summary>思考题</summary>
<b>Test Your Understanding</b>

- Why doesn't the memtable provide a delete API?
  - 原因很多，比如说有这样一个场景：插入——更新——删除，如果真的物理删除了，是有可能去读取到第一次插入的那个旧值的，写个空值就能规避这种问题。此外，LSM 树常常也是需要版本控制的，用写空值替代物理删除既省事效率又高，比直接删除更好。
- Does it make sense for the memtable to store all write operations instead of only the latest version of a key? For example, the user puts a->1, a->2, and a->3 into the same memtable.
  - 对同一个内存表，完全不需要。因为最终起作用的值还是只有最后写入的那个，在单个内存表里面的数据是同一批写入的，也没有回溯或者版本控制的要求。
- Is it possible to use other data structures as the memtable in LSM? What are the pros/cons of using the skiplist?
  - 有肯定是有的，比如 RocksDB 还可以选哈希跳表，用哈希链表和 B+ 树应该也可以。
  - 跳表优势在于读写性能都是 O(log n)，支持 LSM 常见的范围查询（比如下一节中大量存在的 Bound），还具备并发写的能力。劣势在于有额外维护索引的成本，然后读可能会比那帮 O(1) 的家伙慢一些，且性能不稳定，极端情况下性能可能会退化。
- Why do we need a combination of state and state_lock? Can we only use state.read() and state.write()?
  - 组合的优势在于读取 state 通常不需要获取锁，这样有助于提高并发性能，减少锁竞争。
  - 如果能保证复合操作原子性的话理论上应该也可以，但通常不能。
- Why does the order to store and to probe the memtables matter? If a key appears in multiple memtables, which version should you return to the user?
  - 因为新数据会覆盖旧的，内存表的顺序就是新旧的保证。
  - 拿最后写入，也就是版本号最大的那个。
- Is the memory layout of the memtable efficient / does it have good data locality? (Think of how Byte is implemented and stored in the skiplist...) What are the possible optimizations to make the memtable more efficient?
  - 没有，这是跳表这个数据结构决定的。
  - Byte 本质上是指针，可以考虑预分配连续的内存。
- So we are using parking_lot locks in this course. Is its read-write lock a fair lock? What might happen to the readers trying to acquire the lock if there is one writer waiting for existing readers to stop?
  - 显然是非公平读写锁。
  - 可能会优先获取到读锁，导致写饥饿。
- After freezing the memtable, is it possible that some threads still hold the old LSM state and wrote into these immutable memtables? How does your solution prevent it from happening?
  - 不可能。
  - 在冻结内存表之前已经获取到写锁了，而冻结后 imm_memtables 是不可变的。
- There are several places that you might first acquire a read lock on state, then drop it and acquire a write lock (these two operations might be in different functions but they happened sequentially due to one function calls the other). How does it differ from directly upgrading the read lock to a write lock? Is it necessary to upgrade instead of acquiring and dropping and what is the cost of doing the upgrade?
  - 确实有，升级可能导致死锁。
  - 没有必要吧，升级需要重新做状态检查。
</details>

文档最后还有几个思考题，因为没有答案不确定对不对，我把我的想法折叠起来写在了上面，题目就不翻译了，感兴趣的读者可以自行点击展开阅读。最下面甚至还有个 Bonus Tasks，讲道理这东西适合二刷的时候再搞，跳过跳过～

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
你可能想问 borrow_item 这个方法是哪里来的，其实我也不知道，打了`self.`后编辑器自动弹出的提示里面第一个方法就是这个。不过猜也能猜到，这个方法肯定是`#[self_referencing]`这个宏，也就是文档中提到的 ouroboros 库带进来的。与之相对 borrow_map 这个方法也是有的。我贴一段 Kimi 生成的回答：
>在 ouroboros（或类似库）里，只要字段被标记为 #[borrows(...)] 或 #[covariant] / #[not_covariant]，宏就会为 每一个普通字段（即没有被 #[borrows] 的字段）额外生成一组访问器：  
>borrow_<字段名>() —— 只读借用  
>borrow_<字段名>_mut() —— 可变借用  
>into_<字段名>() —— 所有权转移  
>笔者：那也就是除了 iter，另两个都会有

与之相似的还有 with_xxx，同样包括带 mut 的，读者自己看吧。

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
        // 先把当前的迭代器取出来，并迭代到它的下一个值
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
最后还得写 scan。首先映入眼帘的就是这个方法的入参：Bound 是什么玩意？仔细看了下测试 case 才明白，原来 scan 的时候是要获取一个范围内的键值对的，这个 Bound 就是范围的上下界。然后这个方法返回的就是我们刚刚写的 MemTableIterator，检索一番后发现——坏了，没有类似代码，这回没得抄了！

那就直接硬写吧，先不管边界，建一个结构体出来——当我这么想的时候，怎么写怎么不对劲，结构体怎么创建不出来了？actual_data 是个什么玩意？仔细看了下编译器的提示，原来又是 ouroboros 在搞鬼！ouroboros 的过程宏会**自动**给结构体生成一个 Builder，想不用都不行。先搞个框架出来。
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
不出意外，iter_builder 的构造也在搞事情。这个自引用确实太烦了。那到底怎么实现呢？我们知道这是一个 SkipMapRangeIter，而 SkipMapRangeIter 是一个 Range：
```rust,name=mem_table.rs
type SkipMapRangeIter<'a> =
    crossbeam_skiplist::map::Range<'a, Bytes, (Bound<Bytes>, Bound<Bytes>), Bytes, Bytes>;
```
那我们也新建一个 Range 就行了，参数就是传进来的上下界。
```rust,name=mem_table.rs
iter_builder: |map| {
    map.range((map_bound(_lower), map_bound(_upper)))
},
```
此外还得把 map 改成 clone 的。这时候编译就通过了。但能像现在这样直接返回吗？显然不行！item 表示当前键值对，现在还是空的呢。写吧！这段很简单，因为这个时候，Iterator 已经初始化好了，next 拿到的下一个值就是 lower 限定后的、迭代器里的第一个值，所以此处和前面 next 方法完全一样，抄下来即可。完整代码如下：
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
好！两个测试都过了。

Day2 难度猛增，写到这我都有点想放弃了，哎，太痛苦了！万恶的 ouroboros！要是以后每节都来个新的第三方库，我可顶不住啊！

#### Task2：归并迭代器
这节我们要实现的还是这四兄弟，以及一个 create 方法。仔细一看，哦，原来 StorageIterator 是个特质，之前还没留意。

这节不能愣头写，得先知道要我们实现什么。仔细阅读文档，能够得知这里需要合并多个内存表，返回最新的那个值。里面还用了二叉堆排序输出，

前面三个都很简单，无非就是拿的东西变成了被 Option 包起来了的特质泛型，多拆几层就是了：
```rust,name=iterators/merge_iterator.rs
fn key(&self) -> KeySlice {
    self.current.as_ref().unwrap().1.key()
}

fn value(&self) -> &[u8] {
    self.current.as_ref().unwrap().1.value()
}

fn is_valid(&self) -> bool {
    self.current.as_ref().map_or(false, |iter| iter.1.is_valid())
}
```
这里还用到了[错误处理](../rust-learn-note-adv-10/#dai-mo-ren-zhi-de-ying-she)一节中的映射方法 map_or，挺好。

困难的还是 next。我们先来写 create，把困难留在最后。create 比 task1 的还要容易点，老样子，先搭个框子：
```rust,name=iterators/merge_iterator.rs
pub fn create(iters: Vec<Box<I>>) -> Self {
    let mut merge_iterator = MergeIterator {
        iters: iters
            .into_iter()
            .enumerate()
            .map(|(idx, boxed)| HeapWrapper(idx, boxed))
            .collect::<BinaryHeap<_>>(),
        current: None,
    };
    merge_iterator.current = merge_iterator.iters.pop();
    merge_iterator
}
```
iters 就是把入参的 Vec 转换成 BinaryHeap，current 代表当前已经读取了的最小键元素，构造的时候就是从创建好的 iter 里取一个出来，当然得先创建好。这段 iters 的构造是 Kimi 写的，我让他帮我搓一个`Vec<Box<I>>`转换成`BinaryHeap<HeapWrapper<I>>`（其中后者的结构体是`struct HeapWrapper<I: StorageIterator>(pub usize, pub Box<I>)`）的函数，就是这个效果了，不过用到的这几个方法也都是迭代器的常用方法。看上去很合理，排序也已经由堆来实现了，但是不是还缺点什么？看文档怎么说：
>Note that you will need to handle errors (i.e., when an iterator is not valid) and ensure that the latest version of a key-value pair comes out.

懂了，还需要给迭代器加个 filter，把无效的值过滤掉。完整代码如下：
```rust,name=iterators/merge_iterator.rs
pub fn create(iters: Vec<Box<I>>) -> Self {
    let mut merge_iterator = MergeIterator {
        iters: iters
            .into_iter()
            .filter(|i| i.is_valid())
            .enumerate()
            .map(|(idx, boxed)| HeapWrapper(idx, boxed))
            .collect::<BinaryHeap<_>>(),
        current: None,
    };
    merge_iterator.current = merge_iterator.iters.pop();
    merge_iterator
}
```
最后终于到 next 了，开始写之前必须弄明白这个方法需要做什么。简单讲，就是要合并多个内存表（也就是 task1 实现的迭代器），返回“下一个”不重复的最小键，并且当出现相同键时优先选取索引更小的那个迭代器里的值。二叉堆 BinaryHeap 用于维持各子迭代器当前元素的最小值队列，这个二叉堆存了一个 HeapWrapper(idx, iter)。代码里能看到
```rust,name=iterators/merge_iterator.rs
impl<I: StorageIterator> Ord for HeapWrapper<I> {
    fn cmp(&self, other: &Self) -> cmp::Ordering {
        self.1
            .key()
            .cmp(&other.1.key())
            .then(self.0.cmp(&other.0))
            .reverse()
    }
}
```
也就是说比较规则是按 iter 的 key 升序；若相等则按 idx 升序。最后那个 reverse 是因为 Rust 的 BinaryHeap 是大根堆。这里堆中的元素就是待读取的候选，current 则是已经读出来的的元素。

于是，我们需要做的，就是把堆顶的元素和 current 进行比较，清理所有无效和旧的值，并找出新的最小的那个 current。举个例子，I0: [1, 3], I1: [1, 2], I2: [2, 3]，初始 current = (I0, 1)。next 时在堆顶的 I1 发现了 key=1，等于current 的 1，则推进到 2；然后推进 I0 到 3 再压回堆；新的 current 变为最小的 2（来自 I1 或 I2 中索引更小者）。这样键 1 只输出一次。后续其他值也是同理。

最后，文档要求我们不能用`?`，必须自己处理错误，否则会有个测试过不去。综上，我的实现如下：
```rust,name=iterators/merge_iterator.rs
fn next(&mut self) -> Result<()> {
    // 必须这么写，不能直接去调前面实现的 key 方法，否则会报同时可变不可变引用的错误
    let k = self.current.as_ref().unwrap().1.key();
    // 参考文档用 peek_mut 循环取值
    while let Some(mut iter) = self.iters.peek_mut() {
        // 不是我喜欢的类型，直接拒绝（划掉）
        // 不是可用的迭代器，直接弹出，因为已经走到尾了
        if !iter.1.is_valid() {
            PeekMut::pop(iter);
            continue;
        }
        // iters 是已经在堆里排过序的
        // 出现这种情况意味着堆里的最小 key 已经比 current 大了，于是退出循环
        if iter.1.key() > k {
            break;
        }
        // 调用 next 前进一步并自己处理错误
        // 不要用 ?，听人劝吃饱饭
        // 调用 next 是为了去重，当前键已经被 current 拿到了，理论上这个时候取到的 iter.1.key() 应该都等于 k，不会有更小的了
        // 也就是说，这是在弹出其他迭代器的旧数据
        if let Err(e) = iter.1.next() {
            PeekMut::pop(iter);
            return Err(e);
        }
        // 堆空了的时候，即走到尾部的时候，堆里会残留一个空白的无效元素
        // 再次 pop 以保证把已经耗尽了的迭代器从堆里清出去
        if !iter.1.is_valid() {
            PeekMut::pop(iter);
        }
    }
    let mut cur = self.current.take().unwrap();
    // 调用 next 并自己处理错误 + 1
    // current 的迭代器也前进一位
    if let Err(e) = cur.1.next() {
        self.current = self.iters.pop();
        return Err(e);
    }
    // 如果没出错，就把 cur 重新放回堆里，参与下一轮选最小 current 的竞争
    if cur.1.is_valid() {
        self.iters.push(cur);
    }
    // 即从堆中取出当前全局最小的 iter 作为新的 current
    self.current = self.iters.pop();
    Ok(())
}
```
这写的是不是很过瘾？然而，单这一节写代码花了我周末的一下午，写博客又花了一中午，折磨……

#### Task3：融合迭代器
本节最痛苦的部分已经过去了，剩下两个任务都相对轻松一些。

FusedIterator 是 Rust 的一个特质，当然这节并没用到特质，而是自己实现了一个。这节要写的还是这四个方法，轻车熟路了，轻轻松松。

既然文档说测试不测 LsmIterator，这里就先应付一下：
```rust,name=lsm_iterator.rs
fn is_valid(&self) -> bool {
    self.inner.is_valid()
}

fn key(&self) -> &[u8] {
    self.inner.key().raw_ref()
}

fn value(&self) -> &[u8] {
    self.inner.value()
}

fn next(&mut self) -> Result<()> {
    self.inner.next()
}
```
FusedIterator 还是得稍微认真一点，至少 has_errored 这个参数不能 never read 吧：
```rust,name=lsm_iterator.rs
fn is_valid(&self) -> bool {
    !self.has_errored && self.iter.is_valid()
}

fn key(&self) -> Self::KeyType<'_> {
    self.iter.key()
}

fn value(&self) -> &[u8] {
    self.iter.value()
}

fn next(&mut self) -> Result<()> {
    self.iter.next()
}
```
真有这么简单吗？一运行测试果然跪了，仔细看看这节测试在干什么：
```rust,name=week1_day2.rs
let iter = MockIterator::new_with_error(
    vec![
        (Bytes::from("a"), Bytes::from("1.1")),
        (Bytes::from("a"), Bytes::from("1.1")),
    ],
    1,
);
let mut fused_iter = FusedIterator::new(iter);
assert!(fused_iter.is_valid());
assert!(fused_iter.next().is_err());
assert!(!fused_iter.is_valid());
assert!(fused_iter.next().is_err());
assert!(fused_iter.next().is_err());
```
还是没看懂？再来读读代码文件结构体上面的这段注释：
```rust,name=lsm_iterator.rs
/// A wrapper around existing iterator, will prevent users from calling `next` when the iterator is
/// invalid. If an iterator is already invalid, `next` does not do anything. If `next` returns an error,
/// `is_valid` should return false, and `next` should always return an error.
pub struct FusedIterator<I: StorageIterator> {
    iter: I,
    has_errored: bool,
}
```
next 还得进行容错，不能每调用一次就真的去取 next，迭代器不可用的时候要阻止这种行为。这里的 Result 还是 anyhow 包的，不是标准的，返回错误必须得用它的宏。虽然之前就注意到这一点了，但这个时候才第一次被恶心到。然后，调用 next 后也得再检查一下，还是上闭包。
```rust,name=lsm_iterator.rs
fn next(&mut self) -> Result<()> {
    if self.has_errored {
        return Err(anyhow!(""));
    }
    self.iter.next().inspect_err(|_| self.has_errored = true)
}
```

#### Task4：扫描
最后一步，提供一个 scan 接口给引擎调用，只需要实现这一个方法。当然，前面偷懒没写的 LsmIterator 还是得还债的。看到这个上下界，熟悉的感觉又回来了，还没忘记 Day1 的东西吧？老规矩，先搭个框子。
```rust,name=lsm_storage.rs
pub fn scan(
    &self,
    _lower: Bound<&[u8]>,
    _upper: Bound<&[u8]>,
) -> Result<FusedIterator<LsmIterator>> {
    let mut iters = vec![];
    let iter = LsmIterator::new(MergeIterator::create(iters))?;
    Ok(FusedIterator::new(iter))
}
```
iters 里面塞什么呢？肯定就是 scan 要读的东西。再结合这个上下界，很明显，这里面要传的就是 Day1 写的内存表和不可变内存表。差不多就是这样：
```rust,name=lsm_storage.rs
let read_lock = self.state.read();
iters.push(Box::new(read_lock.memtable.scan(_lower, _upper)));
iters.append(&mut read_lock.imm_memtables.clone().into_iter().map(|i| Box::new(i.scan(_lower, _upper))).collect());
```
这样就行了吗？并不行！跑测试一看，怎么被删掉的 key 还在里面呢？说明漏东西了。仔细一想，前面刚实现的 next 方法还没用呢。自然而然，要对这个 iter 使用了。既然是删掉的还在，那就是要把键还在、值没了的条目过滤掉。完整代码如下：
```rust,name=lsm_storage.rs
pub fn scan(
    &self,
    _lower: Bound<&[u8]>,
    _upper: Bound<&[u8]>,
) -> Result<FusedIterator<LsmIterator>> {
    let mut iters = vec![];
    let read_lock = self.state.read();
    iters.push(Box::new(read_lock.memtable.scan(_lower, _upper)));
    iters.append(
        &mut read_lock
            .imm_memtables
            .clone()
            .into_iter()
            .map(|i| Box::new(i.scan(_lower, _upper)))
            .collect(),
    );
    let mut iter = LsmIterator::new(MergeIterator::create(iters))?;
    while !iter.key().is_empty() && iter.value().is_empty() {
        // 这个问号加不加都行，最好还是加
        iter.next()?;
    }
    Ok(FusedIterator::new(iter))
}
```
然后，该还债了。LsmIterator 的 next 方法也要做一波判断，逻辑是一样的：
```rust,name=lsm_iterator.rs
fn next(&mut self) -> Result<()> {
    let res = self.inner.next();
    if res.is_ok() && !self.key().is_empty() && self.value().is_empty() {
        self.next()
    } else {
        res
    }
}
```
运行测试，怎么还是不行？仔细一看报错，原来之前 MergeIterator 的实现也有问题！债越来越多了。

报错是代码中对一个 None 值调用了 unwrap()。这也有判断！MergeIterator 的几兄弟全有这个问题，那就全都要改了。用 or 方法在为空时抛个默认值而非报错就行：
```rust,name=iterators/merge_iterator.rs
fn key(&self) -> KeySlice {
    self.current
        .as_ref()
        .map(|iter| iter.1.key())
        .unwrap_or_else(|| KeySlice::from_slice(&[]))
}

fn value(&self) -> &[u8] {
    self.current
        .as_ref()
        .map(|iter| iter.1.value())
        .unwrap_or(&[])
}

fn next(&mut self) -> Result<()> {
    let Some(iter) = self.current.as_ref() else {
        return Ok(());
    };
    let k = iter.1.key();
    // ...
}
```
行了，终于能通过测试了。

<details>
<summary>思考题</summary>
<b>Test Your Understanding</b>

- What is the time/space complexity of using your merge iterator?
  - O(log N) （指 next 方法）
  - O(n)
- Why do we need a self-referential structure for memtable iterator?
  - 方便处理生命周期，保证迭代器不会比它引用的数据存活更久
- If a key is removed (there is a delete tombstone), do you need to return it to the user? Where did you handle this logic?
  - 不用，会被跳过，参考 scan 方法
- If a key has multiple versions, will the user see all of them? Where did you handle this logic?
  - 不能，只能看到最新的，参考 next 方法
- If we want to get rid of self-referential structure and have a lifetime on the memtable iterator (i.e., MemtableIterator<'a>, where 'a = memtable or LsmStorageInner lifetime), is it still possible to implement the scan functionality?
  - 不确定，感觉也可以，就是很麻烦
- What happens if (1) we create an iterator on the skiplist memtable (2) someone inserts new keys into the memtable (3) will the iterator see the new key?
  - 不行，迭代器还看不到没冻结的新写入的数据，需要冻结以后才行
- What happens if your key comparator cannot give the binary heap implementation a stable order?
  - 堆的排序就不准了，没法确定返回的是新数据还是旧数据
- Why do we need to ensure the merge iterator returns data in the iterator construction order?
  - 为了读的时候能先拿到新数据
-Is it possible to implement a Rust-style iterator (i.e., next(&self) -> (Key, Value)) for LSM iterators? What are the pros/cons?
  - 这题不会……感觉不太行
- The scan interface is like fn scan(&self, lower: Bound<&[u8]>, upper: Bound<&[u8]>). How to make this API compatible with Rust-style range (i.e., key_a..key_b)? If you implement this, try to pass a full range .. to the interface and see what will happen.
  - 上下界的核心就是 map.range，把这段改造一下也能适配，比方说换成有序列表之类的。
- The starter code provides the merge iterator interface to store `Box<I>` instead of I. What might be the reason behind that?
  - Box 最重要的作用就是大对象出栈入堆，容器混用，以及好管理特质对象。好处无非就是这几点。
</details>

这节的思考题也不太简单，不过只要能写出上面的代码，对内存表应该已经有很深的理解了，区区这种问题应该难不倒吧。Bonus Task 就挺难了，短时间估计做不出来，以后再说。

### Day3：Block
这一节终于不是内存表了，要实现的是块。所谓的块，就是 LSM 树在磁盘上的最小存储单元。也就是说，在 LSM 树中，当内存表满了的时候，数据就会以块的形式落盘。

#### Task1：块构造器
这一节终于到我熟悉的领域了，之前扣 parquet 编码的日子还历历在目。这回做的事情也差不多，写就完事了。存储到头来都是这种细活。

这个 Task 需要实现 6 个方法，先来写 builder 里面的。很明显最难的是 add，先把它跳过，其他 3 个都是送分的。
```rust,name=block/builder.rs
impl BlockBuilder {
    /// Creates a new block builder.
    pub fn new(block_size: usize) -> Self {
        Self {
            offsets: vec![],
            data: vec![],
            block_size: block_size,
            first_key: KeyVec::new(),
        }
    }

    /// Adds a key-value pair to the block. Returns false when the block is full.
    /// You may find the `bytes::BufMut` trait useful for manipulating binary data.
    #[must_use]
    pub fn add(&mut self, key: KeySlice, value: &[u8]) -> bool {
        unimplemented!()
    }

    /// Check if there is no key-value pair in the block.
    pub fn is_empty(&self) -> bool {
        self.data.is_empty()
    }

    /// Finalize the block.
    pub fn build(self) -> Block {
        Block {
            data: self.data,
            offsets: self.offsets,
        }
    }
}
```
接下来来攻克 add。这里比较麻烦的是有一个 block_size，也就是需要处理大小相关的东西。first_key 是什么东西？搜索半天单测和代码后发现目前好像没什么用，但是文档中有这句话：
>The block has a size limit, which is target_size. Unless the first key-value pair exceeds the target block size, you should ensure that the encoded block size is always less than or equal to target_size. (In the provided code, the target_size here is essentially the block_size)

注意这个 Unless。这个的意思是说，无论如何至少也得能写进去一组键值对，也就是说第一组键值对不受 block_size 限制。哦，first_key 就是用来存这个的！当 first_key 有内容的时候，就说明现在 add 的不是第一组键值对了！需要判断 block_size 了！

那么，一个 block 究竟有多长呢？文档中写到块长这样
```
----------------------------------------------------------------------------------------------------
|             Data Section             |              Offset Section             |      Extra      |
----------------------------------------------------------------------------------------------------
| Entry #1 | Entry #2 | ... | Entry #N | Offset #1 | Offset #2 | ... | Offset #N | num_of_elements |
----------------------------------------------------------------------------------------------------
```
也就是说，块大小就等于 entry 数量（也就是 data 长度，因为 data 是 u8）加上 Offset 数量 * 2（因为 Offset 是 u16）再加上 Extra 长度（一个 u16 大小，固定为 2）。于是我们开始写代码：
```rust,name=block/builder.rs
/// Adds a key-value pair to the block. Returns false when the block is full.
/// You may find the `bytes::BufMut` trait useful for manipulating binary data.
#[must_use]
pub fn add(&mut self, key: KeySlice, value: &[u8]) -> bool {
    // 根据提示，我们加一个 assert
    assert!(!key.is_empty());
    if self.first_key.is_empty() {
        // 吐槽一下，KeyVec 的这几个接口太难用了，只能转这么一遭
        self.first_key = KeyVec::from_vec(key.raw_ref().to_vec());
    } else if self.block_size
        // 最后别忘了还要加上本次要写的这一对
        < self.data.len() + self.offsets.len() * 2 + 2 + key.len() + value.len()
    {
        return false;
    }
    // 肯定要先写 offset，表示这次写入的字段的起始位置
    self.offsets.push(self.data.len() as u16);
    // 我写完才发现上面提示了可以用 bytes::BufMut，又是个第三方库，确实好用，可以直接 put_u16
    // 不过写都写了，rust 自带的 extend_from_slice 也能用，以后有问题再改
    // 写入 2 字节的 key 长度，to_be_bytes 表示大端写入
    self.data.extend_from_slice(&key.len().to_be_bytes());
    // 写入 key
    self.data.extend_from_slice(&key.raw_ref());
    // 写入 2 字节的 value 长度
    self.data.extend_from_slice(&value.len().to_be_bytes());
    // 写入 value
    self.data.extend_from_slice(&value);

    true
}
```
然后处理编码解码。注意 encoded 返回的 Bytes 就是前面那个注释里的特质的第三方库，这次想不用都不行了。

这次的核心还是块的那张图，就是个纯手搓，实际上没任何难度。详情看注释吧。
```rust,name=block.rs
impl Block {
    /// Encode the internal data to the data layout illustrated in the course
    /// Note: You may want to recheck if any of the expected field is missing from your output
    pub fn encode(&self) -> Bytes {
        // 第一部分即数据部分，也就是键值对已经编好码了，直接 clone 一份就行
        let mut b = self.data.clone();
        // 第二部分，offset，需要写个小循环，逐个取值然后写成 u16
        // 这个 put_u16 就是 bytes::BufMut 特质带过来的方法
        for offset in &self.offsets {
            b.put_u16(*offset);
        }
        // 第三部分，存的数据个数，看 offsets 数组大小即可
        b.put_u16(self.offsets.len() as u16);
        b.into()
    }

    /// Decode from the data layout, transform the input `data` to a single `Block`
    pub fn decode(data: &[u8]) -> Self {
        // 输入是整个的 u8 数组，因此要先找到三部分的切分点
        // 第二、第三部分的切分点非常简单，固定倒数 2 字节就是
        let offset_extra_split = data.len() - 2;
        // 解析出数据的个数
        let num = (&data[offset_extra_split..data.len()]).get_u16() as usize;
        // 根据数据个数，再找到第一、第二部分的切分点
        let data_offset_split = data.len() - 2 - 2 * num;
        // 解码 offsets
        let offset_bytes = &data[data_offset_split..offset_extra_split];
        let offsets: Vec<u16> = offset_bytes
            // 这个方法就是在把两字节拼在一起
            .chunks_exact(2)
            // 闭包用于把两字节解析成 u16
            .map(|chunk| u16::from_be_bytes(chunk.try_into().unwrap()))
            // 转成 Vec
            .collect();
        // 剩下的就是 data，怎么来的还是怎么回去，同样转成 Vec
        let data = data[0..data_offset_split].to_vec();
        Self { data, offsets }
    }
}
```
写到这里，这节的测试除了最后两个，应该都通过了。真的是轻轻松松～

{{ admonition(type="warning", title="注意", text="施工中") }}
```rust,name=

```
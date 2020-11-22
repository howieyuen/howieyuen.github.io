---
author: Yuan Hao
date: 2020-5-14
title: 哈希表
tag: [golang, map]
---

# 1. 引言

粗略的讲，Go 语言中 map 采用的是哈希查找表，由一个key通过哈希函数得到哈希值，64 位系统中就生成一个 64 bit 的哈希值，由这个哈希值将 key 对应到不同的桶（bucket）中，当有多个哈希映射到相同的的桶中时，使用链表解决哈希冲突。

## 1.1 hash函数

首先要知道的就是 map 中哈希函数的作用，go 中 map 使用 hash 作查找，就是将 key 作哈希运算，得到一个哈希值，根据哈希值确定 key-value 落在哪个bucket 的哪个 cell。golang 使用的 hash 算法和 CPU 有关，如果 CPU 支持 aes，那么使用 aes hash，否则使用 memhash。

## 1.2 数据结构

**hmap** 可以理解为 header of map 的缩写，即 map 数据结构的入口。

```go
type hmap struct {
	// map 中的元素个数，必须放在 struct 的第一个位置，因为 内置的 len 函数会从这里读取
	count     int 
	// map状态标识，比如是否在被写或者迁移等，因为map不是线程安全的所以操作时需要判断flags
	flags     uint8
	// log_2 of buckets (最多可以放 loadFactor * 2^B 个元素即6.5*2^B，再多就要 hashGrow 了)
	B         uint8  
	// overflow 的 bucket 的近似数
	noverflow uint16 
	// hash seed，随机哈希种子可以防止哈希碰撞攻击
	hash0     uint32
	// 存储数据的buckets数组的指针， 大小2^B，如果 count == 0 的话，可能是 nil
	buckets    unsafe.Pointer
	// 一半大小的之前的 bucket 数组，只有在 growing 过程中是非 nil
	oldbuckets unsafe.Pointer
	// 扩容进度标志，小于此地址的buckets已迁移完成。
	nevacuate  uintptr
	// 可以减少GC扫描，当 key 和 value 都可以 inline 的时候，就会用这个字段
	extra *mapextra // optional fields
}
```

用 **mapextra** 来存储 key 和 value 都不是指针类型的 map，并且大小都小于 128 字节，这样可以避免 GC 扫描整个 map。

```go
type mapextra struct {
    // 如果 key 和 value 都不包含指针，并且可以被 inline(<=128 字节)
    // 使用 extra 来存储 overflow bucket，这样可以避免 GC 扫描整个 map
    // 然而 bmap.overflow 也是个指针。这时候我们只能把这些 overflow 的指针
    // 都放在 hmap.extra.overflow 和 hmap.extra.oldoverflow 中了
    // overflow 包含的是 hmap.buckets 的 overflow 的 bucket
    // oldoverflow 包含扩容时的 hmap.oldbuckets 的 overflow 的 bucket
    overflow       *[]*bmap
    oldoverflow    *[]*bmap

    // 指向空闲的 overflow bucket 的指针
    nextOverflow *bmap
}
```

**bmap** 可以理解为 buckets of map 的缩写，它就是 map 中 bucket 的本体，即存 key 和 value 数据的“桶”。

```go
type bmap struct {
    // tophash 是 hash 值的高 8 位
    tophash [bucketCnt]uint8
    // 以下字段没有显示定义在bmap，但是编译时编译器会自动添加
    // keys              // 每个桶最多可以装8个key
    // values            // 8个key分别有8个value一一对应
    // overflow pointer  // 发生哈希碰撞之后创建的overflow bucket
}
```

根据哈希函数将 key 生成一个哈希值，其中**低位哈希用来判断桶位置，高位哈希用来确定在桶中哪个c ell**。低位哈希就是哈希值的低 B 位，`hmap` 结构体中的 B，比如 B 为 5，2^5=32，即该 map 有 32 个桶，只需要取哈希值的低 5 位就可以确定当前 key-value 落在哪个桶(bucket)中；高位哈希即 `tophash`，是指哈希值的高 8 bits，根据 `tophash` 来确定 key 在桶中的位置。每个桶可以存储 8 对 key-value，存储结构不是 key/value/key/value...，而是 key/key..value/value，这样可以避免字节对齐时的 padding，节省内存空间。

当不同的 key 根据哈希得到的 `tophash` 和低位 hash 都一样，发生哈希碰撞，这个时候就体现 `overflow pointer` 字段的作用了。桶溢出时，就需要把key-value 对存储在 `overflow bucket`（溢出桶），`overflow pointer` 就是指向 `overflow bucket` 的指针。如果 `overflow bucket` 也溢出了呢？那就再给 `overflow bucket` 新建一个 `overflow bucket`，用指针串起来就形成了链式结构，map 本身有 2^B 个 bucket，只有当发生哈希碰撞后才会在 bucket 后链式增加 `overflow bucket`。

# 2. map内存布局

![memory-layout-of-map.png](/golang/data-structure/map/memory-layout-of-map.png)


## 2.1 扩容

1. 装填因子是否大于 6.5

   装填因子 = 元素个数/桶个数，大于 6.5 时，说明桶快要装满，需要扩容

2. `overflow bucket` 是否太多
   ​当 bucket 的数量 < 2^15，但 `overflow bucket` 的数量大于桶数量
   ​当 bucket 的数量 >= 2^15，但 `overflow bucket` 的数量大于 2^15

**双倍扩容**：装载因子多大，直接翻倍，B+1；扩容也不是申请一块内存，立马开始拷贝，每一次访问旧的 buckets 时，就迁移一部分，直到完成，旧 bucket 被 GC 回收。

**等量扩容**：重新排列，极端情况下，重新排列也解决不了，map 成了链表，性能大大降低，此时哈希种子 hash0 的设置，可以降低此类极端场景的发生。

## 2.2 查找

1. 根据 key 计算出哈希值
2. 根据哈希值低位确定所在 bucket
3. 根据哈希值高 8 位确定在 bucket 中的存储位置
4. 当前 bucket 未找到则查找对应的 `overflow bucket`。
5. 对应位置有数据则对比完整的哈希值，确定是否是要查找的数据
6. 如果当前处于 map 进行了扩容，处于数据搬移状态，则优先从 oldbuckets 查找。

## 2.3 插入

1. 根据 key 计算出哈希值
2. 根据哈希值低位确定所在 bucket
3. 根据哈希值高 8 位确定在 bucket 中的存储位置
4. 查找该 key 是否存在，已存在则更新，不存在则插入

## 2.4 map无序

map 的本质是散列表，而 map 的增长扩容会导致重新进行散列，这就可能使 map 的遍历结果在扩容前后变得不可靠，Go 设计者为了让大家不依赖遍历的顺序，**故意在实现 map 遍历时加入了随机数**，让每次遍历的起点--即起始 bucket 的位置不一样，即不让遍历都从 bucket0 开始，所以即使未扩容时我们遍历出来的 map 也总是无序的。

# 3. 参考资料

- [Golang map底层实现](https://bettertxt.top/post/go-map/)
- [Go语言之map：map的用法到map底层实现分析](https://blog.csdn.net/chenxun_2010/article/details/103768011) 
- [Go语言map底层浅析](https://segmentfault.com/a/1190000018380327)
- [哈希表](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-hashmap/) 
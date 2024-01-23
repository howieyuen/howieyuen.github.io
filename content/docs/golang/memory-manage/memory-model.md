---
author: Yuan Hao
date: 2020-5-14
title: 内存模型
categories: [Golang]
---

Go 语言的内存模型规定了一个 goroutine 可以看到另外一个 goroutine 修改同一个变量的值的条件，
这类似 java 内存模型中内存可见性问题。当多个 goroutine 并发同时存取同一个数据时候必须把并发的存取的操作顺序化，
在 go 中可以实现操作顺序化的工具有高级的通道（channel）通信和同步原语比如 sync 包中的互斥锁（Mutex）、
读写锁（RWMutex）或者和 sync/atomic 中的原子操作。

<!--more-->

## 设计原则

### happens-before

假设 A 和 B 表示一个多线程的程序执行的两个操作。**如果 A happens-before B，那么 A 操作对内存的影响将对执行 B 的线程（且执行 B 之前）可见**。

单一 goroutine 中当满足下面条件时候，对一个变量的写操作 w1 对读操作 r1 可见：

- 读操作 r1 不是发生在写操作 w1 前
- 在读操作 r1 之前，写操作 w1 之后没有其他的写操作 w2 对变量进行了修改

多 goroutine 下则需要满足下面条件才能保证写操作 w1 对读操作 r1 可见：

- 写操作 w1 先于读操作 r1
- 任何对变量的写操作 w2 要先于写操作 w1 或者晚于读操作 r1

关于 channel 的 happens-before 在 Go 的内存模型中提到了三种情况：

1. **带缓冲**的 channel 的**发送操作** happens-before 相应 channel 的**接收操作**完成
2. **不带缓冲**的 channel 的**接收操作** happens-before 相应 channel 的**发送操作**完成
3. **关闭一个 channel** happens-before 从该 channel **接收到最后的返回值 0**

### Synchronization

#### 初始化

如果在一个 goroutine 所在的源码包 p 里面通过 import 命令导入了包 q，那么 q 包里面 go 文件的初始化方法的执行会 happens before 于包 p 里面的初始化方法执行。

#### 创建 goroutine

go 语句启动一个新的 goroutine 的动作 happen before 该新 goroutine 的运行。

#### 销毁 goroutine

一个 goroutine 的销毁操作并不能确保 happen before 程序中的任何事件。

#### 通道通信

在 go 中通道是用来解决多个 goroutines 之间进行同步的主要措施，在多个 goroutines 中，每个对通道进行写操作的 goroutine 都对应着一个从通道读操作的 goroutine。

- 在有缓冲的通道时候向通道写入一个数据总是 happen  before 这个数据被从通道中读取完成。
- 对应无缓冲的通道来说从通道接受（获取叫做读取）元素 happen before 向通道发送（写入）数据完成。
- 从容量为 C 的通道接受第 K 个元素 happen before 向通道第 k+C 次写入完成，比如从容量为 1 的通道接受第 3 个元素 happen before 向通道第 3+1 次写入完成。

### Locks

1. 对应任何 `sync.Mutex` 或 `sync.RWMutex` 类型的变量 I 来说，调用 n 次 `l.Unlock()` 操作 happen before 调用 m 次 `l.Lock()` 操作返回，其中 n<m。
2. 对任何一个 `sync.RWMutex` 类型的变量 l 来说，存在一个次数 n，调用 `l.RLock()`（读锁）操作 happens after 调用 n 次 `l.Unlock()`（释放写锁）并且相应的 `l.RUnlock()`（释放读锁） happen before 调用 n+1 次 `l.Lock()`（写锁）。

### Once

多 goroutine 下同时调用 `once.Do(f)` 时，真正执行 `f()` 函数的 goroutine， happen before 任何其他由于调用 `once.Do(f)` 而被阻塞的 goroutine 返回。

## 参考资料

- [Golang 内存模型](http://ifeve.com/golang-mem/)
- [The Go Memory Model](https://golang.org/ref/mem)
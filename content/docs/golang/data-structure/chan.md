---
author: Yuan Hao
date: 2020-12-14
title: chan
tag: [golang, chan]
---

# 1. 引言
Go 语言中最常见的、也是经常被人提及的设计模式就是 —— 不要通过共享内存的方式进行通信，而是应该通过通信的方式共享内存。

**先入先出**

目前的 Channel 收发操作均遵循了先入先出（FIFO）的设计，具体规则如下：
- 先从 Channel 读取数据的 Goroutine 会先接收到数据；
- 先向 Channel 发送数据的 Goroutine 会得到先发送数据的权利；

# 2. 数据结构
```go
type hchan struct {
	qcount   uint           // 当前队列中剩余元素个数
	dataqsiz uint           // 环形队列长度，即可以存放的元素个数
	buf      unsafe.Pointer // 环形队列指针
	elemsize uint16 // 每个元素的大小
	closed   uint32 // 标识关闭状态
	elemtype *_type // 元素类型
	sendx    uint   // 发送操作处理到的位置
	recvx    uint   // 接收操作处理到的位置
	recvq    waitq  // 等待读消息的 goroutine 队列
	sendq    waitq  // 等待写消息的 goroutine 队列
	lock mutex      // 互斥锁，chan 不允许并发读写
}
```
从数据结构可以看出 channel 由队列、类型信息、goroutine 等待队列组成，下面分别说明其原理。

## 2.1 环形队列
chan 内部实现了一个环形队列作为其缓冲区，队列的长度是创建 chan 时指定的。

下图展示了一个可缓存 6 个元素的 channel 示意图：

```
+-------------+
|    hchan    |
+-------------+
|   qcount=2  |
+-------------+
|  dataqsiz=6 |
+-------------+       +-----------------------+
|     buf     |------>| 0 | 1 | 1 | 0 | 0 | 0 |             
+-------------+       +-----^-------^---------+
|   sendx=3   |-------------|-------|    
+-------------+             |       
|   recvq=1   |-------------+
+-------------+
```

<!-- 
![with-buf.png](/golang/data-structure/chan/with-buf.png)
-->

- dataqsiz 指示了队列长度为 6，即可缓存 6 个元素；
- buf 指向队列的内存，队列中还剩余两个元素；
- qcount 表示队列中还有两个元素；
- sendx 指示后续写入的数据存储的位置，取值 [0, 6)；
- recvx 指示从该位置读取数据，取值 [0, 6)；

## 2.2 等待队列
从 channel 读数据，如果 channel 缓冲区为空或者没有缓冲区，当前 goroutine 会被阻塞。
向 channel 写数据，如果 channel 缓冲区已满或者没有缓冲区，当前 goroutine 会被阻塞。

被阻塞的 goroutine 将会挂在 channel 的等待队列中：
- 因读阻塞的 goroutine 会被向 channel 写入数据的 goroutine 唤醒；
- 因写阻塞的 goroutine 会被从 channel 读数据的 goroutine 唤醒；

下图展示了一个没有缓冲区的 channel，有几个 goroutine 阻塞等待读数据：

```
+-------------+
|    hchan    |
+-------------+
|   qcount=0  |
+-------------+
|  dataqsiz=0 |
+-------------+
|     buf     |      
+-------------+
|   sendx=0   |
+-------------+ 
|   recvx=0   |
+-------------+    +---+    +---+    +---+
|    sendq    |--->| G |--->| G |--->| G |       
+-------------+    +---+    +---+    +---+
|    recvq    |
+-------------+
```
<!--  
![with-waitq.png](/golang/data-structure/chan/with-waitq.png)
-->

注意，一般情况下 recvq 和 sendq 至少有一个为空。只有一个例外，
那就是同一个 goroutine 使用 select 语句向 channel 一边写数据，一边读数据。

## 2.3 类型信息
一个 channel 只能传递一种类型的值，类型信息存储在 hchan 数据结构中。
- elemtype 代表类型，用于数据传递过程中的赋值；
- elemsize 代表类型大小，用于在 buf 中定位元素位置。

## 2.4 锁
一个 channel 同时仅允许被一个 goroutine 读写。

# 3. channel 读写

## 3.1 创建 channel
创建 channel 的过程实际上是初始化 hcha 结构。其中类型信息和缓冲区长度由 make 语句传入，
buf 的大小则与元素 大小和缓冲区长度共同决定。

创建 channel 的部分代码如下所示，只保留了核心创建逻辑：
```go
func makechan(t *chantype, size int) *hchan {
    elem := t.elem
    
    // ...
    
	mem, overflow := math.MulUintptr(elem.size, uintptr(size))
    
    // ...
    
	var c *hchan
	switch {
	case mem == 0:
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		c.buf = c.raceaddr()
	case elem.ptrdata == 0:
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)
	lockInit(&c.lock, lockRankHchan)

    // ...
    
	return c
}
```

## 3.2 发送数据

向一个 channel 中发送数据简单过程如下：
1. 如果等待接收队列 recvq 不为空，说明缓冲区中没有数据或者没有缓冲区，
   此时直接从 recvq 取出 G, 并把数据写入，最后把该 G 唤醒，结束发送过程；
2. 如果缓冲区中有空余位置，将数据写入缓冲区，结束发送过程；
3. 如果缓冲区中没有空余位置，将待发送数据写入 G，将当前 G 加入 sendq，进入睡眠，等待被读 goroutine 唤醒；

简单的流程图总结如下：
{{< mermaid >}}
graph TD
start(开始发送) --> B{recq非空?}
B --> |Y| C[从recvq取出一个G]
C --> C1[数据写入G]
C1 --> C2[唤醒G]
C2 --> H(结束发送)
B --> |N| D{buf有空位?}
D --> |Y| E[将数据写入buf队尾]
E --> H
D --> |N| F[将当前goroutine加入senq,等待被唤醒]
F -.-> G[被唤醒,数据被取走]
G -.-> H
{{< /mermaid >}}

## 3.3 接收数据

从一个 channel 接收数据简单过程如下：
1. 如果等待发送队列 sendq 不为空，且没有缓冲区，直接从 sendq 中取出 G，
   把 G 中数据读出，最后把 G 唤醒，结束读取过程；
2. 如果等待发送队列 sendq 不为空，此时说明缓冲区已满，从缓冲区中首部读出数据，
   把 G 中数据写入缓冲区尾部，把 G 唤醒，结束读取过程；
3. 如果缓冲区中有数据，则从缓冲区取出数据，结束读取过程；
4. 将当前 goroutine 加入 recvq，进入睡眠，等待被写 goroutine 唤醒；

简单的流程图总结如下：
{{< mermaid >}}
graph TD
A(开始接收) --> B{sendq非空?}
B --> |N| F{qcount>0?}
F --> |Y| f0[从buf队头取数据]
f0 --> Z
F --> |N| f1[将当前goroutine假如有recq,等待被唤醒]
f1 -.-> f2[被唤醒,数据已写入]
f2 -.-> Z
B --> |Y| C{有缓冲区?}
C --> |Y| d0[从buf队头取数据]
d0 --> d1[从sendq中取出G]
d1 --> d2[把G中数据写入buf队尾]
d2 --> d3[唤醒G]
d3 --> Z(结束接收)
C --> |N| d1
d1 --> e0[从G中读出数据]
e0 --> d3
{{< /mermaid >}}

## 3.4 关闭 channel

关闭 channel 时会把 recvq 中的 G 全部唤醒，本该写入 G 的数据位置为 nil。把 sendq 中的 G 全部唤醒，但这些 G 会 panic。
除此之外，panic 出现的常见场景还有：
1. 关闭值为 nil 的 channel
2. 关闭已经被关闭的 channel
3. 向已经关闭的 channel 写数据

# 4. 参考资料
- [Go Channel 实现原理精要](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-channel/)
- [Go 专家编程：1.1 chan](https://www.bookstack.cn/read/GoExpertProgramming/chapter01-1.1-chan.md)
---
author: Yuan Hao
date: 2021-01-21
title: pprof
tag: [golang, pprof]
---

# 1. 什么是 pprof

Profiling 是指在程序执行过程中，收集能够反映程序执行状态的数据。
在软件工程中，性能分析（performance analysis，也称为 profiling），
是以收集程序运行时信息为手段研究程序行为的分析方法，是一种动态程序分析的方法。

Go 语言自带的 pprof 库就可以分析程序的运行情况，并且提供可视化的功能。它包含两个相关的库：
- runtime/pprof
  对于只跑一次的程序，例如每天只跑一次的离线预处理程序，调用 pprof 包提供的函数，手动开启性能数据采集。
- net/http/pprof
  对于在线服务，对于一个 HTTP Server，访问 pprof 提供的 HTTP 接口，获得性能数据。
  当然，实际上这里底层也是调用的 runtime/pprof 提供的函数，封装成接口对外提供网络访问。

# 2. pprof 的作用

> 下表来自 [golang pprof 实战](https://blog.wolfogre.com/posts/go-ppof-practice)

类型         |  描述                    | 备注
------------ | ------------------------ | ---------------------------
allocs	     | 内存分配情况的采样信息	  | 可以用浏览器打开，但可读性不高
blocks	     | 阻塞操作情况的采样信息	  | 可以用浏览器打开，但可读性不高
cmdline	     | 当前程序的命令行调用   	  | 可以用浏览器打开，显示编译文件的临时目录
goroutine    | 当前所有协程的堆栈信息	  | 可以用浏览器打开，但可读性不高
heap	     | 堆上内存使用情况的采样信息 |	可以用浏览器打开，但可读性不高
mutex	     | 锁争用情况的采样信息	     | 可以用浏览器打开，但可读性不高
profile	     | CPU 占用情况的采样信息	 | 浏览器打开会下载文件
threadcreate | 系统线程创建情况的采样信息 | 可以用浏览器打开，但可读性不高
trace	     | 程序运行跟踪信息	         | 浏览器打开会下载文件，本文不涉及，可另行参阅 [深入浅出 Go trace](https://mp.weixin.qq.com/s/I9xSMxy32cALSNQAN8wlnQ)

allocs 和 heap 采样的信息一致，不过前者是所有对象的内存分配，而 heap 则是活跃对象的内存分配。

# 3. pprof 如何使用

我们可以通过报告生成、Web 可视化界面、交互式终端三种方式来使用 pprof。

## 3.1 runtime/pprof

拿 CPU profiling 举例，增加两行代码，调用 `pprof.StartCPUProfile` 启动 cpu profiling，
调用 `pprof.StopCPUProfile()` 将数据刷到文件里：

```go
import "runtime/pprof"

var cpuprofile = flag.String("cpuprofile", "", "write cpu profile to file")

func main() {
    // …
        
    pprof.StartCPUProfile(f)
    defer pprof.StopCPUProfile()
    
    // …
}
```

## 3.2 net/http/pprof

启动一个端口（和正常提供业务服务的端口不同）监听 pprof 请求：

```go
import (
    "log"
    "net/http"
    _ "net/http/pprof"
    "regexp"
)

func handler(wr http.ResponseWriter, r *http.Request) {
    var pattern = regexp.MustCompile(`^(\w+)@foo.bar$`)
    account := r.URL.Path[1:]
    res := pattern.FindSubmatch([]byte(account))
    if len(res) > 1 {
        wr.Write(res[1])
    } else {
        wr.Write([]byte("None"))
    }
}

func main() {
    http.HandleFunc("/", handler)
    err := http.ListenAndServe(":8888", nil)
    if err != nil {
        log.Fatal("ListenAndServe:", err)
    }
}
```

pprof 包会自动注册 handler， 处理相关的请求：

```go
// src/net/http/pprof/pprof.go:71
func init() {
    http.Handle("/debug/pprof/", http.HandlerFunc(Index))
    http.Handle("/debug/pprof/cmdline", http.HandlerFunc(Cmdline))
    http.Handle("/debug/pprof/profile", http.HandlerFunc(Profile))
    http.Handle("/debug/pprof/symbol", http.HandlerFunc(Symbol))
    http.Handle("/debug/pprof/trace", http.HandlerFunc(Trace))
}
```

启动服务后，直接在浏览器访问：http://localhost:8888/debug/pprof/，得到如下页面：

<iframe 
src="/golang/tools/pprof/_debug_pprof_.html"
height="680"
width="100%"
frameborder="1">
</iframe>

关于 goroutine 的信息有两个链接，goroutine 和 full goroutine stack dump，
前者是一个汇总的消息，可以查看 goroutines 的总体情况，
后者则可以看到每一个 goroutine 的状态。
页面具体内容的解读可以参考大彬的文章 [实战 Go 内存泄露](https://segmentfault.com/a/1190000019222661)。

# 4. 采样分析

## 4.1 报告生成

报告生成有 2 种方式：
- runtime/pprof 写入本地文件
- 内置页面下载采样文件

下面以内置页面为例：
在 debug/pprof 页面点击 profile，会在后台进行默认 30s 的数据采样，采样完成后，返回给浏览器一个采样文件，其他命令类似。
然后使用以下命令：

```bash
go tool pprof -http=:8080 profile
```

会在本地 8080 端口启动 http 服务，提供可视化界面：

<!-- graph -->

图中的连线代表对方法的调用，连线上的标签代表指定的方法调用的采样值（例如时间、内存分配大小等），
方框的大小与方法运行的采样值的大小有关。
每个方框由两个标签组成：在 cpu profile 中，一个是方法运行的时间占比，
一个是它在采样的堆栈中出现的时间占比（前者是 flat 时间，后者则是 cumulate 时间占比)；
框越大，代表耗时越多或是内存分配越多

点击左上角 VIEW 可以选择不同图形化展示，最常见的火焰图如下：

<!-- flameGraph -->

它和一般的火焰图相比刚好倒过来了，调用关系的展现是从上到下。形状越长，表示执行时间越长。

## 4.2 交互式命令

除了使用内置的页面采样分析，也直接使用如下命令，不需要通过浏览器直接进入~

```bash
# step1：压测工具访问服务
wrk -c500 -t30 -d1m http://localhost:8888/zzz@foo.bar
# step2：采集数据，默认 30s
go tool pprof http://localhost:8888/debug/pprof/profile
```

其他类似的采集数据命令还有：
```bash
# 自定义等待时间 120s
go tool pprof http://localhost:8888/debug/pprof/profile?second=120
    
# 下载 heap profile
go tool pprof http://localhost:8888/debug/pprof/heap

# 下载 goroutine profile
go tool pprof http://localhost:8888/debug/pprof/goroutine

# 下载 block profile
go tool pprof http://localhost:8888/debug/pprof/block

# 下载 mutex profile
go tool pprof http://localhost:8888/debug/pprof/mutex
```

下面开始进入命令行交互式模式，使用 `top`、`list`、`web` 等命令：

```bash
(pprof) top
Showing nodes accounting for 32830ms, 46.48% of 70640ms total
Dropped 489 nodes (cum <= 353.20ms)
Showing top 10 nodes out of 159
      flat  flat%   sum%        cum   cum%
   18950ms 26.83% 26.83%    19700ms 27.89%  syscall.Syscall
    2910ms  4.12% 30.95%     2920ms  4.13%  runtime.nanotime (inline)
    2780ms  3.94% 34.88%     2830ms  4.01%  runtime.walltime
    1960ms  2.77% 37.66%    12060ms 17.07%  runtime.mallocgc
    1280ms  1.81% 39.47%     1680ms  2.38%  runtime.heapBitsSetType
    1200ms  1.70% 41.17%     2360ms  3.34%  runtime.scanobject
    1170ms  1.66% 42.82%     1170ms  1.66%  runtime.nextFreeFast
     960ms  1.36% 44.18%      960ms  1.36%  runtime.memclrNoHeapPointers
     880ms  1.25% 45.43%     3090ms  4.37%  regexp.makeOnePass.func1
     740ms  1.05% 46.48%      900ms  1.27%  runtime.findObject
```

得到 5 列数据：

字段  | 描述
----- | ---
flat  |	本函数的执行耗时
flat% |	flat 占 CPU 总时间的比例。程序总耗时 70640ms，syscall.Syscall 的 18950ms 占了 26.83%
sum%  |	前面每一行的 flat 占比总和
cum   |	累计量。指该函数加上该函数调用的函数总耗时
cum%  |	cum 占 CPU 总时间的比例

上面的结果没啥有价值的信息，都是 go 的底层调用。使用 `list` 正则匹配最耗时的代码：

```bash
(pprof) list syscall.Syscall
Total: 1.18mins
ROUTINE ======================== syscall.Syscall in /usr/local/go/src/syscall/asm_linux_amd64.s
    18.95s     19.70s (flat, cum) 27.89% of Total
         .          .     13:// Trap # in AX, args in DI SI DX R10 R8 R9, return in AX DX
         .          .     14:// Note that this differs from "standard" ABI convention, which
         .          .     15:// would pass 4th arg in CX, not R10.
         .          .     16:
         .          .     17:TEXT ·Syscall(SB),NOSPLIT,$0-56
      10ms      420ms     18:	CALL	runtime·entersyscall(SB)
         .          .     19:	MOVQ	a1+8(FP), DI
         .          .     20:	MOVQ	a2+16(FP), SI
         .          .     21:	MOVQ	a3+24(FP), DX
         .          .     22:	MOVQ	trap+0(FP), AX	// syscall entry
         .          .     23:	SYSCALL
    18.92s     18.92s     24:	CMPQ	AX, $0xfffffffffffff001
      10ms       10ms     25:	JLS	ok
         .          .     26:	MOVQ	$-1, r1+32(FP)
         .          .     27:	MOVQ	$0, r2+40(FP)
         .          .     28:	NEGQ	AX
         .          .     29:	MOVQ	AX, err+48(FP)
         .       20ms     30:	CALL	runtime·exitsyscall(SB)
         .          .     31:	RET
         .          .     32:ok:
         .          .     33:	MOVQ	AX, r1+32(FP)
      10ms       10ms     34:	MOVQ	DX, r2+40(FP)
         .          .     35:	MOVQ	$0, err+48(FP)
         .      320ms     36:	CALL	runtime·exitsyscall(SB)
         .          .     37:	RET
         .          .     38:
         .          .     39:// func Syscall6(trap, a1, a2, a3, a4, a5, a6 uintptr) (r1, r2, err uintptr)
         .          .     40:TEXT ·Syscall6(SB),NOSPLIT,$0-80
         .          .     41:	CALL	runtime·entersyscall(SB)
```
最耗时的 18.92s 都在比较寄存器。

进入下一步，使用 `web` 命令查看调用链，需要提前安装 graphviz 工具，自动调用浏览器展示下图：

<!-- web 图示 -->

图中可明显看出，`regexp.Compile` 耗时明显，因为每次请求过来，都要重新编译正则表达式。

# 5. 参考资料

- [深度解密Go语言之 pprof](https://qcrao.com/2019/11/10/dive-into-go-pprof/)
- [曹大 pprof](https://xargin.com/pprof-and-flaegraph/)
- [Russ Cox 优化过程，并附上代码](https://blog.golang.org/profiling-go-programs)
- [google pprof](https://github.com/google/pprof)
- [使用 pprof 和火焰图调试 golang 应用](https://cizixs.com/2017/09/11/profiling-golang-program/)
- [资源合集](https://jvns.ca/blog/2017/09/24/profiling-go-with-pprof/)
- [Profiling your Golang app in 3 steps](https://coder.today/tech/2018-11-10_profiling-your-golang-app-in-3-steps/)
- [案例，压测 Golang remote profiling and flamegraphs](https://matoski.com/article/golang-profiling-flamegraphs/)
- [煎鱼 pprof](https://segmentfault.com/a/1190000016412013)
- [鸟窝 pprof](https://colobu.com/2017/03/02/a-short-survey-of-golang-pprof/)
- [关于 Go 的 7 种性能分析方法](https://blog.lab99.org/post/golang-2017-10-20-video-seven-ways-to-profile-go-apps.html)
- [pprof 比较全](https://juejin.im/entry/5ac9cf3a518825556534c76e)
- [通过实例来讲解分析、优化过程](https://artem.krylysov.com/blog/2017/03/13/profiling-and-optimizing-go-web-applications/)
- [wolfogre 非常精彩的实战文章](https://blog.wolfogre.com/posts/go-ppof-practice/)
- [实战案例](https://www.cnblogs.com/sunsky303/p/11058808.html)
- [大彬 实战内存泄露](https://segmentfault.com/a/1190000019222661)
- [查找内存泄露](https://www.freecodecamp.org/news/how-i-investigated-memory-leaks-in-go-using-pprof-on-a-large-codebase-4bec4325e192/)

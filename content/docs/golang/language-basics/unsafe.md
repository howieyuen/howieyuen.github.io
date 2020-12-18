---
author: Yuan Hao
date: 2020-12-18
title: unsafe
tag: [golang, unsafe]
---

> 本文转自：[Go 里面的 unsafe 包详解](https://gocn.vip/topics/371)

# The unsafe Package in Golang

Golang 的 unsafe 包是一个很特殊的包。为什么这样说呢？本文将详细解释。

# 来自 go 语言官方文档的警告

unsafe 包的文档是这么说的：

{{< hint danger >}}
导入 unsafe 的软件包可能不可移植，并且不受 Go 1 兼容性指南的保护。
{{< /hint >}}

Go 1 兼容性指南这么说：

{{< hint danger >}}
导入 unsafe 软件包可能取决于 Go 实现的内部属性。我们保留对可能导致程序崩溃的实现进行更改的权利。
{{< /hint >}}

当然包名称暗示 unsafe 包是不安全的。但这个包有多危险呢？让我们先看看 unsafe 包的作用。

# Unsafe包的作用

直到现在（Go1.7），unsafe 包含以下资源：

- 三个函数：
    - `func Alignof(variable ArbitraryType) uintptr`
    - `func Offsetof(selector ArbitraryType) uintptr`
    - `func Sizeof(variable ArbitraryType) uintptr`
- 和一种类型：
    - 类型 `Pointer *ArbitraryType`

这里，ArbitraryType 不是一个真正的类型，它只是一个占位符。

与 Golang 中的大多数函数不同，上述三个函数的调用将始终在编译时求值，而不是运行时。这意味着它们的返回结果可以分配给常量。

{{< hint warning >}}
- unsafe 包中的函数中非唯一调用将在编译时求值。
- 当传递给 len 和 cap 的参数是一个数组值时，内置函数和 cap 函数的调用也可以在编译时被求值。
{{< /hint >}}

除了这三个函数和一个类型外，指针在 unsafe 包也为编译器服务。

出于安全原因，Golang 不允许以下之间的直接转换：
- 两个不同指针类型的值，例如 int64 和 float64。
- 指针类型和 uintptr 的值。

但是借助 `unsafe.Pointer`，我们可以打破 Go 类型和内存安全性，并使上面的转换成为可能。
这怎么可能发生？让我们阅读 unsafe 包文档中列出的规则：
- 任何类型的指针值都可以转换为 `unsafe.Pointer`。
- `unsafe.Pointer` 可以转换为任何类型的指针值。
- `uintptr` 可以转换为 `unsafe.Pointer`。
- `unsafe.Pointer` 可以转换为 `uintptr`。

这些规则与Go规范一致：

{{< hint info >}}
底层类型 uintptr 的任何指针或值都可以转换为指针类型，反之亦然。
{{< /hint >}}

规则表明 `unsafe.Pointe`r 类似于 c 语言中的 `void*` 。当然，`void*` 在 C 语言里是危险的！

在上述规则下，对于两种不同类型 T1 和 T2，可以使 T1 值与 `unsafe.Pointer` 值一致，
然后将 `unsafe.Pointer` 值转换为 T2 值（或 uintptr 值）。
通过这种方式可以绕过 Go 类型系统和内存安全性。当然，滥用这种方式是很危险的。

举个例子：

```go
package main

import (
    "fmt"
    "unsafe"
)
func main() {
    var n int64 = 5
    var pn = &n
    var pf = (*float64)(unsafe.Pointer(pn))
    // now, pn and pf are pointing at the same memory address
    fmt.Println(*pf) // 2.5e-323
    *pf = 3.14159
    fmt.Println(n) // 4614256650576692846
}
```

在这个例子中的转换可能是无意义的，但它是安全和合法的（为什么它是安全的？）。

因此，资源在 unsafe 包中的作用是为 Go 编译器服务，`unsafe.Pointer` 类型的作用是绕过 Go 类型系统和内存安全。

# unsafe.Pointer 和 uintptr

这里有一些关于 `unsafe.Pointer` 和 `uintptr` 的事实：
- `uintptr` 是一个整数类型。
    - 即使 `uintptr` 变量仍然有效，由 `uintptr` 变量表示的地址处的数据也可能被 GC 回收。
- `unsafe.Pointer`是一个指针类型。
    - 但是 `unsafe.Pointer` 值不能被取消引用。
    - 如果 `unsafe.Pointer` 变量仍然有效，则由 `unsafe.Pointer` 变量表示的地址处的数据不会被 GC 回收。
    - `unsafe.Pointer` 是一个通用的指针类型，就像 `*int` 等。

由于 `uintptr` 是一个整数类型，`uintptr` 值可以进行算术运算。
所以通过使用 `uintptr` 和 `unsafe.Pointer`，我们可以绕过限制，`*T` 值不能在 Golang 中计算偏移量：

```go
package main

import (
    "fmt"
    "unsafe"
)

func main() {
    a := [4]int{0, 1, 2, 3}
    p1 := unsafe.Pointer(&a[1])
    p3 := unsafe.Pointer(uintptr(p1) + 2 * unsafe.Sizeof(a[0]))
    *(*int)(p3) = 6
    fmt.Println("a =", a) // a = [0 1 2 6]

    // ...

    type Person struct {
        name   string
        age    int
        gender bool
    }

    who := Person{"John", 30, true}
    pp := unsafe.Pointer(&who)
    pname := (*string)(unsafe.Pointer(uintptr(pp) + unsafe.Offsetof(who.name)))
    page := (*int)(unsafe.Pointer(uintptr(pp) + unsafe.Offsetof(who.age)))
    pgender := (*bool)(unsafe.Pointer(uintptr(pp) + unsafe.Offsetof(who.gender)))
    *pname = "Alice"
    *page = 28
    *pgender = false
    fmt.Println(who) // {Alice 28 false}
}
```

# unsafe 包有多危险

关于 unsafe 包，Ian，Go 团队的核心成员之一，已经确认：
- 在 unsafe 包中的函数的签名将不会在以后的 Go 版本中更改，
- 并且 `unsafe.Pointer` 类型将在以后的 Go 版本中始终存在。

所以，unsafe 包中的三个函数看起来不危险。
go team leader 甚至想把它们放在别的地方。
unsafe 包中这几个函数唯一不安全的是它们调用结果可能在后来的版本中返回不同的值。
很难说这种不安全是一种危险。

看起来所有的 unsafe 包的危险都与使用 `unsafe.Pointer` 有关。
unsafe 包 docs 列出了一些使用 `unsafe.Pointer` 合法或非法的情况。
这里只列出部分非法使用案例：

```go
package main

import (
    "fmt"
    "unsafe"
)

// case A: conversions between unsafe.Pointer and uintptr 
//         don't appear in the same expression
func illegalUseA() {
    fmt.Println("===================== illegalUseA")

    pa := new([4]int)

    // split the legal use
    // p1 := unsafe.Pointer(uintptr(unsafe.Pointer(pa)) + unsafe.Sizeof(pa[0]))
    // into two expressions (illegal use):
    ptr := uintptr(unsafe.Pointer(pa))
    p1 := unsafe.Pointer(ptr + unsafe.Sizeof(pa[0]))
    // "go vet" will make a warning for the above line:
    // possible misuse of unsafe.Pointer

    // the unsafe package docs, https://golang.org/pkg/unsafe/#Pointer,
    // thinks above splitting is illegal.
    // but the current Go compiler and runtime (1.7.3) can't detect
    // this illegal use.
    // however, to make your program run well for later Go versions,
    // it is best to comply with the unsafe package docs.

    *(*int)(p1) = 123
    fmt.Println("*(*int)(p1)  :", *(*int)(p1)) //
}    

// case B: pointers are pointing at unknown addresses
func illegalUseB() {
    fmt.Println("===================== illegalUseB")

    a := [4]int{0, 1, 2, 3}
    p := unsafe.Pointer(&a)
    p = unsafe.Pointer(uintptr(p) + uintptr(len(a)) * unsafe.Sizeof(a[0]))
    // now p is pointing at the end of the memory occupied by value a.
    // up to now, although p is invalid, it is no problem.
    // but it is illegal if we modify the value pointed by p
    *(*int)(p) = 123
    fmt.Println("*(*int)(p)  :", *(*int)(p)) // 123 or not 123
    // the current Go compiler/runtime (1.7.3) and "go vet" 
    // will not detect the illegal use here.

    // however, the current Go runtime (1.7.3) will 
    // detect the illegal use and panic for the below code.
    p = unsafe.Pointer(&a)
    for i := 0; i <= len(a); i++ {
        *(*int)(p) = 123 // Go runtime (1.7.3) never panic here in the tests

        fmt.Println(i, ":", *(*int)(p))
        // panic at the above line for the last iteration, when i==4.
        // runtime error: invalid memory address or nil pointer dereference

        p = unsafe.Pointer(uintptr(p) + unsafe.Sizeof(a[0]))
    }
}

func main() {
    illegalUseA()
    illegalUseB()
}
```

编译器很难检测 Go 程序中非法的 `unsafe.Pointer` 使用。
运行 `go vet` 可以帮助找到一些潜在的错误，但不是所有的都能找到。
同样是 Go 运行时，也不能检测所有的非法使用。
非法 `unsafe.Pointer` 使用可能会使程序崩溃或表现得怪异（有时是正常的，有时是异常的）。
这就是为什么使用不安全的包是危险的。

# 转换 T1 为 T2
对于将 T1 转换为 `unsafe.Pointer`，然后转换为 T2，unsafe 包 docs说：

{{< hint info >}}
如果 T2 比 T1 大，并且两者共享等效内存布局，则该转换允许将一种类型的数据重新解释为另一类型的数据。
{{< /hint >}}

这种“等效内存布局”的定义是有一些模糊的。看起来 Go 团队故意如此。这使得使用 unsafe 包更危险。

由于 Go 团队不愿意在这里做出准确的定义，本文也不尝试这样做 这里，列出了已确认的合法用例的一小部分。

## 合法用例 1：在 `[]T` 和 `[]MyT` 之间转换

在这个例子里，我们用 int 作为 T：

```go
type MyInt int
```

在 Golang 中，`[]int` 和 `[]MyInt` 是两种不同的类型，它们的底层类型是自身。
因此，`[]int` 的值不能转换为 `[]MyInt`，反之亦然。 但是在 `unsafe.Pointer` 的帮助下，转换是可能的：

```go
package main

import (
    "fmt"
    "unsafe"
)

func main() {
    type MyInt int

    a := []MyInt{0, 1, 2}
    // b := ([]int)(a) // error: cannot convert a (type []MyInt) to type []int
    b := *(*[]int)(unsafe.Pointer(&a))

    b[0]= 3

    fmt.Println("a =", a) // a = [3 1 2]
    fmt.Println("b =", b) // b = [3 1 2]

    a[2] = 9

    fmt.Println("a =", a) // a = [3 1 9]
    fmt.Println("b =", b) // b = [3 1 9]
}
```

## 合法用例 2：调用 sync/atomic 包中指针相关的函数

sync/atomic 包中的以下函数的大多数参数和结果类型都是 `unsafe.Pointer` 或 `*unsafe.Pointer`：

- `func CompareAndSwapPointer(addr *unsafe.Pointer, old，new unsafe.Pointer) (swapped bool)`
- `func LoadPointer(addr *unsafe.Pointer) (val unsafe.Pointer)`
- `func StorePointer(addr *unsafe.Pointer, val unsafe.Pointer)`
- `func SwapPointer(addr *unsafe.Pointer, new unsafe.Pointer) (old unsafe.Pointer)`

要使用这些功能，必须导入 unsafe 包。

> 注意： `unsafe.Pointer` 是一般类型，因此 `unsafe.Pointer` 的值可以转换为 `unsafe.Pointer`，反之亦然。

```go
package main

import (
    "fmt"
    "log"
    "time"
    "unsafe"
    "sync/atomic"
    "sync"
    "math/rand"
)

var data *string

// get data atomically
func Data() string {
    p := (*string)(atomic.LoadPointer(
            (*unsafe.Pointer)(unsafe.Pointer(&data)),
        ))
    if p == nil {
        return ""
    } else {
        return *p
    }
}

// set data atomically
func SetData(d string) {
    atomic.StorePointer(
            (*unsafe.Pointer)(unsafe.Pointer(&data)), 
            unsafe.Pointer(&d),
        )
}

func main() {
    var wg sync.WaitGroup
    wg.Add(200)

    for range [100]struct{}{} {
        go func() {
            time.Sleep(time.Second * time.Duration(rand.Intn(1000)) / 1000)

            log.Println(Data())
            wg.Done()
        }()
    }

    for i := range [100]struct{}{} {
        go func(i int) {
            time.Sleep(time.Second * time.Duration(rand.Intn(1000)) / 1000)
            s := fmt.Sprint("#", i)
            log.Println("====", s)

            SetData(s)
            wg.Done()
        }(i)
    }

    wg.Wait()

    fmt.Println("final data = ", *data)
}
```

# 结论
- unsafe 包用于 Go 编译器，而不是 Go 运行时。
- 使用 unsafe 作为程序包名称只是让你在使用此包是更加小心。
- 使用 `unsafe.Pointer` 并不总是一个坏主意，有时我们必须使用它。
- Golang 的类型系统是为了安全和效率而设计的。
  但是在 Go 类型系统中，安全性比效率更重要。
  通常 Go 是高效的，但有时安全真的会导致 Go 程序效率低下。
  unsafe 包用于有经验的程序员通过安全地绕过 Go 类型系统的安全性来消除这些低效。
- unsafe 包可能被滥用并且是危险的。
---
author: Yuan Hao
date: 2020-5-14
title: reflect
tag: [golang, reflect]
---

# 1. 引言

计算机中提到的反射一般是指，**程序借助某种手段检查自己结构的一种能力**，通常就是借助编程语言中定义的类型（`types`）。因此，反射是建立在类型系统上的。

**go是静态类型化**，每个变量都有一个静态类型，也就是说，在编译时，变量的类型就已经确定。不显示地去做强制类型转换，不同类型之间是无法相互赋值的。

有一种特殊的类型叫做接口（`interface`），一个接口表示的是一组方法集合。**一个接口变量能存储任何具体的值，只要这个值实现了这个接口的方法集合**。比如io包中的 `Reader` 和 `Writer`，`io.Reader` 接口变量能够保存任意实现了 `Read()` 方法的类型所定义的值。

一个特殊接口就是空接口 `interface{}`，任何值都可以说实现了空接口，因为空接口中没有定义方法，所以**空接口可以保存任何值**。

**一个接口类型变量存储了一对值：赋值给这个接口变量的具体值 + 这个值的类型描述符**。更进一步的讲，这个“值”是实现了这个接口的底层具体数据项（underlying concrete data item)，而这个“类型”是描述了具体数据项（item）的全类型（full type）。

所以反射是干嘛的呢？**反射是一种检查存储在接口变量中的（类型/值）对的机制**。reflect 包中提供的 2 个类型 `Type` 和 `Value`，提供了访问接口值的 `reflect.Type` 和 `reflect.Value` 部分。

# 2. 三大法则

## 2.1. Reflection goes from interface value to reflecton object

从 `interface{}` 变量可以反射出反射对象

```go
type MyInt int32

func main() {
    var x MyInt = 7
    v := reflect.ValueOf(x)
    t := reflect.TypeOf(x)
    fmt.Println("type:", t)        // type: main.MyInt
    fmt.Println("value:", v)       // value: 7
    fmt.Println("kind:", v.Kind()) // kind: int32
    fmt.Println("type:", v.Type()) // type: main.MyInt
    x = MyInt(int32(v.Int))        // v.Int returns a int64
}
```

`reflect.Value的Type` 返回的是静态类型 `MyInt`，而 `kind()` 方法返回的是底层类型 `int32`；为了保持 API 简单，value 的 `Setter` 和 `Getter` 类型方法操作，是包含某个值的最大类型，`v.Int()` 返回的是 `int64`，必要时转化成实际类型。

## 2.2 Reflection goes from reflection object to interface value

从反射对象可以获取 `interface{}` 变量；

```go
type MyInt int32

func main() {
    var x MyInt = 7
    v := reflect.ValueOf(x)
    y := v.Interface().(int32)
    fmt.Println(y) // 7
}
```

对于一个 `reflect.Value`，可以用 `Interface()` 方法恢复成一个接口值，效果就是包类型和值打包成接口，并返回结果。

## 2.3 To modify a reflection object, the value must be settable

要修改反射对象，其值必须可设置；

```go
func main() {
    var x float64 = 3.4
    v := reflect.ValueOf(x)
    // panic: reflect: reflect.flag.mustBeAssignable using addressable value
    v.SetFloat(7.1) 
    
    p := reflect.ValueOf(&X)
    // panic: reflect: reflect.flag.mustBeAssignable using addressable value
    p.SetFloat(7.1) 
    
    e := reflect.ValueOf(&X).Elem()
    // OK
    e.SetFloat(7.1) 
}
```

如果我们想通过反射来修改变量 `x`，我们必须把我们想要修改的值的指针传给一个反射库。Go 语言的函数调用都是值传递的，所以我们只能先获取指针对应的 `reflect.Value`，再通过 `reflect.Value.Elem` 方法迂回的方式得到可以被设置的变量。我们通过如下所示的代码理解这个过程：

```go
func main() {
    i := 1
    v := &i
    *v = 10
}
```

如果不能直接操作 `i` 变量修改其持有的值，我们就只能获取 `i` 变量所在地址并使用 `*v` 修改所在地址中存储的整数。

# 3. 相关资料

- [Go语言中反射包的实现原理](https://studygolang.com/articles/2157)
- [4.3反射](https://draveness.me/golang/docs/part2-foundation/ch04-basic/golang-reflect/)
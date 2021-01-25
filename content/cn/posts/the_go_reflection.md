---
title: "5分钟系列 -「Go Reflection」"
date: 2018-09-12
categories: ['note', 'tech']
draft: false
---

很多语言都支持反射，Go 语言也不例外。那么什么是反射呢？总的来说，反射就是计算机程序语言可以在运行时动态访问、检查和修改任意对象本身状态和行为的能力。不同语言的反射特性有着不同的工作方式，有些语言还不支持反射特性。我们今天主要来看看在 Go 语言中，反射是怎么工作的。

在开始之前，先安利一篇 Go 官方出品的介绍反射机制的博客：[The Laws of Reflection](https://blog.golang.org/laws-of-reflection)

这篇笔记将我个人对于 Go 反射机制的补充与总结，感兴趣的读者建议去看原文。

## 反射的适用场景

需要说明的是，Go 语言是一门静态的编译型语言，也就是在编译过程中，编译器就能发现一些类型错误，但是它并不能发现反射相关代码的错误，这种错误往往需要在运行时才能被发现，所以一旦反射代码出错，将直接导致程序 panic 而退出，所以要尽量避免适用反射。此外，反射相关代码的可读性往往较差，执行效率也比正常 Go 语言代码低不少。综上，除非以下特殊情况必须使用反射外，我们应经量避免适用 Go 的这一语言特性。

适用于反射的场景：

1. 某段函数或者方法需要处理的数据类型不确定，会包含一些列的可能类型，这个时候我们就需要使用反射动态获取要处理的数据类型，基于数据类型的不同调用不同的处理程序。一个非常典型的例子是我们通过反序列化得到一段一个类型为`map[string]interface{}`的数据结构，然后通过反射机制递归地获取每个字段的类型。

2. 在某些场景下我们需要根据特性的某些条件决定调用哪个函数，也就是说需要在运行期间获取函数以及函数调用的参数来动态地执行函数。典型的例子是现在主流的`RPC`框架的实现机制，`RPC`服务器端维护函数名到函数反射值的映射，`RPC`客户端通过网络传递函数名、参数列表给`RPC`服务器端后，`RPC`服务器解析为反射值，调用执行函数，最后再将函数的返回值打包通过网络返回给`RPC`客户端。

> Note: 当然，可能还有其他适用于反射的适用场景，这里只是罗列常用的使用场景。

## 反射的实现基础

首先需要明确一点，Go 反射的实现基础是类型。在 Go 语言中，每个变量都有一个**静态类型**，也就是在编译阶段就检查的类型，比如`int`, `string`, `map`, `struct`等。需要注意的是，这个静态类型是指变量声明的时候指定的类型，并不一定不是底层真正存储的数据类型。例如：

```
type MyInt int

var i int
var j MyInt
```

上面的代码中，虽然变量`i`和`j`的真正存储的数据类型都是`int`，但是对于编译器来说，它们是不同的类型，要进行显示的类型转换才能相互赋值。

有的变量可能除了**静态类型**之外，还会有**动态类型**。所谓动态类型就是指变量值实际存储的类型信息，举个例子来说明：

```
var r io.Reader 

tty, err := os.OpenFile("/dev/tty", os.O_RDWR, 0)
if err != nil {
    return nil, err
}

r = tty

var empty interface{}
empty = tty
```

首先声明`r`的类型是`io.Reader`，也就是说这是`r`的静态类型，此时它的动态类型为`nil`，并且它的动态值也是`nil`。接下来的`r=tty`语句，将`r`的动态类型变成`*os.File`，动态值则变成非空，表示打开的文件对象。这时，`r`可以用`<value,type>`对来表示为：`<tty, *os.File>`。

同时，我们知道，Go 语言里面的任意对象都是空接口(`interface{}`)类型，因为任意对象都实现了零个或者多个方法。于是语句`empty = tty`将`*os.File`对象赋予空借口类型的变量`empty`。

事实上，反射与空接口类型密切相关。任意一个空接口(`interface{}`)类型的对象都主要是由两部分组成：

1. 类型(Type)
2. 数据值(Value)

![go-interface.jpg](https://i.loli.net/2021/01/24/G1UcFoJByQZdhrS.jpg)

其实如果要细分的话，可以分为无函数的`eface`和有函数的`iface`:

1. 无函数的`eface`

![eface.jpg](https://i.loli.net/2021/01/24/iTb4XqGA2eZaOD7.jpg)

2. 有函数的`iface`

![iface.jpg](https://i.loli.net/2021/01/24/lsSLRVKAaEqBf7D.jpg)


`iface`描述的是非空接口，它包含类型包含的方法集；与之相对的是`eface`，描述的是空接口(`interface{}`)，不包含任何方法。

## 反射的基本类型与函数

`reflect`标准库中定义了两个最重要的反射对象类型，它们提供很多函数来获取存储在接口里的类型：

1. reflect.Type
2. reflect.Value

其中，`reflect.Type`定义为接口类型，主要提供关于类型相关的信息，所以它和`_type`关联比较紧密，从`Type`接口的[定义](https://golang.org/pkg/reflect/#Type)中可以看到非常多有趣的方法，其中`MethodByName` 可以获取当前类型对应方法的引用、`Implements`可以判断当前类型是否实现了某个接口等；

```
type Type interface {
    Align() int
    FieldAlign() int
    Method(int) Method
    MethodByName(string) (Method, bool)
    NumMethod() int
    Name() string
    PkgPath() string
    Size() uintptr
    String() string
    Kind() Kind
    Implements(u Type) bool
    ...
}
```

而`reflect.Value`为普通结构体，它结合了`_type`和`data`，没有任何对外暴露的成员变量，但是却提供了很多方法让我们获取或者写入`reflect.Value`结构体中存储的数据，因此可以通过它获取甚至改变类型的值。

```
type Value struct {
    // contains filtered or unexported fields
}

func (v Value) Addr() Value
func (v Value) Bool() bool
func (v Value) Bytes() []byte
func (v Value) Float() float64
...
```

同时，`reflect`标准库还提供了两个基础的关于反射的函数来获取这两种反射对象类型：

1. func TypeOf(i interface{}) Type 
2. func ValueOf(i interface{}) Value

`TypeOf`函数用来提取一个接口中值的类型信息。由于它的输入参数是一个空接口(`interface{}`)，调用此函数时，实参会先被转化为空接口(`interface{}`)类型；`ValueOf`函数同样也是接受一个空接口(`interface{}`)，因此实参也要做类型转换，而函数的返回值是`reflect.Value`表示参数值存储的实际变量值，它能提供实际变量的各种信息，如类型结构体指针、真实数据的地址、标志位等。

特别需要说明的是，通过`ValueOf`函数获取到的`reflect.Value`反射对象值有以下两个重要的方法：

1. `Type()`方法获取变量的类型信息，等同于调用`reflect.TypeOf()`函数；
2. `Interface()()`方法获取反射变量的接口类型信息，相当于是`ValueOf`函数的方向操作；

![go-reflect-tye-function.jpg](https://i.loli.net/2021/01/24/ypBaDPOXx7vE6eo.jpg)

## 反射的三大法则

根据 Go 官方关于反射的博客，反射有三大法则：
    
> 1. Reflection goes from interface value to reflection object.
> 2. Reflection goes from reflection object to interface value.
> 3. To modify a reflection object, the value must be settable.

反射的第一条法则说的是，基于反射我们可以将接口类型对象转化为反射类型的对象，由此来检测存储在接口类型对象中的类型和值，上面提到的`reflect.TypeOf`和`reflect.ValueOf`就是完成这个转换的两个最重要函数：

举个例子，
```
var x float64 = 3.4
t := reflect.TypeOf(x)
v := reflect.ValueOf(x)
fmt.Println("type:", t)
fmt.Println("type:", v.Type())
fmt.Println("kind is float64:", v.Kind() == reflect.Float64)
fmt.Println("value:", v.Float())
```

将会打印出：

```
type: float64
type: float64
kind is float64: true
value: 3.4
```

上述简单的例子说明我们可以通过`reflect.TypeOf`和`reflect.ValueOf`函数将接口类型的变量`x`转化为反射类型的对象`t`与`v`，由此何以获取该变量的其他元信息，包括当前类型相关数据和操作，例如通过`Kind()`方法获取他存储的是基底层类型等：

- 对于结构体，可以获取字段的数量并通过下标和字段名获取字段；
- 对于哈希表，可以获取哈希表的 Key 类型；
- 对于函数或方法，可以获得入参和返回值的类型；
- ...



反射的第二条法则与第一条法则正好相反，反射对象还可以原成成接口类型的变量，`reflect.Value`类型的对象可以通过调用`Interface()`方法获得`interface{}`类型的接口变量：

```
v := reflect.ValueOf(1)
v.Interface{}.(int)
```

从上面的例子可以看到，反射对象转换成接口对象过程就像从接口对象到反射对象的镜面过程一样，从接口对象到反射对象需要经过从基本类型到接口类型的类型转换和从接口类型到反射对象类型的转换，反过来的话，所有的反射对象也都需要先转换成接口类型，再通过强制类型转换变成原始类型。

反射的第三条法则与值是否可以被更改相关的，它说的是如果需要修改一个反射变量，那么它必须是可设置的。反射变量可设置的本质是它存储了原变量本身，这样对反射变量的操作，就会反映到原变量本身；反之，如果反射变量不能代表原变量，那么操作了反射变量，不会对原变量产生任何影响，这会给使用者带来疑惑。所以第二种情况在语言层面是不被允许的。

举个例子：

```
var x float64 = 3.4
v := reflect.ValueOf(x)
v.SetFloat(7.1) // Error: will panic.
```

```
$ go run reflect.go
panic: reflect: reflect.flag.mustBeAssignable using unaddressable value
```

执行上面的代码会panic，因为反射变量`v`不能代表接口变量`x`本身的反射对象，因为调用 Go 语言函数调用是传值，所以`reflect.ValueOf(x)`这一函数执行的时候传入的实参只是形参的拷贝，所以`v`并不是`x`真正的反射对象映射。所以 Go 语言规定因此对`v`进行操作是被禁止的。

可设置(settable)是反射对象`reflect.Value`的一个性质，但不是所有的反射对象都是可被设置的。

如果想要上面的栗子可以运行，我们只需要在函数调用时传递参数的指针就可以解决：

```
var x float64 = 3.4
p := reflect.ValueOf(&x)
fmt.Println("type of p:", p.Type())
fmt.Println("settability of p:", p.CanSet())
```

但是虽然程序可以运行，但是从输出结果来看通过指针获取到的反射对象还是不可设置的：

```
type of p: *float64
settability of p: false
```

原因是反射对象`p`还不是代表`x`，`p`是反射对象里面的指针类型，其真正指向的类型需要通过`p.Elem()`来获取，也就是说才`p.Elem()`代表`x`：

```
v := p.Elem()
v.SetFloat(7.1)
fmt.Println(v.Interface()) // 7.1
fmt.Println(x) // 7.1
```

总之，如果想要操作原变量，反射变量`Value`必须对应到原接口变量的地址才行。

## 总结

反射是 Go 语言比较重要的特性之一，很多框架都依赖 Go 语言的反射机制实现一些动态的功能。Go 语言作为一门静态语言，它的设计目标就是尽量追求简单，但是有时候却缺少一定的表达能力，好在 Go 语言通过提供的反射机制提供了动态特性来弥补它在语法上的一些劣势。通过本文我们大致了解了在 Go 语言中反射的使用场景，反射的基本实现机制，同时也介绍了反射的基本类型与函数和三大法则。整体来讲，Go 反射是围绕着三者进行的，分别是`Type`、`Value`以及`Interface`，三者相辅相成。`reflect`标准库实现了运行时的反射能力，能够让 Go 程序操作不同类型的对象，可以使用`reflect.TypeOf`函数从静态接口类型(`interface{}`)中获取动态类型信息并通过`reflect.ValueOf()`函数获取数据的运行时表示，通过这两个函数和`reflect`标准库其他方法就可以得到更强大的表达能力。

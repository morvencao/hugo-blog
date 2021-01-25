---
title: "5分钟系列 -「Go Interface & Composition」"
date: 2018-02-26
categories: ['note', 'tech']
draft: false
---

### Go语言接口与鸭子类型的关系

什么是“鸭子类型”？

> If it looks like a duck, swims like a duck, and quacks like a duck, then it probably is a duck.

以上引用自维基百科的解释描述了什么是“鸭子类型”，所谓鸭子类型，是动态编程语言的一种对象推断策略，它更关注对象能如何被使用，而不是对象的类型本身。

传统的静态语言，如Java, C++等，它们必须显示声明实现了某个接口，之后，才能用在任何需要这个接口的地方，否则编译也不会通过，这也是静态语言比动态语言更安全的原因。但是Go语言作为一门“现代”静态语言，使用的是动态编程语言的对象推断策略，它更关注对象能如何被使用，而不是对象的类型本身。也就是说，Go语言引入了动态语言的便利，同时又会进行静态语言的类型检查，因此，它采用了折中的做法：不要求类型显示地声明实现了某个接口，只要实现了相关的方法即可，编译器就能检测到。

举个例子，下面的代码片段先定义一个接口，和使用此接口作为参数的函数：

```
type IGreeting interface {
    greeting()
}

func sayHello(i IGreeting) {
    i.greeting()
}
```

接下来再定义两个结构体：

```
type A struct {}
func (a A) greeting() {
    fmt.Println("Hi, I am A!")
}

type B struct {}
func (b B) greeting() {
    fmt.Println("Hi, I am B!")
}
```

最后，在`main`函数里调用`sayHello()`函数：

```
func main() {
    a := A{}
    b := B{}

    sayHello(a)
    sayHello(b)
}
```

运行程序之后的输出：

```
Hi, I am A!
Hi, I am B!
```

在`main`函数中，调用调用`sayHello()`函数时，传入了`a`和`b`对象，它们并没有显式地声明实现了`IGreeting`类型，只是实现了接口所规定的`greeting()`函数。然后编译器在调用`sayHello()` 函数时，会隐式地将`a`和`b`对象转换`成IGreeting`类型，这也是静态语言的类型检查功能。

可以看出，鸭子类型是一种动态语言的风格，在这种风格中，一个对象有效的语义，不是由继承自特定的类或实现特定的接口，而是由它“当前方法和属性的集合”决定。Go语言作为一种静态语言，通过接口实现了“鸭子类型”，实际上是Go语言的编译器在其中作了隐匿的转换工作。

### 值接收者和指针接收者

在Go语言中，一切皆是类型，包括原始类型的int，bool，string，还包括内置的map，slice，甚至func都是类型，这个javascript倒是有些相似之处。

而方法能给Go语言中的类型添加新的行为，它和函数的区别在于方法有一个接收者，给一个函数添加一个接收者，那么它就变成了方法。接收者可以是值接收者，也可以是指针接收者。

总体来说，在调用方法的时候，**值类型既可以调用值接收者的方法，也可以调用指针接收者的方法；指针类型既可以调用指针接收者的方法，也可以调用值接收者的方法。**也就是说，不管方法的接收者是什么类型，该类型的值和指针都可以调用，不必严格符合接收者的类型。

来看一个例子：

```
package main

import (
	"fmt"
	"math"
)

type Vertex struct {
	X, Y float64
}

func (v Vertex) Abs() float64 {
	return math.Sqrt(v.X*v.X + v.Y*v.Y)
}

func (v *Vertex) Scale(f float64) {
	v.X = v.X * f
	v.Y = v.Y * f
}

func main() {
    // v是值类型
	v := Vertex{3, 4}
    fmt.Println(v.Abs())
    // 值类型 调用接收者也是指针类型的方法
    v.Scale(10)
    // 值类型 调用接收者也是值类型的方法
	fmt.Println(v.Abs())

    // p是指针类型
	p := &Vertex{4, 3}
    // 指针类型 调用接收者是值类型的方法
	fmt.Println(p.Abs())
    // 指针类型 调用接收者也是指针类型的方法
	p.Scale(10)
    fmt.Println(p.Abs())
}
```

调用了`Scale()`方法之后，不管调用者是值类型还是指针类型，它的值都改变了。实际上，当类型和方法的接收者类型不同时，其实是编译器在背后做了一些工作，用一个表格来呈现：

| - | 接收者也是值类型 | 调用接收者也是指针类型 |
| ------ | ------ | ----- |
| 值类型 | 方法调用者的一个副本，类似于“传值” | 使用值的“引用”（地址）来调用方法，上例中`v.Scale(10)`实际上隐式地转化为`(&v).Scale(10)` |
| 指针类型 | 指针隐式地解引用为“值”，上例中`p.Abs()`实际上隐式地转化为`(*p).Abs()` | 实际上也是“传值”，方法内部的操作会直接影响调用者，类似于指针传参 |

那么，为什么不管方法的接收者是什么类型，该类型的值和指针都可以调用？这里面实际上通过Go语言的语法糖在起作用。

划重点：**实现了接收者是值类型的方法，相当于自动实现了接收者是指针类型的方法；而实现了接收者是指针类型的方法，不会自动生成对应接收者是值类型的方法。**

一个简单的例子就能明白：

```
package main

import "fmt"

type iReadWriter interface {
    read()
    write()
}

type ReadWriter struct {
    content string
}

func (rw ReadWriter) read() {
    fmt.Printf("I am reading %s\n", rw.content)
}

func (rw *ReadWriter) write() {
    fmt.Printf("I am writing %s\n", rw.content)
}

func main() {
    var rw iReadWriter = &ReadWriter{"hello"}
    rw.read()
    rw.write()
}
```

程序运行结果：

```
I am reading hello
I am writing hello
```

但是如果把`main`函数的第一条语句换一下：

```
func main() {
    var rw iReadWriter = ReadWriter{"hello"}
    rw.read()
    rw.write()
}
```

运行之后报错：

```
./prog.go:23:9: cannot use ReadWriter literal (type ReadWriter) as type iReadWriter in assignment:
	ReadWriter does not implement iReadWriter (write method has pointer receiver)
```

这两处代码的差别了是第一次是将`&ReadWriter`赋给了`iReadWriter`类型的变量`rw`；第二次则是将`ReadWriter` 赋给了`iReadWriter`类型的变量`rw`。第二次报错是说，`ReadWriter`没有实现`iReadWriter`，这就是说`ReadWriter`类型并没有实现`write`方法；从表面上看，`*ReadWriter`类型也没有实现`read`方法，但是因为`ReadWriter`类型实现了`read`方法，所以让`*ReadWriter`类型自动拥有了`read`方法。

也就是说，接收者是指针类型的方法，很可能在方法中会对接收者的属性进行更改操作，从而影响接收者；而对于接收者是值类型的方法，在方法中不会对接收者本身产生影响。所以，当实现了一个接收者是值类型的方法，就可以自动生成一个接收者是对应指针类型的方法，因为两者都不会影响接收者。但是，当实现了一个接收者是指针类型的方法，如果此时自动生成一个接收者是值类型的方法，原本期望对接收者的改变（通过指针实现），现在无法实现，因为值类型会产生一个拷贝，不会真正影响调用者。

### 值接收者还是指针接收者

如果方法的接收者是值类型，无论调用者是对象还是对象指针，修改的都是对象的副本，不影响调用者；如果方法的接收者是指针类型，则调用者修改的是指针指向的对象本身。

使用指针作为方法的接收者的理由：

1. 方法能够修改接收者指向的值；
2. 避免在每次调用方法时复制该值，在值的类型为大型结构体时，这样做会更加高效；


是使用值接收者还是指针接收者，不是由该方法是否修改了调用者（也就是接收者）来决定，而是应该基于该类型的本质。

如果类型具备“原始的本质”，也就是说它的成员都是由Go语言里内置的原始类型，如字符串，整型值等，那就定义值接收者类型的方法；而对于内置的引用类型，如slice，map，interface，channel，这些类型比较特殊，声明他们的时候，实际上是创建了一个`header`， 对于它们也是直接定义值接收者类型的方法。这样，调用函数时，是直接拷贝这些类型的`header`，而`header`本身就是为复制设计的。如果类型具备非原始的本质，不能被安全地复制，这种类型总是应该被共享，那就定义指针接收者的方法。比如Go语言标准库里面定义的文件结构体（struct File）就不应该被复制，应该只有一份实体。

### 组合与继承

在Go语言中中一切皆是类型，既包括Go语言里内置的原始类型，如字符串，整型值等，也包括内置的引用类型，如slice，map，interface，channel，func，当然还有用户自定义的类型等。这点和面向对象的概念有点像，但是又不完全等同，反而特别类似于javascript的语法糖。

那么Go语言如何实现类似于Java等面向对象语言中的“继承”呢？

答案是Go语言使用“组合”实现类似于继承的概念。举个例子：

```
type animal struct {
    name string
    age int
}

type cat struct {
    animal
    name string
}

func main() {
    c := cat{animal:animal{name:"animal",age:2},name:"cat"}
    fmt.Println(c)
    fmt.Println("name",c.name)
    fmt.Println("name",c.animal.name)
    fmt.Println("age",c.age)
    fmt.Println("age",c.animal.age)
}
```

程序的输出如下：

```
{{animal 2} cat}
name cat
name animal
age 2
age 2
```

可以看到，`animal`的age属性被组合进了`cat`，成为了`cat`的属性，`animal`和`cat`通过组合达到继承的关系。但是当`animal`的属性和`cat`的属性同名，被组合者的属性会覆盖被组合者的同名属性，也就是说这里`cat`的name属性不会覆盖`animal`的name属性，要访问`animal`的name需要通过被组合者间接访问，也就是需要通过`cat.animal.name`来访问animal`自身的name属性。

类似于struct结构体定义的对象，接口对象中可以组合其它接口对象，这种方式等效于在接口中添加其它接口的方法。举个例子：

```
type Reader interface {
    read()
}

type Writer interface {
    write()
}

//定义上述两个接口的实现类
type MyReadWrite struct{}

func (mrw *MyReadWrite) read() {
    fmt.Println("MyReadWrite...read")
}

func (mrw *MyReadWrite) write() {
    fmt.Println("MyReadWrite...write")
}

//定义一个接口，组合了上述两个接口
type ReadWriter interface {
    Reader
    Writer
}

//上述接口等价于
type ReadWriterV2 interface {
    read()
    write()
} 

//ReadWriter和ReadWriterV2两个接口是等效的，因此可以相互赋值
func interfaceTest() {
    mrw := &MyReadWrite{}
    //mrw对象实现了read()方法和write()方法，因此可以赋值给ReadWriter和ReadWriterV2
    var rw1 ReadWriter = mrw
    rw1.read()
    rw1.write()

    fmt.Println("------")
    var rw2 ReadWriterV2 = mrw
    rw2.read()
    rw2.write()

    //同时，ReadWriter和ReadWriterV2两个接口对象可以相互赋值
    rw1 = rw2
    rw2 = rw1
}
```

如果结构struct实现了接口interface要求的所有方法，所以可以将struct赋值给相应的该接口interface；而如果struct1如果组合的struct了，那么struct1也就实现了接口interface的所有方法的实现，所以可以将struct1赋值给该接口interface。Go语言正是通过这种组合来实现面向对象语言中继承的概念。

Go语言正是通过提供别样的结构体&接口组合的机制，让一个结构体/接口包含另一个结构体/接口类型的匿名成员，这样就可以通过简单的点运算符`x.f`来访问匿名成员链中嵌套的`x.d.e.f`成员。同样的规则，内嵌匿名成员链的方法也会提升为外部类型的方法。

但是需要注意的是：

1. 组合一个类型，这个内部类型的方法就变成了外部类型的方法，但是当它被调用时，方法的接受者是内部类型(被组合类型)，而非外部类型。

```
type Job struct {
    Command string
    *log.Logger
}

func (job *Job)Start() {
    job.Log("starting now...")
    ...
    job.Log("started.")
}
```

上面这个Job例子，即使组合后调用的方式变成了`job.Log(…)`，但`Log`方法的接收者仍然是`log.Logger`指针，因此在`Log`方法中也不可能访问到`job`的其他成员方法和变量。

2. 被组合类型不是基类

如果对传统的面向对象语言中“类”中的继承比较熟悉的话，很可能会倾向于将"被组合类型"看作一个基类，而“外部类型”看作其子类或者继承类，或者将“外部类型”看作"is a"被组合类型，但这样理解是错误的：

```
type Point struct{ X, Y float64 }

type ColoredPoint struct {
    Point
    Color color.RGBA
}

func (p Point) Distance(q Point) float64 {
    dX := q.X - p.X
    dY := q.Y - p.Y
    return math.Sqrt(dX*dX + dY*dY)
}

red := color.RGBA{255, 0, 0, 255}
blue := color.RGBA{0, 0, 255, 255}
var p = ColoredPoint{Point{1, 1}, red}
var q = ColoredPoint{Point{5, 4}, blue}
fmt.Println(p.Distance(q.Point)) // "5"

p.Distance(q) // compile error: cannot use q (ColoredPoint) as Point
```

请注意上面例子中对`Distance`方法的调用，`Distance`方法有一个参数是`Point`类型，但`q`并不是一个`Point`类，所以尽管`q`有着`Point`这个被组合类型，我们也必须要显式地选择它，尝试直接传`q`的话会报错：

实际上，从实现的角度来考虑问题，内嵌字段会指导编译器去生成额外的包装方法来委托已经声明好的方法，和下面的形式是等价的：

```
func (p ColoredPoint) Distance(q Point) float64 {
    return p.Point.Distance(q)
}
```

当`Point.Distance()`被以上编译器生成的包装方法调用时，它的接收器值是`p.Point`，而非`p`。

3. 匿名冲突(duplicate field) 和隐式名字

匿名成员也有一个隐式的名字，以其类型名称（去掉包名部分）作为成员变量的名字。因此不能同一级同时包含两个类型相同的匿名成员，这会导致名字冲突。

```
type Logger struct {
    Level int
}

type MyJob struct {
    *Logger
    Name string
    *log.Logger // duplicate field Logger
}
```

以下两点都间接说明匿名组合不是继承：

1. 匿名成员有隐式的名字
2. 匿名可能冲突(duplicate field)

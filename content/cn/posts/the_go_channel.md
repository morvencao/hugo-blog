---
title: "5分钟系列 -「Go Routine & Channel」"
date: 2018-04-26
categories: ['note', 'tech']
draft: false
---

说到Go语言，不得不提一下Go语言的并发编程。Go从语言层面增加了对并发编程的良好支持，不像Python、Java等其他语言使用Thread库来新建线程，同时使用线程安全队列库来共享数据。Go语言对于并发编程的支持依赖于Go语言的两个基础概念：`Go Routine`和`Go Channel`。

> Note: 也许我们还对并发(Concurrency)和并行(Parallelism)傻傻分不清楚，在这里再次强调两者的不同点：
> 
> Concurrency is about dealing with lots of things at once. Parallelism is about doing lots of things at once.
>
> 也就是说，并发是在同一时间处理多件事情，往往是通过编程的手段，目的是将CPU的利用率提到最高；而并行是在同一时间做多件事情，需要多核CPU的支持。

## Go Routine

Go Routine是Go语言并行编程的核心概念之一，有人将它称作为“协程”，是比Thread更轻量级的并发单元，完全处于用户态并由Go语言运行时管理，最小Go Routine只需极少的栈内存(大约是4~5KB)，这样十几个Go Routine的规模可能体现在底层就是五六个线程的大小，最高同时运行成千上万个并发任务；同时，Go语言内部实现了Go Routine之间的内存共享使得它比Thread更高效，更易用，我们不必再使用类似于晦涩难用的线程安全的队列库来同步数据。

### 创建Go Routine

要创建一个Go Routine，我们只需要在函数调⽤语句前添加`go`关键字，Go语言的调度器会自动将其安排到合适的系统线程上执行。

```
go f(x, y, z)
```

会启动一个新的 Go 程并执行`f(x, y, z)`，`f`, `x`, `y` 和 `z` 的求值发生在当前的Go Routine中，而`f`的执行发生在新的Go Routine中。


实际上，我们在并发编程的过程中经常将一个大的任务分成好几块可以并行的小任务，为每一个小任务创建一个Go Routine。当一个程序启动时，其主函数即在一个单独的Go Routine中运行，我们叫它`main routine`，然后在主函数中使用`go`关键字来创建其他的Go Routine：

```
func subTask() {
    i := 0
    for {
        i++
        fmt.Printf("new goroutine: i = %d\n", i)
        time.Sleep(1 * time.Second)
    }
}

func main() {
    go subTask() // Create go rountine to execute sub task

    i := 0
    // main goroutine
    for {
        i++
        fmt.Printf("main goroutine: i = %d\n", i)
        time.Sleep(1 * time.Second)
    }
}
```

需要注意的是，`main routine`退出之后，由它创建的其他Go Routine也会自动退出。这就提醒我们，在并发编程的时候，一定要在`main routine`退出之前优雅的关闭其他的Go Routine。

### Runtime包

说到Go Routine，不得不提[Runtime包](https://golang.org/pkg/runtime/)。

> Package runtime contains operations that interact with Go's runtime system, such as functions to control goroutines. It also includes the low-level type information used by the reflect package; see reflect's documentation for the programmable interface to the run-time type system.

可以看到Runtime包主要是负责和与Go语言运行时打交道的接口程序包，它可以控制Go Routine，通过反射机制动态获取运行时底层信息。这里，我们主要关注Runtime包控制Go Routine的接口。

1）Gosched

`runtime.Gosched()`的作用是让当前Go Routine主动让出CPU时间片，让Go语言调度器安排其他等待的Go Routine的运行，并在下次某个时候从该位置恢复执行，非常类似于Java线程库的`Thread.yield`。

举个例子：

```
func main() {
    runtime.GOMAXPROCS(1)
    exit := make(chan int)
    go func() {
        defer close(exit)
        go func() {
            fmt.Println("b")
        }()
    }()

    for i := 0; i < 6; i++ {
        if i == 4 {
            runtime.Gosched() // switch go routine
        }
        fmt.Println("a:", i)
    }
    <-exit
}
```

上述这段程序的输出：

```
a: 0
a: 1
a: 2
a: 3
b
a: 5
a: 6
```

2）Goexit

`runtime.Goexit()`主要用于立即终止当前Go Routine的执⾏，Go语言调度器确保所有已注册的`defer`语句被调用执行。

```
func main() {
    go func() {
        defer fmt.Println("A.defer")

        func() {
            defer fmt.Println("B.defer")
            runtime.Goexit() // exit current go routine, import "runtime"
            fmt.Println("B") // never execute this
        }()

        fmt.Println("A")
    }()

    // for loop
    for {
    }
}
```

程序的输出：

```
B.defer
A.defer
```

3) GOMAXPROCS

如果要在Go Routine中使用多核，可以使用`runtime.GOMAXPROCS()`函数设置可以并行计算的CPU核数的最大值，并返回之前的值，当参数小于1时使用默认值。

```
func main() {
    //runtime.GOMAXPROCS(1)
    runtime.GOMAXPROCS(2)

    var wg sync.WaitGroup // import "sync"
    wg.Add(2)

    fmt.Println("Starting Go Routines")
    go func() {
        defer wg.Done()

        for char := 'a'; char < 'a'+26; char++ {
            fmt.Printf("%c ", char)
        }
    }()

    go func() {
        defer wg.Done()

        for number := 1; number < 27; number++ {
            fmt.Printf("%d ", number)
        }
    }()

    fmt.Println("Waiting To Finish")
    wg.Wait()

    fmt.Println("\nTerminating Program")
}
```

## Go Channel

有了Go Routine，我们可以并发地启动多个子任务，极大地提高处理的效率，但是当多个子任务之间有数据要同步怎么办？比如说我有两个子任务，子任务2必须等子任务1将处理了某个数据之后才能启动，怎么保证这样的数据共享与同步？

Go Routine在相同的地址空间中运行，因此可以通过[sync包](https://golang.org/pkg/sync/)提供的锁机制对要访问的共享内存进行同步，不过这在Go语言中并不经常用到，更方便的办法是使用Go Channel。

Go Channel是并发的Go Routine之间进行通信的一种方式，它与Unix中的管道机制类似，底层是一种先入先出（FIFO）的队列。

### Go Channel的类型与声明

**Go Channel的类型**

Go Channel是类型相关的，一个channel只能传递一种类型的数据，类型的定义格式如下：

```
ChannelType = ( "chan" | "chan" "<-" | "<-" "chan" ) ElementType
```

箭头(<-)的指向就是数据的流向，如果没有指定方向，那么channel就是双向的，既可以接收数据，也可以发送数据。`ElementType`指定channel传递的数据类型。

```
chan T          // Send or Receive Data of type T
chan<- T        // only Send Data of type T
<-chan int      // only Receive Data of type T
```

Go Channel必须先创建再使用，和`map`和`slice`数据类型一样，我们使用`make`来创建Go Channel:

```
make(chan T[, capacity])
```

可选的第二个参数`capacity`代表Go Channel最多可容纳元素的数量，代表Go Channel的缓存区的大小。如果没有设置容量，或者容量设置为0, 说明Go Channel没有缓存，只有发送者和接收者都准备好了后它们的通讯才会发生(之前一直阻塞)。如果设置了缓存，只有buffer满了后发送者才会被阻塞，而只有缓存空了后接收者才会被阻塞。

> Note: 一个值为`nil`的Go Channel不会通信

### Go Channel的操作

Go Channel的操作符是箭头`<-`，支持multi-valued assignment：

```
ch <- v       // Send value 'v' to channel 'ch'
v := <-ch     // Receive value fron channel 'ch' and assign it to v
v, ok := <-ch // Receive value fron channel 'ch' and assign it to v, get the status to 'ok'
```

上面第三个例子的返回结果中`ok`用来检查Go Channel的状态，如果`ok`是`false`，表明接收的`v`是Channel传递类型的零值，这个Go Channel被关闭了或者为空。

**发送**

发送操作是用来往Go Channel中发送数据，如`ch <- 3`，它的定义如下:

```
SendStmt = Channel "<-" Expression
Channel  = Expression
```

在通讯(communication)开始前channel和expression必选先求值出来(evaluated)，比如下面的(3+4)先计算出7然后再发送给channel。

```
c := make(chan int)
defer close(c)
go func() { c <- 3 + 4 }()
i := <-c
fmt.Println(i)
```

发送操作被执行前通讯一直被阻塞着。如前所言，对于无缓存的Go Channel而言，只有在接收者准备好后发送操作才被执行；如果有缓存，并且缓存未满，则发送操作会立刻被执行。

> Note:
> - 向一个已经被close的Go Channel中发送数据会导致`run-time panic`
> - 向nil的Go Channel中发送数据会一直被阻塞

**接收**

`<-ch`用来从Go Channel中接收数据，对于无缓存的Go Channel而言，只有在发送者准备好后接收操作才被执行；如果有缓存，并且缓存不为空，则接收操作会立刻被执行。

发送者可通过`close`关闭一个信道来表示没有需要发送的值了。接收者可以通过为接收表达式分配第二个参数来测试信道是否被关闭：若没有值可以接收且信道已被关闭，那么在执行完：

```
v, ok := <-ch
```

之后`ok`会被设置为`false`。

> Note:
> - 从nil的Go Channel中接收数据会一直被block
> - 只有发送者才能关闭信道，而接收者不能
> - 从一个被close的channel中接收数据不会被阻塞，而是立即返回，接收完已发送的数据后会返回元素类型的零值

**Range**

Go语言中的经常使用`for ... range`从Go Channel中读取所有值，直到它被关闭：

```
func main() {
	c := make(chan int)
	go func() {
		for i := 0; i < 10; i = i + 1 {
			c <- i
		}
		close(c)
	}()
	for i := range c {
		fmt.Println(i)
	}
	fmt.Println("Finished")
}
```

上面的代码片段中，`range c`产生的迭代值为向Go Channel中发送的值，它会一直迭代直到Channel `c`被关闭。如果将上面的例子中如果把`close(c)`注释掉，程序会一直阻塞在`for …… range`那一行。

**Select**

`select`语句类似于`switch`语句，只是用来处理多个GO Routine通过Go Channel来实现并发通信的。`select`对应的`case`子句可以是发送表达式，也可以是接收表达式，亦或者`default`表达式，`select`子句可以选择一组可能的发送操作和接收操作去处理；如果有多个`case`子句都可以运行，`select`会随机公平地选出一个执行；如果没有`case`子句满足处理条件，则会默认选择`default`去处理；如果没有`default`子句存在，则`select`语句会一直被阻塞，直到某个`case`需要被处理。

> Note： 最多允许存在一个`default`子句，它可以放在`case`子句列表的任何位置，但一般会将它放在最后；如果没有`default`子句，只有`nil`的Go Channel上的`select`语句会一直被阻塞。

```
func fibonacci(c, quit chan int) {
	x, y := 0, 1
	for {
		select {
		case c <- x:
			x, y = y, x+y
		case <-quit:
			fmt.Println("quit")
			return
		}
	}
}
func main() {
	c := make(chan int)
	quit := make(chan int)
	go func() {
		for i := 0; i < 10; i++ {
			fmt.Println(<-c)
		}
		quit <- 0
	}()
	fibonacci(c, quit)
}
```

`select`语句和`switch`语句一样，它不是循环，它只会选择一个`case`来处理，如果想一直处理Go Channel，可以在外面加一个无限的`for`循环：

```
for {
	select {
	case c <- x:
		x, y = y, x+y
	case <-quit:
		fmt.Println("quit")
		return
	}
}
```

**Timer和Ticker**

`select`语句一个非常重要的应用就是**超时处理**。 之前提到，如果没有`case`需要处理，`select`语句就会一直阻塞着，这时我们就需要一个处理超时的`case`。

下面这个例子我们会在2秒后往channel `c`中发送一个数据，但是`select`设置为1秒超时，因此我们会打印出`timeout 1`，而不是`result 1`：

```
import "time"
import "fmt"

func main() {
    c := make(chan string, 1)
    go func() {
        time.Sleep(time.Second * 2)
        c <- "result 1"
    }()
    select {
    case res := <-c:
        fmt.Println(res)
    case <-time.After(time.Second * 1):
        fmt.Println("timeout 1")
    }
}
```

利用的是`time.After`方法，它返回一个类型为`<-chan Time`的单向的Go Channel，在指定的时间发送一个当前时间给返回的Go Channel中。

事实上，`timer`是一个定时器，代表未来的某个事件，在创建`timer`的时候可以告诉`timer`要等待多长时间，它将创建并返回一个Go Channel，在将来的那个时间向那个Go Channel提供了一个时间值。下面的例子中第二行会阻塞2秒钟左右的时间，直到时间到了才会继续执行：

```
timer1 := time.NewTimer(time.Second * 2)
<-timer1.C
fmt.Println("Timer 1 expired")
```

`ticker`是一个定时触发的计时器，它会以一个间隔(interval)向Go Channel发送一个事件(当前时间)，接收者可以以固定的时间间隔从Go Channel中读取事件。下面的例子每500毫秒触发一次，可以观察输出的时间：

```
ticker := time.NewTicker(time.Millisecond * 500)
go func() {
	for t := range ticker.C {
		fmt.Println("Tick at", t)
	}
}()
```

`timer`和`ticker`都可以通过`Stop()`方法来停止。一旦它停止，接收者不再会从返回的Go Channel中接收到数据了。

## sync.Mutex

我们已经看到Go Channel非常适合在各个Go Routine之间进行通信。但是如果我们并不需要通信呢？比如说，若我们只是想保证每次只有一个Go Routine能够访问一个共享的变量，从而避免冲突？

这里涉及的概念叫做 **互斥(mutual exclusion)** ，我们通常使用 **互斥锁(Mutex)** 这一数据结构来提供这种机制。

Go语言标准库中提供了`sync.Mutex`互斥锁类型及其两个方法：

```
Lock
Unlock
```

我们可以通过在代码前调用`Lock`方法，在代码后调用`Unlock`方法来保证一段代码的互斥执行。也可以用`defer `语句来保证互斥锁一定会被解锁。

```
// SafeCounter is safe to use concurrently.
type SafeCounter struct {
	mu sync.Mutex
	v  map[string]int
}

// Inc increments the counter for the given key.
func (c *SafeCounter) Inc(key string) {
	c.mu.Lock()
	defer c.mu.Unlock()
	// Lock so only one goroutine at a time can access the map c.v.
	c.v[key]++
	
}

// Value returns the current value of the counter for the given key.
func (c *SafeCounter) Value(key string) int {
	return c.v[key]
}

func main() {
	c := SafeCounter{v: make(map[string]int)}
	for i := 0; i < 1000; i++ {
		go c.Inc("somekey")
	}
	time.Sleep(time.Second)
	fmt.Println(c.Value("somekey"))
}
```

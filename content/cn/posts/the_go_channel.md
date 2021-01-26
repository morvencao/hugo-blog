---
title: "Go 协程与管道"
date: 2018-04-26
categories: ['note', 'tech']
draft: false
---

说到 Go 语言，不得不提 Go 语言的并发编程。Go 从语言层面增加了对并发编程的良好支持，不像 Python、Java 等其他语言使用 Thread 库来新建线程，同时使用线程安全队列库来共享数据。Go 语言对于并发编程的支持依赖于 Go 语言的两个基础概念：Go 协程（Routine）和Go 管道（Channel）。

> Note: 也许我们还对并发(Concurrency)和并行(Parallelism)傻傻分不清楚，在这里再次强调两者的不同点：
> 
> Concurrency is about dealing with lots of things at once. Parallelism is about doing lots of things at once.
>
> 也就是说，并发是在同一时间处理多件事情，往往是通过编程的手段，目的是将 CPU 的利用率提到最高；而并行是在同一时间做多件事情，需要多核 CPU 的支持。

## Go 协程

Go 协程是 Go 语言并行编程的核心概念之一，是比 Thread 更轻量级的并发单元，完全处于用户态并由 Go 语言运行时管理，最小 Go 协程只需极少的栈内存(大约是4~5KB)，这样十几个Go 协程的规模可能体现在底层就是五六个线程的大小，最高同时运行成千上万个并发任务；同时，Go 语言内部实现了 Go 协程之间的内存共享使得它比 Thread 更高效，更易用，我们不必再使用类似于晦涩难用的线程安全队列库来同步数据。

### 创建 Go 协程

要创建一个Go 协程，我们只需要在函数调⽤语句前添加 `go` 关键字，Go 语言的调度器会自动将其安排到合适的系统线程上执行。

```golang
go f(x, y, z)
```

会启动一个新的 Go 协程并执行 `f(x, y, z)`。

> Note: 上面启动新的 Go 协程时，`f`, `x`, `y` 和 `z` 的求值发生在当前的 Go 协程中，而` f` 的执行发生在新的Go 协程中。

实际上，我们在并发编程的过程中经常将一个大的任务分成好几块可以并行执行的小任务，为每一个小任务创建一个 Go 协程。当一个程序启动时，其主函数即在一个单独的 Go 协程中运行，我们叫它 main routine，然后在主函数中使用 `go` 关键字来创建其他的 Go 协程：

```golang
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

需要注意的是，`main routine` 退出之后，由它创建的其他 Go 协程也会自动退出。这就提醒我们，在并发编程的时候，一定要在 `main routine` 退出之前优雅的关闭其他的 Go 协程。

### runtime

说到 Go 协程，不得不提 [runtime包](https://golang.org/pkg/runtime/)。

> Package runtime contains operations that interact with Go's runtime system, such as functions to control goroutines. It also includes the low-level type information used by the reflect package; see reflect's documentation for the programmable interface to the run-time type system.

可以看到 runtime 包主要是负责和与 Go 语言运行时打交道的接口程序包，它可以控制 Go 协程，通过反射机制动态获取运行时底层信息。这里，我们主要关注 runtime 包控制 Go 协程的接口。

1. Gosched

`runtime.Gosched()` 的作用是让当前 Go 协程主动让出 CPU 时间片，让 Go 语言调度器安排其他等待的 Go 协程运行，并在下次某个时候从该位置恢复执行，非常类似于 Java 线程库的 `Thread.yield`。

举个例子：

```golang
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

`runtime.Goexit()` 主要用于立即终止当前 Go 协程的执⾏，Go 语言调度器确保所有已注册的 `defer` 语句被调用执行。

```golang
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

如果要在 Go 协程中使用多核，可以使用 `runtime.GOMAXPROCS()` 函数设置可以并行计算的 CPU 核数的最大值，并返回之前的值，当参数小于1时使用默认值。

```golang
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

## Go 管道

有了Go 协程，我们可以并发地启动多个子任务，极大地提高处理的效率，但是当多个子任务之间有数据要同步怎么办？比如说我有两个子任务，子任务2必须等子任务1将处理了某个数据之后才能启动，怎么保证这样的数据共享与同步？

Go 协程在相同的地址空间中运行，因此可以通过 [sync包](https://golang.org/pkg/sync/) 提供的锁机制对要访问的共享内存进行同步，不过这在 Go 语言中并不经常用到，更方便的办法是使用 Go 管道。

Go 管道是并发的Go 协程之间进行通信的一种方式，它与 Unix 中的管道机制类似，底层是一种先入先出（FIFO）的队列。

### Go 管道的类型

1. Go 管道的类型

Go 管道是类型相关的，一个管道只能传递一种类型的数据，类型的定义格式如下：

```
ChannelType = ( "chan" | "chan" "<-" | "<-" "chan" ) ElementType
```

箭头(<-)的指向就是数据的流向，如果没有指定方向，那么管道就是双向的，既可以接收数据，也可以发送数据。`ElementType` 指定管道传递的数据类型。

```
chan T          // Send or Receive Data of type T
chan<- T        // only Send Data of type T
<-chan int      // only Receive Data of type T
```

Go 管道必须先创建再使用，和 `map` 以及 `slice` 数据类型一样，我们使用 `make` 函数来创建 Go 管道：

```golang
make(chan T[, capacity])
```

可选的第二个参数 `capacity` 代表 Go 管道最多可容纳元素的数量，代表Go 管道的缓存区的大小。如果没有设置容量，或者容量设置为0, 说明Go 管道没有缓存，只有发送者和接收者都准备好了后它们的通讯才会发生(之前一直阻塞)。如果设置了缓存，只有缓存满了后发送者才会被阻塞，而只有缓存空了后接收者才会被阻塞。

> Note: 一个值为 `nil` 的 Go 管道不会通信

### Go 管道的操作

Go 管道的操作符是箭头 `<-`，也支持multi-valued assignment：

```golang
ch <- v       // Send value 'v' to channel 'ch'
v := <-ch     // Receive value fron channel 'ch' and assign it to v
v, ok := <-ch // Receive value fron channel 'ch' and assign it to v, get the status to 'ok'
```

上面第三个例子的返回结果中 `ok` 用来检查 Go 管道的状态，如果 `ok` 的值是 `false` ，则接收者 `v` 的值是管道传递类型的零值，这个 Go 管道被关闭了或者为空。

1. 发送

发送操作是用来往 Go 管道中发送数据，如 `ch <- 3`，它的定义如下：

```golang
SendStmt = Channel "<-" Expression
Channel  = Expression
```

在通讯开始前 channel 和 expression 必选先求值，比如下面的 `(3+4)` 先计算出结果然后再发送给管道：

```golang
c := make(chan int)
defer close(c)
go func() { c <- 3 + 4 }()
i := <-c
fmt.Println(i)
```

发送操作被执行前通讯一直被阻塞着。如前所言，对于无缓存的 Go 管道而言，只有在接收者准备好后发送操作才被执行；如果有缓存，并且缓存未满，则发送操作会立刻被执行。

> Note: 关于管道操作还需要注意以下两点：
> - 向一个已经被关闭的 Go 管道中发送数据会导致 `run-time panic`
> - 向空（nil）的 Go 管道中发送数据会一直被阻塞

2. 接收

`<-ch` 用来从 Go 管道中接收数据，对于无缓存的 Go 管道而言，只有在发送者准备好后接收操作才被执行；如果有缓存，并且缓存不为空，则接收操作会立刻被执行。

发送者可通过 `close()` 函数关闭一个信道来表示没有需要发送的值了。接收者可以通过为接收表达式分配第二个参数来测试信道是否被关闭：若没有值可以接收且信道已被关闭，那么在执行完：

```golang
v, ok := <-ch
```

之后 `ok` 会被设置为 `false`。

> Note:
> - 从空（nil）的 Go 管道中接收数据会一直被阻塞
> - 只有发送者才能关闭信道，而接收者不能
> - 从一个已经被关闭的 Go 管道中接收数据不会被阻塞，而是立即返回，接收完已发送的数据后会返回元素类型的零值

3. Range

Go 语言中的经常使用 `for ... range` 从 Go 管道中读取所有值，直到它被关闭：

```golang
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

上面的代码片段中，`range c` 产生的迭代值为向 Go 管道中发送的值，它会一直迭代直到管道 `c` 被关闭。如果将上面的例子中如果把 `close(c)` 注释掉，程序会一直阻塞在 `for …… range` 那一行。

4. Select

`select` 语句类似于 `switch` 语句，只是用来处理多个 GO 协程通过 Go 管道来实现并发通信的。`select` 对应的 `case` 子句可以是发送表达式，也可以是接收表达式，亦或者` default` 表达式，`select` 子句可以选择一组可能的发送操作和接收操作去处理；如果有多个 `case` 子句都可以运行，`select` 会随机公平地选出一个执行；如果没有 `case` 子句满足处理条件，则会默认选择 `default` 去处理；如果没有 `default` 子句存在，则 `select` 语句会一直被阻塞，直到某个 `case` 需要被处理。

> Note： 最多允许存在一个 `default` 子句，它可以放在 `case` 子句列表的任何位置，但一般会将它放在最后；如果没有 `default` 子句，只有 `nil` 的Go 管道上的 `select` 语句会一直被阻塞。

```golang
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

`select` 语句和 `switch` 语句一样，它不是循环，它只会选择一个 `case` 来处理，如果想一直处理 Go 管道，可以在外面加一个无限的 for 循环：

```golang
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

5. Timer & Ticker

`select` 语句一个非常重要的应用就是**超时处理**。 之前提到，如果没有 `case` 需要处理，`select` 语句就会一直阻塞着，这时我们就需要一个处理超时的 `case`。

下面这个例子我们会在2秒后往管道 `c` 中发送一个数据，但是 `select` 设置为1秒超时，因此我们会打印出 `timeout 1`，而不是 `result 1`：

```golang
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

利用的是 `time.After` 方法，它返回一个类型为 `<-chan Time` 的单向的 Go 管道，在指定的时间发送一个当前时间给返回的 Go 管道中。

事实上，`timer` 是一个定时器，代表未来的某个事件，在创建 `timer` 的时候可以告诉 `timer` 要等待多长时间，它将创建并返回一个 Go 管道，在将来的那个时间向那个 Go 管道提供了一个时间值。下面的例子中第二行会阻塞2秒钟左右的时间，直到时间到了才会继续执行：

```golang
timer1 := time.NewTimer(time.Second * 2)
<-timer1.C
fmt.Println("Timer 1 expired")
```

`ticker` 是一个定时触发的计时器，它会以一个间隔(interval)向 Go 管道发送一个事件(当前时间)，接收者可以以固定的时间间隔从 Go 管道中读取事件。下面的例子每500毫秒触发一次，可以观察输出的时间：

```golang
ticker := time.NewTicker(time.Millisecond * 500)
go func() {
	for t := range ticker.C {
		fmt.Println("Tick at", t)
	}
}()
```

`timer` 和 `ticker` 都可以通过 `Stop()` 方法来停止。一旦它停止，接收者不再会从返回的 Go 管道中接收到数据了。

## 同步原语与锁

Go 语言作为一个原生支持协程的语言，当提到并发编程时，往往都离不开锁这一概念。锁是一种并发编程中的同步原语（Synchronization Primitives），它能保证多个 Go 协程在访问同一片内存时不会出现竞争条件（Race condition）等问题。

Go 语言在 [sync](https://pkg.go.dev/sync) 包中提供了用于同步的一些基本原语，包括常见的 `sync.Mutex`、`sync.RWMutex`、`sync.WaitGroup`、`sync.Once` 和 `sync.Cond`：

> Note: Go 语言基础的同步原语是一种相对原始的同步机制，在多数情况下，我们都应该使用抽象层级的更高的管道实现同步。

1. Mutex

Go 语言的 `sync.Mutex` 由两个字段 `state` 和 `sema` 组成。其中 `state` 表示当前互斥锁的状态，而 `sema` 是用于控制锁状态的信号量。

```golang
type Mutex struct {
	state int32
	sema  uint32
}
```

上述两个字段加起来只占8字节空间的结构体表示了 Go 语言中的互斥锁。其中 `state` 表示互斥锁的状态，最低三位分别表示 `mutexLocked`、`mutexWoken` 和 `mutexStarving`，在默认情况下，互斥锁的所有状态位都是`0`，`int32` 中的不同位分别表示了不同的状态；剩下的位置用来表示当前有多少个 `Goroutine` 在等待互斥锁的释放：

`sync.Mutex` 有两种工作模式：**正常模式**和**饥饿模式``。在正常模式下，锁的等待者会按照先进先出的顺序获取锁。但是刚被唤起的 Go 协程与新创建的 Go 协程竞争时，大概率会获取不到锁，为了减少这种情况的出现，一旦 Go 协程超过1毫秒没有获取到锁，它就会将当前互斥锁切换饥饿模式，防止部分 Go 协程被饿死。

从编程的角度来看，`sync.Mutex` 互斥锁涉及到两个方法：`Lock` 和 `Unlock`。

我们可以通过在代码前调用 `Lock` 方法，在代码后调用 `Unlock` 方法来保证一段代码的互斥执行。也可以用 `defer` 语句来保证互斥锁一定会被解锁。

```golang
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

2. RWMutex

读写互斥锁 `sync.RWMutex` 是细粒度的互斥锁，它不限制资源的并发读，但是读写、写写操作无法并行执行。一般来说，常见服务的资源读写比例会非常高，因为大多数的读请求之间不会相互影响，所以我们可以分离读写操作，以此来提高服务的性能。

| - | 读 | 写 |
|  :---: | :---: | :---: |
| 读 | Y | N |
| 写 | N | N |

`sync.RWMutex` 中总共包含以下5个字段：

```golang
type RWMutex struct {
	w           Mutex
	writerSem   uint32
	readerSem   uint32
	readerCount int32
	readerWait  int32
}
```

其中，`w` 提供复用互斥锁提供的能力；`writerSem` 和 `readerSem` 则分别用于写等待读和读等待写：`readerCount` 存储了当前正在执行的读操作数量；`readerWait` 表示当写操作被阻塞时等待的读操作个数；

- 对于“写锁”来说，获取写锁时会先阻塞写锁的获取，后阻塞读锁的获取，这种策略能够保证读操作不会被连续的写操作饿死。
- 对于“读锁”来说，读锁的加锁方法会通过 `sync/atomic.AddInt32` 将 `readerCount` 加一。如果该方法返回负数，则其他 Go 协程获得了写锁，当前 Go 协程陷入休眠等待锁的释放；如果该方法的结果为非负数，则没有 Go 协程获得写锁，当前方法会成功返回；读锁的解锁过程基本是相反的过程。

3. WaitGroup

`sync.WaitGroup` 可以等待一组 Go 协程的返回，常见的使用场景是批量发出 RPC 或 HTTP 请求：

```golang
requests := []*Request{...}
wg := &sync.WaitGroup{}
wg.Add(len(requests))

for _, request := range requests {
    go func(r *Request) {
        defer wg.Done()
        res, err := service.call(r)
    }(request)
}
wg.Wait()
```

通过 `sync.WaitGroup` 将原本顺序执行的代码在多个 Go 协程中并发执行，加快程序处理的速度。

`sync.WaitGroup` 结构体定义中只包含两个成员变量：

```golang
type WaitGroup struct {
	noCopy noCopy
	state1 [3]uint32
}
```

其中，`noCopy` 是一个特殊的私有结构体，保证 `sync.WaitGroup` 不会被开发者通过再赋值的方式拷贝；`state1` 则存储着状态和信号量。

`sync.WaitGroup` 结构体对外暴露了三个方法：`sync.WaitGroup.Add`、`sync.WaitGroup.Wait` 和 `sync.WaitGroup.Done`。其中 `sync.WaitGroup.Done` 只是向 `sync.WaitGroup.Add` 方法传入了`-1`，`sync.WaitGroup` 必须在 `sync.WaitGroup.Wait` 方法返回之后才能被重新使用；`sync.WaitGroup.Done` 只是对 `sync.WaitGroup.Add` 方法的简单封装，我们可以向 `sync.WaitGroup.Add` 方法传入任意负数（需要保证计数器非负）快速将计数器归零以唤醒等待的 Go 协程；
可以同时有多个 Go 协程等待当前 `sync.WaitGroup` 计数器的归零，这些 Go 协程会被同时唤醒；

4. Once

`sync.Once` 可以保证在 Go 程序运行期间的某段代码只会执行一次，举例来说：

```golang
func main() {
    o := &sync.Once{}
    for i := 0; i < 10; i++ {
        o.Do(func() {
            fmt.Println("only once")
        })
    }
}
```

程序运行结果：

```bash
$ go run main.go
only once
```

`sync.Once` 结构体中都只包含一个用于标识代码块是否执行过的 `done` 以及一个互斥锁 `sync.Mutex`：

```golang
type Once struct {
	done uint32
	m    Mutex
}
```

`sync.Once.Do` 是 `sync.Once` 结构体对外唯一暴露的方法，该方法会接收一个入参为空的函数：如果传入的函数已经执行过，会直接返回；如果传入的函数没有执行过，会调用 `sync.Once.doSlow` 执行传入的函数：

```golang
func (o *Once) Do(f func()) {
	if atomic.LoadUint32(&o.done) == 0 {
		o.doSlow(f)
	}
}

func (o *Once) doSlow(f func()) {
	o.m.Lock()
	defer o.m.Unlock()
	if o.done == 0 {
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}
```

作为用于保证函数执行次数的 `sync.Once` 结构体，它使用互斥锁和 `sync/atomic` 包提供的方法实现了某个函数在程序运行期间只能执行一次的语义。在使用该结构体时，我们也需要注意以下的问题：`sync.Once.Do` 方法中传入的函数只会被执行一次，哪怕函数中发生了 panic；两次调用 `sync.Once.Do` 方法传入不同的函数只会执行第一次调传入的函数；

5. Cond

`sync.Cond` 可以让一组 Go 协程都在满足特定条件时被唤醒，`sync.Cond` 结构体在初始化时都需要传入一个互斥锁。举例来说：

```golang
var status int64

func main() {
	c := sync.NewCond(&sync.Mutex{})
	for i := 0; i < 10; i++ {
		go listen(c)
	}
	time.Sleep(1 * time.Second)
	go broadcast(c)

	ch := make(chan os.Signal, 1)
	signal.Notify(ch, os.Interrupt)
	<-ch
}

func broadcast(c *sync.Cond) {
	c.L.Lock()
	atomic.StoreInt64(&status, 1)
	c.Broadcast()
	c.L.Unlock()
}

func listen(c *sync.Cond) {
	c.L.Lock()
	for atomic.LoadInt64(&status) != 1 {
		c.Wait()
	}
	fmt.Println("listen")
	c.L.Unlock()
}
```

程序运行结果：

```
$ go run main.go
listen
...
listen
```

上述代码同时运行了10 个 Go 协程通过 `sync.Cond.Wait` 等待特定条件的满足；1个 Go 协程会调用 `sync.Cond.Broadcast` 唤醒所有陷入等待的 Go 协程；调用 `sync.Cond.Broadcast` 方法后，上述代码会打印出10次 “listen” 并结束调用。

`sync.Cond` 对外暴露的 `sync.Cond.Wait` 方法会将当前 Go 协程陷入休眠状态；`sync.Cond.Signal` 和 `sync.Cond.Broadcast` 将唤醒陷入休眠的 Go 协程，它们的实现有一些细微的差别：`sync.Cond.Signal` 方法会唤醒队列最前面的 Go 协程，而 `sync.Cond.Broadcast` 方法会唤醒队列中全部的 Go 协程。

需要注意的是，`sync.Cond` 不是一个常用的同步机制，但是在条件长时间无法满足时，与使用 `for {}` 进行忙碌等待相比，`sync.Cond` 能够让出处理器的使用权，提供 CPU 的利用率。此外，还需要说明的是：

- `sync.Cond.Wait` 在调用之前一定要使用获取互斥锁，否则会触发程序崩溃；
- `sync.Cond.Signal` 唤醒的 Go 协程都是队列最前面、等待最久的 Go 协程；
- `sync.Cond.Broadcast` 会按照一定顺序广播通知等待的全部 Go 协程；

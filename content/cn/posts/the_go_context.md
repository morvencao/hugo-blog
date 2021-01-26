---
title: "Go 上下文"
date: 2018-06-12
categories: ['note', 'tech']
draft: false
---

从 Go 语言从1.7开始，正式将 [context](golang.org/x/net/context) 即“上下文”包引入官方标准库。事实上，我们在 Go 语言编程中经常遇到“上下文”，不论是在一般的服务器代码还是复杂的并发处理程序中，它都起到很重要的作用。今天，我们就来深入探讨一下它的实现以及最佳实践。

官方文档对于 context 包的解释是：

> Package context defines the Context type, which carries deadlines, cancelation signals, and other request-scoped values across API boundaries and between processes.

简单来说，“上下文”可以理解为程序单元的一个运行状态、现场、快照。其中上下是指存在上下层的传递，上层会把内容传递给下层，而程序单元则指的是 Go 协程。每个 Go 协程在执行之前，都要先知道程序当前的执行状态，通常将这些执行状态封装在一个“上下文”变量中，传递给要执行的 Go 协程中。而 context 包是专门用来简化处理针对单个请求的多个 Go 协程与请求截止时间、取消信号以及请求域数据等相关操作的。一个常遇到的例子是在 Go 语言实现的服务器程序中，每个网络请求一般需要创建单独的Go 协程进行处理，这些 Go 协程有可能涉及到多个 API 的调用，进而可能会开启其他的 Go 协程；由于这些 Go 协程都是在处理同一个网络请求，所以它们往往需要访问一些共享的资源，比如用户认证令牌环、请求截止时间等；而且如果请求超时或者被取消后，所有的 Go 协程都应该马上退出并且释放相关的资源。使用上下文可以让 Go 语言开发者方便地实现这些 Go 协程之间的交互操作，跟踪并控制这些 Go 协程，并传递请求相关的数据、取消 Go 协程的信号或截止日期等。

![go-context.jpg](https://i.loli.net/2021/01/30/3EGqicXUKPL8NVB.jpg)

## 上下文的数据结构

context 包中核心数据结构是一种嵌套的结构或者说是单向继承的结构。基于最初的上下文（也叫做“根上下文”），开发者可以根据使用场景的不同定义自己的方法和数据来继承“根上下文”；正是上下文这种分层的组织结构，允许开发者在每一层上下文中定义一些不同的特性，这种层级式的组织也使得上下文易于扩展、职责清晰。

context 包中最基础的数据结构是 `Context` 接口：

```golang
type Context interface {
    // Done returns a channel that is closed when this Context is canceled or times out.
    Done() <-chan struct{}

    // Err indicates why this context was canceled, after the Done channel is closed.
    Err() error

    // Deadline returns the time when this Context will be canceled, if any.
    Deadline() (deadline time.Time, ok bool)

    // Value returns the value associated with key or nil if none.
    Value(key interface{}) interface{}
}
```

`Context` 接口定义了4个方法：

- `Done()` 方法返回一个只读的任何类型管道；使用该方法开发者可以在接收到上下文的取消请求，或者截止时间到了之后做一些清理操作，然后退出 Go 协程，释放相关资源，也可以调用 `Err()` 方法得到上下文被取消的原因，或者调用 `Value()` 方法得到上下文中的相关值；
- `Err()` 方法返回上下文被取消的原因，此方法一般是在 `Done()` 方法返回的管道有数据的时候（表明相应的上下文被取消）调用；
- `Deadline()` 方法即设置上下文的截止时间，到了这个时间上下文会自动发起取消请求；当第二个返回值 `ok` 为 `false` 时表示没有设置截止时间，如果需要取消的话，需要调用取消函数进行取消；
-  `Value()` 方法通过“键”获取上下文上绑定的“值”，这个方法是线程安全的，与 `Err()` 方法一样，一般是在 `Done()` 方法返回的管道有数据的时候调用；

## 上下文的实现原理

既然 Context 被定义为一个接口，那么我们就会想到开发者要用它就得实现接口所定义的4个方法。但上下文推荐的用法并不是这样，context 包里面定义了一个 Context 接口的实现叫作 `emptyCtx`：

```golang
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {

	return
}

func (*emptyCtx) Done() <-chan struct{} {

	return nil
}

func (*emptyCtx) Err() error {

	return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {

	return nil
}
```

以上我们看到的 `emptyCtx` 结构体定义，发现 `emptyCtx` 是一个不能被取消，没有设置截止时间也没有没有携带任何值的 Context 接口的实现。但我们一般并不直接使用它，而是通过 context 包中的两个工厂方法来来基于 `emptyCtx` 创建两种不同的 Context 对象，需要开始上下文的时候都是以这两个 Context 对象作为最顶层的“根上下文”，然后再衍生出其他的“子上下文”，最终这些上下文被组织成一棵树状结构；这样，当一个上下文被取消时，所有继承自它的上下文都会被自动取消。这个两个上下文对象的工厂方法定义如下：

```golang
var (
	background = new(emptyCtx)
	todo = new(emptyCtx)
)

func Background() Context {
	return background
}

func TODO() Context {
	return todo
}
```

- `Background()` 工厂方法得到上下文的默认值，主要用于 `main()` 函数、初始化以及测试代码中，作为上下文这个树结构的“根上下文”，它不能被取消；
- `TODO()` 工厂方法创建的是一个 todo 上下文，一般使用的比较少，应该仅在不确定应该使用哪种上下文时使用；

## 上下文的继承衍生

我们在上面介绍过所有的 Context 对象被组织单向继承的树状结构，一旦有了“根上下文”，那么是如何衍生更多的“子上下文” 的呢？这就要依靠 context 包为提供的一系列 `With` 函数了。

```golang
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
func WithDeadline(parent Context, deadline time.Time) (ctx Context, cancel CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (ctx Context, cancel CancelFunc)
func WithValue(parent Context, key, val interface{}) Context
```

通过这4个 `With` 函数，开发者就可以创建上下文树，树的每个节点都可以有任意多个子节点，节点层级可以有任意多个。

- `WithCancel` 函数，传递一个“父 context”作为参数，返回“子 context”，以及一个取消函数用来取消返回的 context；
- `WithDeadline` 函数，和 `WithCancel` 函数类似，但会多传递一个截止时间参数，意味着到了这个时间点，会自动取消对应的 context，当然也可以不等到这个时候，可以提前通过取消函数进行取消；
- `WithTimeout` 函数和 `WithDeadline` 函数基本一样，只不过传递的参数是上下文超时时间，即多少时间后自动取消对应的 context；
- `WithValue` 函数和其他3个函数不同，它不是用来取消上下文的，只是为了生成一个绑定了键值对数据的 context，这个绑定的数据可以通过 `context.Value` 方法访问，一般想要通过上下文来传递数据时可以通过这个方法；

## 上下文最佳实践

- “根上下文” 一般为 background 上下文，通过调用 `context.Background()` 得到；
- 不要把上下文对象放在结构体定义中，而是以参数的方式显示地在函数间传递；
- 一般把上下文作为第一个参数传递给入口请求和出口请求链路上的每一个函数，变量名推荐使用 `ctx`；
- 不要传递值为 `nil` 的上下文给函数或者方法，否则在追踪的时候，就会断了上下文树的连接；
- 使用上下文的 `Value()` 方法应该传递必须的数据，不要什么数据都使用 `Value()` 方法传递；上下文传递数据是线程安全的；
- 可以把一个上下文对象传递给任意多个 Go 协程，对它执行取消操作时，所有 Go 协程都会接收到取消信号；

下面是我个人总结的关于上下文典型使用实例：

1. 使用 `Done()` 方法主动取消上下文：

```golang
func process(ctx context.Context, wg *sync.WaitGroup) error {
	defer wg.Done()
	respC := make(chan int)
	// business logic
	go func() {
		time.Sleep(time.Second * 2)
		respC <- 10
	}()
	// wait for signal
	for {
		select {
		case <-ctx.Done():
			fmt.Println("cancel")
			return errors.New("cancel")
		case r := <-respC:
			fmt.Println(r)
		}
	}
}

func main() {
	wg := new(sync.WaitGroup)
	ctx, cancel := context.WithCancel(context.Background())
	wg.Add(1)
	go process(ctx, wg)
	time.Sleep(time.Second * 5)
	// trigger context cancel
	cancel()
	// wait for gorountine exit...
	wg.Wait()
}
```

运行程序的输出：

```
10
cancel
```

2. 超时自动取消上下文：

```golang
func process(ctx context.Context, wg *sync.WaitGroup) error {
	defer wg.Done()

	for i := 0; i < 1000; i++ {
		select {
		case <-time.After(2 * time.Second):
			fmt.Println("processing... ", i)

		// receive cancelation signal in this channel
		case <-ctx.Done():
			fmt.Println("Cancel the context ", i)
			return ctx.Err()
		}
	}
	return nil
}

func main() {
    wg := new(sync.WaitGroup)
	ctx, cancel := context.WithTimeout(context.Background(), 4*time.Second)
	defer cancel()

	wg.Add(1)
	go process(ctx, wg)
	wg.Wait()
}
```

运行程序的输出：

```
processing...  0
processing...  1
Cancel the context  2
```

3. 通过上下文的 `WithValue()` 方法在 Go 协程之间传递数据

```golang
func main() {
	ctx, cancel := context.WithCancel(context.Background())
	valueCtx := context.WithValue(ctx, "mykey", "myvalue")

	go watch(valueCtx)
	time.Sleep(10 * time.Second)
	cancel()

	time.Sleep(5 * time.Second)
}

func watch(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			fmt.Println(ctx.Value("mykey"), "is cancel")
			return
		default:
			fmt.Println(ctx.Value("mykey"), "int goroutine")
			time.Sleep(2 * time.Second)
		}
	}
}
```

运行程序的输出：

```
myvalue int goroutine
myvalue int goroutine
myvalue int goroutine
myvalue int goroutine
myvalue int goroutine
myvalue int goroutine
myvalue is cancel
```

## 总结

Go 语言中的“上下文”的主要作用还是在多个 Go 协程或者模块之间同步取消信号或者截止日期，用于减少对资源的消耗和长时间占用，避免资源浪费，虽然传值也是它的功能之一，但是这个功能我们还是很少用到。在真正使用传值的功能时我们也应该非常谨慎，不能将请求的所有参数都使用“上下文”进行传递，这是一种非常差的设计，比较常见的使用场景是传递请求对应用户的认证令牌以及用于进行分布式追踪的请求 ID。

在 Go 语言编程中，通常不能直接杀死 Go 协程，协程的关闭一般会用“管道 + `select`”的方式来控制。但是在某些场景下，例如为了处理一个请求衍生了很多 Go 协程，这些协程之间需要共享一些全局变量、有共同的截止时间等，而且可以同时被关闭，这种情况下再使用“管道 + `select`”就比较麻烦，而可以通过“上下文”就可以轻松应对。

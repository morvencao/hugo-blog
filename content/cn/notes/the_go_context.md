---
title: "理解Go Context"
date: 2019-05-02
type: "notes"
draft: false
---

从go1.7开始，正式将`context(golang.org/x/net/context)`，即“上下文”包引入官方标准库。事实上，我们经常见到它，不论是在一般的服务器代码还是在复杂的并发处理程序中，它都起到很重要的作用。今天，我们就来深入研究一下它的实现以及最佳实践。

官方文档对于`context`包的解释是：

> Package context defines the Context type, which carries deadlines, cancelation signals, and other request-scoped values across API boundaries and between processes.

简单来说，`context`包是专门用来简化处理针对单个请求的多个goroutine与请求截止时间、取消信号以及请求域的数据等相关操作。一个简单的例子是在典型的go服务器程序中，每个网络请求都需要创建单独的goroutine进行处理，这些goroutine有可能涉及多个API的调用，进而可能会开启其他的goroutine；由于这些goroutine都是在处理同一个网络请求，所以它们往往需要访问一些共享的资源，比如用户认证token、请求截止时间等；而且如果请求超时或者被取消后，所有的goroutine都应该马上退出并且释放相关的资源。使用`context`，即“上下文”，可以让go开发者方便地实现这些多个goroutine之间的交互操作，跟踪并控制这些goroutine，并传递request相关的数据、取消goroutine的signal或截止日期等。


### Context结构

`context`包中核心的数据结构是一种嵌套的结构或者说是单向继承的结构。基于最初的context（根context），开发者可以根据使用场景的不同定义自己的方法和数据来继承根context；正是context这种分层的组织结构，允许开发者在每一层context都定义一些不同的特性，这种层级式的组织也使得context易于扩展，职责清晰。

`context`包中的最基础的数据结构是一个`Context`接口：

```
type Context interface {
    // Done returns a channel that is closed when this Context is canceled
    // or times out.
    Done() <-chan struct{}

    // Err indicates why this context was canceled, after the Done channel
    // is closed.
    Err() error

    // Deadline returns the time when this Context will be canceled, if any.
    Deadline() (deadline time.Time, ok bool)

    // Value returns the value associated with key or nil if none.
    Value(key interface{}) interface{}
}
```

`Context`接口的定义包括4个方法：

- `Done`方法返回一个只读的任何类型channel；使用`Done`方法开发者可以在接收到context的取消请求，或者截止时间到了之后做一些清理操作，然后退出goroutine，释放相关资源，也可以调用`Err`方法得到context被取消的原因，或者调用`Value`方法得到上下文中的相关值；
- `Err`方法返回context被取消的错误原因，此方法一般是在`Done`方法返回的channel有数据的时候（表明context被取消）调用；
- `Deadline`方法即设置context的截止时间，到了这个时间context会自动发起取消请求；当第二个返回值`ok`为`false`时表示没有设置截止时间，如果需要取消的话，需要调用取消函数进行取消；
-  `Value`方法通过key获取context上绑定的值，这个方法是线程安全的，与`Err`方法一样，一般是在`Done`方法返回的channel有数据的时候（表明context被取消）调用；

### Context的实现

既然Context是一个接口，那么我们就会想到开发者要用它就得实现它所定义的4个方法。但context推荐的用法并不是这样，context包里面定义了一个Context的实现：

```
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

以上我们看到的`emptyCtx`结构体定义，发现`emptyCtx`是一个不能被取消，没有设置截止时间也没有没有携带任何值的context的实现。但我们一般并不直接使用它，而是通过`context`包中的两个工厂方法来来基于`emptyCtx`创建两种不同的context，需要开始上下文的时候都是以这两个作为最顶层的根context，然后再衍生出其他的子context，最终这些context被组织成一棵树状结构；这样，当一个context被取消时，所有继承自它的context都会被自动取消。这个两个context的定义如下：

```
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


- `Background`工厂方法得到一个background的context，主要用于`main`函数、初始化以及测试代码中，作为context这个树结构的根context，它不能被取消；
- `TODO`工厂方法创建的是一个todo的context，一般使用的比较少；

### Context的继承

我们在上面介绍过所有的context对象被组织单向继承的树状结构，一旦有了根context，那么是如何衍生更多的子context的呢？这就要依靠context包为提供的一系列With函数了。

```
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)

func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)

func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)

func WithValue(parent Context, key, val interface{}) Context
```

通过这4个With函数，开发者就可以创建context树，树的每个节点都可以有任意多个子节点，节点层级可以有任意多个。

- `WithCancel`函数，传递一个父Context作为参数，返回子Context，以及一个取消函数用来取消context；
- `WithDeadline`函数，和`WithCancel`函数类似，但会多传递一个截止时间参数，意味着到了这个时间点，会自动取消对应的context，当然也可以不等到这个时候，可以提前通过取消函数进行取消；
- `WithTimeout`函数和`WithDeadline`函数基本一样，只不过传递的参数context超市时间，即多少时间后自动取消对应的context；
- `WithValue`函数和其他3个函数不同，它不是用来取消context的，只是为了生成一个绑定了一个键值对数据的context，这个绑定的数据可以通过`context.Value`方法访问，一般想要通过上下文来传递数据时可以通过这个方法；

### Context最佳实践

- 根context一般为Background，通过`context.Background()`得到；
- 不要把context对象放在结构体定义中，而是以参数的方式显示地在函数间传递；
- 一般把context作为第一个参数传递给入口请求和出口请求链路上的每一个函数，变量名推荐使用`ctx`；
- 不要传递值为`nil`的context给函数或者方法，否则在追踪的时候，就会断了context树的连接；
- context的`Value`方法应该传递必须的数据，不要什么数据都使用`Value`方法传递；context传递数据是线程安全的；
- 可以把一个context对象传递给任意多个gorotuine，对它执行取消操作时，所有goroutine都会接收到取消信号；

### Context典型使用实例

1. 使用`Done`方法主动取消context：

```
func process(ctx context.Context, wg *sync.WaitGroup) error {
	defer wg.Done()
	respC := make(chan int)
	// business logic
	go func() {
		time.Sleep(time.Second * 5)
		respC <- 10
	}()
	// wait for signal
	select {
	case <-ctx.Done():
		fmt.Println("cancel")
		return errors.New("cancel")
	case r := <-respC:
		fmt.Println(r)
		return nil
	}
}

func main() {
	wg := new(sync.WaitGroup)
	ctx, cancel := context.WithCancel(context.Background())
	wg.Add(1)
	go process(ctx, wg)
	time.Sleep(time.Second * 2)
	// trigger context cancel
	cancel()
	// wait for gorountine exit...
	wg.Wait()
}
```

2. 超时自动取消context：

```
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
	go process(ctx. wg)
	wg.Wait()
}
```

3. 通过`WithValue`在goroutine之间传递数据：

```
func main() {
	ctx, cancel := context.WithCancel(context.Background())
	valueCtx := context.WithValue(ctx, key, "myvalue")

	go watch(valueCtx)
	time.Sleep(10 * time.Second)
	cancel()

	time.Sleep(5 * time.Second)
}

func watch(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			fmt.Println(ctx.Value(key), "is cancel")
			return
		default:
			fmt.Println(ctx.Value(key), "int goroutine")
			time.Sleep(2 * time.Second)
		}
	}
}
```

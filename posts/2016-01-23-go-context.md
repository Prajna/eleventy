---
title: Go Context
description: go context机制
date: 2016-01-23
tags:
	- golang
layout: layouts/post.njk
---
context库是go一个比较有意思的标准库，其它语言里面我还真的没见过这么设计的。

大概100%的人吐槽过go里面
```go
if err != nil {
    ...
}
```

满天飞，也大概有100%的人吐槽过go函数几乎都必返回一个error；

但是我还真没见过大家吐槽
```go
func blabla(ctx context.Context, ...) ... {
    ...
}
```
满天飞的。

我们来看Google官方是怎么说的：
```english
At Google, we require that Go programmers pass a Context parameter as the first argument to every function on the call path between incoming and outgoing requests. (...) It provides simple control over timeouts and cancelation and ensures that critical values like security credentials transit Go programs properly.

在Google，我们要求所有go程序员都把context当成第一个参数，从头传到尾。在取消和超时上它提供了很简单的控制，也保证了重要的值（比如crendentials）可以稳稳到达go程序。
```
我猜大家并不吐槽的原因，只是因为大家写go并不会像Google这么控制狂，所有函数都必须以context.Context为第一个参数吧。

下面用一个简单的例子，来说明一下context到底在`cancel` `timeout`上面有什么作用。

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	// Set a duration.
	duration := 150 * time.Millisecond

	// Create a context that is both manually cancellable and will signal
	// cancel at the specified duration.
	ctx, cancel := context.WithTimeout(context.Background(), duration)
	defer cancel()

	// Create a channel to receive a signal that work is done.
	ch := make(chan int, 1)

	// Ask the goroutine to do some work for us.
	go func() {
		// Simulate work.
		time.Sleep(50 * time.Millisecond)

		// Report the work is done.
		ch <- 100
	}()

	// Wait for the work to finish. If it takes too long, move on.
	select {
	case d := <-ch:
		fmt.Println("work complete", d)

	case <-ctx.Done():
		fmt.Println("work cancelled")
	}
}
```
运行结果：
```shell
$ go run main.go
work complete 100
```
这段代码做了下面的事：
- 定义了一个timeout context，规定在150ms之后就超时。
- 另起了一个goroutinue，睡了50ms后往ch里面塞了个数字。
- main中捕捉到了100，打印出来work complete 100。

如果在goroutine中把sleep时间从50改到250，最终结果就是
```shell
$ go run main.go
work cancelled
```
原因也很好理解，250ms > 150ms，于是走向了下一条分支也就是`ctx.Done()`。

那么在这个context到底发生了什么？它是怎么控制超时的？

```go
ctx, cancel := context.WithTimeout(context.Background(), duration)
```
在这一句，就体现了context一个很重要的设计理念：derived。
首先我们调用了一个context.Background()，它返回了一个interface context。在go中，只要实现以下就是一个context：
```go
type Context interface {
    Done() <-chan struct{}
    Err() error
    Deadline() (deadline time.Time, ok bool)
    Value(key interface{}) interface{}
}
```
而context.Background()返回的实际struct是什么呢？是其内部一个私有的`emptyCtx`。
```go
// An emptyCtx is never canceled, has no values, and has no deadline. It is not
// struct{}, since vars of this type must have distinct addresses.
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
从代码也可以看到，这个`emptyCtx`什么也不干。但是我们都会先创建一个`emptryCtx`，然后由它去生成其它`context`，这样它们就会像一颗树一样，共享所有的操作和值。

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}
```
在`context`库里面`WithTimeout`直接调用了`WithDeadline`。

```go
// WithDeadline returns a copy of the parent context with the deadline adjusted
// to be no later than d. If the parent's deadline is already earlier than d,
// WithDeadline(parent, d) is semantically equivalent to parent. The returned
// context's Done channel is closed when the deadline expires, when the returned
// cancel function is called, or when the parent context's Done channel is
// closed, whichever happens first.
//
// Canceling this context releases resources associated with it, so code should
// call cancel as soon as the operations running in this Context complete.
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
		// The current deadline is already sooner than the new one.
		return WithCancel(parent)
	}
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  d,
	}
	propagateCancel(parent, c)
	dur := time.Until(d)
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded) // deadline has already passed
		return c, func() { c.cancel(false, Canceled) }
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.err == nil {
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded)
		})
	}
	return c, func() { c.cancel(true, Canceled) }
}
```
前面的两个`if`都是保护性质的，从第一个`if`也可以看到`context`的嵌套特性。

在`if`过后，我们可以看到它新建了一个`timerCtx`，而其中又新建了一个`cancelCtx`，这个`cancelCtx`是整个`context`的重中之重。

```go
type cancelCtx struct {
	Context // 保存parent context

	mu       sync.Mutex // 保护数据
	done     chan struct{} // 用来标识为是否被关闭
	children map[canceler]struct{} // 保存所有的canceler，canceler是个接口，在context库里面实现此接口的有timerCtx和cancelCtx
	err      error // 当cancel之后赋予值，否则为nil
}
```
从这个struct里面可以看到，要同步调用`cancel`，就必须要构建起父子`context`之间的关系，所以`children`这个map赋值是又是重中之重。那么`context`是在哪里构建这个关系的呢？是紧接着的`propagateCancel`方法。

```go
// propagateCancel arranges for child to be canceled when parent is.
func propagateCancel(parent Context, child canceler) {
	done := parent.Done()
	if done == nil {
		return // parent is never canceled
	}

	select {
	case <-done:
		// parent is already canceled
		child.cancel(false, parent.Err())
		return
	default:
	}

	if p, ok := parentCancelCtx(parent); ok {
		p.mu.Lock()
		if p.err != nil {
			// parent has already been canceled
			child.cancel(false, p.err)
		} else {
			if p.children == nil {
				p.children = make(map[canceler]struct{})
			}
			p.children[child] = struct{}{}
		}
		p.mu.Unlock()
	} else {
		atomic.AddInt32(&goroutines, +1)
		go func() {
			select {
			case <-parent.Done():
				child.cancel(false, parent.Err())
			case <-child.Done():
			}
		}()
	}
}
```
因为要实现联级`cancel`，根本无需关心没有实现`canceler`接口的`valueCtx`，只需要关心`cancelCtx`和`timerCtx`。
- 找到第一个非空的`Done()`的`context`实例。
- 如果是`canceler` interface的实现，那么直接就可以利用它的children。如果发现已经`cancel`，就可以直接调用child的`cancel`；如果没有，那就加入`children`。
- 如果不是，那就得加一个goroutine，在parent取消的时候，也取消child。

总结：
- `context`接口只有四个方法。
- `context`库中只有四个实现了`context`接口的struct，`emptyCtx` `cancelCtx` `timerCtx`和`valueCtx`。
- `timerCtx`也是利用了`cancelCtx`，所以`cancelCtx`才是重中之重。
- `cancelCtx`实现的重点只有两个，如何联级取消 + 如何构建树形结构。
- 树形结构在`propagateCancel`中构建，往上追溯到第一个`cancelCtx`，然后利用它的`children`。
- 联级取消就是在`propagateCancel`构建好之后，循环调用`children`的`cancel`。
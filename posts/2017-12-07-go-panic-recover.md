---
title: Go Panic和Recover(1)
description: 会写一写我在网上看到的关于Go当中的panic和recover机制。
date: 2017-12-07
tags:
	- golang
layout: layouts/post.njk
---
关于Go当中的panic和recover，网上的文章也是挺铺天盖地的。今天我想在这边把自己的一些实践 & 想法写下来。

谈到panic和recover，就一定要带上defer。最近电面某大厂，面试官一句recover和defer有关系么，直接让我把他们刷了下来。

# *defer*
`defer`语句是go程序员非常熟悉的。它的主要用处就是用来做一些清理工作。

```go
// 这个程序是把一个文件的内容，拷贝到另外一个文件。
func CopyFile(dstName, srcName string) (written int64, err error) {
    src, err := os.Open(srcName)
    if err != nil {
        return
    }

    dst, err := os.Create(dstName)
    if err != nil {
        return
    }

    written, err = io.Copy(dst, src)
    dst.Close()
    src.Close()
    return
}
```

上面这段代码如果正常工作的话，是没问题的。但是如果os.Create这一步出错，那么src就不会被正确关闭。在这种情况下，我们需要把`Close()`都写到defer中去。
```go
func CopyFile(dstName, srcName string) (written int64, err error) {
    src, err := os.Open(srcName)
    if err != nil {
        return
    }
    defer src.Close()

    dst, err := os.Create(dstName)
    if err != nil {
        return
    }
    defer dst.Close()

    return io.Copy(dst, src)
}
```
相信大家对这样的重构都已经习以为常了。

# *panic* & *recover*
`panic`，在Go语言作者看来，是一个用来控制`逻辑流转`的关键词。这个可能跟大家印象中不太一样，大家印象中的panic就是出大错误的时候调用一下，但又不像`os.Exit(1)`那么直挺挺。

我们可以先从`panic`的运转方式来看一下：
* 函数F调用panic
* 函数F的后续调用将停止
* 函数F中的所有defer将被正常执行
* 函数F返回到它的caller。

对于caller来说，函数F的返回也等同于一个panic，所以上面四步会再循环一下，回到caller的caller...然后接着循环。就像冒泡泡一样，在当前goroutine中所有函数都跑完之后，程序崩溃。

听起来是不是非常熟悉？没错，就像是其它语言中的抛错一样(ruby raise, java throw)，如果在当前的代码块中没有catch住，就会把错误一层层往上抛，直到最顶上一层。

而recover则是起到了catch的作用，可以把控制流再次夺回来。

`recover`的机制是这样的：
* 它只在defer中生效。
* 在panic状态下，recover会得到当时传给panic的值，并且阻止panic继续往上冒泡。

下面有几个例子，大家可以来看看recover是怎么工作的。
```go
func main() {
	defer fmt.Println("in main")
	if err := recover(); err != nil {
		fmt.Println(err)
	}

	panic("unknown err")
}
```
```shell
$ go run main.go
in main
panic: unknown err

goroutine 1 [running]:
main.main()
        /Users/prajna/main.go:11
exit status 2
```
可以看得到，这个程序并没有正常退出。因为`recover`调用在`panic`之前，在非panic状态下调用`recover`，它会返回nil而且不会有任何效果。所以我们才需要在`defer`中调用`recover`，这样子才会有效果。

```go
package main

import (  
    "fmt"
)

func recoverFullName() {
    if r := recover(); r!= nil {
        fmt.Println("recovered from ", r)
    }
}

func fullName(firstName *string, lastName *string) {  
    defer recoverFullName()
    if firstName == nil {
        panic("runtime error: first name cannot be nil")
    }
    if lastName == nil {
        panic("runtime error: last name cannot be nil")
    }
    fmt.Printf("%s %s\n", *firstName, *lastName)
    fmt.Println("returned normally from fullName")
}

func main() {  
    defer fmt.Println("deferred call in main")
    firstName := "Elon"
    fullName(&firstName, nil)
    fmt.Println("returned normally from main")
}
```
```shell
$ go run main.go
recovered from  runtime error: last name cannot be nil
returned normally from main
deferred call in main
```
这个例子就可以看得到，整个程序并没有因为`panic`而崩溃，因为`recover`把它从panic状态下拯救了回来。在`recoverFullName`中调用的`recover`，其返回值正是`panic`的参数。值得注意的是，在`recover`调用之后，panic状态得以消除，控制权回到了caller的手中，在这个例子里面也就是main函数；而不是接着panic下面的语句继续执行。

`defer` & `recover` & `panic`这三者的结合，让我们不用`err != nil`满街飞，而采用控制流程的方式来写代码。而它们内部的实现，我们在下一篇讲解。

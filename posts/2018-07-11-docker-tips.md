---
title: Dockerfile的一些小技巧(1)
description: build dockerfile无非就是两个考量选项：速度、大小。这一篇里面我只讲大小。
date: 2018-07-11
tags:
	- docker
layout: layouts/post.njk
---
衡量Dockerfile写得好不好，就只有两个维度：build的速度，以及build出来的image大小。

这一篇里面我会先讲大小，速度篇会详细详解大小篇中用到的一些技术对速度的影响。image的大小，在multi-stage出现之后，已经出现了质的变化。它让我们可以从之前的build中把最需要部分取出来，从而大大减少image大小。

## multi-stage
---
来看最简单的hello world
```c
int main () {
  puts("Hello, world!");
  return 0;
}
```
```dockerfile
FROM gcc
COPY hello.c .
RUN gcc -o hello hello.c
CMD ["./hello"]
```
有点经验的人都知道这样不行，编译出来的`hello`可能只有几kb，但是整个镜像因为有了整一套gcc，所以大小会严重超标。

```dockerfile
FROM gcc AS first
WORKDIR /src
COPY hello.c .
RUN gcc -o hello hello.c
FROM ubuntu
COPY --from=first /src/hello .
CMD ["./hello"]
```
multi-stage语法相信大家也比较熟悉了，需要注意的是这里我多加了一句
```dockerfile
WORKDIR /src
```
这样主要是为了在多个stage之间拷贝文件时，可以使用绝对路径，相信我这样子会减少很多问题。

经过multi-stage后，image大小从1.14G变成了64.2MB，抛弃了整个gcc带来的变动就是这么大。

## 进一步优化
---
#### 选择更小的镜像
在上面的第二步stage，我们选择了`ubuntu`，假如我们选择体积更小的`alpine`，或者更极端一点，`scratch`呢？
```go
package main

import "fmt"

func main () {
  fmt.Println("Hello, world!")
}
```
```dockerfile
FROM golang
COPY hello.go .
RUN go build hello.go
FROM scratch
COPY --from=0 /go/hello .
CMD ["./hello"]
```
可以看得到，这个dockerfile可以正常编译，可以正常输出`hello world`，大小也是只有2MB。但是有点经验的人都知道，`scratch`真的很难用
- 没有shell，你就没法正常用RUN和CMD。
- 没有一些常用命令，好比如说`ping` `netstat`等等。
- 最重要的一点是，没有`libc`。

C语言是默认动态链接的，go如果用到了某些库也是动态链接的（在非常复杂的go程序里面，libc几乎是必须的）。

那么我们怎么解决这个问题呢？
#### 动态不行，那就静态
---
```dockerfile
RUN gcc -o hello hello.c -static
```
编译出来的hello比原来动态编译下要大很多（几百 vs 几十），但这样子它就可以放进`scratch`中了
而对于go语言来说
```dockerfile
ENV CGO_ENABLED=0
RUN go build hello.go
```
也可以达到去除动态链接的效果。

#### 手动把需要的库拷贝进镜像
---
这一招太难维护，不推荐这么干。

#### 用一个带libc的小镜像
---
比如`busybox:glibc`，或者`alpine`。但是这儿需要注意的一点，`alpine`的`libc`，并不是我们正常认知的那个`GNU C`，而是自己搞的一套`musl`。

能在`GNU C`下正常工作的，并不能在`musl`下正常工作。这就意味着，如果我们的RUN最终是跑在`alpine`中的话，之前所有的编译也最好都是在`alpine`中进行的。

```dockerfile
# 这样是行不通的
FROM gcc
COPY hello.c .
RUN gcc -o hello hello.c

FROM alpine
COPY --from=0 hello .
CMD ["./hello"]
```
```dockerfile
# 而这样是可以的
FROM alpine
RUN apk add build-base
COPY hello.c .
RUN gcc -o hello hello.c

FROM alpine
COPY --from=0 hello .
CMD ["./hello"]
```

上面说的都是编译型语言，那么解释型语言呢？

### 解释型语言
---
其实解释型语言，跟编译型语言原理是差不多。以`ruby`为例
- 它需要一个vm，我们可以类比成`gcc`或者`go`。
- 它需要下载很多`gem`，这个过程我们可以类比成`compile`。实际上很多`gem`确实就需要编译，所以也是需要`gcc`或者`libc`的。
- 在multi-stage的后面，将`gem`和`ruby`程序本身拷贝到另外一个镜像中，就可以达到瘦身的效果。

需要注意的是，ruby `gem`的体积一般都比较大（几百kB到几MB），一个`rails`程序可能会用到几十个甚至上百个`gem`，瘦身的效果往往并不尽如人意。




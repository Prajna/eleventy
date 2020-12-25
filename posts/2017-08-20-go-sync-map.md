---
title: Go sync.Map
description: go 1.9终于有了类似Java的concurrenthashmap。
date: 2017-08-20
tags:
	- golang
layout: layouts/post.njk
---
Go 1.9终于引进了sync.Map。

Go官方文档一直都标示过，map并不是并发安全的。并发读冇问题，并发写在并发量大下会有问题。
比如我们要写一个counter

```go
package main

func main() {
	counter := make(map[string]int)
	go func() {
		for {
			_ = counter["1"]
		}
	}()
	go func() {
		for {
			counter["2"] = 2
		}
	}()
	select {}
}
```
```shell
$ go run -race main.go

...
fatal error: concurrent map read and map write
...
```
这句`fatal error: concurrent map read and map write`在[源码](https://github.com/golang/go/blob/846dce9d05f19a1f53465e62a304dea21b99f910/src/runtime/map_fast64.go#L20-L22)中标记出来了，当hashWriting（也就是4）的时候，
```go
hashWriting  = 4 // a goroutine is writing to the map
```
就表明有一个goroutine正在写map，所以就直接抛错。

于是在go官方博客中直接就弄了个非常...简陋的解决方案。亲，既然读的时候有可能写，那你就加个锁吧。
```go
package main

import "sync"

func main() {
	var counter = struct {
		sync.RWMutex
		m map[string]int
	}{m: make(map[string]int)}

	go func() {
		for {
			counter.RWMutex.RLock()
			_ = counter.m["1"]
			counter.RWMutex.RUnlock()
		}
	}()
	go func() {
		for {
			counter.RWMutex.Lock()
			counter.m["2"] = 2
			counter.RWMutex.Unlock()
		}
	}()
	select {}
}
```
这样确实不会出错，但是在大并发下性能也肯定是急速下降的。

好在在1.9，go终于推出来了sync.Map。今天我们就来看一下sync.Map为了实现并发而又不用上面RWMutex的做法，做了哪些事情。整体的代码加上注释只有384行，称得上非常简洁。

首先定义了一个sync.Map struct
```go
type Map struct {
	mu Mutex

	// read contains the portion of the map's contents that are safe for
	// concurrent access (with or without mu held).
	//
	// The read field itself is always safe to load, but must only be stored with
	// mu held.
	//
	// Entries stored in read may be updated concurrently without mu, but updating
	// a previously-expunged entry requires that the entry be copied to the dirty
	// map and unexpunged with mu held.
	read atomic.Value // readOnly

	// dirty contains the portion of the map's contents that require mu to be
	// held. To ensure that the dirty map can be promoted to the read map quickly,
	// it also includes all of the non-expunged entries in the read map.
	//
	// Expunged entries are not stored in the dirty map. An expunged entry in the
	// clean map must be unexpunged and added to the dirty map before a new value
	// can be stored to it.
	//
	// If the dirty map is nil, the next write to the map will initialize it by
	// making a shallow copy of the clean map, omitting stale entries.
	dirty map[interface{}]*entry

	// misses counts the number of loads since the read map was last updated that
	// needed to lock mu to determine whether the key was present.
	//
	// Once enough misses have occurred to cover the cost of copying the dirty
	// map, the dirty map will be promoted to the read map (in the unamended
	// state) and the next store to the map will make a new dirty copy.
	misses int
}
```
其中的`read`是一个`atomic.Value`，它其实上是一个`readonly` struct
```go
// readOnly is an immutable struct stored atomically in the Map.read field.
type readOnly struct {
	m       map[interface{}]*entry
	amended bool // true if the dirty map contains some key not in m.
}
```
其中可以看到，它用了两个`map[interface{}]*entry`，一个是read，一个是dirty。下面会详细介绍它们是怎么协同工作的。

对于一个map正常的`read`操作，sync.Map是这么实现的：
```go
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
	read, _ := m.read.Load().(readOnly) 
	e, ok := read.m[key] // 读`read`，看所传的`key`是否存在
	if !ok && read.amended { // 如果不存在，
		m.mu.Lock() // 则加锁。
		read, _ = m.read.Load().(readOnly) 
		e, ok = read.m[key] // 第二次读`read`中是否存在`key`（因为有可能在第一读和加锁之间map发生了*异动*，这个*异动*在后面讲）
		if !ok && read.amended { // 如果还是不存在，则去`dirty`中读。
			e, ok = m.dirty[key]
			m.missLocked() // *异动*
		}
		m.mu.Unlock()
	}
	if !ok {
		return nil, false
	}
	return e.load()
}

func (m *Map) missLocked() {
	m.misses++ // 无论`dirty`中存在不存在此`key`，都将`misses`加1。
	if m.misses < len(m.dirty) {
		return
	}
	m.read.Store(readOnly{m: m.dirty}) // 如果发现`misses` 小于 `len(dirty)`，则将`dirty`提升为`read`。
	m.dirty = nil
	m.misses = 0 // 将`dirty`设置成`nil`，misses清零
}
```

`delete`操作是这么实现的：
```go
func (m *Map) Delete(key interface{}) {
	m.LoadAndDelete(key) // 实际上是调用了LoadAndDelete
}

func (m *Map) LoadAndDelete(key interface{}) (value interface{}, loaded bool) {
	read, _ := m.read.Load().(readOnly) 
	e, ok := read.m[key] // 第一次读`read`
	if !ok && read.amended { // 如果不存在，
		m.mu.Lock() // 则加锁。
		read, _ = m.read.Load().(readOnly) // 第二次读`read`
		e, ok = read.m[key]
		if !ok && read.amended { // 如果还是不存在，读`dirty`
			e, ok = m.dirty[key]
			delete(m.dirty, key) // 如果`dirty`中有，那么直接用默认的`delete`删除
			m.missLocked()
		}
		m.mu.Unlock()
	}
	if ok {
		return e.delete() // 但如果read中有，那不能够普通删除。
	}
	return nil, false
}

func (e *entry) delete() (value interface{}, ok bool) {
	for {
		p := atomic.LoadPointer(&e.p)
		if p == nil || p == expunged {
			return nil, false
		}
		if atomic.CompareAndSwapPointer(&e.p, p, nil) { // 原子操作，将e.p置换成nil
			return *(*interface{})(p), true
		}
	}
}
```
这个`atomic.CompareAndSwapPointer`具体是什么意思呢？
- 如果是`delete`中的key，直接用内置`delete`删除，就当无事发生过。
- 如果是`read`中的key，那么并不是真实的删除，而是将`entry`中的`p`置换成`nil`。等着下次提升dirty的时候，就自然删除了。

而最为复杂的`write`操作又是这么实现的：
```go
func (m *Map) Store(key, value interface{}) {
	read, _ := m.read.Load().(readOnly)
	if e, ok := read.m[key]; ok && e.tryStore(&value) { // 读read，如果read中存在，而且entry并没有被标记成expunged，直接写。
		return 
	}

	m.mu.Lock() // 如果read中不存在；或者read中存在，但是entry被标记成expunged，加锁
	read, _ = m.read.Load().(readOnly) // 再读read
	if e, ok := read.m[key]; ok { // 这一块在最后讲
		if e.unexpungeLocked() {
			m.dirty[key] = e
		}
		e.storeLocked(&value)
	} else if e, ok := m.dirty[key]; ok { // 如果在dirty中，那么保存在dirty的entry中
		e.storeLocked(&value)
	} else {
		if !read.amended { // amended这个标记位，表示没有元素在dirty中，所以是第一次往dirty中写。这种情况只会发生在初始化或者dirty刚被提升过。
			m.dirtyLocked()
			m.read.Store(readOnly{m: read.m, amended: true})
		}
		m.dirty[key] = newEntry(value)
	}
	m.mu.Unlock() // 解锁
}

func (m *Map) dirtyLocked() {
    // 需要在这里make dirty
	if m.dirty != nil {
		return
	}

	read, _ := m.read.Load().(readOnly)
    m.dirty = make(map[interface{}]*entry, len(read.m))
    // 在Delete()时，read中那些p被标记为nil的entry，p会被标记成expunged。而其它正常的entry，会被拷贝到dirty中。因为dirty和read指向了同一个entry，所以并不会增加很多开销。
	for k, e := range read.m {
		if !e.tryExpungeLocked() {
			m.dirty[k] = e
		}
	}
}

// 在这里将nil转化成expunged
func (e *entry) tryExpungeLocked() (isExpunged bool) {
	p := atomic.LoadPointer(&e.p)
	for p == nil {
		if atomic.CompareAndSwapPointer(&e.p, nil, expunged) {
			return true
		}
		p = atomic.LoadPointer(&e.p)
	}
	return p == expunged
}
```

而最绕的一块：
```go
if e, ok := read.m[key]; ok {
		if e.unexpungeLocked() {
			m.dirty[key] = e
		}
		e.storeLocked(&value)
	}
```
这一块是什么意思？我们可以一步步分解来看
- 初始化一个sync.Map，这时候`read`和`dirty`什么都没有。
- 我们往里面随便写几个，再读几个，导致了一次dirty提升：`read`中有1 => "1", 2 => "2"，dirty为nil。
- 这时候我们删除1。这时候，我们会从read中读到，所以"1"这个entry的p会被置为nil。
- 此时，我们再加一个3 => "3"，但此时dirty是nil。
- 我们需要跑一次dirtyLocked，将read中的元素copy到dirty中。在这个过程中，1不会被拷贝，因为p为nil，我们只会将它的p置为expunged。
- 所以这时候，read 1 => "1"(expunged)，2 => "2"，dirty 2 => "2", 3 => "3"
- 这时候我们要写入1 => "4"
- 我们会发现dirty中根本没有1，因为在上次的dirtyLocked时并没有把它copy过去，这时候我们就需要将read中1的p设为nil，再在dirty创建 1 => "4"。

这样就营造了一个，1并不在`read`中，但是在`dirty`中的假象。下次无论是`Delete()`还是`Load()`，都可以直接从`dirty`中读或者删除。

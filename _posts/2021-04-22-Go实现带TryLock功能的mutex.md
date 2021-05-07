---
layout: post
title:  Go语言实现带TryLock功能的mutex
tags: go
---

我们知道，`mutex`是操作系统提供的一种同步原语，用来保证对于共享资源互斥的访问。各个编程语言通过关键字、数据结构、库等方式提供了类似的功能：**在多线程程序中，并发安全地访问共享变量**。

在golang中，官方提供`sync.Mutex`互斥锁，使用方式也很简单：

```golang
var mu sync.Mutex
mu.Lock()
// do something
mu.Unlock()
```

`Lock`的调用会阻塞当前goroutine，直到成功获取锁。在其他编程语言（比如Java）中，也提供了非阻塞尝试获取锁的方式。golang能否实现这样的功能呢？无论是否获取成功，都立即返回，成功返回true，失败返回false。官方没有提供这样的包，但是我们可以很容易的使用`channel`实现这个功能。

```go
type C chan struct{}

func NewC() C {
	ch := make(chan struct{}, 1)
	return ch
}

func (c *C) Lock() {
	(*c) <- struct{}{}
}

func (c *C) UnLock() {
	<-(*c)
}
```

注意我们使用的`channel`类型是`struct{}`，因为我们不需要使用`channel`来传递实际数据，只是同步信号。而且我们使用容量为1的`buffered channel`，这样第一个获取锁的`goroutine`不会阻塞。

以上代码模拟了一个基本的互斥锁，那么怎么实现非阻塞Lock呢？

Go语言提供了`select`语句来使得一个`goroutine`可以监听多个`channel`是否可读或者可写（有点像网络编程中的IO多路复用）。如果`case`对应的`channel`读写操作还不能执行，就会执行`defaul`t对应的语句。

```go
func (c *C) TryLock() bool {
	select {
	case *c <- struct{}{}:
		return true
	default:
		return false
	}
}
```

为TryLock指定超时时间就很简单了：

```go
func (c *C) TryLockWithTimeOut(d time.Duration) bool {
	t := time.NewTimer(d)
	select {
	case <-t.C:
		return false
	case *c <- struct{}{}:
		t.Stop()
		return true
	}
}
```

以上，我们通过对`channel`的简单封装，实现了非阻塞地尝试获取锁，以及一定超时时间内尝试获取锁的功能。

写段代码来验证一下吧

```go
func main() {
	n1 := int64(0)
	n2 := int64(0)
	c := NewC()

	wg := sync.WaitGroup{}
	for i := 0; i < 10000; i++ {
		wg.Add(1)
		go func() {
			if c.TryLock() {
				n1++
				c.UnLock()
			} else {
				atomic.AddInt64(&n2, 1)
			}
			wg.Done()
		}()
	}
	wg.Wait()

	fmt.Printf("total: %v, success: %v, fail: %v\n", n1+n2, n1, n2)
}
```


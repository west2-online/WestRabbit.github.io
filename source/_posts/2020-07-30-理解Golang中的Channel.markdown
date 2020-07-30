---
title:      "理解 Golang 中的 Channel"
date:       2020-07-30
author:     "SunPengfei"
tags:
     - 18级
     - Golang
---
# Channel

> Do not communicate by sharing memory; instead, share memory by communicating.

Channel 是 Go 中的一个核心类型，它的设计充分体现了利用通信来共享内存。
Channel 具有以下特点：
- 并发安全
- 用于在 Goroutine 之间通信
- 其语义是先入先出（FIFO）
- 可以阻塞 Goroutine

# Channel 的基本使用

我们先看看 Channel 的基本使用方法：
```go
package main
import "fmt"

func main() {
    c := make(chan int) // 创建一个 Channel

    go func() {
        c <- 1 // 发送数据
    }()

    x := <-c // 接收数据

    fmt.Println(x)
}
```
以上代码对应了 Channel 的创建、发送和接收数据。
我们用 Go 自带的命令 `go tool compile -S main.go` 可以将以上代码翻译成 Go 汇编代码。
分析汇编代码，我们可以看到：
- `c := make(chan int)` 创建语句对应以下的汇编代码，可以看到 Channel 在底层调用的是 **runtime.makechan** 函数
```
leaq    type.chan int(SB), AX
movq    AX, (SP)
movq    $1, 8(SP)
pcdata  $1, $0
call    runtime.makechan(SB)
```
- `c <- 1` 发送数据语法对应以下的汇编代码，可以看到底层调用的是 **runtime.chansend1** 函数
```
movq    "".c+32(SP), AX
movq    AX, (SP)
leaq    ""..stmp_0(SB), AX
movq    AX, 8(SP)
pcdata  $1, $1
call    runtime.chansend1(SB)
```
- `x := <-c` 接收数据语法对应以下的汇编代码，可以看到底层调用的是 **runtime.chanrecv1** 函数
```
movq    $0, ""..autotmp_9+64(SP)
movq    "".c+72(SP), AX
movq    AX, (SP)
leaq    ""..autotmp_9+64(SP), AX
movq    AX, 8(SP)
pcdata  $1, $0
call    runtime.chanrecv1(SB)
movq    ""..autotmp_9+64(SP), AX
```

# Channel 的创建

Channel 最终调用的是 **runtime.makechan** 函数，函数原型为 `func makechan(t *chantype, size int) *hchan`，这个函数的作用其实就是初始化一个 **runtime.hchan** 结构体，所以我们先来看一下 **runtime.hchan** 结构体
```go
type hchan struct {
	qcount   uint           // buffer 循环队列中的已经存在的数据个数
	dataqsiz uint           // buffer 循环队列中的数据最大长度
	buf      unsafe.Pointer // buffer 循环队列指针
	elemsize uint16         // buffer 中的元素大小
	elemtype *_type         // buffer 中的元素类型
	sendx    uint           // buffer 中已经发送的索引位置
	recvx    uint           // buffer 中已经接收的索引位置
	recvq    waitq          // 等待接收的 Goroutine 的双向链表
	sendq    waitq          // 等待发送的 Goroutine 的双向链表
	closed   uint32         // Channel 是否关闭, 0 代表未关闭

	lock mutex
}
```
![runtime.hchan 的结构图](hchan.png)
注意： **上图的发送和接受队列应该是一个双向链表**

接下来我们来看一下 **runtime.makechan** 函数：
```go
// 参数 block 表示发送操作是否阻塞
// 参数 ep 是发送数据的地址
// 返回结果表示 是否发送成功
func makechan(t *chantype, size int) *hchan {
	elem := t.elem

	// 元素类型大小限制
	if elem.size >= 1<<16 {
		throw("makechan: invalid channel element type")
	}
	// 对齐限制
	if hchanSize%maxAlign != 0 || elem.align > maxAlign {
		throw("makechan: bad alignment")
	}

	// 检查确认 Channel 的容量是否合适
	mem, overflow := math.MulUintptr(elem.size, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	var c *hchan
	switch {
	case mem == 0:
		// 如果当前 Channel 中不存在缓冲区，那么就只会为 runtime.hchan 分配一段内存空间
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		c.buf = c.raceaddr()
	case elem.ptrdata == 0:
		// 如果当前 Channel 中存储的类型不是指针类型，就会为当前的 Channel 和底层的数组分配一块连续的内存空间
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// 在默认情况下会单独为 runtime.hchan 和缓冲区分配内存
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

	//统一更新 runtime.hchan 的 elemsize、elemtype 和 dataqsiz 几个字段
	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)

	if debugChan {
		print("makechan: chan=", c, "; elemsize=", elem.size, "; elemalg=", elem.alg, "; dataqsiz=", size, "\n")
	}
	return c
}
```
可以看到 **runtime.makechan** 函数是根据 Channel 中收发元素的类型和缓冲区的大小初始化 **runtime.hchan** 结构体和缓冲区，最后返回的是一个 **runtime.hchan** 的结构体。

# Channel 的发送

上面我们看到发送数据是调用的是 **runtime.chansend1** 函数：
```go
func chansend1(c *hchan, elem unsafe.Pointer) {
	chansend(c, elem, true, getcallerpc())
}
```
可以看到最终调用的其实是 **runtime.chansend** 函数:
```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	// 检查 Channel 是否为空
	if c == nil {
		// 非阻塞直接返回 false
		if !block {
			return false
		}
		// 向一个空的 Channel 发送数据就会永远阻塞该 Goroutine
		gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	 if debugChan {
        print("chansend: chan=", c, "\n")
    }

    if raceenabled {
        racereadpc(c.raceaddr(), callerpc, funcPC(chansend))
    }

    // 下面这段注释主要解释了为何不进行加锁就可以判断非阻塞操作是否可以立即返回(不太理解就不翻译了)
    // Fast path: check for failed non-blocking operation without acquiring the lock.
    //
    // After observing that the channel is not closed, we observe that the channel is
    // not ready for sending. Each of these observations is a single word-sized read
    // (first c.closed and second c.recvq.first or c.qcount depending on kind of channel).
    // Because a closed channel cannot transition from 'ready for sending' to
    // 'not ready for sending', even if the channel is closed between the two observations,
    // they imply a moment between the two when the channel was both not yet closed
    // and not ready for sending. We behave as if we observed the channel at that moment,
    // and report that the send cannot proceed.
    //
    // It is okay if the reads are reordered here: if we observe that the channel is not
    // ready for sending and then observe that it is not closed, that implies that the
    // channel wasn't closed during the first observation.
    // 如果（buffer 大小为0且没有等待接收的 Goroutine）或（buffer 已经满了）就直接返回false
	if !block && c.closed == 0 && ((c.dataqsiz == 0 && c.recvq.first == nil) ||
	(c.dataqsiz > 0 && c.qcount == c.dataqsiz)) {
		return false
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock)

	// 检查 Channel 是否关闭
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}

	// 如果已经有 Goroutine 处于读等待状态，那么直接把数据发送给最先开始等待的 Goroutine（FIFO）
	if sg := c.recvq.dequeue(); sg != nil {
		// 在 send 函数里面会将处于阻塞状态的 Goroutine 唤醒，并将数据发送给他
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

	// 如果 buffer 还没满就把数据放进 buffer 队列
	if c.qcount < c.dataqsiz {
		// qp 指向 buffer 已经发送的索引位置
		qp := chanbuf(c, c.sendx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
		}
		// 把数据从 ep 复制到 qp
		typedmemmove(c.elemtype, qp, ep)
		// 发送的索引位置+1
		c.sendx++
		// 发送的索引位置等于队列大小就归零（循环队列）
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		// buffer 数据数量+1
		c.qcount++
		unlock(&c.lock)
		// 返回 true 表示数据发送成功
		return true
	}

	// 该操作是非阻塞的就直接返回 false
	if !block {
		unlock(&c.lock)
		return false
	}

	// 接下来就是阻塞该 Goroutine，等待有 Goroutine 来读数据
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	// 将该 Goroutine 压入等待发送的队列
	c.sendq.enqueue(mysg)
	// 将当前 goroutine 挂起，会在下文的 recv 函数里面被唤醒
	goparkunlock(&c.lock, waitReasonChanSend, traceEvGoBlockSend, 3)
	// Ensure the value being sent is kept alive until the
	// receiver copies it out. The sudog has a pointer to the
	// stack object, but sudogs aren't considered as roots of the
	// stack tracer.
	KeepAlive(ep)

	// 唤醒之后做一些收尾工作
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	if gp.param == nil {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	mysg.c = nil
	releaseSudog(mysg)
	return true
}
```

# Channel 的接收
上面我们看到发送数据是调用的是 **runtime.chanrecv1** 函数：
```go
func chanrecv1(c *hchan, elem unsafe.Pointer) {
	chanrecv(c, elem, true)
}
```
可以看到最终调用的其实是 **runtime.chanrecv** 函数:
```go
// 参数 ep 是接收数据的地址
// 返回值：
// selected 用于 select{}，表示是否执行对应的 case ... <- channel
// received 用于判断是否接收到了数据
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {

	if debugChan {
		print("chanrecv: chan=", c, "\n")
	}

	if c == nil {
		if !block {
			return
		}
		// 从空 Channel 中接收数据会永远阻塞 该 Goroutine
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	// 这里的情况和上面发送的代码类似，都是在不获取锁的情况下判断是否可以直接返回
	// 不获取锁可以提高性能
	if !block && (c.dataqsiz == 0 && c.sendq.first == nil ||
		c.dataqsiz > 0 && atomic.Loaduint(&c.qcount) == 0) &&
		atomic.Load(&c.closed) == 0 {
		return
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock)
	// Channel 已经关闭，且 buffer 中不存在数据了
	if c.closed != 0 && c.qcount == 0 {
		if raceenabled {
			raceacquire(c.raceaddr())
		}
		unlock(&c.lock)
		if ep != nil {
			typedmemclr(c.elemtype, ep)
		}
		return true, false
	}

	// 从等待发送队列中取出一个 Goroutine
	if sg := c.sendq.dequeue(); sg != nil {
		// 上面我们介绍到 Goroutine 会挂起等待被读取，在 recv 函数里面就会把 Goroutine 唤醒并读取它的数据
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}

	// 如果 buffer 中从在数据那就直接从 buffer 里面读取数据
	if c.qcount > 0 {
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
		}
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}

	// Channel 没有数据，且非阻塞，直接返回
	if !block {
		unlock(&c.lock)
		return false, false
	}

	// 阻塞等待数据，和上面阻塞等待发送数据类似
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil
	// 将当前 Goroutine 压入等待接收的队列
	c.recvq.enqueue(mysg)
	// 挂起当前 Goroutine，会在上文的 send 函数里面被唤醒
	goparkunlock(&c.lock, waitReasonChanReceive, traceEvGoBlockRecv, 3)

	// 唤醒之后做一些收尾工作
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	closed := gp.param == nil
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, !closed
}
```

# 小结
本文简单的分析了 Channel 的创建、发送和接收数据的底层原理，但是对于 Goroutine 挂起后如何被唤醒没有进行深入的分析，如果有机会的话**下次一定**。

# 参考资料
- [Go 语言设计与实现](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-channel/#64-channel "Go 语言设计与实现")
- [Go 语言原本](https://changkun.de/golang/zh-cn/part2runtime/ch09lang/chan/#channel- "Go 语言原本")
- [golang channel底层实现](http://yangxikun.com/golang/2019/10/15/golang-channel.html "golang channel底层实现")
- [Go 语言问题集](https://www.bookstack.cn/read/qcrao-Go-Questions/channel-%E5%90%91%20channel%20%E5%8F%91%E9%80%81%E6%95%B0%E6%8D%AE%E7%9A%84%E8%BF%87%E7%A8%8B%E6%98%AF%E6%80%8E%E6%A0%B7%E7%9A%84.md "Go 语言问题集")
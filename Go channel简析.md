# Go channel简析
channel是golang中最核心的feature之一，因此理解channel的原理对于学习和使用golang非常重要。  
channel是goroutine之间通信的一种方式，可以类比成Unix中的进程的通信方式——管道。

## CSP模型
在讲channel之前，有必要先提一下CSP模型。传统的并发模型主要分为Actor模型和CSP模型，CSP模型全称为communicating sequential processes，CSP模型由并发执行实体（进程，线程或协程），和消息通道组成，实体之间通过消息通道发送消息进行通信。和Actor模型不同，CSP模型关注的事消息发送的载体，即通道，而不是发送消息的执行实体。Go语言的并发模型参考了CSP理论，其中执行实体对应的是goroutine，消息通道对应的就是channel。  
## channel介绍
channel提供了一种通信机制，通过它，一个goroutine可以向另一个goroutine发送消息。channel本身还需关联一个类型，也就是channel可以发送数据的类型。例如：发送int类型消息的channel写作chan int。
## channel创建
channel使用内置的make函数创建，下面声明了一个chan int类型的channel。
```go
ch := make(chan int)
```  
c和map类似，make创建了一个底层数据结构的引用，当赋值或者参数传递时，只是拷贝了一个channel引用，指向相同的channel对象。和其他引用类型一样，channel的空值为nil。使用==可以对类型相同的channel进行比较，只有指向相同对象或同为nil时，才返回true。
## channel的读写操作
```go
ch := make(chan int)

//write to channel
ch <- x

//read from channel
x <- ch

//another way to read
x = <-ch
```  
channel一定要初始化后才能进行读写操作，否则会永久阻塞。
## 关闭channel
golang提供了内置的close函数对channel进行关闭操作。
```go
ch := make(chan int)

close(ch)
```  
有关channel的关闭，需要注意以下事项：
* 关闭一个未初始化（nil）的channel会产生panic
* 重复关闭同一个channel会产生panic
* 向一个已关闭的channel中发送消息会产生panic
* 从已关闭的channel读消息不会产生panic，且能读出channel中还未被读取的消息，若消息均已读出，则会读到该类型的零值。从一个已关闭的channel中读取消息永远不会阻塞，并且会返回一个位false的ok-idiom，可以用它来判断channel是否关闭。
* 关闭channel会产生一个广播机制，所有向channel读取消息的goroutine都会收到消息
```go
ch := make(chan int, 10)
ch <- 11
ch <- 12

close(ch)

for x := range ch {
    fmt.Println(x)
}

x,ok := <- ch
fmt.Println(x, ok)
```  
输出：  
11  
12  
0 false  
## channel的类型
channel分为不带缓存的channel和带缓存的channel。
### 无缓存的channel
从无缓存的channel中读消息会阻塞，直到有goroutine向该channel中发送消息；同理，向无缓存的channel中发送消息也会阻塞，直到有goroutine从channel中读取消息。
### 有缓存的channel
有缓存的channel的声明方式为指定make函数的第二个参数，该参数为channel缓存的容量。
```go
ch := make(chan int, 10)
```  
有缓存的channel类似一个阻塞队列（采用环形数组实现）。当缓存未满时，向channel中发送消息时不会阻塞，当缓存满时，发送操作将被阻塞，直到有其他goroutine从中读取消息；相应的，当channel中消息不为空时，读取消息不会出现阻塞，当channel为空时，读取操作会造成阻塞，直到有goroutine向channel中写入消息。
```go
ch := make(chan int, 3)

//bloced,read from empty buffered channel
<- ch
```
```go
ch := make(chan int, 3)
ch <- 1
ch <- 2
ch <- 3

//blocked,send to full buffered channel
ch <- 4
```  
通过len函数可以获得chan中的元素个数，通过cap函数可以得到channel的缓存长度。
## channel的用法
### goroutine通信
看一个effective go中的例子
```go
c := make(chan int) //Allocate a channel

//Start the sort in a goroutine; when it completes, signal on the channel
go func() {
    list.Sort()
    c <- 1 //Send a signal; value does not matter
}()

doSomethingForAWhile()
<-c
```  
主goroutine会阻塞，直到执行sort的goroutine完成。
### range遍历
channel也可以使用range取值，并且会一直从channel中读取数据，直到有goroutine对该channel执行close操作，循环才会结束。
```go
//consumer worker
ch := make(chan int, 10)
for x := range ch {
    fmt.Println(x)
}
```  
等价于
```go
for {
    x, ok := <- ch
    if !ok {
        break
    }
    fmt.Println(x)
} 
```  
### 配合select使用
select用法类似于IO多路复用，可以同时监听多个channel的消息状态，看下面的例子
```go
select {
    case <- ch1:
    ...
    case <- ch2:
    ..
    default:
    ...
} 
```  
* select可以同时监听多个channel的写入或读取
* 执行select时，若只有一个case通过（不阻塞），则执行这个case块。
* 若有多个case通过，则随机挑选一个case执行
* 若所有case均阻塞，且定义了default模块，则执行default模块。若未定义default模块，则select语句阻塞，直到有case被唤醒。
* 使用break会跳出select块。

### 1. 设置超时时间
```go
ch := make(chan struct{})

//finish task while send msg to ch
go doTask(ch)

timeout := time.After(5 * time.Second)
select {
    case <- ch:
        fmt.Println("task finished")
    case <- timeout:
        fmt.Println("task timeout")
}
```
### 2. quite channel
有一些场景中，一些worker goroutine需要一直循环处理信息，直到收到quit信号
```go
msgCh := make(chan struct{})
quitCh := make(chan struct{})
for {
    select {
        case <- msgCh:
            doWork()
        case <- quitCh:
            finish()
            return
    }
}
```  
### 单向channel
即只可写入或只可读取的channel，事实上，channel只读或只写都没有意义，所谓的单向channel其实只是声明时用，比如：
```go
func foo(ch chan<- int) <- chan int {...}
```  
chan <- int 表示一个只可写入的channel，<- chan int 表示一个只可读取的channel。上面这个函数约定了foo内只能从ch中写入数据，返回一个只能读取的channel，虽然使用普通的channel也没有问题，但这样在方法声明时约定可以防止channel被滥用，这种预防机制发生在编译期间。
## channel源码分析
channel的主要实现在src/runtime/chan.go中，以下源码均基于go1.9.2。源码阅读时为了更好地理解channel特性，帮助正确合理的使用channel，阅读代码的过程可以回忆前面章节的channel特性。
### channel类结构
channel相关类定义如下：
```go
//channel类型定义
type hchan struct {
    //channel中的元素数量，len
    qcount uint //total data in the queue

    //channel的大小，cap
    dataqsiz uint //size of the circular queue

    //channel的缓冲区，环形数组实现
    buf unsafe.Pointer //points to an array of dataqsiz elements

    //单个元素的大小
    elemsize uint16

    //close标志位
    closed uint32

    //元素的类型
    elemtype *_type //element type

    //send和 receive的索引，用于实现环形数组队列
    sendx uint //send index
    recvx uint //receive index

    // recv goroutine 等待队列
    recvq    waitq  // list of recv waiters
    
    // send goroutine 等待队列
    sendq    waitq  // list of send waiters

    // lock protects all fields in hchan, as well as several
    // fields in sudogs blocked on this channel.
    //
    // Do not change another G's status while holding this lock
    // (in particular, do not ready a G), as this can deadlock
    // with stack shrinking.
    lock mutex
}

// 等待队列的链表实现
type waitq struct {    
    first *sudog       
    last  *sudog       
}

// in src/runtime/runtime2.go
// 对 G 的封装
type sudog struct {
    // The following fields are protected by the hchan.lock of the
    // channel this sudog is blocking on. shrinkstack depends on
    // this for sudogs involved in channel ops.

    g          *g
    selectdone *uint32 // CAS to 1 to win select race (may point to stack)
    next       *sudog
    prev       *sudog
    elem       unsafe.Pointer // data element (may point to stack)

    // The following fields are never accessed concurrently.
    // For channels, waitlink is only accessed by g.
    // For semaphores, all fields (including the ones above)
    // are only accessed when holding a semaRoot lock.

    acquiretime int64
    releasetime int64
    ticket      uint32
    parent      *sudog // semaRoot binary tree
    waitlink    *sudog // g.waiting list or semaRoot
    waittail    *sudog // semaRoot
    c           *hchan // channel
} 
```  
可以看到，channel的主要组成有：一个环形数组实现的队列，用于存储消息元素；两个链表实现的goroutine等待队列，用于存储阻塞在recv和send操作上的goroutine；一个互斥锁，用于各个属性变动的同步。
### channel make实现
```go
func makechan(t *chantype, size int64) *hchan {
    elem := t.elem

    // compiler checks this but be safe.
    if elem.size >= 1<<16 {
        throw("makechan: invalid channel element type")
    }
    if hchanSize%maxAlign != 0 || elem.align > maxAlign {
        throw("makechan: bad alignment")
    }
    if size < 0 || int64(uintptr(size)) != size || (elem.size > 0 && uintptr(size) > (_MaxMem-hchanSize)/elem.size) {
        panic(plainError("makechan: size out of range"))
    }

    var c *hchan
    
    if elem.kind&kindNoPointers != 0 || size == 0 {
        // case 1: channel 不含有指针
        // case 2: size == 0，即无缓冲 channel
        // Allocate memory in one call.
        // Hchan does not contain pointers interesting for GC in this case:
        // buf points into the same allocation, elemtype is persistent.
        // SudoG's are referenced from their owning thread so they can't be collected.
        // TODO(dvyukov,rlh): Rethink when collector can move allocated objects.
        
        // 在堆上分配连续的空间用作 channel
        c = (*hchan)(mallocgc(hchanSize+uintptr(size)*elem.size, nil, true))
        if size > 0 && elem.size != 0 {
            c.buf = add(unsafe.Pointer(c), hchanSize)
        } else {
            // race detector uses this location for synchronization
            // Also prevents us from pointing beyond the allocation (see issue 9401).
            c.buf = unsafe.Pointer(c)
        }
    } else {
        // 有缓冲 channel 初始化
        c = new(hchan)
        // 堆上分配 buf 内存
        c.buf = newarray(elem, int(size))
    }
    c.elemsize = uint16(elem.size)
    c.elemtype = elem
    c.dataqsiz = uint(size)

    if debugChan {
        print("makechan: chan=", c, "; elemsize=", elem.size, "; elemalg=", elem.alg, "; dataqsiz=", size, "\n")
    }
    return c
}
```  
make的过程还比较简单，需要注意的一点是，当元素不含指针的时候，会将整个hchan分配成一个连续的空间。
### channel send
```go
// entry point for c <- x from compiled code
//go:nosplit
func chansend1(c *hchan, elem unsafe.Pointer) {
    chansend(c, elem, true, getcallerpc(unsafe.Pointer(&c)))
}

/*
 * generic single channel send/recv
 * If block is not nil,
 * then the protocol will not
 * sleep but return if it could
 * not complete.
 *
 * sleep can wake up with g.param == nil
 * when a channel involved in the sleep has
 * been closed.  it is easiest to loop and re-run
 * the operation; we'll see that it's now closed.
 */
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {

    // 前面章节说道的，当 channel 未初始化或为 nil 时，向其中发送数据将会永久阻塞
    if c == nil {
        if !block {
            return false
        }
        
        // gopark 会使当前 goroutine 休眠，并通过 unlockf 唤醒，但是此时传入的 unlockf 为 nil, 因此，goroutine 会一直休眠
        gopark(nil, nil, "chan send (nil chan)", traceEvGoStop, 2)
        throw("unreachable")
    }

    if debugChan {
        print("chansend: chan=", c, "\n")
    }

    if raceenabled {
        racereadpc(unsafe.Pointer(c), callerpc, funcPC(chansend))
    }

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
    if !block && c.closed == 0 && ((c.dataqsiz == 0 && c.recvq.first == nil) ||
        (c.dataqsiz > 0 && c.qcount == c.dataqsiz)) {
        return false
    }

    var t0 int64
    if blockprofilerate > 0 {
        t0 = cputicks()
    }

    // 获取同步锁
    lock(&c.lock)

    // 之前章节提过，向已经关闭的 channel 发送消息会产生 panic
    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("send on closed channel"))
    }

    // CASE1: 当有 goroutine 在 recv 队列上等待时，跳过缓存队列，将消息直接发给 reciever goroutine
    if sg := c.recvq.dequeue(); sg != nil {
        // Found a waiting receiver. We pass the value we want to send
        // directly to the receiver, bypassing the channel buffer (if any).
        send(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true
    }

    // CASE2: 缓存队列未满，则将消息复制到缓存队列上
    if c.qcount < c.dataqsiz {
        // Space is available in the channel buffer. Enqueue the element to send.
        qp := chanbuf(c, c.sendx)
        if raceenabled {
            raceacquire(qp)
            racerelease(qp)
        }
        typedmemmove(c.elemtype, qp, ep)
        c.sendx++
        if c.sendx == c.dataqsiz {
            c.sendx = 0
        }
        c.qcount++
        unlock(&c.lock)
        return true
    }

    if !block {
        unlock(&c.lock)
        return false
    }
    
    // CASE3: 缓存队列已满，将goroutine 加入 send 队列
    // 初始化 sudog
    // Block on the channel. Some receiver will complete our operation for us.
    gp := getg()
    mysg := acquireSudog()
    mysg.releasetime = 0
    if t0 != 0 {
        mysg.releasetime = -1
    }
    // No stack splits between assigning elem and enqueuing mysg
    // on gp.waiting where copystack can find it.
    mysg.elem = ep
    mysg.waitlink = nil
    mysg.g = gp
    mysg.selectdone = nil
    mysg.c = c
    gp.waiting = mysg
    gp.param = nil
    // 加入队列
    c.sendq.enqueue(mysg)
    // 休眠
    goparkunlock(&c.lock, "chan send", traceEvGoBlockSend, 3)

    // 唤醒 goroutine
    // someone woke us up.
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
从send代码中可以看到，之前章节提到的一些特性都在代码中有所体现。  
send有以下几种情况：  
* 有goroutine阻塞在channel recv队列上，此时缓存队列为空，直接将消息发送给receive goroutine，只产生一次复制。
* 当channel缓存队列有剩余空间时，将数据放到队列里，等待接收，接收后总共产生两次复制
* 当channel缓存队列已满时，将当前goroutine加入send队列并阻塞
### channel receive
```go
// entry points for <- c from compiled code
//go:nosplit
func chanrecv1(c *hchan, elem unsafe.Pointer) {
    chanrecv(c, elem, true)
}

//go:nosplit
func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
    _, received = chanrecv(c, elem, true)
    return
}

// chanrecv receives on channel c and writes the received data to ep.
// ep may be nil, in which case received data is ignored.
// If block == false and no elements are available, returns (false, false).
// Otherwise, if c is closed, zeros *ep and returns (true, false).
// Otherwise, fills in *ep with an element and returns (true, true).
// A non-nil ep must point to the heap or the caller's stack.
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    // raceenabled: don't need to check ep, as it is always on the stack
    // or is new memory allocated by reflect.

    if debugChan {
        print("chanrecv: chan=", c, "\n")
    }

    // 从 nil 的 channel 中接收消息，永久阻塞
    if c == nil {
        if !block {
            return
        }
        gopark(nil, nil, "chan receive (nil chan)", traceEvGoStop, 2)
        throw("unreachable")
    }

    // Fast path: check for failed non-blocking operation without acquiring the lock.
    //
    // After observing that the channel is not ready for receiving, we observe that the
    // channel is not closed. Each of these observations is a single word-sized read
    // (first c.sendq.first or c.qcount, and second c.closed).
    // Because a channel cannot be reopened, the later observation of the channel
    // being not closed implies that it was also not closed at the moment of the
    // first observation. We behave as if we observed the channel at that moment
    // and report that the receive cannot proceed.
    //
    // The order of operations is important here: reversing the operations can lead to
    // incorrect behavior when racing with a close.
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
    
    // CASE1: 从已经 close 且为空的 channel recv 数据，返回空值
    if c.closed != 0 && c.qcount == 0 {
        if raceenabled {
            raceacquire(unsafe.Pointer(c))
        }
        unlock(&c.lock)
        if ep != nil {
            typedmemclr(c.elemtype, ep)
        }
        return true, false
    }

    // CASE2: send 队列不为空
    // CASE2.1: 缓存队列为空，直接从 sender recv 元素
    // CASE2.2: 缓存队列不为空，此时只有可能是缓存队列已满，从队列头取出元素，并唤醒 sender 将元素写入缓存队列尾部。由于为环形队列，因此，队列满时只需要将队列头复制给 reciever，同时将 sender 元素复制到该位置，并移动队列头尾索引，不需要移动队列元素
    if sg := c.sendq.dequeue(); sg != nil {
        // Found a waiting sender. If buffer is size 0, receive value
        // directly from sender. Otherwise, receive from head of queue
        // and add sender's value to the tail of the queue (both map to
        // the same buffer slot because the queue is full).
        recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true, true
    }

    // CASE3: 缓存队列不为空，直接从队列取元素，移动头索引
    if c.qcount > 0 {
        // Receive directly from queue
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
    
    if !block {
        unlock(&c.lock)
        return false, false
    }
    
    // CASE4: 缓存队列为空，将 goroutine 加入 recv 队列，并阻塞
    // no sender available: block on this channel.
    gp := getg()
    mysg := acquireSudog()
    mysg.releasetime = 0
    if t0 != 0 {
        mysg.releasetime = -1
    }
    // No stack splits between assigning elem and enqueuing mysg
    // on gp.waiting where copystack can find it.
    mysg.elem = ep
    mysg.waitlink = nil
    gp.waiting = mysg
    mysg.g = gp
    mysg.selectdone = nil
    mysg.c = c
    gp.param = nil
    c.recvq.enqueue(mysg)
    goparkunlock(&c.lock, "chan receive", traceEvGoBlockRecv, 3)

    // someone woke us up
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
### channel close
```go
func closechan(c *hchan) {
    if c == nil {
        panic(plainError("close of nil channel"))
    }

    lock(&c.lock)
    
    // 重复 close，产生 panic
    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("close of closed channel"))
    }

    if raceenabled {
        callerpc := getcallerpc(unsafe.Pointer(&c))
        racewritepc(unsafe.Pointer(c), callerpc, funcPC(closechan))
        racerelease(unsafe.Pointer(c))
    }

    c.closed = 1

    var glist *g

    // 唤醒所有 reciever
    // release all readers
    for {
        sg := c.recvq.dequeue()
        if sg == nil {
            break
        }
        if sg.elem != nil {
            typedmemclr(c.elemtype, sg.elem)
            sg.elem = nil
        }
        if sg.releasetime != 0 {
            sg.releasetime = cputicks()
        }
        gp := sg.g
        gp.param = nil
        if raceenabled {
            raceacquireg(gp, unsafe.Pointer(c))
        }
        gp.schedlink.set(glist)
        glist = gp
    }

    // 唤醒所有 sender，并产生 panic
    // release all writers (they will panic)
    for {
        sg := c.sendq.dequeue()
        if sg == nil {
            break
        }
        sg.elem = nil
        if sg.releasetime != 0 {
            sg.releasetime = cputicks()
        }
        gp := sg.g
        gp.param = nil
        if raceenabled {
            raceacquireg(gp, unsafe.Pointer(c))
        }
        gp.schedlink.set(glist)
        glist = gp
    }
    unlock(&c.lock)

    // Ready all Gs now that we've dropped the channel lock.
    for glist != nil {
        gp := glist
        glist = glist.schedlink.ptr()
        gp.schedlink = 0
        goready(gp, 3)
    }
}
```
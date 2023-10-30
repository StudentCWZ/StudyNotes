# 第三讲 Golang Channel 原理
## 0 前言

1. Golang channel

   <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/Golang%20channel.png" alt="Golang channel" style="zoom:50%;" />

2. 用过 go 的都知道 channel，无需多言，直接开整！

## 1 核心数据结构

1. Golang channel 核心数据结构

   <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/Golang%20channel%20%E6%A0%B8%E5%BF%83%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84.png" alt="Golang channel 核心数据结构" style="zoom:50%;" />

### 1-1 hchan

1. hchan 数据结构如下

   ```Go
   type hchan struct {
       qcount   uint           // total data in the queue
       dataqsiz uint           // size of the circular queue
       buf      unsafe.Pointer // points to an array of dataqsiz elements
       elemsize uint16
       closed   uint32
       elemtype *_type // element type
       sendx    uint   // send index
       recvx    uint   // receive index
       recvq    waitq  // list of recv waiters
       sendq    waitq  // list of send waiters
   
       lock mutex
   }
   ```

2. hchan： channel 数据结构
   - **qcount**: 当前 channel 中存在多少个元素
   - **dataqsize**: 当前 channel 能存放的元素容量
   - **buf**: channel 中用于存放元素的环形缓冲区
   - **elemsize**: channel 元素类型的大小
   - **closed**: 标识 channel 是否关闭
   - **elemtype**: channel 元素类型
   - **sendx**: 发送元素进入环形缓冲区的 index
   - **recvx**: 接收元素所处的环形缓冲区的 index
   - **recvq**: 因接收而陷入阻塞的协程队列
   - **sendq**: 因发送而陷入阻塞的协程队列

### 1-2 waitq

1. waitq 数据结构如下

   ```Go
   type waitq struct {
       first *sudog
       last  *sudog
   }
   ```

2. waitq：阻塞的协程队列
   - **first**: 队列头部
   - **last**: 队列尾部

### 1-3 sudog

1. sudog 数据结构如下

   ```Go
   type sudog struct {
       g *g
   
       next *sudog
       prev *sudog
       elem unsafe.Pointer // data element (may point to stack)
       // ...
       c        *hchan 
   }
   ```

2. sudog：用于包装协程的节点
   - **g**: goroutine，协程
   - **next**: 队列中的下一个节点
   - **prev**: 队列中的前一个节点
   - **elem**: 读取/写入 channel 的数据的容器
   - **c**: 标识与当前 sudog 交互的 chan

## 2 构造函数

1. 构造过程如下

   <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/channel%20%E6%9E%84%E9%80%A0%E5%87%BD%E6%95%B0.png" alt="channel 构造函数" style="zoom:50%;" />

2. channel 构造函数如下

   ```Go
   func makechan(t *chantype, size int) *hchan {
       elem := t.elem
   
       // ...
       mem, overflow := math.MulUintptr(elem.size, uintptr(size))
       if overflow || mem > maxAlloc-hchanSize || size < 0 {
           panic(plainError("makechan: size out of range"))
       }
   
       var c *hchan
       switch {
       case mem == 0:
           // Queue or element size is zero.
           c = (*hchan)(mallocgc(hchanSize, nil, true))
       case elem.ptrdata == 0:
           // Elements do not contain pointers.
           // Allocate hchan and buf in one call.
           c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
           c.buf = add(unsafe.Pointer(c), hchanSize)
       default:
           // Elements contain pointers.
           c = new(hchan)
           c.buf = mallocgc(mem, elem, true)
       }
   
       c.elemsize = uint16(elem.size)
       c.elemtype = elem
       c.dataqsiz = uint(size)
       
       lockInit(&c.lock, lockRankHchan)
   
       return
   }
   ```

   - **判断申请内存空间大小是否越界，mem 大小为 element 类型大小与 element 个数相乘后得到，仅当无缓冲型 channel 时，因个数为 0 导致大小为 0**
   - **根据类型，初始 channel，分为 无缓冲型、有缓冲元素为 struct 型、有缓冲元素为 pointer 型 channel**
   - **倘若为无缓冲型，则仅申请一个大小为默认值 96 的空间**
   - **如若有缓冲的 struct 型，则一次性分配好 96 + mem 大小的空间，并且调整 chan 的 buf 指向 mem 的起始位置**
   - **倘若为有缓冲的 pointer 型，则分别申请 chan 和 buf 的空间，两者无需连续**
   - **对 channel 的其余字段进行初始化，包括元素类型大小、元素类型、容量以及锁的初始化**

## 3 写流程

### 3-1 两类异常情况处理

1. 两类异常情况处理

   ```Go
   func chansend1(c *hchan, elem unsafe.Pointer) {
       chansend(c, elem, true, getcallerpc())
   }
   
   
   func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
       if c == nil {
           gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
           throw("unreachable")
       }
   
   
       lock(&c.lock)
   
   
       if c.closed != 0 {
           unlock(&c.lock)
           panic(plainError("send on closed channel"))
       }
   }
   ```

   - **对于未初始化的 chan，写入操作会引发死锁**

   - **对于已关闭的 chan，写入操作会引发 panic**

### 3-2 case1：写时存在阻塞读协程

1. 写时存在阻塞读协程过程：

   <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/%E5%86%99%E6%97%B6%E5%AD%98%E5%9C%A8%E9%98%BB%E5%A1%9E%E8%AF%BB%E5%8D%8F%E7%A8%8B.png" alt="写时存在阻塞读协程" style="zoom:50%;" />

2. 源码如下

   ```Go
   func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
       // ...
   
       lock(&c.lock)
   
       // ...
   
       if sg := c.recvq.dequeue(); sg != nil {
           // Found a waiting receiver. We pass the value we want to send
           // directly to the receiver, bypassing the channel buffer (if any).
           send(c, sg, ep, func() { unlock(&c.lock) }, 3)
           return true
       }
       
       // ..
   }
   ```

   - **加锁**
   - **从阻塞度协程队列中取出一个 goroutine 的封装对象 sudog**
   - **在 send 方法中，会基于 memmove 方法，直接将元素拷贝交给 sudog 对应的 goroutine**
   - **在 send 方法中会完成解锁动作**

### 3-3 case2：写时无阻塞读协程但环形缓冲区仍有空间

1. 写时无阻塞读协程但环形缓冲区仍有空间过程：

   <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/%E5%86%99%E6%97%B6%E6%97%A0%E9%98%BB%E5%A1%9E%E8%AF%BB%E5%8D%8F%E7%A8%8B%E4%BD%86%E7%8E%AF%E5%BD%A2%E7%BC%93%E5%86%B2%E5%8C%BA%E4%BB%8D%E6%9C%89%E7%A9%BA%E9%97%B4.png" alt="写时无阻塞读协程但环形缓冲区仍有空间" style="zoom:50%;" />

2. 源码

   ```Go
   func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
       // ...
       lock(&c.lock)
       // ...
       if c.qcount < c.dataqsiz {
           // Space is available in the channel buffer. Enqueue the element to send.
           qp := chanbuf(c, c.sendx)
           typedmemmove(c.elemtype, qp, ep)
           c.sendx++
           if c.sendx == c.dataqsiz {
               c.sendx = 0
           }
           c.qcount++
           unlock(&c.lock)
           return true
       }
   
   
       // ...
   }
   ```

   - **加锁**
   - **将当前元素添加到环形缓冲区 sendx 对应的位置**
   - **sendx++**
   - **qcount++**
   - **解锁，返回**

### 3-4 case3：写时无阻塞读协程且环形缓冲区无空间

1. 写时无阻塞读协程且环形缓冲区无空间过程

   <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/%E5%86%99%E6%97%B6%E6%97%A0%E9%98%BB%E5%A1%9E%E8%AF%BB%E5%8D%8F%E7%A8%8B%E4%B8%94%E7%8E%AF%E5%BD%A2%E7%BC%93%E5%86%B2%E5%8C%BA%E6%97%A0%E7%A9%BA%E9%97%B4.png" alt="写时无阻塞读协程且环形缓冲区无空间" style="zoom:50%;" />

   

2. 源码

   ```Go
   func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
       // ...
       lock(&c.lock)
   
       // ...
       gp := getg()
       mysg := acquireSudog()
       mysg.elem = ep
       mysg.g = gp
       mysg.c = c
       gp.waiting = mysg
       c.sendq.enqueue(mysg)
       
       atomic.Store8(&gp.parkingOnChan, 1)
       gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
       
       gp.waiting = nil
       closed := !mysg.success
       gp.param = nil
       mysg.c = nil
       releaseSudog(mysg)
       return true
   }
   ```

   - **加锁**
   - **构造封装当前 goroutine 的 sudog 对象**
   - **完成指针指向，建立 sudog、goroutine、channel 之间的指向关系**
   - **把 sudog 添加到当前 channel 的阻塞写协程队列中**
   - **park 当前协程**
   - **倘若协程从 park 中被唤醒，则回收 sudog(sudog能被唤醒，其对应的元素必然已经被读协程取走)**
   - **解锁，返回**

### 3-5 写流程整体串联

1. 写流程整体串联过程

   <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/%E5%86%99%E6%B5%81%E7%A8%8B%E6%95%B4%E4%BD%93%E4%B8%B2%E8%81%94.png" alt="写流程整体串联" style="zoom:50%;" />

## 4 读流程

### 4-1 异常 case1：读空 channel

1. 读空 channel

   ```Go
   func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
       if c == nil {
           gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
           throw("unreachable")
       }
       // ...
   }
   ```

   - **park 挂起，引起死锁**

### 4-2 异常 case2：channel 已关闭且内部无元素

1. channel 已关闭且内部无元素

   ```Go
   func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
     
       lock(&c.lock)
   
   
       if c.closed != 0 {
           if c.qcount == 0 {
               unlock(&c.lock)
               if ep != nil {
                   typedmemclr(c.elemtype, ep)
               }
               return true, false
           }
           // The channel has been closed, but the channel's buffer have data.
       } 
   
   
       // ...
   }
   ```

   - **直接解锁返回即可**

### 4-3 case3：读时有阻塞的写协程

1. 读时有阻塞的写协程过程：

   <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/%E8%AF%BB%E6%97%B6%E6%9C%89%E9%98%BB%E5%A1%9E%E7%9A%84%E5%86%99%E5%8D%8F%E7%A8%8B.png" alt="读时有阻塞的写协程" style="zoom:50%;" />

2. 源码

   ```Go
   func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
      
       lock(&c.lock)
   
   
       // Just found waiting sender with not closed.
       if sg := c.sendq.dequeue(); sg != nil {
           recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
           return true, true
        }
   }
   
   func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
   	if c.dataqsiz == 0 {
   		if ep != nil {
   			// copy data from sender
   			recvDirect(c.elemtype, sg, ep)
   		}
   	} else {
   		// Queue is full. Take the item at the
   		// head of the queue. Make the sender enqueue
   		// its item at the tail of the queue. Since the
   		// queue is full, those are both the same slot.
   		qp := chanbuf(c, c.recvx)
   		if ep != nil {
   			typedmemmove(c.elemtype, ep, qp)
   		}
   		
   		typedmemmove(c.elemtype, qp, sg.elem)
   		c.recvx++
   		if c.recvx == c.dataqsiz {
   			c.recvx = 0
   		}
   		c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
   	}
   	sg.elem = nil
   	gp := sg.g
   	unlockf()
   	gp.param = unsafe.Pointer(sg)
   	sg.success = true
   	goready(gp, skip+1)
   }
   ```

   - **加锁**
   - **从阻塞写协程队列中获取到一个写协程**
   - **倘若 channel 无缓冲区，则直接读取写协程元素，并唤醒写协程**
   - **倘若 channel 有缓冲区，则读取缓冲区头部元素，并将写协程元素写入缓冲区尾部后唤醒写写成**
   - **解锁，返回**

### 4-4 case4：读时无阻塞写协程且缓冲区有元素

1. 读时无阻塞写协程且缓冲区有元素过程

   <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/%E8%AF%BB%E6%97%B6%E6%97%A0%E9%98%BB%E5%A1%9E%E5%86%99%E5%8D%8F%E7%A8%8B%E4%B8%94%E7%BC%93%E5%86%B2%E5%8C%BA%E6%9C%89%E5%85%83%E7%B4%A0.png" alt="读时无阻塞写协程且缓冲区有元素" style="zoom:50%;" />

2. 源码

   ```Go
   func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
   
   
       lock(&c.lock)
   
   
       if c.qcount > 0 {
           // Receive directly from queue
           qp := chanbuf(c, c.recvx)
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
   }
   ```

   - **加锁**
   - **获取到 recvx 对应位置的元素**
   - **recvx++**
   - **qcount--**
   - **解锁，返回**

### 4-5 case5：读时无阻塞写协程且缓冲区无元素

1. 读时无阻塞写协程且缓冲区无元素过程

   <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/%E8%AF%BB%E6%97%B6%E6%97%A0%E9%98%BB%E5%A1%9E%E5%86%99%E5%8D%8F%E7%A8%8B%E4%B8%94%E7%BC%93%E5%86%B2%E5%8C%BA%E6%97%A0%E5%85%83%E7%B4%A0.png" alt="读时无阻塞写协程且缓冲区无元素" style="zoom:50%;" />

2. 源码

   ```Go
   func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
       lock(&c.lock)
   
       gp := getg()
       mysg := acquireSudog()
       mysg.elem = ep
       gp.waiting = mysg
       mysg.g = gp
       mysg.c = c
       gp.param = nil
       c.recvq.enqueue(mysg)
       atomic.Store8(&gp.parkingOnChan, 1)
       gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)
   
       gp.waiting = nil
       success := mysg.success
       gp.param = nil
       mysg.c = nil
       releaseSudog(mysg)
       return true, success
   }
   ```

   - **加锁**
   - **构造封装当前 goroutine 的 sudog 对象**
   - **完成指针指向，建立 sudog、goroutine、channel 之间的指向关系**
   - **把 sudog 添加到当前 channel 的阻塞读协程队列中**
   - **park 当前协程**
   - **倘若协程从 park 中被唤醒，则回收 sudog(sudog能被唤醒，其对应的元素必然已经被写入)****
   - **解锁，返回**

### 4-6 读流程整体串联

1. 读流程整体串联过程

   <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/%E8%AF%BB%E6%B5%81%E7%A8%8B%E6%95%B4%E4%BD%93%E4%B8%B2%E8%81%94.png" alt="读流程整体串联" style="zoom:50%;" />

## 5 阻塞与非阻塞模式

1. 在上述源码分析流程中，均是以阻塞模式为主线进行讲述，忽略非阻塞模式的有关处理逻辑
2. 此处阐明两个问题：
   - 非阻塞模式下，流程逻辑有何区别？
   - 何时会进入非阻塞模式？

### 5-1 非阻塞模式逻辑区别

1. 非阻塞模式下，读/写 channel 方法通过一个 bool 型的响应参数，用以标识是否读取/写入成功
   - **所有需要使得当前 goroutine 被挂起的操作，在非阻塞模式下都会返回 false**
   - **所有是的当前 goroutine 会进入死锁的操作，在非阻塞模式下都会返回 false**
   - **所有能立即完成读取/写入操作的条件下，非阻塞模式下会返回 true**

### 5-2 何时进入非阻塞模式

1. 默认情况下，读/写 channel 都是阻塞模式，只有在 select 语句组成的多路复用分支中，与 channel 的交互会变成非阻塞模式：

   ```Go
   ch := make(chan int)
   select{
     case <- ch:
     default:
   }
   ```

### 5-3 代码一览

1. 代码一览

   ```Go
   func selectnbsend(c *hchan, elem unsafe.Pointer) (selected bool) {
   	return chansend(c, elem, false, getcallerpc())
   }
   
   func selectnbrecv(elem unsafe.Pointer, c *hchan) (selected, received bool) {
   	return chanrecv(c, elem, false)
   }
   ```

   - 在 select 语句包裹的多路复用分支中，读和写 channel 操作会被汇编为 selectnbrecv 和 selectnbsend 方法，底层同样复用 chanrecv 和 chansend 方法，但此时由于第三个入参 block 被设置为 false，导致后续会走进非阻塞的处理分支

## 6 两种读 channel 的协议

1. 读取 channel 时，可以根据第二个 bool 型的返回值用以判断当前 channel 是否已处于关闭状态：

   ```Go
   ch := make(chan int, 2)
   got1 := <- ch
   got2,ok := <- ch
   ```

2. 实现上述功能的原因是，两种格式下，读 channel 操作会被汇编成不同的方法：

   ```Go
   func chanrecv1(c *hchan, elem unsafe.Pointer) {
       chanrecv(c, elem, true)
   }
   
   
   //go:nosplit
   func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
       _, received = chanrecv(c, elem, true)
       return
   }
   ```

## 7 关闭

1. 关闭 channel 过程

   <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/%E5%85%B3%E9%97%AD%20channel%20%E8%BF%87%E7%A8%8B.png" alt="关闭 channel 过程" style="zoom:50%;" />

2. 源码如下

   ```Go
   func closechan(c *hchan) {
       if c == nil {
           panic(plainError("close of nil channel"))
       }
   
       lock(&c.lock)
       if c.closed != 0 {
           unlock(&c.lock)
           panic(plainError("close of closed channel"))
       }
   
       c.closed = 1
   
       var glist gList
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
           gp := sg.g
           gp.param = unsafe.Pointer(sg)
           sg.success = false
           glist.push(gp)
       }
   
       // release all writers (they will panic)
       for {
           sg := c.sendq.dequeue()
           if sg == nil {
               break
           }
           sg.elem = nil
           gp := sg.g
           gp.param = unsafe.Pointer(sg)
           sg.success = false
           glist.push(gp)
       }
       unlock(&c.lock)
   
       // Ready all Gs now that we've dropped the channel lock.
       for !glist.empty() {
           gp := glist.pop()
           gp.schedlink = 0
           goready(gp, 3)
     }
   }
   ```

   - **关闭未初始化过的 channel 会 panic**
   - **加锁**
   - **重复关闭 channel 会 panic**
   - **将阻塞读协程队列中的协程节点统一添加到 glist**
   - **将阻塞写协程队列中的协程节点统一添加到 glist**
   - **唤醒 glist 当中的所有协程**

## 一道考题

1. 要求实现一个 map

   - **面向高并发**

   - **只存在插入和查询操作 O(1)**

   - **查询时，若 key 存在，直接返回 val；若 key 不存在，阻塞直到 key val 对被放入后，获取 val 返回； 等待指定时长仍未放入，返回超时错误**

   - **写出真实代码，不能有死锁或者 panic 风险**

     ```Go
     type MyConcurrentMap struct{
       // ...
     }
     
     func (m *MyConcurrentMap)Put(k,v int){
       // ...
     }
     
     func (m *MyConcurrentMap)Get(k int, maxWaitingDuration time.Duration)(int,error){
       // ...
     }
     ```


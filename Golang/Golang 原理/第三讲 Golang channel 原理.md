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

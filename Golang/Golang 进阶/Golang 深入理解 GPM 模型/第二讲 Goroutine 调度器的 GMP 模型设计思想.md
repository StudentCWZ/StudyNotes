# Goroutine 调度器的 GMP 模型设计思想
## GMP 模型简介

1. GMP
    + G: Goroutine ==> 协程
    + P: Processor ==> 处理器
    + M: Thread ==> 内核线程
2. 全局队列：存放等待运行的 G
3. P 的本地队列
    + 存放等待运行的 G
    + 数量限制：不超过 256 G
    + 优先将新创建的 G 放在 P 的本地队列中，如果满了会放在全局队列中
4. P 列表
    + 程序启动时创建
    + 最多有 GOMAXPROCS 个(可配置)
5. M 列表：当前操作系统分配到当前 Go 程序的内核线程数
6. P 和 M 的数量
    + P 的数量问题
        * 环境变量 $GOMAXPROCS
        * 在程序中通过 runtime.GOMAXPROCS() 来设置
    + M 的数量问题
        * Go 语言本身是限定 M 的最大量是 10000(忽略)
        * runtime/debug 包中的 SetMaxThreads 函数来设置
        * 有一个 M 阻塞，会创建一个新的 M
        * 如果有一个 M 空闲，那么就会回收或者睡眠

## 调度器的设计策略

1. 复用线程：避免频繁的创建、销毁线程，而是对线程的复用。
    - work stealing 机制：当本线程无可运行的 G 时，尝试从其他线程绑定的 P 偷取 G，而不是销毁线程。
    - hand off 机制：当本线程因为 G 进行系统调用阻塞时，线程释放绑定的 P ，把 P 转移给其他空闲的线程执行。
2. 利用并行
    - GOMAXPROCS 设置 P 的数量，最多有 GOMAXPROCS 个线程分布在多个 CPU 上同时运行
3. 抢占
    - 在 coroutine 中要等待一个协程主动让出 CPU 才执行下一个协程，在 Go 中，一个 Goroutine 最多占用 CPU 10 ms，防止其他 goroutine 被饿死
4. 全局队列
    - 当 M 执行 work stealing 从其他 P 偷不到 G 时，它可以从全局 G 队列获取 G

## go 指令的调度流程

1. 流程：
    - 我们通过 go func() 来创建一个 goroutine
    - 有两个存储 G 的队列，一个是局部调度器 P 的本地队列、一个是全局 G 队列。新创建的 G 会先保存在 P 的本地队列中，如果 P 的本地队列已经满了就会保存在全局的队列中
    - G 只能运行在 M 中，一个 M 必须持有一个 P，M 与 P 是 1:1 的关系。M 会从 P 的本地队列弹出一个可执行状态的 G 来执行，如果 P 的本地队列为空，就会想向其他的 MP 组合偷出一个可执行的 G 来执行
    - 一个 M 调度 G 的过程是一个循环机制；
    - 当 M 执行某一个 G 时如果发生了 syscall 或其余阻塞操作，M 会阻塞，如果当前有一些 G 在执行，runtime 会把这个线程 M 从 P 中摘除(detach)，然后创建一个新的操作系统的线程(如果空闲的线程可用就复用空闲线程)来服务于这个 P;
    - 当 M 系统调用结束时候，这个 G 会尝试获取一个空闲的 P 执行，并放入到这个 P 的本地队列。如果获取不到 P，那么这个线程 M 变成休眠状态，加入到空闲线程中，然后这个 G 会被放入全局队列中

## 调度器的生命周期

1. M0:
    - M0 是启动程序后编号为 0 的主线程，这个 M 对应的实例会在 runtime.m0 中，不需要在 heap 上分配，M0 负责执行初始化操作和启动第一个 G，再之后 M0 就跟其他的 M 一样了
2. G0:
    - G0是每次启动一个 M 都会第一个创建的 goroutine，G0 仅用于负责调度的 G，G0 不指向任何可执行的函数，每个 M 都会有一个自己的 G0。在调度或系统调用时会使用 G0 的栈空间，全局变量的 G0 是 M0 的 G0

## 可视化的 GMP 编程

1. 基本的 trace 的编程
    - 创建 trace 文件: `f, err := os.Create("trace.out")`
    - 启动 trace: `trace.Start(f)`
    - 停止 trace: `tarce.Stop()`
    - go build 并且运行之后，会得到一个 trace.out 文件

2. 通过 go tool trace 工具打开 trace 文件: go tool trace trace.out

3. 通过 Debug trace 查看 GMP 信息：GODEBUG=schedtrace=1000 ./可执行程序

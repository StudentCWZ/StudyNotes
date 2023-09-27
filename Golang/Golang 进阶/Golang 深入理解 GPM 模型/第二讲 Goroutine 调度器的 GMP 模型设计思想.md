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

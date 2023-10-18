# 第二讲 Golang map 底层原理
## 0 前言

1. map 是一种如此经典的数据结构，各种语言中都对 map 着一套宏观流程相似、但技术细节百花齐放的实现方式。
2. 古语有云，大道至简，但这更多体现在原理和使用层面，从另一个角度而言，在认知上越基础的东西反而往往有着越复杂的实现细节。
3. 那么，诸位英雄好汉中又有哪几位有勇气随我一同深入 golang 的 map 源码，抱着不通透不罢休的态度，对其底层原理进行一探究竟呢？

## 1 基本用法

### 1-1 概述

1. map 又称字典，是一种常用的数据结构，核心特征包含下述三点：

   - 存储基于 key-value 对映射的模式

   - 基于 key 维度实现存储数据的去重

   - 读、写、删操作控制，时间复杂度 O(1)

### 1-2 初始化

#### 1-2-1 几种初始化方法

1. golang 中，对 map 的初始化分为以下几种方式：

   - 通过 make 关键字进行初始化，同时指定 map 预分配的容量

     ```Go
     myMap1 := make(map[int]int, 2)
     ```

   - 通过 make 关键字进行初始化，不显式声明容量，因此默认容量 为 0

     ```Go
     myMap2 := make(map[int]int)
     ```

   - 初始化操作连带赋值，一气呵成

     ```Go
     myMap3 := map[int]int{
       1: 2,
       3: 4,
     }
     ```

#### 1-2-2 key 的类型要求

1. map 中，key 的数据类型必须为可比较的类型，需要关注，不可比较的类型包括：切片、map 和函数。

#### 1-2-3 读

读 map 分为下面两种方式：

- 第一种方式是直接读，倘若 key 存在，则获取到对应的 val，倘若 key 不存在或者 map 未初始化，会返回 val 类型的零值作为兜底

  ```Go
  v1 := myMap[10]
  ```

- 第二种方式是读的同时添加一个 bool 类型的 flag 标识是否读取成功. 倘若 ok == false，说明读取失败， key 不存在，或者 map 未初始化

  ```Go
  v2, ok := myMap[10]
  ```

- 此处同一种语法能够实现不同返回值类型的适配，是由于代码在汇编时，会根据返回参数类型的区别，映射到不同的实现方法

#### 1-2-4 写

1. 写操作的语法如下

   ```Go
   myMap[5] = 6
   ```

2. 须注意的一点是，倘若 map 未初始化，直接执行写操作会导致 panic

   ```Go
   const plainError string
   panic(plainError("assignment to entry in nil map"))
   ```

#### 1-2-5 删

1. 删除方法如下:

   ```Go
   delete(myMap, 5)
   ```

2. 执行 delete 方法时，倘若 key 存在，则会从 map 中将对应的 key-value 对删除；倘若 key 不存在或 map 未初始化，则方法直接结束，不会产生显式提示

#### 1-2-6 遍历

1. 遍历分为下面两种方式：

   - 基于 k,v 依次承接 map 中的 key-value 对

     ```Go
     for k,v := range myMap{
       // ...
     }
     ```

   - 基于 k 依次承接 map 中的 key，不关注 val 的取值

     ```Go
     for k := range myMap{
       // ...
     }
     ```

2. 需要注意的是，在执行 map 遍历操作时，获取的 key-value 对并没有一个固定的顺序，因此前后两次遍历顺序可能存在差异

#### 1-2-7 并发冲突

1. map 不是并发安全的数据结构，倘若存在并发读写行为，会抛出 fatal error

2. 具体规则是：

   - 并发读没有问题

   - 并发读写中的写是广义上的，包含写入、更新、删除等操作

   - 读的时候发现其他 goroutine 在并发写，抛出 fatal error

   - 写的时候发现其他 goroutine 在并发写，抛出 fatal error

     ```Go
     fatal("concurrent map read and map write")
     fatal("concurrent map writes")
     ```

3. 需要关注，此处并发读写会引发 fatal error，是一种比 panic 更严重的错误，无法使用 recover 操作捕获

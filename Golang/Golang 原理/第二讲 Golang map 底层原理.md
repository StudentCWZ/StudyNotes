# 第二讲 Golang map 底层原理
## 0 前言

1. map 是一种如此经典的数据结构，各种语言中都对 map 着一套宏观流程相似、但技术细节百花齐放的实现方式。
2. 古语有云，大道至简，但这更多体现在原理和使用层面，从另一个角度而言，在认知上越基础的东西反而往往有着越复杂的实现细节。
3. 那么，诸位英雄好汉中又有哪几位有勇气随我一同深入 Golang 的 map 源码，抱着不通透不罢休的态度，对其底层原理进行一探究竟呢？

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

   - 基于 k, v 依次承接 map 中的 key-value 对

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

## 2 核心原理

### 2-1 概述

1. map 又称为 hash map，在算法上基于 hash 实现 key 的映射和寻址；在数据结构上基于桶数组实现 key-value 对的存储。

2. 以一组 key-value 对写入 map 的流程为例进行简述：

   - 通过哈希方法取得 key 的 hash 值
   - hash 值对桶数组长度取模，确定其所属的桶
   - 在桶中插入 key-value 对

3. hash 的性质，保证了相同的 key 必然产生相同的 hash 值，因此能映射到相同的桶中，通过桶内遍历的方式锁定对应的 key-value 对。

4. 因此，只要在宏观流程上，控制每个桶中 key-value 对的数量，就能保证 map 的几项操作都限制为常数级别的时间复杂度。

5. Golang map 数据结构

   <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/Golang%20map%20%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84.png" alt="Golang map 数据结构" style="zoom:50%;" />

### 2-2 hash

1. hash 译作散列，是一种将任意长度的输入压缩到某一固定长度的输出摘要的过程，由于这种转换属于压缩映射，输入空间远大于输出空间，因此不同输入可能会映射成相同的输出结果. 此外，hash 在压缩过程中会存在部分信息的遗失，因此这种映射关系具有不可逆的特质。

   - **hash 的可重入性：相同的 key，必然产生相同的 hash 值**

   - **hash 的离散性：只要两个 key 不相同，不论其相似度的高低，产生的 hash 值会在整个输出域内均匀地离散化**

     <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/hash%20%E7%A6%BB%E6%95%A3%E6%80%A7.png" alt="hash 离散性" style="zoom:50%;" />

   - **hash 的单向性：企图通过 hash 值反向映射回 key 是无迹可寻的**

     <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/hash%20%E7%9A%84%E5%8D%95%E5%90%91%E6%80%A7.png" alt="hash 的单向性" style="zoom:50%;" />

   - **hash 冲突：由于输入域(key)无穷大，输出域(hash 值)有限，因此必然存在不同 key 映射到相同 hash 值的情况，称之为 hash 冲突.**

     <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/hash%20%E5%86%B2%E7%AA%81.png" alt="hash 冲突" style="zoom:50%;" />

### 2-3 桶数组

1. map 中，会通过长度为 2 的整数次幂的桶数组进行 key-value 对的存储：

   - **每个桶固定可以存放 8 个 key-value 对**

   - **倘若超过 8 个 key-value 对打到桶数组的同一个索引当中，此时会通过创建桶链表的方式来化解这一问题**

     <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/%E6%A1%B6%E6%95%B0%E7%BB%84.png" alt="桶数组" style="zoom:50%;" />

### 2-4 map 解决 hash 冲突

1. 首先，由于 hash 冲突的存在，不同 key 可能存在相同的 hash 值。

2. 再者，hash 值会对桶数组长度取模，因此不同 hash 值可能被打到同一个桶中。

3. 综上，不同的 key-value 可能被映射到 map 的同一个桶当中。

4. 此时最经典的解决手段分为两种：拉链法和开放寻址法。

   - **拉链法**

     - 拉链法中，将命中同一个桶的元素通过链表的形式进行链接，因此很便于动态扩展。

       <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/%E6%8B%89%E9%93%BE%E6%B3%95.png" alt="拉链法" style="zoom: 50%;" />

   - **开放寻址法**

     - 开放寻址法中，在插入新条目时，会基于一定的探测策略持续寻找，直到找到一个可用于存放数据的空位为止。

       <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/%E5%BC%80%E6%94%BE%E5%AF%BB%E5%9D%80%E6%B3%95.png" alt="开放寻址法" style="zoom:50%;" />

   - **对标拉链还有开放寻址法，两者的优劣对比**

     |  **方法**  |                           **优点**                           |
     | :--------: | :----------------------------------------------------------: |
     |   拉链法   |               简单常用；无需预先为元素分配内存               |
     | 开放寻址法 | 无需额外的指针用于链接元素；内存地址完全连续，可以基于局部性原理，充分利用 CPU 高速缓存 |

5. 在 map 解决 hash/分桶 冲突问题时，实际上结合了拉链法和开放寻址法两种思路. 以 map 的插入写流程为例，进行思路阐述：

   - **桶数组中的每个桶，严格意义上是一个单向桶链表，以桶为节点进行串联**

   - **每个桶固定可以存放 8 个 key-value 对**

   - **当 key 命中一个桶时，首先根据开放寻址法，在桶的 8 个位置中寻找空位进行插入**

   - **倘若桶的 8 个位置都已被占满，则基于桶的溢出桶指针，找到下一个桶，重复第 3 步**

   - **倘若遍历到链表尾部，仍未找到空位，则基于拉链法，在桶链表尾部续接新桶，并插入 key-value 对**

     <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/map%20%E8%A7%A3%E5%86%B3%E5%86%B2%E7%AA%81%E6%96%B9%E6%B3%95.png" alt="map 解决冲突方法" style="zoom:50%;" />

### 2-5 扩容优化性能

1. 倘若 map 的桶数组长度固定不变，那么随着 key-value 对数量的增长，当一个桶下挂载的 key-value 达到一定的量级，此时操作的时间复杂度会趋于线性，无法满足诉求。

2. 因此在实现上，map 桶数组的长度会随着 key-value 对数量的变化而实时调整，以保证每个桶内的 key-value 对数量始终控制在常量级别，满足各项操作为 O(1) 时间复杂度的要求。

3. map 扩容机制的核心点包括：

   - **扩容分为增量扩容和等量扩容**

   - **当桶内 key-value 总数/桶数组长度 > 6.5 时发生增量扩容，桶数组长度增长为原值的两倍**

   - **当桶内溢出桶数量大于等于 2^B 时( B 为桶数组长度的指数，B 最大取 15)，发生等量扩容，桶的长度保持为原值**

   - **采用渐进扩容的方式，当桶被实际操作到时，由使用者负责完成数据迁移，避免因为一次性的全量数据迁移引发性能抖动**

     <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/map%20%E6%89%A9%E5%AE%B9%E8%BF%87%E7%A8%8B.png" alt="map 扩容过程" style="zoom:50%;" />


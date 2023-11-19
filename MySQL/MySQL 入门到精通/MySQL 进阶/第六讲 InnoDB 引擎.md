# 第六讲 InnoDB 引擎

## 6.1 逻辑存储结构

1. **表空间(ibd 文件)，一个 mysql 实例可以对应多个表空间，用于存储记录、索引等数据。**
2. **段，分为数据(Leaf node segment)、索引段(Non-leaf node segment)、回滚段(Rollback segment)，InnoDB 是索引组织表，数据段就是 B+ 树的叶子节点，索引段即为 B+ 树的非叶子节点。段用来管理多个 Extent(区)。**
3. **区，表空间的单元结构，每个区的大小为 1 M。默认情况下，InnoDB 存储引擎大小为 16 k，即一个区中一共有 64 个连续的页。**
4. **页，是 InnoDB 存储引擎磁盘管理的最小单元，每个页的大小默认为 16 KB，为了保证页的连续性，InnoDB 存储引擎每次从磁盘申请 4-5 个区**
5. **行，InnoDB 存储引擎数据是按行进行存放的。**
   - **trx_id: 每次对某条记录进行改动的时，都会把对应的事务 id 赋值给 trx_id 隐藏列**
   - **roll_pointer: 每次对某条记录进行改动的时，都会把旧的版本写入到 undo 日志中，然后这个隐藏列就相当于一个指针，可以通过它来找到该记录修改前的信息**

## 6.2 架构

1. MySQL 5.5 版本开始，默认使用 InnoDB 存储引擎，它擅长事务处理，具有崩溃恢复特性，在日常开发中使用非常广泛。下面是 InnoDB 架构图，左侧为内存结构，右侧为磁盘结构

   <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/MySQL%20%E8%BF%9B%E9%98%B6%E7%AF%87-InnoDB%20%E6%9E%B6%E6%9E%84.png" alt="MySQL 进阶篇-InnoDB 架构" style="zoom:50%;" />

### 6.2.1 内存架构

1. Buffer Pool: 缓冲池是主内存中的一个区域，里面可以缓存磁盘上经常操作的真实数据，在执行增删改查操作时，先操作缓冲池中的数据(若缓冲池没有数据，则从磁盘加载并缓存)，然后再以一定频率刷新到磁盘，从而减少磁盘 IO，加快处理速度。
2. 注意：这里的 Buffer Pool 跟 Server 层的查询缓存是完全不同的，Buffer Pool 属于是 InnoDB 引擎层的范畴，而且 Server 层的 查询缓存在 MySQL 8.0 版本已经去掉了。
3. 缓冲池以 Page 页为单位，底层采用链表数据结构管理 Page。根据状态，将 Page 分为三种类型:
   - **free page: 空闲 page，未被使用**
   - **clean page: 被使用 page，数据没有被修改过**
   - **dirty page: 脏页，被使用 page，数据被修改，页中数据与磁盘的数据产生了不一致**
4. **Change Buffer: 更改缓冲区(针对于非唯一二级索引页)，在执行 DML 语句时，如果这些数据 Page 没有在 Buffer Pool 中，不会直接操作磁盘，而会将数据变更存在更改缓冲区 Change Buffer 中，在未来数据被读取的时候，再将数据合并并恢复到 Buffer Pool 中，再将合并后的数据刷新到磁盘中。**
5. **Change Buffer 的意义是什么？**
   - 与聚集索引不同，二级索引通常是非唯一的，并且以相对随机的顺序插入二级索引。同样，删除和更新可能会影响索引树中不能相邻的二级索引页，如果每一次都操作磁盘，会造成大量的磁盘 I/O。有了 Change Buffer 之后，我们可以在缓冲池中进行合并处理，减少磁盘 IO
6. Adaptivate Hash Index: 自适应 hash 索引，用于优化对 Buffer Pool 数据的查询，InnoDB 存储引擎会监控对表各个索引页的查询，如果观察到 hash 索引可以提升速度，则建立哈希索引，称之为自适应 hash 索引
   - **自适应哈希索引，无需人工干预，是系统根据情况自动完成的**
   - **参数: adaptive_hash_index**

7. **Log Buffer: 日志缓冲区，用来保存要写入到磁盘中的 log 日志数据(redo log、undo log)，默认大小为 16 MB，日志缓冲区的日志会定期刷新到磁盘中。如果需要更新、插入或删除许多行的事务，增加日志缓冲区的大小可以节省磁盘 I/O**

8. **Log Buffer 对应的参数:**

   - **innodb_log_buffer_size: 缓冲区大小**

   - **innodb_flush_log_at_trx_commit: 日志刷新到磁盘的时机**

     ```SQL
     show variables like '%log_buffer_size%';
     show variables like '%flush_log%';
     ```

9. **innodb_flush_log_at_trx_commit 参数含义**
   - **1: 日志在每次事务提交时写入并刷新到磁盘**
   - **0: 每秒将日志写入并刷新到磁盘一次**
   - **2: 日志在每次事务提交后写入，并每秒刷新到磁盘的一次**

### 6.2.2 磁盘架构

1. **System TableSpace: 系统表空间是更改缓冲区的存储区域。如果表是在系统表空间而不是每个表文件或通用表空间中创建的，它也可能包含表和索引数据。(MySQL 5.x 版本总还包含 InnoDB 数据字典，undolog 等)**

2. System TableSpace 对应的参数:

   - **innodb_data_file_path**

3. 示例

   ```SQL
   show variables like '%data_file_path%';
   ```

4. **File-Per-Table: 每个表的文件表空间包含单个 InnoDB 表的数据和索引，并存储在文件系统上的单个数据文件中**

5. File-Per-Table 对应的参数：

   - **innodb_file_per_table**

6. 示例

   ```SQL
   show variables like '%file_per_table%';
   ```

7. **General TableSpace: 通用表空间，需要通过 CREATE TABLESPACE 语法创建通用表空间，在创建表时，可以指定该表空间。**

8. General TableSpace 相关语法

   ```SQL
   CREATE TABLESPACE xxx ADD DATAFILE 'file_name' ENGINE = engine_name;
   ```

9. **Undo Tablespaces: 撤销表空间，MySQL 实例在初始化时，会自动创建两个默认的 undo 表空间(初始大小 16 M)，用于存储 undo log 日志**
10. **Temporary Tablespace: InnoDB 使用会话临时表空间和全局临时表空间，存储用户创建的临时表等数据**
11. **Doublewrite Buffer files: 双写缓冲区，InnoDB 引擎将数据页从 Buffer Pool 刷新到磁盘前，先将数据页写入双写缓冲区文件中，便于系统异常时恢复数据**
    - #ib_16384_0.dblwr
    - #ib_16384_1.dblwr
12. **Redo Log: 重做日志，是用来实现事务的持久性。该日志文件由两部分组成：重做日志缓冲区(redo log buffer)以及重做日志文件(redo log)，前者在内存中，后者在磁盘中，当事务提交之后会把所有修改信息存到该日志中，用于在刷新脏页到磁盘时，发生错误时，进行数据恢复使用**
13. 以循环的方式写入重做日志文件，涉及两个文件：
    - **ib_logfile0**
    - **ib_logfile1**

### 6.2.3 后台线程

1. **Master Thread**

   - 核心后台线程，负责调度其他线程，还负责将缓冲池中的数据异步刷新到磁盘中，保存数据的一致性，还包括脏页的刷新、合并插入缓存、undo 页的回收

2. **IO Thread**

   - 在 InnoDB 存储引擎中大量使用了 AIO 来处理 IO 请求，这样可以极大地提高数据库的性能，而 IO Thread 主要负责这些 I/O 请求的回调

     |       线程类型       | 默认个数 |             职责             |
     | :------------------: | :------: | :--------------------------: |
     |     Read thread      |    4     |         负责读写操作         |
     |     Write thread     |    4     |          负责写操作          |
     |      Log thread      |    1     |  负责将日志缓冲区刷新到磁盘  |
     | Insert buffer thread |    1     | 负责将写缓冲区内容刷新到磁盘 |

3. 查看 InnoDB 状态信息来查看线程

   ```SQL
   show engine innodb status;
   ```

4. **Purge Thread**
   - 主要用于回收事务已经提交了的 undo log，在事务提交之后，undo log 可能不用了，就用它来回收
5. **Page Cleaner Thread**
   - 协助 Master Thread 刷新脏页到磁盘的线程，它可以减轻 Master Thread 的工作压力，减少阻塞

## 6.3 事务原理

## 6.4 MVCC


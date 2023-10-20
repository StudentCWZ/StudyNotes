# 第三讲 SQL 优化

## 3-1 插入数据优化

1. **insert 优化**

   - **批量插入**

     ```SQL
     insert into tb_test values(1, 'Tom'), (2, 'Cat'), (3, 'Jerry');
     ```

   - **手动提交事务**

     ```SQL
     start transaction;
     insert into tb_test values(1, 'Tom'), (2, 'Cat'), (3, 'Jerry');
     insert into tb_test values(4, 'Tom'), (5, 'Cat'), (6, 'Jerry');
     insert into tb_test values(7, 'Tom'), (8, 'Cat'), (9, 'Jerry');
     commit;
     ```

   - **主键顺序插入**

   - **大批量插入数据**

     - 如果一次性需要插入大批量数据，使用 insert 语句插入性能较低，此时可以使用 MySQL 数据库提供的 load 指令进行插入。操作如下

       ```SQL
       # 客户端连接服务端，加上参数 --local-infile
       mysql --local-inlife -u root -p
       # 设置全局参数 local_infile 为 1，开启从本地加载文件导入数据的开关
       set global local_infile = 1;
       # 执行 load 指令将准备好的数据，加载到表结构中
       load data local file 'root/sql1.log' into table `tb_user` fields terminated by ',' lines terminated by '\n';
       ```

2. **主键顺序插入性能高于乱序插入**

## 3-2 主键优化

1. 数据的组织方式
   - 在 InnoDB 存储引擎中，表数据都是根据主键顺序组织存放的，这种存储方式的表称为**索引组织表(index organized table IOT)**。
2. 页分裂
   - 页可以为空，也可以填充一半，也可以填充 100 %。每个页包含了 2 - N 行数据(如果一行数据过大，就会行溢出)，根据主键排列
   - MySQL 页分裂(Page Split)指的是当一个数据页(Page)已经满了，再次插入新数据时，MySQL 不得不将该页分裂成两个或多个页的过程。
   - 相对于主键顺序插入，乱序插入更容易造成页分裂，导致插入性能下降
3. 页合并
   - 当删除一行记录时，实际上记录并没有被物理删除，只是记录被标记(flaged)为删除并且它的空间变得允许被其他记录声明使用
   - 当页中删除的记录达到 MERGE_THRESOLD(默认为页的 50 %)，InnoDB 会开始寻找最靠近的页(前或后)看看是否可以将两个页合并以优化空间使用

4. 主页设计原则
   - **满足业务需求的情况下，尽量降低主键的长度**
   - **插入数据时，尽量选择顺序插入，选择使用 AUTO_INCREMENT 自增主键**
   - **尽量不要使用 UUID 做主键或者是其他自然主键，比如身份证号**
   - **业务操作时，避免对主键的修改**

## 3-3 order by 优化

## 3-4 group by 优化

## 3-5 limit 优化

## 3-6 count 优化

## 3-7 update 优化

 

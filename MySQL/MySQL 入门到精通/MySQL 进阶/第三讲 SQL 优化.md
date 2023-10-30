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

1. Using filesort: 通过表的索引或全表扫描，读取满足条件的数据行，然后在排序缓冲区 sort buffer 中完成排序操作，所有不是通过索引直接返回的排序结果都叫 FileSort 排序。

2. Using index: 通过有序索引顺序扫描直接返回有序数据，这种情况即为 Using index，不需要额外排序，操作效率高

   ```SQL
   # 没有创建索引时，根据 age, phone 进行排序
   explain select id, age, phone from tb_user order by age, phone;
   
   # 创建索引
   create index idx_user_age_phone on tb_user(age, phone);
   
   # 创建索引后，根据 age, phone 进行升序排序
   explain select id, age, phone from tb_user order by age, phone;
   # 创建索引后，根据 age, phone 进行降序排序
   explain select id, age, phone from tb_user order by age, phone desc;
   # 创建索引后，根据 age, phone 进行降序排序 ==> Using index;Using filesort: 违背最左前缀法则
   explain select id, age, phone from tb_user order by phone, age desc;
   # Using index;Using filesort
   explain select id, age, phone from tb_user order by age asc, phone desc;
   # 创建索引
   create index idx_user_phone_ad on tb_user(age asc, phone desc);
   # Using index ==> 建立索引，优化得到的结果
   explain select id, age, phone from tb_user order by age asc, phone desc;
   ```

3. 小结
   - **根据排序字段建立合适索引，多字段排序时，也遵循最左前缀法则**
   - 尽量使用覆盖索引
   - 多字段排序，一个升序一个降序，此时需要注意联合索引在创建时的规则(ASC/DESC)
   - 如果不可避免地出现 filesort，大数据量排序时，可以适当增大排序缓冲区大小 sort_buffer_size(默认为 256 k)

## 3-4 group by 优化

1. 在分组操作时，可以通过索引来提高效率

2. 分组操作时，索引的使用也是满足最左前缀法则的

   ```SQL
   # 删除掉目前的联合索引 idx_user_pro_age_sta
   drop index idx_user_pro_age_sta on tb_user;
   # 执行分组操作，根据 profession 字段分组 ==> Using temproary
   explain select profession, count(*) from tb_user group by profession;
   # 创建索引
   create index idx_user_pro_age_sta on tb_user(profession, age, status);
   # 执行分组操作，根据 profession 字段分组 ==> Using index ==> 执行效率更高
   explain select profession, count(*) from tb_user group by profession;
   # 执行分组操作，根据 profession, age 字段分组 ==> Using index
   explain select profession, age, count(*) from tb_user group by profession, age;
   # 执行分组操作，根据 age 字段分组 ==> Using index;Using temproary
   explain select age, count(*) from tb_user group by age;
   # 执行分组操作，根据 age 字段分组 ==> Using index
   explain select age, count(*) from tb_user where profession = '软件工程' group by age;
   ```

## 3-5 limit 优化

1. 一个常见又非常头疼的问题就是 limit 20000, 10，此时需要 MySQL 排序前 2000010 记录，仅仅返回 2000000 - 2000010 的记录，其他记录丢弃，查询排序的代价非常大。

2. 优化思路：一般分页查询时，通过创建**覆盖索引**能够比较好地提高性能，可以通过覆盖索引加子查询形式行进优化

   ```SQL
   # SQL 优化前
   explain select * from tb_sku limit 2000000, 10;
   # SQL 优化后
   explain select * from tb_sku t, (select id from tb_sku order by id limit 2000000, 10) a where t.id = a.id;
   ```

## 3-6 count 优化

1. MyISAM 引擎把一个表的总行数存在磁盘上，因此执行 count(*) 的时候会直接返回这个数，效率很高(注意：如果后面跟 where 条件，效率就不高了)
2. InnoDB 引擎就麻烦了，它执行 count(*) 的时候，需要把数据一行一行地从引擎里面读出来，然后累积计数
3. **优化思路：自己计数**
4. **count 的几种用法**
   - count() 是一个聚合函数，返回的结果集，一行行地判断，如果 count 函数的参数不是 NULL、累计值就加 1，否则不加，最后返回累计值
   - 用法: `count(*)`、`count(主键)`、`count(字段)`、`count(1)`
5. count(主键)
   - InnoDB 引擎会遍历整张表，把每一行的主键 id 值都取出来，返回服务层。服务层拿到主键后，直接按行进行累加(主键不可能为 null)
6. count(字段)
   - 没有 not null 约束: InnoDB 引擎会遍历整张表把每一行的字段值都取出来，返回服务层，服务层判断是否为 null，不为 null，技术累加
   - 有 not null 约束: InnoDB 引擎会遍历整张表把每一行的字段值都取出来，返回给服务层，直接按行进行累加
7. count(1)
   - InnoDB 引擎遍历整张表，但不取值。服务层对于返回的每一行，放一个数字 1 进去，直接按行进行累加
8. count(*)
   - InnoDB 引擎并不会把全部字段取出来，而是专门做了优化，不取值，服务层直接按行进行累加
9. `按照效率排序的话，count(字段) < count(主键 id) < count(1) ≈ count(*)`，所以尽量使用 count(*)

## 3-7 update 优化

1. update 语句执行注意事项

   ```SQL
   # update 语句 ==> 行锁
   update student set no = '2000100100' where id = 1;
   
   # update 语句 ==> 行锁
   update student set no = '2000100105' where name = "韦一笑;
   ```

2. **InnoDB 的行锁是针对索引加的锁，不是针对记录加的锁，并且该索引不能失效，否则会从行锁升级为表锁，并发性能就会降低**

   

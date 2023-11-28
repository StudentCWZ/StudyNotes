# 第七讲 MySQL 管理

## 7.1 系统数据库

1. MySQL 数据库安装完成后，自带了四个数据库，具体作用如下:

   |       数据库       |                             含义                             |
   | :----------------: | :----------------------------------------------------------: |
   |       mysql        | 存储 MySQL 服务器正常运行所需要的各种信息(时区、主从、用户、权限等) |
   | information_schema | 提供了访问数据库元数据的各种表和视图，包含数据库、表、字段类型和访问权限等 |
   | performance_schema | 为了 MySQL 服务器运行时状态提供了一个底层监控功能，主要用于收集数据库的服务器性能参数 |
   |        sys         | 包含一系列方便 DBA 和开发人员利用 performance_schema 性能数据库进行性能调优和诊断的视图 |

## 7.2 常用工具

### 7.2.1 mysql

1. 该 mysql 不是指 mysql 服务，而是指 mysql 的客户端工具

2. 语法

   ```bash
   mysql [options] [database]
   ```

3. 选项

   ```
   -u, --user=name # 指定用户名
   -p, --password[=name] # 指定密码
   -h, --host=name # 指定服务器 IP 或域名
   -p, --port=port # 指定连接端口
   -e, --excute=name # 执行 MySQL 语句并退出
   ```

4. `-e` 选项可以在 mysql 客户端执行 SQL 语句，而不用连接到 mysql 数据库再执行，对于一些批处理脚本，这种方式尤其方便

   ```bash
   mysql -uroot -p123456 db01 -e 'select * from stu';
   ```

### 7.2.2 mysqladmin

1. mysqladmin 是一个执行管理操作的客户端程序。可以用它来检查服务器的配置的当前状态、创建并删除数据库等

2. 通过帮助文档查看选项

   ```mysql
   mysqladmin –-help
   ```

3. 示例

   ```bash
   mysqladmin -uroot -p123456 version
   mysqladmin -uroot -p123456 create db02
   mysqladmin -uroot -p123456 -e 'show databases'
   ```

### 7.2.4 mysqlbinlog

1. 由于服务器生成的二进制日志文件以二进制格式保存，所以如果想要检查这些文本的文本格式，就会使用到 mysqlbinlog 日志管理工具

2. 语法

   ```bash
   mysqlbinlog [options] log-files1 log-files2
   ```

3. 选项

   ```bash
   -d, --database=name # 指定数据库名称，只列出指定的数据相关操作
   -o, --offset        # 忽略掉日志中的前 n 行命令
   -r, --result		# 将输出的文本格式日志输出到指定文件
   -s, --sort-form		# 显示简单的格式，省略掉一些信息
   --start-datatime=date1 --stop-datetime=date2 # 指定日期间隔内的所有日志
   --start-position=pos1 --stop-datetime=pos2 # 指定位置间隔内的所有日志
   ```

### 7.2.5 mysqlshow

1. mysqlshow 客户端对象查找工具，用来很快地查找存在哪些数据库，数据库中的表、表中的列或者索引

2. 语法

   ```SQL
   mysqlshow [options] [db_name[table_name[col_bname]
   ```

3. 示例

   ```
   # 查询每个数据库的表数量及表中记录的数量
   mysqlshow -uroot -p1234 --count
   
   # 查询 test 库中每个表中的字段数，及行数
   mysqlshow -uroot -p1234 test --count
   
   # 查询 test 库中 book 表中的详细情况
   mysqlshow -uroot -p1234 test book --count
   ```

### 7.2.6 mysqldump

1. mysqldump 客户端工具用来备份数据库或在不同数据库之间进行数据迁移。备份内容包括创建表，及插入表的 SQL 语句

2. 语法

   ```bash
   mysqldump [options] db_name [tables]
   mysqldump [options] --database/-B db1 [db2 db3 ...]
   mysqldump [options] --all-databases/-A
   ```

3. 连接选项

   ```bash
   -u, --user=name		# 指定用户名
   -p, --password		# 指定密码
   -h, --host=name		# 指定服务器 IP 或域名
   -p，--port          # 指定链接端口
   ```

4. 输出选项

   ```bash
   --add-drop-database		# 在每个数据库创建语句前加入 drop database 语句
   --add-drop-table		# 在每个表创建语句前加入 drop tables 语句，默认开启，不开启(--skip-add-drop-table)
   -n, --no-create-db		# 不包含数据表的创建语句
   -t, --no-create-info	# 不包含数据表的创建语句
   -d--no-data				# 不包含数据
   -T, --tab=name			# 自动生成两个文件，一个 sql 文件，创建表结构的语句，一个 txt 文件，数据文件
   ```

### 7.2.7 mysqlimport/source

1. mysqlimport 是客户端数据导入工具，用来导入 mysqldump 加 -T 参数后导出的文本文件

2. 语法

   ```bash
   mysqlimport [options] db_name textfile1 [textfile2]
   ```

3. 示例

   ```bash
   mysqlimport -uroot -p1234 test /tmp/city.txt
   ```

4. 如果需要导入 sql 文件，可以使用 mysql 中的 source 指令

   ```bash
   source /root/xxxxx.sql
   ```




### MySQL执行流程

#### 1. 基本流程图

![MySQL流程](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/MySQL%E6%B5%81%E7%A8%8B.png)

#### 2. 连接器

- 进行连接的命令`mysql -u root -p`

- MySQL显示当前有多少连接数：`show processlist`

  ```mysql
  mysql> show processlist;
  +----+-----------------+----------------+------+---------+------+------------------------+------------------+
  | Id | User            | Host           | db   | Command | Time | State                  | Info             |
  +----+-----------------+----------------+------+---------+------+------------------------+------------------+
  |  5 | event_scheduler | localhost      | NULL | Daemon  | 3209 | Waiting on empty queue | NULL             |
  |  8 | root            | localhost:6833 | stu  | Query   |    0 | init                   | show processlist |
  +----+-----------------+----------------+------+---------+------+------------------------+------------------+
  2 rows in set, 1 warning (0.00 sec)
  ```

- 查看最大连接时间`show varialbes like 'wait_timeout'`

  ```mysql
  mysql> show variables like 'wait_timeout';
  +---------------+-------+
  | Variable_name | Value |
  +---------------+-------+
  | wait_timeout  | 28800 |
  +---------------+-------+
  1 row in set, 1 warning (0.02 sec)
  ```

- 结束连接`kill connection +id`

  - 但是不能杀死`event_scheduler`的进程

    - `event_scheduler`是MySQL定时器的开关，在MySQL中用来执行定期任务的，当遇上主从库时，如果先建立主从库，在到MySQL中建立定期任务，则从库不会出现多次复制问题，如果先在主库建立定期任务，再建立从库，则会发生从库复制混乱的问题。参考资料：https://www.cnblogs.com/fander/p/10309687.html

    - 查看数据库定时器是否开关

      ```mysql
      mysql> show variables like '%event_scheduler%';
      +-----------------+-------+
      | Variable_name   | Value |
      +-----------------+-------+
      | event_scheduler | ON    |
      +-----------------+-------+
      1 row in set, 1 warning (0.01 sec)
      ```

    - 关闭定时器`set global event_scheduler = off`

      ```mysql
      mysql> show variables like '%event_scheduler%';
      +-----------------+-------+
      | Variable_name   | Value |
      +-----------------+-------+
      | event_scheduler | ON    |
      +-----------------+-------+
      1 row in set, 1 warning (0.01 sec)
      
      mysql> SET GLOBAL event_scheduler = OFF;
      Query OK, 0 rows affected (0.01 sec)
      
      mysql> show variables like '%event_scheduler%';
      +-----------------+-------+
      | Variable_name   | Value |
      +-----------------+-------+
      | event_scheduler | OFF   |
      +-----------------+-------+
      1 row in set, 1 warning (0.00 sec)
      
      mysql> show processlist;
      +----+------+----------------+------+---------+------+-------+------------------+
      | Id | User | Host           | db   | Command | Time | State | Info             |
      +----+------+----------------+------+---------+------+-------+------------------+
      |  9 | root | localhost:9838 | stu  | Query   |    0 | init  | show processlist |
      +----+------+----------------+------+---------+------+-------+------------------+
      1 row in set, 1 warning (0.00 sec)
      ```

      

- 查看MySQL最大连接数`show variables like 'max_connections'`

  ```mysql
  mysql> show variables like 'max_connections';
  +-----------------+-------+
  | Variable_name   | Value |
  +-----------------+-------+
  | max_connections | 151   |
  +-----------------+-------+
  1 row in set, 1 warning (0.01 sec)
  ```

#### 查询缓存

如果这条语句有过查询，并且数据库没有进行修改过，就直接在缓存里面查找并返回，MySQL8.0已经废弃。因为大多数时候缓存是查不到的。

#### 解析器

首先进行**词法分析**，找出关键字和非关键字，再进行**语法分析**，看是否满足MySQL的基本语法，如果满足则构建出语法树。如果不满足，就会进行语法报错。

```mysql
mysql> create table stu if not existe;
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'if not existe' at line 1
```

#### 执行器

**预处理器：**

- 检查 SQL 查询语句中的表或者字段是否存在；
- 将 `select *` 中的 `*` 符号，扩展为表上的所有列；

```mysql
mysql> select * from stu;
ERROR 1146 (42S02): Table 'stu.stu' doesn't exist
```

**优化器：**

将负责确认SQL使用查询方案的确定，使用`explain`可以查看SQL语句执行的计划

```mysql
mysql> explain select * from students where id =1;
+----+-------------+----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table    | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | students | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL  |
+----+-------------+----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```

当我们创建索引

```mysql
mysql> CREATE INDEX idx_name ON students (name);
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

再进行查询时,mysql会选择索引方案：

```mysql
mysql> explain select * from students where id >1 and name like '王%';
+----+-------------+----------+------------+-------+------------------+----------+---------+------+------+----------+-----------------------+
| id | select_type | table    | partitions | type  | possible_keys    | key      | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+----------+------------+-------+------------------+----------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | students | NULL       | range | PRIMARY,idx_name | idx_name | 206     | NULL |    1 |    80.00 | Using index condition |
+----+-------------+----------+------------+-------+------------------+----------+---------+------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
```

#### 执行器

执行器将于索引引擎进行交互，主要有三种方式

- 主键索引查询
- 全表扫描
- 索引下推

**主键索引查询：**

```mysql
select * from students where id =1;
```

当执行以上语句时，优化器决定了访问的类型为const，也就是使用主键索引查询一条记录。则执行器会执行以下内容：

- 先调用`read_first_record`,将`id=1`给存储引擎
- 存储引擎在B+树上查找记录，如果有则返回，如果不存在，则返回错误
- 执行器检查结果，如果符合查询条件则返回给客户端，如果不符合，则忽略
- 执行器的查询过程是一个while循环，执行器调用`read_record`函数，这个函数指向-1，查询结束

**全表扫描**

和主键索引查询不同的是，全表扫描，执行器的访问类型为ALL，所以，`read_record`函数将会指向第一次查询到的结果的下一条语句，直到全部查询完。然后进行返回。

**索引下推**

![image-20240113111115280](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/image-20240113111115280.png)

当进行联合索引时，查询到`id>1`后不立即进行回表，而是说判断`name like '王%'`是否满足，如果满足则进回表，并返回给上一层。

#### 回表

回表就是先通过普通索引扫描出数据所在的行，再通过行主键ID 取出索引中未包含的数据。如果select一次能查询出所有的数据，那么就不需要再进行回表。

**InnoDB的两大索引**

**聚集索引 （clustered index）**

InnoDB聚集索引的叶子节点存储行记录，因此， InnoDB必须要有且只有一个聚集索引。

- 如果表定义了主键，则Primary Key 就是聚集索引；
- 如果表没有定义主键，则第一个非空唯一索引（Not NULL Unique）列是聚集索引；
- 否则，InnoDB会创建一个隐藏的row-id作为聚集索引；

**普通索引（secondary index）**

普通索引也叫二级索引，除聚簇索引外的索引都是普通索引，即非聚簇索引。

InnoDB的普通索引叶子节点存储的是主键（聚簇索引）的值，而MyISAM的普通索引存储的是记录指针。

**索引存储结构：**

**聚簇索引**

id 是主键，所以是聚簇索引，其叶子节点存储的是对应行记录的数据。

![image-20240113120043853](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/image-20240113120043853.png)

**普通索引**

age是普通索引（二级索引），非聚簇索引的叶子节点存储的是聚簇索引的值，即主键ID的值。

![image-20240113120709997](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/image-20240113120709997.png)

**搜索过程：**

我们根据上表可以看到，当使用聚簇索引时，一次就可以返回全部内容，当使用普通索引时，需要找到主键，并且进行回表

**如何防止回表：**

使用覆盖索引：只需要在一棵索引树上就能获取SQL所需的所有列数据，无需回表查询。

**如何实现覆盖索引？**

常见的方法是将被查询的字段，建立到联合索引中。

解释性SQL的explain的输出结果Extra字段为Using index时表示触发了索引覆盖。

方式一、select语句中的内容要么是索引内容，要么是主键内容

```mysql
mysql> explain select id,age from students where age >20;
+----+-------------+----------+------------+-------+---------------+---------+---------+------+------+----------+--------------------------+
| id | select_type | table    | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+----------+------------+-------+---------------+---------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | students | NULL       | range | idx_age       | idx_age | 4       | NULL |    3 |   100.00 | Using where; Using index |
+----+-------------+----------+------------+-------+---------------+---------+---------+------+------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)
```

![image-20240113121652636](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/image-20240113121652636.png)

方式二、构建组合索引

```mysql
mysql> create index idx_name_idx_age on students(`age`,`name`);
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> show index from students;
+----------+------------+------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| Table    | Non_unique | Key_name         | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Visible | Expression |
+----------+------------+------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| students |          0 | PRIMARY          |            1 | id          | A         |           5 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| students |          1 | idx_name_idx_age |            1 | age         | A         |           5 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| students |          1 | idx_name_idx_age |            2 | name        | A         |           5 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
+----------+------------+------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
3 rows in set (0.00 sec)

mysql> explain select id,name,age from students where age>20;
+----+-------------+----------+------------+-------+------------------+------------------+---------+------+------+----------+--------------------------+
| id | select_type | table    | partitions | type  | possible_keys    | key              | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+----------+------------+-------+------------------+------------------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | students | NULL       | range | idx_name_idx_age | idx_name_idx_age | 4       | NULL |    3 |   100.00 | Using where; Using index |
+----+-------------+----------+------------+-------+------------------+------------------+---------+------+------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)

```

![image-20240113122948959](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/image-20240113122948959.png)

**索引覆盖优化的场景**

全表count查询的优化，当count查询一个用了索引的列时，会使用索引。

![image-20240113123722254](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/image-20240113123722254.png)

**列查询回表优化**

使用前文的联合索引。

**分页查询优化**

![image-20240113124615436](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/image-20240113124615436.png)

展示表信息的几种方式：
```mysql
mysql> DESCRIBE students;
+------------+-----------------+------+-----+---------+----------------+
| Field      | Type            | Null | Key | Default | Extra          |
+------------+-----------------+------+-----+---------+----------------+
| id         | int             | NO   | PRI | NULL    | auto_increment |
| name       | varchar(50)     | NO   | MUL | NULL    |                |
| gender     | enum('男','女') | NO   |     | NULL    |                |
| age        | int             | NO   | MUL | NULL    |                |
| student_id | varchar(20)     | NO   | UNI | NULL    |                |
+------------+-----------------+------+-----+---------+----------------+
5 rows in set (0.01 sec)

mysql> SHOW TABLE STATUS LIKE 'students';
+----------+--------+---------+------------+------+----------------+-------------+-----------------+--------------+-----------+----------------+---------------------+---------------------+------------+--------------------+----------+----------------+---------+
| Name     | Engine | Version | Row_format | Rows | Avg_row_length | Data_length | Max_data_length | Index_length | Data_free | Auto_increment | Create_time
| Update_time         | Check_time | Collation          | Checksum | Create_options | Comment |
+----------+--------+---------+------------+------+----------------+-------------+-----------------+--------------+-----------+----------------+---------------------+---------------------+------------+--------------------+----------+----------------+---------+
| students | InnoDB |      10 | Dynamic    |    5 |           3276 |       16384 |               0 |        16384 |         0 |              6 | 2024-01-13 10:50:45 | 2024-01-13 10:38:32 | NULL       | utf8mb4_0900_ai_ci |     NULL |                |         |
+----------+--------+---------+------------+------+----------------+-------------+-----------------+--------------+-----------+----------------+---------------------+---------------------+------------+--------------------+----------+----------------+---------+
1 row in set (0.01 sec)

mysql> -- 查看表详细信息
mysql> SHOW FULL COLUMNS FROM students;
+------------+-----------------+--------------------+------+-----+---------+----------------+---------------------------------+---------+
| Field      | Type            | Collation          | Null | Key | Default | Extra          | Privileges                      | Comment |
+------------+-----------------+--------------------+------+-----+---------+----------------+---------------------------------+---------+
| id         | int             | NULL               | NO   | PRI | NULL    | auto_increment | select,insert,update,references |         |
| name       | varchar(50)     | utf8mb4_0900_ai_ci | NO   | MUL | NULL    |                | select,insert,update,references |         |
| gender     | enum('男','女') | utf8mb4_0900_ai_ci | NO   |     | NULL    |                | select,insert,update,references |         |
| age        | int             | NULL               | NO   | MUL | NULL    |                | select,insert,update,references |         |
| student_id | varchar(20)     | utf8mb4_0900_ai_ci | NO   | UNI | NULL    |                | select,insert,update,references |         |
+------------+-----------------+--------------------+------+-----+---------+----------------+---------------------------------+---------+
5 rows in set (0.00 sec)

mysql> show index from students;
+----------+------------+------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| Table    | Non_unique | Key_name   | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Visible | Expression |
+----------+------------+------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| students |          0 | PRIMARY    |            1 | id          | A         |           5 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| students |          0 | student_id |            1 | student_id  | A         |           5 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| students |          1 | idx_age    |            1 | age         | A         |           5 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| students |          1 | idx_name   |            1 | name        | A         |           5 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
+----------+------------+------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
4 rows in set (0.01 sec)
```



问题：

```mysql
mysql> explain select * from students where gender ='男';
+----+-------------+----------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table    | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+----------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | students | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    5 |    50.00 | Using where |
+----+-------------+----------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

mysql>
mysql> EXPLAIN SELECT * FROM students WHERE age = 20;
+----+-------------+----------+------------+------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table    | partitions | type | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+----------+------------+------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | students | NULL       | ref  | idx_age       | idx_age | 4       | const |    1 |   100.00 | NULL  |
+----+-------------+----------+------------+------+---------------+---------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)

mysql> EXPLAIN SELECT * FROM students WHERE age > 20;
+----+-------------+----------+------------+-------+---------------+---------+---------+------+------+----------+-----------------------+
| id | select_type | table    | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+----------+------------+-------+---------------+---------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | students | NULL       | range | idx_age       | idx_age | 4       | NULL |    3 |   100.00 | Using index condition |
+----+-------------+----------+------------+-------+---------------+---------+---------+------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
```

以上三种查询中是否使用了回表

- 不唯一值当索引，它的查询过程是怎么样子的？

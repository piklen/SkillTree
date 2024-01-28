## MySQL一行记录是怎么存储的？

#### 常见的面试问题有：

- MySQL 的 NULL 值会占用空间吗？
- MySQL 怎么知道 varchar(n) 实际占用数据的大小？
- varchar(n) 中 n 最大取值为多少？
- 行溢出后，MySQL 是怎么处理的？
- MySQL中的Null值是怎么存放的？

#### MySQL中数据存放在哪？

MySQL中的数据是存储在磁盘上的，一般可以通过`SHOW VARIABLES LIKE 'datadir'; `

进行查看

![image-20240128172424423](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/image-20240128172424423.png)

`.idb`文件存储的是表数据。

#### 表空间文件的结构组成

表空间由4部分组成：行->页->区->段。

数据存储的最小单位是行，16KB组成一个页，MySQL中如果使用的是innoDB的话，读取数据时按页进行读写的。

当数据量大时，为某个索引分配空间时则不是按照页来分配，而是按照区来分配，一个区大小为1MB,可以分配64个大小为16KB的连续的页。

多个区组成一个段，段一般分为数据段、索引段和回滚段等。

- 索引段：存放 B + 树的非叶子节点的区的集合；
- 数据段：存放 B + 树的叶子节点的区的集合；
- 回滚段：存放的是回滚数据的区的集合， MVCC 利用了回滚段实现了多版本查询数据。

![表空间结构](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/%E8%A1%A8%E7%A9%BA%E9%97%B4%E7%BB%93%E6%9E%84.png)

#### InnoDB 行格式有哪些？

| 行格式     | 紧凑存储 | 增强可变长度列存储 | 大索引键前缀 | 压缩支持 | 支持的表空间类型                | 所需文件格式          |
| ---------- | -------- | ------------------ | ------------ | -------- | ------------------------------- | --------------------- |
| REDUNDANT  | 否       | 否                 | 否           | 否       | system, file-per-table, general | Antelope or Barracuda |
| COMPACT    | 是       | 否                 | 否           | 否       | system, file-per-table, general | Antelope or Barracuda |
| DYNAMIC    | 是       | 是                 | 是           | 否       | system, file-per-table, general | Barracuda             |
| COMPRESSED | 是       | 是                 | 是           | 是       | file-per-table, general         | Barracuda             |

REDUNDANT与COMPACT  的结构区别是，COMPACT将NULL值表单独列出来了，而且COMPACT少了少了n_field和1byte_offs_flag两个字段。

REDUNDANT：

![REDUNDANT](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/1.png)

COMPACT：

<img src="https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/COMPACT.png" alt="COMPACT" />

- DB_ROW_ID(row_id) : 当表没有定义主键，则选择unique键作为主键，如果仍没有，则默认添加一个名为DB_ROW_ID的隐藏列作为主键，占用6个字节。也就是说这个列只有当没有主键也没有唯一索引时才存在，当成是**隐形的主键**

- DB_TRX_ID(transaction_id): 事务id,占用6字节

- DB_ROLL_PTR(roll_pointer): 占用7个字节,回滚指针(后面MVCC的时候会用到)

#### MySQL 怎么知道 varchar(n) 实际占用数据的大小？

MySQL通过读取**变长字段长度列表**来获取varchar(n)的实际占用大小。

在innoDB中变长字段长度列表中是逆序存储变长的列值的长度的，且不存储NULL值。

#### MySQL中的Null值是怎么存放的？

MySQL中的NULL值存放在null值列表中。

- 二进制位的值为`1`时，代表该列的值为NULL。
- 二进制位的值为`0`时，代表该列的值不为NULL。
- ULL 值列表必须用整数个字节的位表示（1字节8位），如果使用的二进制位个数不足整数个字节，则在字节的高位补 `0`。
- 存储过程也是逆序存储的。与列的存储顺序相反

#### NULL值列表与变长字段长度列表是必须的吗？

这两者都不是必须：

- 当表结构不存在变长直段如：`VAARCHAR `、`VARBINARY`、`BLOB`、`TEXT`等变长字段，那就不存在变长字段长度列表。
- 当表不为不存在可为NULL值的字段是，则NULL值列表不存在。

#### MySQL 的 NULL 值会占用空间吗？

NULL不会存储在行格式中的真实数据部分，NULL值列表最少会占用一字节的空间。

#### varchar(n) 中 n 最大取值为多少？

MySQL 规定除了 TEXT、BLOBs 这种大对象类型之外，其他所有的列（不包括隐藏列和记录头信息）占用的字节长度加起来不能超过 65535 个字节。

同时要考虑到是否为空值、编码类型、变长字段长度列表的空间。

####  行溢出后，MySQL 是怎么处理的？

MySQL 中磁盘和内存交互的基本单位是页，一个页的大小一般是 `16KB`，也就是 `16384字节`，而一个 varchar(n) 类型的列最多可以存储 `65532字节`，当一个页存不了一条数据，就会发生行溢出，多的数据会存储到溢出页中。

在Compact行格式中先存储部分数据，然后用20个字节存储溢出页的地址

![行溢出](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/%E8%A1%8C%E6%BA%A2%E5%87%BA.png)

在Dynamic中记录的真实数据处不会存储该列的一部分数据，只存储 20 个字节的指针来指向溢出页。

![行溢出2](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/%E8%A1%8C%E6%BA%A2%E5%87%BA2.png)

---

参考资料：
 [MySQL 一行记录是怎么存储的？](https://xiaolincoding.com/mysql/base/row_format.html)

[MySQL的一行数据是如何存储的](https://juejin.cn/post/7097040621210173454)


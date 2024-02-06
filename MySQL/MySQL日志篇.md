### MySQL日志篇

####  undo log（回滚日志）、redo log（重做日志） 、binlog （归档日志）基本作用是什么？

- undo log（回滚日志）：是 Innodb 存储引擎层生成的日志，实现了事务中的原子性，主要用于事务回滚和 MVCC。
- redo log（重做日志）：是 Innodb 存储引擎层生成的日志，实现了事务中的持久性，主要用于掉电等故障恢复；
- binlog （归档日志）：是 Server 层生成的日志，主要用于数据备份和主从复制；

#### undo log

##### 为什么需要undo log?

当我执行增删改时，就算没有输入begin开启事务，MySQL也会自动的隐式开启事务。执行完语句后就自动提交事务了。

当在执行一条语句过程中，如果MySQL突然崩溃。我们就需要undo log 来回滚事务。从而保证了事务的原子性。

undo log 的基本执行流程如下：

![1707210855761](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/1707210855761.png)

##### undo log 是怎么操作的？

- 当事务需要插入一条数据时，我们把这条记录的主键值记录下来，回滚时，我们把这个主键对应的记录删除即可。
- 当事务删除一条记录时，我们把这条记录的内容都记下来，回滚时，我们把这条记录插入到表中。
- 当更新一条数据时，我们把原来的数值保存下来，回滚时，更新为原来的数据就好了。

不同的事务操作产生的undo log 日志格式不一样。

更新事务产生的 undo log 格式都有一个 roll_pointer 指针和一个 trx_id 事务id：

- 通过 trx_id 可以知道该记录是被哪个事务修改的；
- 通过 roll_pointer 指针可以将这些 undo log 串成一个链表，这个链表就被称为版本链；

![image-20240206173506308](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/image-20240206173506308.png)

##### undo log 的基本作用是什么？

- 实现事务回滚，保障事务的原子性。
- 实现MVCC关键因素之一。ReadView+undo log 可以实现MVCC。

#### 缓冲池（Buffer Pool）

为了提高数据库的读写性能，InnoDB存储引擎设计了缓冲池，当查询、修改的数据在缓冲池中时，我们直接对缓冲池中的数据进行查询和修改。

##### 缓冲池中包含什么内容？

![image-20240206175109343](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/image-20240206175109343.png)

**查询一条记录，就只需要缓冲一条记录吗？**

并不是。它会直接把一整个页加载到缓冲池中，然后再通过页目录去查找值。

#### redo log

##### 为什么需要redo log?

因为Buffer Pool是基于内存的，当断电时，数据就会丢失。所以当我们修改缓存中的数据时，我们需要用redo log 来保存胀页里面的数据。

之后，InnoDB 引擎会在适当的时候，由后台线程将缓存在 Buffer Pool 的脏页刷新到磁盘里，这就是 **WAL （Write-Ahead Logging）技术**。

**WAL 技术指的是， MySQL 的写操作并不是立刻写到磁盘上，而是先写日志，然后在合适的时间再写到磁盘上**。

过程如下：
![Clip_2024-02-06_18-02-03](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/Clip_2024-02-06_18-02-03.png)

##### 什么是redo log?

redo log 是物理日志，记录了某个数据页做了什么修改。比如**对 XXX 表空间中的 YYY 数据页 ZZZ 偏移量的地方做了AAA 更新**，每当执行一个事务就会产生这样的一条或者多条物理日志。当系统崩溃时，因为redo log 做了持久化，所以可以根据redo log 将所有数据恢复到最新的状态。

##### redo log 与undo log的区别是什么？

- redo log 记录了此次事务「**完成后**」的数据状态，记录的是更新**之后**的值；
- undo log 记录了此次事务「**开始前**」的数据状态，记录的是更新**之前**的值；

事务提交之前发生了崩溃，重启后会通过 undo log 回滚事务，事务提交之后发生了崩溃，重启后会通过 redo log 恢复事务，如下图：

![Clip_2024-02-06_19-53-10](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/Clip_2024-02-06_19-53-10.png)

##### 为什么需要redo log？

- **实现事务的持久性，让 MySQL 有 crash-safe（崩溃恢复） 的能力**，能够保证 MySQL 在任何时间段突然崩溃，重启后之前已提交的记录都不会丢失；
- **将写操作从「随机写」变成了「顺序写」**，提升 MySQL 写入磁盘的性能。

##### redo log是直接写入磁盘的吗？

并不是，redo log 有它自己的缓存，也就是redo log buffer,当产生一条redo log时，先写入缓存中，后续再持久化到磁盘中。

![事务恢复](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/redologbuf.webp)

##### redo log buffer什么时候写入到磁盘中？

- MySQL 正常关闭时；
- 当 redo log buffer 中记录的写入量大于 redo log buffer 内存空间的一半时，会触发落盘；
- InnoDB 的后台线程每隔 1 秒，将 redo log buffer 持久化到磁盘。
- 每次事务提交时都将缓存在 redo log buffer 里的 redo log 直接持久化到磁盘。

可以设置`innodb_flush_log_at_trx_commit`的值来控制写入磁盘的时机：

![img](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/innodb_flush_log_at_trx_commit2.drawio.png)

##### redo log 是怎么保存更迭保存的？

一般情况下，InnoDB存储引擎有1个重做日志文件组（redo log Group),它有两个文件，然后日志是循环写入这两个组的：

![重做日志文件组](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/%E9%87%8D%E5%81%9A%E6%97%A5%E5%BF%97%E6%96%87%E4%BB%B6%E7%BB%84.drawio.png)

![重做日志文件组写入过程](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/%E9%87%8D%E5%81%9A%E6%97%A5%E5%BF%97%E6%96%87%E4%BB%B6%E7%BB%84%E5%86%99%E5%85%A5%E8%BF%87%E7%A8%8B.drawio.png)

redo log 是循环写的方式，相当于一个环形，InnoDB 用 write pos 表示 redo log 当前记录写到的位置，用 checkpoint 表示当前要擦除的位置，如下图：

![img](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/checkpoint.png)

- write pos 和 checkpoint 的移动都是顺时针方向；
- write pos ～ checkpoint 之间的部分（图中的红色部分），用来记录新的更新操作；
- check point ～ write pos 之间的部分（图中蓝色部分）：待落盘的脏数据页记录；

擦除的过程就是写入磁盘的过程。

#### binlog

binlog只记录表结构和表数据修改，不会记录查询类操作。

##### redo log 与binlog 有什么区别？

- binlog是MySQL的server层实现的，所有存储引擎都可以用，redo log的innoDB引擎的特色。
- binlog有三种模式statement：就是每一条SQL语句都被记录下来，一般被称为逻辑日志，但是会在动态函数失效，比如说uuid,now。ROW：记录的是数据最终被修改成了什么样子，会使得日志文件过大。MIXED:包含以上两种模式，自动选择。
- redo log是物理日志，记录的是对数据页的修改。
- binlog使用的是追加的方式，redo log是循环写。
- binlog 用于备份恢复、主从复制，redo log 用于掉电等故障恢复。
- binlog可以完整恢复数据库，但是redo log 不行，因为redo log 是循环写。

#### MySQL主从复制是怎么实现的？

主从复制依赖于binlog，主数据库把binlog发送到从数据库，然后从数据库再单独开线程读取relay log 中的内容。如下图：

![MySQL 主从复制过程](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6%E8%BF%87%E7%A8%8B.drawio.png)

当我们有了主从数据库时，我们就可以使得读写进行分离，写操作用主数据库，读操作用从数据库。

![MySQL 主从架构](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/%E4%B8%BB%E4%BB%8E%E6%9E%B6%E6%9E%84.drawio.png)

一般情况下数据库的设置为一个主库一般跟 2～3 个从库（1 套数据库，1 主 2 从 1 备主）。

##### MySQL主从复制的基本模型

- 同步复制：复制到从数据库，再把结果返回给客户端，性能很差，可用性也很差。
- 异步复制：MySQL主库不会等binlog同步到各个从库，就直接返回到客户端。有数据丢失的风险。
- 半同步复制：有部分从库同步好了数据，就可以返回成功的结果了，这样不会丢失数据。

##### binlog 什么时候写入磁盘？

刚开始时，MySQL会给每个事务都开一片内存用于缓存binlog,当满了时，就需要暂存到磁盘。

binlog写入磁盘的过程如下：
![binlog cach](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/binlogcache.drawio.png)

MySQL可以使用`sync_binlog`来控制写入的频率。当值为0时（也就是默认），每次提交事务都只 write，不 fsync，后续交由操作系统决定何时将数据持久化到磁盘；sync_binlog = 1 的时候，表示每次提交事务都会 write，然后马上执行 fsync；sync_binlog =N(N>1) 的时候，表示每次提交事务都 write，但累积 N 个事务后才 fsync。

一般会 sync_binlog 设置为 100~1000 中的某个数值。

#### 一般事务的两阶段提交的过程是怎么样的？

在 MySQL 的 InnoDB 存储引擎中，开启 binlog 的情况下，MySQL 会同时维护 binlog 日志与 InnoDB 的 redo log，为了保证这两个日志的一致性，MySQL 使用了内部 XA 事务。

客户端执行 commit 语句或者在自动提交的情况下，MySQL 内部开启一个 XA 事务，**分两阶段来完成 XA 事务的提交**，如下图：

![Clip_2024-02-07_00-38-41](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/Clip_2024-02-07_00-38-41.png)

- **prepare 阶段**：将 XID（内部 XA 事务的 ID） 写入到 redo log，同时将 redo log 对应的事务状态设置为 prepare，然后将 redo log 持久化到磁盘（innodb_flush_log_at_trx_commit = 1 的作用）；
- **commit 阶段**：把 XID 写入到 binlog，然后将 binlog 持久化到磁盘（sync_binlog = 1 的作用），接着调用引擎的提交事务接口，将 redo log 状态设置为 commit，此时该状态并不需要持久化到磁盘，只需要 write 到文件系统的 page cache 中就够了，因为只要 binlog 写磁盘成功，就算 redo log 的状态还是 prepare 也没有关系，一样会被认为事务已经执行成功；

#### 出现异常重启，MySQL会怎么进行操作？

![Clip_2024-02-07_00-40-56](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/Clip_2024-02-07_00-40-56.png)

- 时刻A时，redo log已经写入了，但是binlog 没有写入，所以回滚事务。
- 时刻B时，redo log 、binlog都已经写入了，所以提交事务。

**以 binlog 写成功为事务提交成功的标识**

#### 两阶段提交有什么问题？

- 磁盘I/O次数高。因为每次事务都会造成redo log、binlog的写入磁盘的操作。
- 锁竞争激烈。在一个事务获取到锁时才能进入 prepare 阶段，一直到 commit 阶段结束才能释放锁，下个事务才可以继续进行 prepare 操作。

##### 怎么去解决？

**组提交**：MySQL 引入了 binlog 组提交（group commit）机制，当有多个事务提交的时候，会将多个 binlog 刷盘操作合并成一个，从而减少磁盘 I/O 的次数。

- **flush 阶段**：多个事务按进入的顺序将 binlog 从 cache 写入文件（不刷盘）；
- **sync 阶段**：对 binlog 文件做 fsync 操作（多个事务的 binlog 合并一次刷盘）；
- **commit 阶段**：各个事务按顺序做 InnoDB commit 操作；

**每个阶段都有一个队列**，每个阶段有锁进行保护，保证了事务的写入顺序。

![每个阶段都有一个队列](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/commit_4.png)

**MySQL5.7开始，redo log 也有组提交了**，
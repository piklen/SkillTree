### MySQL内存篇

#### 为什么需要Buffer Pool?

为了提高性能减少读取磁盘数据的次数，提高读写性能。

MySQL结构如图所示：

![img](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/%E7%BC%93%E5%86%B2%E6%B1%A0.drawio.png)

有了缓冲池后读取数据和修改数据都优先查找缓冲池里面的。

#### buffer pool如何设置大小？

默认情况下，MySQL设置的buffer pool 大小为128MB。

可以设置 `innodb_buffer_pool_size` 参数来设置 Buffer Pool 的大小，一般建议设置成可用物理内存的 60%~80%。

#### buffer pool的基本结构是怎么样的？

![img](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/bufferpool%E5%86%85%E5%AE%B9.drawio.png)

为了更好的管理这些在 Buffer Pool 中的缓存页，InnoDB 为每一个缓存页都创建了一个**控制块**，控制块信息包括（存页的表空间、页号、缓存页地址、链表节点）

![img](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/%E7%BC%93%E5%AD%98%E9%A1%B5.drawio.png)

#### 如何管理buffer pool?

**空闲页：增加**一个free链表，来快速找到空闲页。结构如下：

![img](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/freelist.drawio.png)

**脏页**：设计出flush链表来标记脏页。结构如下：
![img](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/Flush.drawio.png)

如何提高缓存命中率？

> 使用LRU算法来提升缓存的命中率。

那么buffer pool的样式将如下图所示：
![img](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/bufferpoll_page.png)

图中：

- Free Page（空闲页），表示此页未被使用，位于 Free 链表；
- Clean Page（干净页），表示此页已被使用，但是页面未发生修改，位于LRU 链表。
- Dirty Page（脏页），表示此页「已被使用」且「已经被修改」，其数据和磁盘上的数据已经不一致。当脏页上的数据写入磁盘后，内存数据和磁盘数据一致，那么该页就变成了干净页。脏页同时存在于 LRU 链表和 Flush 链表。

**但是MySQL并没有使用简单的LRU,原因如下：**

- 预读失效；MySQL 在加载数据页时，会提前把它相邻的数据页一并加载进来，目的是为了减少磁盘 IO。但是可能这些**被提前加载进来的数据页，并没有被访问**，相当于这个预读是白做了，这个就是**预读失效**。使用简单的LRU算法可能造成，预读页一直不会被访问。
- Buffer Pool 污染；当查询的数据量极大时，buffer pool的空间很小时，导致大量热数据被淘汰。从而需要大量的磁盘I/O。导致性能急剧下降。

**如何进行改进预读失效：**

让预读的页停留在 Buffer Pool 里的时间要尽可能的短，让真正被访问的页才移动到 LRU 链表的头部，从而保证真正被读取的热数据留在 Buffer Pool 里的时间尽可能长。

MySQL设置了两个区域，old和young区域。结构如下图所示：
![img](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/young%252Bold.png)

可以通过 `innodb_old_blocks_pct` 参数来设置。

**预读的页就只需要加入到 old 区域的头部，当页被真正访问的时候，才将页插入 young 区域的头部**。

##### 如何解决Buffer Pool污染问题？

提高进入young区域的门槛。进入到 young 区域条件增加了一个**停留在 old 区域的时间判断**。

- 如果后续的访问时间与第一次访问的时间**在某个时间间隔内**，那么**该缓存页就不会被从 old 区域移动到 young 区域的头部**；
- 如果后续的访问时间与第一次访问的时间**不在某个时间间隔内**，那么**该缓存页移动到 young 区域的头部**；

这个间隔时间是由 `innodb_old_blocks_time` 控制的，默认是 1000 ms。

也就说，**只有同时满足「被访问」与「在 old 区域停留时间超过 1 秒」两个条件，才会被插入到 young 区域头部**，这样就解决了 Buffer Pool 污染的问题 。

#### 脏页什么时候被写入磁盘？

- 当 redo log 日志满了的情况下，会主动触发脏页刷新到磁盘；
- Buffer Pool 空间不足时，需要将一部分数据页淘汰掉，如果淘汰的是脏页，需要先将脏页同步到磁盘；
- MySQL 认为空闲时，后台线程会定期将适量的脏页刷入到磁盘；
- MySQL 正常关闭之前，会把所有的脏页刷入到磁盘；



#### 参考资料

> [揭开 Buffer Pool 的面纱](https://xiaolincoding.com/mysql/buffer_pool/buffer_pool.html)
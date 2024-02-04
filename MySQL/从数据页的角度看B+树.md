### 从数据页的角度看B+树

#### InnoDB 是如何存储数据的？

- InnoDB 的数据是按「数据页」为单位来读写的,InnoDB 数据页的默认大小是 16KB。

- 数据页主要包括以下几个方面的内容：

  ![图片](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/243b1466779a9e107ae3ef0155604a17.png)

![图片](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/fabd6dadd61a0aa342d7107213955a72-1707035664756-5.png)

File Header 中有两个指针，分别指向上一个数据页和下一个数据页，连接起来的页相当于一个双向的链表，如下图：

![图片](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/557d17e05ce90f18591c2305871af665.png)

数据页中有目录页，目录页中按照槽来分配，槽指向的是分组的最后一个节点的位置。

InnoDB中数据页内容如下：
![图片](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/261011d237bec993821aa198b97ae8ce.png)

通过二分查找对数据页内容进行查找。

#### B+树是如何进行查询的？

数据页样式如下：
![图片](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/7c635d682bd3cdc421bb9eea33a5a413.png)

B+树有以下特点：

- 只有叶子节点（最底层的节点）才存放了数据，非叶子节点（其他上层节）仅用来存放目录项作为索引。
- 非叶子节点分为不同层次，通过分层来降低每一层的搜索量；
- 所有节点按照索引键大小排序，构成一个双向链表，便于范围查询；

B+树通过二分查找进行内容的查询。

#### 聚簇索引和二级索引

- 聚簇索引的叶子节点存放的是实际数据，所有完整的用户记录都存放在聚簇索引的叶子节点；
- 二级索引的叶子节点存放的是主键值，而不是实际数据。
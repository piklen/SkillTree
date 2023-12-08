### SDS数据结构

1. **基本数据结构**

```C
struct sdshdr {
    //记录buf数组中已使用字节的数量
    //等于SDS所保存字符串的长度
    int len;
    //记录buf数组中未使用字节的数量
    int free;
    //字节数组，用于保存字符串
    char buf[];
};
```

SDS遵循C字符串以空字符结尾的惯例，保存空字符的1字节空间不计算在SDS的len属性里面，并且为空字符分配额外的1字节空间，以及添加空字符到字符串末尾等操作，

除了用来保存数据库中的字符串值之外， SDS 还被用作缓冲区（buffer）： AOF 模块中的 AOF 缓冲区， 以及客户端状态中的输入缓冲区， 都是由 SDS 实现的， 在之后介绍 AOF 持久化和客户端状态的时候， 我们会看到 SDS 在这两个模块中的应用。

1. **特性：**

- 常数复杂度获取字符串长度

    在len属性中记录了SDS本身的长度，所以获取一个SDS长度的复杂度仅为O（1）。

- 杜绝缓冲区溢出

    当SDS API需要对SDS进行修改时，API会先检查SDS的空间是否满足修改所需的要求，如果不满足的话，API会自动将SDS的空间扩展至执行修改所需的大小，然后才执行实际的修改操作，所以使用SDS既不需要手动修改SDS的空间大小，也不会出现前面所说的缓冲区溢出问题。

- 减少修改字符串时带来的内存重分配次数

    通过未使用空间，SDS实现了空间预分配和惰性空间释放两种优化策略。

- 空间预分配用于优化SDS的字符串增长操作

    当SDS的API对一个SDS进行修改，并且需要对SDS进行空间扩展的时候，程序不仅会为SDS分配修改所必须要的空间，还会为SDS分配额外的未使用空间。

    如果对SDS进行修改之后，SDS的长度（也即是len属性的值）将小于1MB，那么程序分配和len属性同样大小的未使用空间，这时SDS len属性的值将和free属性的值相同。

    如果对SDS进行修改之后，SDS的长度将大于等于1MB，那么程序会分配1MB的未使用空间。

- 惰性空间释放用于优化SDS的字符串缩短操作

    当SDS的API需要缩短SDS保存的字符串时，程序并不立即使用内存重分配来回收缩短后多出来的字节，而是使用free属性将这些字节的数量记录起来，并等待将来使用。

- 二进制安全

    SDS使用len属性的值而不是空字符来判断字符串是否结束 

- 兼容部分C字符串函数

    这些API总会将SDS保存的数据的末尾设置为空字符，并且总会在为buf数组分配空间时多分配一个字节来容纳这个空字符，这是为了让那些保存文本数据的SDS可以重用一部分＜string.h＞库定义的函数。

1. **SDS API**

![weread_image_65272724591657.jpeg](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/imgs/weread_image_65272724591657.jpeg)

### 链表

链表提供了高效的节点重排能力，以及顺序性的节点访问方式，并且可以通过增删节点来灵活地调整链表的长度。

除了链表键之外，发布与订阅、慢查询、监视器等功能也用到了链表，Redis服务器本身还使用链表来保存多个客户端的状态信息，以及使用链表来构建客户端输出缓冲区（output buffer）

1. **基本数据结构**

双向链表

```C
typedef struct listNode {
    // 前置节点
    struct listNode * prev;
    // 后置节点
    struct listNode * next;
    //节点的值
    void * value;
}listNode;
```

Redis链表

```C
typedef struct list {
    //表头节点，使用的结构是上边的双向链表
    listNode * head;
    //表尾节点
    listNode * tail;
    //链表所包含的节点数量
    unsigned long len;
    //节点值复制函数
    void *(*dup)(void *ptr);
    //节点值释放函数
    void (*free)(void *ptr);
    //节点值对比函数
    int (*match)(void *ptr,void *key);
} list;
```

![weread_image_158859762455173.jpeg](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/imgs/weread_image_158859762455173.jpeg)

1. **特性**
❑双端：链表节点带有prev和next指针，获取某个节点的前置节点和后置节点的复杂度都是O（1）。
❑无环：表头节点的prev指针和表尾节点的next指针都指向NULL，对链表的访问以NULL为终点。
❑带表头指针和表尾指针：通过list结构的head指针和tail指针，程序获取链表的表头节点和表尾节点的复杂度为O（1）。
❑带链表长度计数器：程序使用list结构的len属性来对list持有的链表节点进行计数，程序获取链表中节点数量的复杂度为O（1）。
❑多态：链表节点使用void*指针来保存节点值，并且可以通过list结构的dup、free、match三个属性为节点值设置类型特定函数，所以链表可以用于保存各种不同类型的值。

2. **链表API**

![weread_image_159141803762878.jpeg](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/imgs/weread_image_159141803762878.jpeg)

### **字典** （hashtable)

字典，又称为符号表（symbol table）、关联数组（associative array）或映射（map），是一种用于保存键值对（key-value pair）的抽象数据结构。

1. **基本数据结构**

```C
typedef struct dictEntry {
    //键
    void *key;
    //值
    union{
        void *val;
        uint64_tu64;
        int64_ts64;
    } v;
    //指向下个哈希表节点，形成链表
    struct dictEntry *next;
} dictEntry;
```

```C
typedef struct dictType {
    //计算哈希值的函数
    unsigned int (*hashFunction)(const void *key);
    //复制键的函数
    void *(*keyDup)(void *privdata, const void *key);
    //复制值的函数
    void *(*valDup)(void *privdata, const void *obj);
    //对比键的函数
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    //销毁键的函数
    void (*keyDestructor)(void *privdata, void *key);
    //销毁值的函数
    void (*valDestructor)(void *privdata, void *obj);
} dictType;

```

```C
typedef struct dict {
    //类型特定函数
    dictType *type;
    //私有数据
    void *privdata;
    //哈希表
    dictht ht[2];
    // rehash索引
    //当rehash不在进行时，值为-1
    in trehashidx; /* rehashing not in progress if rehashidx == -1 */
} dict;
```

❑type属性是一个指向`dictType`结构的指针，每个`dictType`结构保存了一簇用于操作特定类型键值对的函数，Redis会为用途不同的字典设置不同的类型特定函数。
❑而`privdata`属性则保存了需要传给那些类型特定函数的可选参数

一般情况下，**字典只使用ht[0]哈希表，ht[1]哈希表只会在对ht[0]哈希表进行rehash时使用。**除了ht[1]之外，另一个和rehash有关的属性就是rehashidx，它记录了rehash目前的进度，如果目前没有在进行rehash，那么它的值为-1。

![weread_image_160264972792346.jpeg](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/imgs/weread_image_160264972792346.jpeg)

1. **相关特性**

- 字典被广泛用于实现Redis的各种功能，其中包括数据库和哈希键。

- Redis中的字典使用哈希表作为底层实现，每个字典带有两个哈希表，一个平时使用，另一个仅在进行rehash时使用。

- 当字典被用作数据库的底层实现，或者哈希键的底层实现时，Redis使用MurmurHash2算法来计算键的哈希值。

- 哈希表使用链地址法来解决键冲突，被分配到同一个索引上的多个键值对会连接成一个单向链表。

- 在对哈希表进行扩展或者收缩操作时，程序需要将现有哈希表包含的所有键值对rehash到新哈希表里面，并且这个rehash过程并不是一次性地完成的，而是渐进式地完成的。

**如何解决哈希冲突？**

Redis的哈希表使用链地址法（separate chaining）来解决键冲突，每个哈希表节点都有一个next指针，多个哈希表节点可以用next指针构成一个单向链表，被分配到同一个索引上的多个节点可以用这个单向链表连接起来，这就解决了键冲突的问题。**使用头插法，故时间复杂度为$O(1)$**

**为什么要进行rehash？**

随着操作的不断执行，哈希表保存的键值对会逐渐地增多或者减少，为了让哈希表的负载因子（load factor）维持在一个合理的范围之内，当哈希表保存的键值对数量太多或者太少时，程序需要对哈希表的大小进行相应的扩展或者收缩。扩展和收缩哈希表的工作可以通过执行rehash（重新散列）操作来完成

**rehash过程：**

1）为字典的ht[1]哈希表分配空间，这个哈希表的空间大小取决于要执行的操作，以及ht[0]当前包含的键值对数量（也即是ht[0].used属性的值）：

- 如果执行的是扩展操作，那么ht[1]的大小为第一个大于等于ht[0].used*2的$2^n$（2的n次方幂）；

- 如果执行的是收缩操作，那么ht[1]的大小为第一个大于等于ht[0].used的$2^n$。

2）将保存在ht[0]中的所有键值对rehash到ht[1]上面：rehash指的是重新计算键的哈希值和索引值，然后将键值对放置到ht[1]哈希表的指定位置上。
3）当ht[0]包含的所有键值对都迁移到了ht[1]之后（ht[0]变为空表），释放ht[0]，将ht[1]设置为ht[0]，并在ht[1]新创建一个空白哈希表，为下一次rehash做准备。

**执行rehash的条件：**

1）服务器目前没有在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于1。
2）服务器目前正在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于5。
其中哈希表的负载因子可以通过公式：


```C
//负载因子=哈希表已保存节点数量/哈希表大小
load_factor = ht[0].used / ht[0].size
```

另一方面，当哈希表的负载因子小于0.1时，程序自动开始对哈希表执行收缩操作。 

**rehash执行的是渐进式的**

1）为ht[1]分配空间，让字典同时持有ht[0]和ht[1]两个哈希表。
2）在字典中维持一个索引计数器变量rehashidx，并将它的值设置为0，表示rehash工作正式开始。
3）在rehash进行期间，每次对字典执行添加、删除、查找或者更新操作时，程序除了执行指定的操作以外，还会顺带将ht[0]哈希表在rehashidx索引上的所有键值对rehash到ht[1]，当rehash工作完成之后，程序将rehashidx属性的值增一。
4）随着字典操作的不断执行，最终在某个时间点上，ht[0]的所有键值对都会被rehash至ht[1]，这时程序将rehashidx属性的值设为-1，表示rehash操作已完成。

**在渐进式hash中怎么去保证数据一致性是值得考虑的问题**？

1. **字典API**

![weread_image_165407025469551.jpeg](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/imgs/weread_image_165407025469551.jpeg)

### 跳跃表

1. **基本数据结构**

```C
typedef struct zskiplistNode {
    //层
    struct zskiplistLevel {
        //前进指针
        struct zskiplistNode *forward;
        //跨度
        unsigned int span;
    } level[];
    //后退指针
    struct zskiplistNode *backward;
    //分值
    double score;
    //成员对象
    robj *obj;
} zskiplistNode;
```

```C
typedef struct zskiplist {
    //表头节点和表尾节点
    structz skiplistNode *header, *tail;
    //表中节点的数量
    unsigned long length;
    //表中层数最大的节点的层数
    int level;
} zskiplist;

```

![weread_image_168149527772411.jpeg](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/imgs/weread_image_168149527772411.jpeg)

**跳表的遍历过程是怎么进行遍历的？**

1）迭代程序首先访问跳跃表的第一个节点（表头），然后从第四层的前进指针移动到表中的第二个节点。
2）在第二个节点时，程序沿着第二层的前进指针移动到表中的第三个节点。
3）在第三个节点时，程序同样沿着第二层的前进指针移动到表中的第四个节点。
4）当程序再次沿着第四个节点的前进指针移动时，它碰到一个NULL，程序知道这时已经到达了跳跃表的表尾，于是结束这次遍历。

1. **相关特性**

- 跳跃表（skiplist）是一种有序数据结构，它通过在每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的。

- 跳跃表支持平均O（logN）、最坏O（N）复杂度的节点查找，还可以通过顺序性操作来批量处理节点。

- 字典被广泛用于实现Redis的各种功能，其中包括数据库和哈希键。

- Redis中的字典使用哈希表作为底层实现，每个字典带有两个哈希表，一个平时使用，另一个仅在进行rehash时使用。

- 当字典被用作数据库的底层实现，或者哈希键的底层实现时，Redis使用MurmurHash2算法来计算键的哈希值。

- 哈希表使用链地址法来解决键冲突，被分配到同一个索引上的多个键值对会连接成一个单向链表。

- 在对哈希表进行扩展或者收缩操作时，程序需要将现有哈希表包含的所有键值对rehash到新哈希表里面，并且这个rehash过程并不是一次性地完成的，而是渐进式地完成的。

**跳表中的跨度是怎么去确定的？？？**

跨度实际上是用来计算排位（rank）的：在查找某个节点的过程中，将沿途访问过的所有层的跨度累计起来，得到的结果就是目标节点在跳跃表中的排位。

1. **跳表API**

![weread_image_169086124736272.jpeg](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/imgs/weread_image_169086124736272.jpeg)

### 整数集合（intset）

1. **基本数据结构**

```C
typedef struct intset {
    //编码方式
    uint32_t encoding;
    //集合包含的元素数量
    uint32_t length;
    //保存元素的数组
    int8_t contents[];
} intset;
```

![weread_image_172759242427423.jpeg](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/imgs/weread_image_172759242427423.jpeg)

**contents这个数组里面是怎么保证有序性的？**

1. **相关特性**

- 整数集合是集合键的底层实现之一。

- 整数集合的底层实现为数组，这个数组以有序、无重复的方式保存集合元素，在有需要时，程序会根据新添加元素的类型，改变这个数组的类型。

- 升级操作为整数集合带来了操作上的灵活性，并且尽可能地节约了内存。

- 整数集合只支持升级操作，不支持降级操作。

**升级操作**

每当我们要将一个新元素添加到整数集合里面，并且新元素的类型比整数集合现有所有元素的类型都要长时，整数集合需要先进行升级（upgrade），然后才能将新元素添加到整数集合里面。

升级整数集合并添加新元素共分为三步进行：
1）根据新元素的类型，扩展整数集合底层数组的空间大小，并为新元素分配空间。
2）将底层数组现有的所有元素都转换成与新元素相同的类型，并将类型转换后的元素放置到正确的位上，而且在放置元素的过程中，需要继续维持底层数组的有序性质不变。
3）将新元素添加到底层数组里面。

**如果插入元素之后不需要升级操作，那么系统会做什么操作？**

1. **相关API**

![weread_image_173360727032662.jpeg](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/imgs/weread_image_173360727032662.jpeg)

### 压缩列表（ziplist）

压缩列表（ziplist）是列表键和哈希键的底层实现之一。当一个列表键只包含少量列表项，并且每个列表项要么就是小整数值，要么就是长度比较短的字符串，那么Redis就会使用压缩列表来做列表键的底层实现。 

![weread_image_173863376254189.jpeg](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/imgs/weread_image_173863376254189.jpeg)

![weread_image_173869009397572.jpeg](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/imgs/weread_image_173869009397572.jpeg)

每个压缩列表节点都由**previous_entry_length、encoding、content**三个部分组成

![weread_image_218082681336329.jpeg](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/imgs/weread_image_218082681336329.jpeg)



节点的previous_entry_length属性以字节为单位，记录了压缩列表中前一个节点的长度。previous_entry_length属性的长度可以是1字节或者5字节：
❑如果前一节点的长度小于254字节，那么previous_entry_length属性的长度为1字节：前一节点的长度就保存在这一个字节里面。
❑如果前一节点的长度大于等于254字节，那么previous_entry_length属性的长度为5字节：其中属性的第一字节会被设置为0xFE（十进制值254），而之后的四个字节则用于保存前一节点的长度。

因为节点的previous_entry_length属性记录了前一个节点的长度，所以程序可以通过指针运算，根据当前节点的起始地址来计算出前一个节点的起始地址。

压缩列表的从表尾向表头遍历操作就是使用这一原理实现的，只要我们拥有了一个指向某个节点起始地址的指针，那么通过这个指针以及这个节点的previous_entry_length属性，程序就可以一直向前一个节点回溯，最终到达压缩列表的表头节点。

节点的encoding属性记录了节点的content属性所保存数据的类型以及长度：
❑一字节、两字节或者五字节长，值的最高位为00、01或者10的是字节数组编码：这种编码表示节点的content属性保存着字节数组，数组的长度由编码除去最高两位之后的其他位记录；
❑一字节长，值的最高位以11开头的是整数编码：这种编码表示节点的content属性保存着整数值，整数值的类型和长度由编码除去最高两位之后的其他位记录；

![weread_image_218880612078577.jpeg](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/imgs/weread_image_218880612078577.jpeg)

![weread_image_218888796128730.jpeg](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/imgs/weread_image_218888796128730.jpeg)

节点的content属性负责保存节点的值，节点值可以是一个字节数组或者整数，值的类型和长度由节点的encoding属性决定。
图7-9展示了一个保存字节数组的节点示例：
❑编码的最高两位00表示节点保存的是一个字节数组；
❑编码的后六位001011记录了字节数组的长度11；
❑content属性保存着节点的值"hello world"。

![weread_image_219024162740814.jpeg](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/imgs/weread_image_219024162740814.jpeg)

常用api

![weread_image_219392184312444.jpeg](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/imgs/weread_image_219392184312444.jpeg)

❑压缩列表是一种为节约内存而开发的顺序型数据结构。
❑压缩列表被用作列表键和哈希键的底层实现之一。
❑压缩列表可以包含多个节点，每个节点可以保存一个字节数组或者整数值。
❑添加新节点到压缩列表，或者从压缩列表中删除节点，可能会引发连锁更新操作，但这种操作出现的几率并不高

### 对象

```C
typedef struct redisObject {
    //类型
    unsigned type:4;
    //编码
    unsigned encoding:4;
    //指向底层实现数据结构的指针
    void *ptr;
    // ...
} robj
```

Redis中的每个对象都由一个redisObject结构表示，该结构中和保存数据有关的三个属性分别是type属性、encoding属性和ptr属性

常见的对象类型

![weread_image_221629757831590.jpeg](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/imgs/weread_image_221629757831590.jpeg)



键值基本类型：

对于Redis数据库保存的键值对来说，**键总是一个字符串对象**，而**值则可以是字符串对象、列表对象、哈希对象、集合对象或者有序集合对象的其中一种**，因此：
❑当我们称呼一个数据库键为“字符串键”时，我们指的是“这个数据库键所对应的值为字符串对象”；
❑当我们称呼一个键为“列表键”时，我们指的是“这个数据库键所对应的值为列表对象”。

![weread_image_222611446125435.jpeg](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/imgs/weread_image_222611446125435.jpeg)

通过encoding属性来设定对象所使用的编码，而不是为特定类型的对象关联一种固定的编码，极大地提升了Redis的灵活性和效率，因为Redis可以根据不同的使用场景来为一个对象设置不同的编码，从而优化对象在某一场景下的效率。

#### 字符串

**Redis中字符串组成raw与embstr的区别是什么？**

- **raw 和 embstr 的区别**

其实 embstr 编码是专门用来保存短字符串的一种优化编码，raw 和 embstr 的区别：

embstr与raw都使用redisObject和sds保存数据，区别在于，embstr的使用只分配一次内存空间（因此redisObject和sds是连续的），而raw需要分配两次内存空间（分别为redisObject和sds分配空间）。因此与raw相比，embstr的好处在于创建时少分配一次空间，删除时少释放一次空间，以及对象的所有数据连在一起，寻找方便。而embstr的坏处也很明显，如果字符串的长度增加需要重新分配内存时，整个redisObject和sds都需要重新分配空间，因此redis中的embstr实现为只读。

浮点数在redis中也是以字符串的形式保存。

字符串api

![weread_image_225533125701820.jpeg](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/imgs/weread_image_225533125701820.jpeg)

#### 列表

列表对象的编码可以是**ziplist或者linkedlist**。
ziplist编码的列表对象使用压缩列表作为底层实现，每个压缩列表节点（entry）保存了一个列表元素。

**编码转换**
当列表对象可以同时满足以下两个条件时，列表对象使用ziplist编码：

❑列表对象保存的所有字符串元素的长度都小于64字节；

❑列表对象保存的元素数量小于512个；不能满足这两个条件的列表对象需要使用linkedlist编码。

**列表api**

![weread_image_22797267821927.jpeg](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/imgs/weread_image_22797267821927.jpeg)



#### 哈希

哈希对象的编码可以是ziplist或者hashtable。

使用ziplist的实现方式：

ziplist编码的哈希对象使用压缩列表作为底层实现，每当有新的键值对要加入到哈希对象时，程序会先将保存了键的压缩列表节点推入到压缩列表表尾，然后再将保存了值的压缩列表节点推入到压缩列表表尾。

**问题？**

这种实现方式下，如何保证查询时$O(1)$的？

编码转化：

❑哈希对象保存的所有键值对的键和值的字符串长度都小于64字节；

❑哈希对象保存的键值对数量小于512个；不能满足这两个条件的哈希对象需要使用hashtable编码。

基本api

![weread_image_5958789557517.jpeg](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/imgs/weread_image_5958789557517.jpeg)

#### 集合（set）

集合对象的编码可以是intset或者hashtable。

intset编码的集合对象使用整数集合作为底层实现，集合对象包含的所有元素都被保存在整数集合里面。

hashtable编码的集合对象使用字典作为底层实现，字典的每个键都是一个字符串对象，每个字符串对象包含了一个集合元素，而字典的值则全部被设置为NULL。**只用字典的键不用值**

编码转化
当集合对象可以同时满足以下两个条件时，对象使用intset编码：

❑集合对象保存的所有元素都是整数值；

❑集合对象保存的元素数量不超过512个。

不能满足这两个条件的集合对象需要使用hashtable编码。

常用api

![weread_image_6362963911738.jpeg](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/imgs/weread_image_6362963911738.jpeg)

#### 有序集合（zset）

有序集合的编码可以是ziplist或者skiplist。

ziplist编码的压缩列表对象使用压缩列表作为底层实现，每个集合元素使用两个紧挨在一起的压缩列表节点来保存，第一个节点保存元素的成员（member），而第二个元素则保存元素的分值（score）。

压缩列表内的集合元素按分值从小到大进行排序，分值较小的元素被放置在靠近表头的方向，而分值较大的元素则被放置在靠近表尾的方向。

skiplist编码的有序集合对象使用zset结构作为底层实现，一个zset结构同时包含一个字典和一个跳跃表：

```Go
typedef struct zset {
    zskiplist *zsl;
    dict *dict;
} zset
```

![weread_image_6881239185186.jpeg](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/imgs/weread_image_6881239185186.jpeg)

编码转化

当有序集合对象可以同时满足以下两个条件时，对象使用ziplist编码：
❑有序集合保存的元素数量小于128个；
❑有序集合保存的所有元素成员的长度都小于64字节；
不能满足以上两个条件的有序集合对象将使用skiplist编码。

基本api

![weread_image_7180973041426.jpeg](https://hruoxuan.oss-cn-shenzhen.aliyuncs.com/imgs/weread_image_7180973041426.jpeg)




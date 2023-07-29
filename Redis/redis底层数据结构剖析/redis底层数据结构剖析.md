---
highlight: a11y-dark
---

# redis 数据库介绍

Redis 是一种键值（Key-Value）数据库。相对于关系型数据库（比如：MySQL），Redis 也被叫作非关系型数据库。
像 MySQL 这样的关系型数据库，表的结构比较复杂，会包含很多字段，可以通过 SQL 语句，来实现非常复杂的查询需求。而 Redis 中只包含“键”和“值”两部分，只能通过“键”来查询“值”。正是因为这样简单的存储结构，也让 Redis 的读写效率非常高。

除此之外，Redis 主要是作为内存数据库来使用，也就是说，数据是存储在内存中的。尽管它经常被用作内存数据库，但是，它也支持将数据存储在硬盘中，也就是所说的持久化。

Redis 中，键的数据类型是字符串，但是为了丰富数据存储的方式，方便开发者使用，值的数据类型有很多，常用的数据类型有这样几种，它们分别是字符串、列
表、字典、集合、有序集合。


## 字符串（string）

先来看下 Redis 的 RedisObject 的数据结构，如下所示：：

```C
typedef struct redisObject {
    // 对外的类型 string list set hash zset等 4bit
    unsigned type:4;
    // 底层存储方式 4bit
    unsigned encoding:4;
    // LRU 时间 24bit
    unsigned lru:LRU_BITS;
    // 引用计数  4byte
    int refcount;
    // 指向对象的指针  8byte
    void *ptr;
} robj;
```

我们都知道，Redis 是由 C 语言编写的。在 C 语言中，字符串标准形式是以空字符\0 作为结束符的，但是 Redis 里面的字符串却没有直接沿用 C 语言的字符串。主要是因为 C 语言中获取字符串长度可以调用 strlen 这个标准函数，这个函数的时间复杂度是 O(N)，由于 Redis 是单线程的，这个时间复杂度代价有点高。

对于不同的对象，Redis 会使用不同的类型来存储。对于同一种类型 type 会有不同的存储形式 encoding。对于 string 类型的字符串，其底层编码方式共有三种，分别为 int、embstr 和 raw。

- int：当存储的字符串全是数字时，此时使用 int 方式来存储；
- embstr：当存储的字符串长度小于 44 个字符时，此时使用 embstr 方式来存储；
- raw：当存储的字符串长度大于 44 个字符时，此时使用 raw 方式来存储；

使用 object encoding key 可以查看 key 对应的 encoding 类型，如下所示：

![str1.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4fa48dd6bc2c483fb77da9e480bde2a0~tplv-k3u1fbpfcp-watermark.image)

对于 embstr 和 raw 这两种 encoding 类型，其存储方式还不太一样。对于 embstr 类型，它将 RedisObject 对象头和 SDS 对象在内存中地址是连在一起的，但对于 raw 类型，二者在内存地址不是连续的。如下图所示：

![str2.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a608aa59f2a14da9a1f17daa80004d30~tplv-k3u1fbpfcp-watermark.image)

### SDS

在介绍 string 类型的存储类型时，我们说到，对于 embstr 和 raw 两种类型其存储方式不一样，但 ptr 指针最后都指向一个 SDS 的结构。那什么是 SDS 呢？Redis 中的字符串称之为简单动态字符串（Simple Dynamic String），简称为 SDS。

与普通 C 语言的原始字符串结构相比，sds 多了一个 sdshdr 的头部信息，sdshdr 基本数据结构如下所示：

```c
struct sdsshr<T>{
    T len;//数组长度
    T alloc;//数组容量
    unsigned  flags;//sdshdr类型
    char buf[];//数组内容
}
```

可以看出，SDS 的结构有点类似于 Java 中的 ArrayList。buf[]表示真正存储的字符串内容，alloc 表示所分配的数组的长度，len 表示字符串的实际长度，并且由于 len 这个属性的存在，Redis 可以在 O(1)的时间复杂度内获取数组长度。

SDS 的优势：

- 长度获取时间复杂度 O(1)；
- 避免频繁内存分配；
- 二进制安全；

## 列表（list）

我们先来看列表。列表这种数据类型支持存储一组数据。这种数据类型对应两种实现方法，一种是压缩列表（ziplist），另一种是双向循环链表（其实优化后为 quicklist）。

当列表中存储的数据量比较小的时候，列表就可以采用压缩列表的方式实现。具体需要同时满足下面两个条件：

- 列表中保存的单个数据（有可能是字符串类型的）小于 64 字节；
- 列表中数据个数少于 512 个。

我们先来看下 redis 用 c 语言实现列表的结构：

```c
//定义链表节点的结构体
typedf struct listNode{
    //前一个节点
    struct listNode *prev;
    //后一个节点
    struct listNode *next;
    //当前节点的值的指针
    void *value;
}listNode;

```

### ziplist

关于压缩列表，我这里稍微解释一下。它并不是基础数据结构，而是 Redis 自己设计的一种数据存储结构。它有点儿类似数组，通过一片连续的内存空间，来存储数据。不过，它跟数组不同的一点是，它允许存储的数据大小不同。具体的存储结构也非常简单，你可以看我下面画的这幅图。

![ziplist.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/80b5f1f797a843dd83f55a79173ac828~tplv-k3u1fbpfcp-watermark.image)

ziplist 结构源码如下所示：

```C
typedf struct ziplist<T>{
    //压缩列表占用字符数
    int32 zlbytes;
    //最后一个元素距离起始位置的偏移量，用于快速定位最后一个节点
    int32 zltail_offset;
    //元素个数
    int16 zllength;
    //元素内容
    T[] entries;
    //结束位 0xFF
    int8 zlend;
}ziplist
```

现在来看看，压缩列表中的“压缩”两个字该如何理解？  
听到“压缩”两个字，直观的反应就是节省内存。之所以说这种存储结构节省内存，是相较于数组的存储思路而言的。我们知道，数组要求每个元素的大小相同，如果要存储不同长度的字符串，那就需要用最大长度的字符串大小作为元素的大小（假设是 20 个字节）。那当存储小于 20 个字节长度的字符串的时候，便会浪费部分存储空间。听起来有点儿拗口，我画个图解释一下。

![list2.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/079c8d917b404c64b7edf997244ab3b5~tplv-k3u1fbpfcp-watermark.image)

压缩列表这种存储结构，一方面比较节省内存，另一方面可以支持不同类型数据的存储。而且，因为数据存储在一片连续的内存空间，通过键来获取值为列表类型的数据，读取的效率也非常高。

### quicklist

当列表中存储的数据量比较大的时候，也就是不能同时满足刚刚讲的两个条件的时候，列表就要通过双向循环链（优化后其实是 qiucklist）表来实现了。

在[链表](https://juejin.cn/post/6961771129014845448)里，已经讲过双向循环链表这种数据结构了，如果不记得了，你可以先回去复习一下。这里着重看一下 Redis 中双向链表的编码实现方式。

Redis 的这种双向链表的实现方式，非常值得借鉴。它额外定义一个 list 结构体，来组织链表的首、尾指针，还有长度等信息。这样，在使用的时候就会非常方便。

qucikList 是由 zipList 和双向链表 linkedList 组成的混合体。它将 linkedList 按段切分，每一段使用 zipList 来紧凑存储，多个 zipList 之间使用双向指针串接起来。示意图如下所示：

![quicklist.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4ce7dc1bf9fb44a4b41f727c6361b1b2~tplv-k3u1fbpfcp-watermark.image)

quicklist 结构定义如下：

```c
typedf struct quicklist{
    //指向头结点
    quicklistNode* head;
    //指向尾节点
    quicklistNode* tail;
    //元素总数
    long count;
    //quicklistNode节点的个数
    int nodes;
    //压缩算法深度
    int compressDepth;
    ...
}quickList
```

## 字典（dict）

字典类型用来存储一组数据对。每个数据对又包含键值两部分。字典类型也有两种实现方式。一种是刚刚讲到的压缩列表，另一种是散列表。

同样，只有当存储的数据量比较小的情况下，Redis 才使用压缩列表来实现字典类型。具体需要满足两个条件：

- 字典中保存的键和值的大小都要小于 64 字节；
- 字典中键值对的个数要小于 512 个。

当不能同时满足上面两个条件的时候，Redis 就使用散列表来实现字典类型。Redis 使用 MurmurHash2 这种运行速度快、随机性好的哈希算法作为哈希函数。对于哈希冲突问题，Redis 使用链表法来解决。除此之外，Redis 还支持散列表的动态扩容、缩容。

当数据动态增加之后，散列表的装载因子会不停地变大。为了避免散列表性能的下降，当装载因子大于 1 的时候，Redis 会触发扩容，将散列表扩大为原来大小的 2 倍左右（具体值需要计算才能得到，如果感兴趣，你可以去阅读[源码](https://github.com/redis/redis/blob/unstable/src/dict.c)）。

当数据动态减少之后，为了节省内存，当装载因子小于 0.1 的时候，Redis 就会触发缩容，缩小为字典中数据个数的大约 2 倍大小（这个值也是计算得到的，如果感兴趣，你也可以去阅读源码）。

dict 数据结构示意图如下：

![dict.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/603c652cacc84545a6a236da29611d36~tplv-k3u1fbpfcp-watermark.image)

## 集合（set）

集合这种数据类型用来存储一组不重复的数据。这种数据类型也有两种实现方法，一种是基于有序数组，另一种是基于散列表。

当要存储的数据，同时满足下面这样两个条件的时候，Redis 就采用有序数组，来实现集合这种数据类型。

- 存储的数据都是整数；
- 存储的数据元素个数不超过 512 个。

当不能同时满足这两个条件的时候，Redis 就使用散列表来存储集合中的数据。

## 有序集合（zset）

有序集合这种数据类型，我们在[跳表](https://juejin.cn/post/6962735884844138533)里已经详细讲过了。它用来存储一组数据，并且每个数据会附带一个得分。通过得分的大小，我们将数据组织成跳表这样的数据结构，以支持快速地按照得分值、得分区间获取数据。

实际上，跟 Redis 的其他数据类型一样，有序集合也并不仅仅只有跳表这一种实现方式。当数据量比较小的时候，Redis 会用压缩列表来实现有序集合。具体点说就是，使用压缩列表来实现有序集合的前提，有这样两个：

- 所有数据的大小都要小于 64 字节；
- 元素个数要小于 128 个。

# 总结

以上是 Redis 五大数据类型的数据结构介绍，也总结了一些思维脑图，由于文章篇幅有限，这里就只放列表（list）的思维脑图，如想获取其它总结的思维脑图，可前往公众【**Go 键盘侠**】，回复“**脑图**”即可。

![listlast.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d8c0719697814b6a80d6f5bf26393e24~tplv-k3u1fbpfcp-watermark.image)


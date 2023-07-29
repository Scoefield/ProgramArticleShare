## 前言

今天我们来聊聊“LRU“缓存淘汰算法。其实LRU缓存淘汰算法是链表的一个经典应用场景，而缓存是一种提高数据读取性能的技术，在软件设计及开发中都应用非常广泛，比如常见的有CPU缓存、数据库缓存、浏览器缓存等。

而缓存的大小是有限的，当缓存达到上限时，我们应该考虑哪些数据需要被清掉，哪些数据应该被保留？这就需要根据缓存淘汰策略来决定了。

## 常见缓存淘汰策略

常见的策略一般有以下三种：

- 先进先出FIFO（First In，First Out）
- 最少使用策略LFU（Least Frequently Used）
- 最近最少使用策略LRU（Least Recently Used）

举个例子，比如你买了很多机械键盘，但有一天发现，这些键盘有点多，太占房间的位置了，你需要卖掉一些，腾出空间。此时，你会考虑该选择哪些键盘卖掉呢？回过头来思考下，你的选择标准是不是和上面的三种策略有点像？

言归正传，今天分享的就是：LRU缓存淘汰算法的介绍，以及如何用链表来实现LRU缓存淘汰算法。

## 数组和链表结构

相比数组来说，链表是一种稍微复杂一点的数据结构。对于初学者来说，掌握起来也要比数组稍难一些。但是数组和链表算是比较基础、也比较常用的数据结构，我们常常将会放到 一块儿来比较。先来看看，这两者有什么区别。

从底层的存储结构上来看一看。为了比较直观地对比，我画了一张图，如下所示：
![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1b0585cf57cf4f429d847289e3480c69~tplv-k3u1fbpfcp-watermark.image)

从图中我们看出，数组内存存储空间是连续的，对内存的要求比较高。比如要申请一个 1G 大小的数组，如果内存中没有连续的、也没有足够大的存储空间时，即使内存的剩余总可用空间大于 1G，仍然会申请失败。

而链表则相反，它并不需要一块连续的内存空间，它是通过“指针”(内存地址)的方式将一组零散的内存块结合起来使用，那么如果我们申请的是 1G 大小的链表时，会申请成功，不会有问题。

链表结构通常有以下三种：

- 单向链表
- 双向链表
- 循环链表

这里就不展开赘述了，如果有需要后续会补充双向链表和循环链表的介绍，单向链表的可以参考前面写的一篇文章：Golang实现单向链表的基础操作。

## 数组和链表性能对比

通过前面内容的介绍说明，数组和链表是两种不同的数据结构。因为它们的内存存储方式有区别，所以它们在插入、删除、访问操作的时间复杂度是不一样的，如下所示：
![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4739e6943a6245a394bbf3dd43e369e9~tplv-k3u1fbpfcp-watermark.image)

不过数组和链表的对比，并不能局限于时间复杂度，而且在实际的程序开发中，不能仅仅利用复杂度分析就决定使用哪个数据结构来存储数据。

数组简单易用，在实现上使用的是连续的内存空间，可以借助CPU的缓存机制，预读数组中的数据，所以访问效率更高。而链表在内存中并不是连续存储，所以对CPU缓存不是很好，不能有效的预读。

数组的缺点是大小固定，一经声明就要占用整块连续内存空间。如果声明的数组过大，系统可能没有足够的连续内存空间分配给它，导致“内存不足（out of memory）”；如果声明的数组过小，则可能出现不够用的情况，这时只能再申请一个更大的内存空间，把原数组拷贝进去，非常耗时。而链表没有大小的限制，支持动态扩容，这也是它与数组最大的区别。

如果你的程序对内存使用比较苛刻，则可以考虑选择使用数组，因为链表中的每个结点都需要额外的存储空间去存储指向下一个结点的指针，所以链表的内存消耗会大点，而且对链表进行频繁的插入、删除操作会导致频繁的内存申请和释放，容易造成内存碎片，如果是Go语言，就可能会导致频繁的GC。所以，在我们实际的开发中，要根据具体的开发场景，来权衡选择使用哪种数据结构。

## LRU算法介绍

以上关于数组和链表的相关知识已经讲完了。现在我们来思考下：如何基于链表实现LRU缓存淘汰算法？

那么学习了链表之后，应该会有这样的思路：首先维护一个有序链表，然后越靠近链表尾部的结点说明是越早之前访问的。当有一个新的数据访问时，需要从链表头开始顺序遍历链表，此时会有以下这几种情况：

如果此数据之前已经被缓存在链表中了，我们遍历得到这个数据对应的结点，并将其从原来的位置删除，然后再插入到链表的头部。

如果此数据没有在缓存链表中，又可以分为两种情况：
1. 如果此时缓存未满，则将此结点直接插入到链表的头部；
2. 如果此时缓存已满，则链表的尾结点删除，将新的数据结点插入链表的头部。

这样就用链表实现了一个LRU缓存算法了。现在我们来看下缓存访问的时间复杂度是多少。因为不管缓存有没有满，我们都需要遍历一遍链表，所以这种基于链表的实现思路，缓存访问的时间复杂度为O(n)。

实际上，可以继续优化这个实现思路，比如引入散列表（Hash table）来记录每个数据的位置，将缓存访问的时间复杂度降到O(1)。这个优化方案现在就不详细说了，如果大家感兴趣的可以作为课后思考题来解答。

## LRU缓存淘汰算法实现

基于Golang实现LRU算法，完整代码如下：
```go
package main

import "fmt"

// LRU缓存结构
type LRUCache struct {
	Cap int				// 缓存大小
	Map map[int]*Node	// 存储节点数据的map
	Head *Node			// 头节点
	Last *Node			// 尾节点
}

// 双向链表 节点
type Node struct {
	Key int			// key
	Val int			// 数据值
	Pre *Node		// 前一个节点的指针
	Next *Node		// 下一个节点的指针
}

// 创建 LRU 缓存结构，一些初始化操作
func NewLRUCache(cap int) *LRUCache {
	cache := &LRUCache{
		Cap:  cap,
		Map:  make(map[int]*Node, cap),
		Head: &Node{},
		Last: &Node{},
	}

	// 双向链表 初始化
	cache.Head.Next = cache.Last
	cache.Last.Pre = cache.Head
	return cache
}

// 设置头节点
func (l *LRUCache) setHeader(node *Node) {
	l.Head.Next.Pre = node
	node.Next = l.Head.Next
	l.Head.Next = node
	node.Pre = l.Head
}

// 删除节点
func (l *LRUCache) remove(node *Node) {
	l.Head.Next = node.Next
	node.Next.Pre = l.Head
}

// 通过 key 获取数据
// 获取不到直接返回 -1
// 获取到，则先删除获取到的节点，在将该节点放到头节点
// 返回获取的值
func (l *LRUCache) Get(key int) int {
	node, ok := l.Map[key]
	if !ok {
		return -1
	}
	l.remove(node)
	l.setHeader(node)
	return node.Val
}

// 加入缓存操作
// 先通过 key 获取节点数据
// 如果获取到，则删除掉该节点
// 如果获取不到，则判断缓存是否满了
// 如果缓存满了，则删掉最后一个节点数据
// 最后将节点数据放到头部
func (l *LRUCache) Put(key, value int) {
	node, ok := l.Map[key]
	if ok {
		l.remove(node)
	} else {
		if len(l.Map) == l.Cap {
			delete(l.Map, l.Last.Pre.Key)
			l.remove(l.Last.Pre)
		}
		node = &Node{Key:key, Val:value}
		l.Map[key] = node
	}

	node.Val = value
	l.setHeader(node)
}

// 主函数，LRU算法测试
func main() {
	lruCache := NewLRUCache(3)

	val := lruCache.Get(2)
	fmt.Println(val)

	lruCache.Put(2, 22)
	lruCache.Put(3, 33)

	val = lruCache.Get(2)
	fmt.Println(val)

	lruCache.Put(4, 44)
	lruCache.Put(5, 55)

	val = lruCache.Get(3)
	fmt.Println(val)
}
```
代码有注释说明，这里不再对代码做解释了。

以上示例代码已归档到 **GitHub** ，如有需要，可自行下载：https://github.com/Scoefield/gokeyboardman/tree/main/lrudemo 。

## 总结

今天主要分享了数组和链表这两种数据结构，以及如何用链表实现LRU缓存淘汰算法。  
- 这两种数据结构比较基础、也比较常用，不过链表要比数组稍微复杂。  
- 链表更适合插入、删除操作频繁的场景，查询的时间复杂度较高。  
- 不过，在具体软件开发中，要对数组和链表的各种性能进行对比，综合来选择使用哪种数据结构。
- 最后就是用Golang实现了LRU算法，可以参考学习，举一反三。

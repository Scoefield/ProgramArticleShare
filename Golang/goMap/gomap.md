---
highlight: dark
---

# Map

用了 Go 语言那么久，该回过头来看看 map 的底层到底是怎么实现的。

**map** 其实可以理解为一种**哈希表**数据结构。几乎所有的编程语言都会有数组和哈希表这两种数据结构，有的编程语言将数组称为列表，比如：Python，而有的语言将 map 称作字典或者映射。无论如何命名或者如何实现，数组和哈希是两种设计集合元素的思路，数组用于表示元素的序列，而哈希表示的是键值对之间映射关系。

以下哈希表即为**Map**。

## 设计原理

哈希表是计算机中最重要的数据结构之一，不仅是因为它 𝑂(1) 的时间复杂度，而且读写性能非常好，它还提供了键值之间的映射。想要实现一个性能优异的哈希表，需要注意两个关键点：

- 哈希函数的选择
- 哈希冲突的解决

### 哈希函数

实现哈希表的关键点在于哈希函数的选择，哈希函数的选择在很大程度上能够决定哈希表的读写性能。

实际的方式是让哈希函数的结果能够尽可能的均匀分布，然后通过相应的方法来解决哈希冲突的问题。哈希函数映射的结果一定要尽可能均匀，结果不均匀的哈希函数会带来更多的哈希冲突以及更差的读写性能。

如果使用结果分布较为均匀的哈希函数，那么哈希的增删改查的时间复杂度为 𝑂(1)；但是如果哈希函数的结果分布不均匀，那么所有操作的时间复杂度可能会达到 𝑂(𝑛)，由此看来，选择使用好的哈希函数是至关重要的。

### 冲突解决

前面所提到的，在通常情况下，哈希函数是不一定完美的，当输入的键足够多也会产生冲突，所以仍然存在发生哈希碰撞的可能，这时就需要一些方法来解决哈希碰撞的问题，常见方法的有：**开放寻址法**和**拉链法**。

#### 开放寻址法

**开放寻址法**是一种在哈希表中解决哈希碰撞的比较普通的方法，这种方法的核心思想是：**依次探测和比较数组中的元素以判断目标键值对是否存在于哈希表中**，如果我们使用开放寻址法来实现哈希表，那么实现哈希表底层的数据结构就是数组，不过因为数组的长度有限。

当我们向当前哈希表写入新的数据时，如果发生了冲突，就会将键值对写入到下一个索引不为空的位置，如下：

![开放寻址法.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2066f1668cd34ff8b6a477320452c0a7~tplv-k3u1fbpfcp-watermark.image)

如上图所示，当 Key3 与已经存入哈希表中的两个键值对 Key1 和 Key2 发生冲突时，Key3 会被写入 Key2 后面的空闲位置。当我们再去读取 Key3 对应的值时就会先获取键的哈希并取模，这会先帮助我们找到 Key1，找到 Key1 后发现它与 Key 3 不相等，所以会继续查找后面的元素，直到内存为空或者找到目标元素。

![开放寻址法2.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/78c696b156104081a3ffbdd43a1584f7~tplv-k3u1fbpfcp-watermark.image)

开放寻址法中对性能影响最大的是**装载因子**，它是数组中元素的数量与数组大小的比值。随着装载因子的增加，线性探测的平均用时就会逐渐增加，这会影响哈希表的读写性能。当装载率超过 70% 之后，哈希表的性能就会急剧下降，而一旦装载率达到 100%，整个哈希表就会完全失效，这时查找和插入任意元素的时间复杂度都是 𝑂(𝑛) 的，这时需要遍历数组中的全部元素，所以在实现哈希表时一定要关注装载因子的变化。

#### 拉链法

与开放地址法相比，拉链法是哈希表最常见的实现方法，大多数的编程语言都用拉链法实现哈希表，它的实现比较开放地址法稍微复杂一些，但是平均查找的长度也比较短，各个用于存储节点的内存都是动态申请的，可以节省比较多的存储空间。

实现拉链法一般会使用数组加上链表，不过一些编程语言会在拉链法的哈希中引入红黑树以优化性能，拉链法会使用链表数组作为哈希底层的数据结构，我们可以将它看成可以扩展的二维数组：

![拉链法.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f3dd3a752779488c9a6a74c3525407be~tplv-k3u1fbpfcp-watermark.image)

如上图所示，当我们需要将一个键值对 (Key6, Value6) 写入哈希表时，键值对中的键 Key6 都会先经过一个哈希函数，哈希函数返回的哈希会帮助我们选择一个桶，和开放地址法一样，选择桶的方式是直接对哈希返回的结果取模：

```go
index := hash("Key6") % array.len
```

选择了 2 号桶后就可以遍历当前桶中的链表了，在遍历链表的过程中会遇到以下两种情况：

1. 找到键相同的键值对 — 更新键对应的值；
2. 没有找到键相同的键值对 — 在链表的末尾追加新的键值对；

## map 底层数据结构

Go 语言同时使用了多个数据结构组合表示哈希表（map），其中 runtime.hmap 是最核心的结构体，我们先来了解一下该结构体的内部字段：

```go
type hmap struct {
	count     int
	flags     uint8
	B         uint8
	noverflow uint16
	hash0     uint32

	buckets    unsafe.Pointer
	oldbuckets unsafe.Pointer
	nevacuate  uintptr

	extra *mapextra
}

type mapextra struct {
	overflow    *[]*bmap
	oldoverflow *[]*bmap
	nextOverflow *bmap
}
```

1. `count` 表示当前哈希表中的元素数量；
2. `B` 表示当前哈希表持有的 `buckets` 数量，但是因为哈希表中桶的数量都 2 的倍数，所以该字段会存储对数，也就是 len(buckets) == 2^B；
3. `hash0` 是哈希的种子，它能为哈希函数的结果引入随机性，这个值在创建哈希表时确定，并在调用哈希函数时作为参数传入；
   4 `oldbuckets` 是哈希在扩容时用于保存之前 `buckets` 的字段，它的大小是当前 `buckets` 的一半；

![hmap.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/561d2e339e4d40119af819d7c3a77485~tplv-k3u1fbpfcp-watermark.image)

如上图所示哈希表 `runtime.hmap` 的桶是 `runtime.bmap`。每一个 `runtime.bmap` 都能存储 8 个键值对，当哈希表中存储的数据过多，单个桶已经装满时就会使用 `extra.nextOverflow` 中桶存储溢出的数据。

上述两种不同的桶在内存中是连续存储的，我们在这里将它们分别称为正常桶和溢出桶，上图中黄色的 `runtime.bmap` 就是正常桶，绿色的 `runtime.bmap` 是溢出桶。

桶的结构体 `runtime.bmap` 在 Go 语言源代码中的定义只包含一个简单的 `tophash` 字段，`tophash` 存储了键的哈希的高 8 位，通过比较不同键的哈希的高 8 位可以减少访问键值对次数以提高性能：

```go
type bmap struct {
	tophash [bucketCnt]uint8
}
```

在运行期间，`runtime.bmap` 结构体其实不止包含 `tophash` 字段，因为哈希表中可能存储不同类型的键值对，而且 Go 语言也不支持泛型，所以键值对占据的内存空间大小只能在编译时进行推导。`runtime.bmap` 中的其他字段在运行时也都是通过计算内存地址的方式访问的，所以它的定义中就不包含这些字段。

## map 初始化

Go 语言中哈希表的实现，首先要分析的是 Go 语言初始化哈希的两种方法 — 通过字面量和运行时。

### 字面量

目前的现代编程语言基本都支持使用字面量的方式初始化哈希，一般都会使用 key: value 的语法来表示键值对，Go 语言中也不例外：

```go
hash := map[string]int{
	"1": 2,
	"3": 4,
	"5": 6,
}
```

我们需要在初始化哈希时声明键值对的类型，这种使用字面量初始化的方式最终都会通过 `cmd/compile/internal/gc.maplit` 初始化，我们来分析一下该函数初始化哈希的过程：

```go
func maplit(n *Node, m *Node, init *Nodes) {
	a := nod(OMAKE, nil, nil)
	a.Esc = n.Esc
	a.List.Set2(typenod(n.Type), nodintconst(int64(n.List.Len())))
	litas(m, a, init)

	entries := n.List.Slice()
        // 注意此处判断
	if len(entries) > 25 {
		...
		return
	}

	// Build list of var[c] = expr.
	// Use temporaries so that mapassign1 can have addressable key, elem.
	...
}
```

当哈希表中的元素数量少于或者等于 25 个时，编译器会将字面量初始化的结构体转换成以下的代码，将所有的键值对一次加入到哈希表中：

```go
hash := make(map[string]int, 3)
hash["1"] = 2
hash["3"] = 4
hash["5"] = 6
```

这种初始化的方式与的**数组**和**切片**几乎完全相同，由此看来集合类型的初始化在 Go 语言中有着相同的处理逻辑。

一旦哈希表中元素的数量超过了 25 个，编译器会创建两个数组分别存储键和值，这些键值对会通过如下所示的 for 循环加入哈希：

```go
hash := make(map[string]int, 26)
vstatk := []string{"1", "2", "3", ... ， "26"}
vstatv := []int{1, 2, 3, ... , 26}
for i := 0; i < len(vstak); i++ {
    hash[vstatk[i]] = vstatv[i]
}
```

这里展开的两个切片 vstatk 和 vstatv 还会被编辑器继续展开，无论使用哪种方法，使用字面量初始化的过程都会使用 Go 语言中的关键字 make 来创建新的哈希并通过最原始的 [] 语法向哈希追加元素。

### 运行时

当创建的哈希被分配到栈上并且其容量小于 BUCKETSIZE = 8 时，Go 语言在编译阶段会使用如下方式快速初始化哈希，这也是编译器对小容量的哈希做的优化：

```go
var h *hmap
var hv hmap
var bv bmap
h := &hv
b := &bv
h.buckets = b
h.hash0 = fashtrand0()
```

除了上述特定的优化之外，无论 make 是从哪里来的，只要我们使用 make 创建哈希，Go 语言编译器都会在类型检查期间将它们转换成 `runtime.makemap`，使用字面量初始化哈希也只是语言提供的辅助工具，最后调用的都是 `runtime.makemap`：

```go
unc makemap(t *maptype, hint int, h *hmap) *hmap {
	mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
	if overflow || mem > maxAlloc {
		hint = 0
	}

	if h == nil {
		h = new(hmap)
	}
	h.hash0 = fastrand()

	B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B

	if h.B != 0 {
		var nextOverflow *bmap
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}
	return h
}
```

这个函数会按照下面的步骤执行：

1. 计算哈希占用的内存是否溢出或者超出能分配的最大值；
2. 调用 `runtime.fastrand` 获取一个随机的哈希种子；
3. 根据传入的 `hint` 计算出需要的最小需要的桶的数量；
4. 使用 `runtime.makeBucketArray` 创建用于保存桶的数组；

`runtime.makeBucketArray` 会根据传入的 B 计算出的需要创建的桶数量并在内存中分配一片连续的空间用于存储数据：

```go
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
	base := bucketShift(b)
	nbuckets := base
	if b >= 4 {
		nbuckets += bucketShift(b - 4)
		sz := t.bucket.size * nbuckets
		up := roundupsize(sz)
		if up != sz {
			nbuckets = up / t.bucket.size
		}
	}

	buckets = newarray(t.bucket, int(nbuckets))
	if base != nbuckets {
		nextOverflow = (*bmap)(add(buckets, base*uintptr(t.bucketsize)))
		last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.bucketsize)))
		last.setoverflow(t, (*bmap)(buckets))
	}
	return buckets, nextOverflow
}
```

- 当桶的数量小于 24 时，由于数据较少、使用溢出桶的可能性较低，会省略创建的过程以减少额外开销；
- 当桶的数量多于 24 时，会额外创建 2𝐵−4 个溢出桶；

根据上述代码，我们能确定在正常情况下，正常桶和溢出桶在内存中的存储空间是连续的，只是被 `runtime.hmap` 中的不同字段引用，当溢出桶数量较多时会通过 `runtime.newobject` 创建新的溢出桶。

## map 读写操作

### 读写

哈希表作为一种数据结构，我们肯定要分析它的常见操作，首先就是读写操作的原理。哈希表的访问一般都是通过下标或者遍历进行的：

```go
_ = hash[key]

for k, v := range hash {
    // k, v
}
```

数据结构的写一般指的都是增加、删除和修改，增加和修改字段都使用索引和赋值语句，而删除字典中的数据需要使用关键字 delete：

```go
hash[key] = value
hash[key] = newValue
delete(hash, key)
```

### 扩容

前面在介绍哈希的写入过程时其实省略了扩容操作，随着哈希表中元素的逐渐增加，哈希的性能会逐渐恶化，所以我们需要更多的桶和更大的内存保证哈希的读写性能：

```go
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	...
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		goto again
	}
	...
}
```

`runtime.mapassign` 函数会在以下两种情况发生时触发哈希的扩容：

1. 装载因子已经超过 6.5；
2. 哈希使用了太多溢出桶；

不过因为 Go 语言哈希的扩容不是一个原子的过程，所以 `runtime.mapassign` 还需要判断当前哈希是否已经处于扩容状态，避免二次扩容造成混乱。

根据触发的条件不同扩容的方式分成两种：

1. 如果这次扩容是溢出的桶太多导致的，那么这次扩容就是等量扩容 `sameSizeGrow`，`sameSizeGrow` 是一种特殊情况下发生的扩容；
2. 当我们持续向哈希中插入数据并将它们全部删除时，如果哈希表中的数据量没有超过阈值，就会不断积累溢出桶造成缓慢的内存泄漏。引入了 sameSizeGrow 通过复用已有的哈希扩容机制解决该问题，一旦哈希中出现了过多的溢出桶，它会创建新桶保存数据，垃圾回收会清理老的溢出桶并释放内存。

扩容的入口是 `runtime.hashGrow`：

```go
func hashGrow(t *maptype, h *hmap) {
	bigger := uint8(1)
	if !overLoadFactor(h.count+1, h.B) {
		bigger = 0
		h.flags |= sameSizeGrow
	}
	oldbuckets := h.buckets
	newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)

	h.B += bigger
	h.flags = flags
	h.oldbuckets = oldbuckets
	h.buckets = newbuckets
	h.nevacuate = 0
	h.noverflow = 0

	h.extra.oldoverflow = h.extra.overflow
	h.extra.overflow = nil
	h.extra.nextOverflow = nextOverflow
}
```

哈希在扩容的过程中会通过 `runtime.makeBucketArray` 创建一组新桶和预创建的溢出桶，随后将原有的桶数组设置到 `oldbuckets` 上并将新的空桶设置到 `buckets` 上，溢出桶也使用了相同的逻辑更新，下图展示了触发扩容后的哈希：

![hmap2.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5a2e1655b0744623b41f3690267e3544~tplv-k3u1fbpfcp-watermark.image)

## 小结

Go 语言使用拉链法来解决哈希碰撞的问题实现了哈希表，它的访问、写入和删除等操作都在编译期间转换成了运行时的函数或者方法。

哈希在每一个桶中存储键对应哈希的前 8 位，当对哈希进行操作时，这些 `tophash` 就成为可以帮助哈希快速遍历桶中元素的缓存。

哈希表的每个桶都只能存储 8 个键值对，一旦当前哈希的某个桶超出 8 个，新的键值对就会存储到哈希的溢出桶中。

随着键值对数量的增加，溢出桶的数量和哈希的装载因子也会逐渐升高，超过一定范围就会触发扩容，扩容会将桶的数量翻倍，元素再分配的过程也是在调用写操作时增量进行的，不会造成性能的瞬时巨大抖动。

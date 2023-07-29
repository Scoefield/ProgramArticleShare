
## 前言

我们通常用 Golang 来开发并构建高并发场景下的服务，但是由于 Golang 内建的GC机制多少会影响服务的性能，因此，为了减少频繁GC，Golang提供了对象重用的机制，也就是使用sync.Pool构建对象池。

## sync.Pool介绍

首先sync.Pool是可伸缩的临时对象池，也是并发安全的。其可伸缩的大小会受限于内存的大小，可以理解为是一个存放可重用对象的容器。sync.Pool设计的目的就是用于存放已经分配的但是暂时又不用的对象，而且在需要用到的时候，可以直接从该pool中取。

pool中任何存放的值可以在任何时候被删除而不会收到通知。另外，在高负载下pool对象池可以动态的扩容，而在不使用或者说并发量不高时对象池会收缩。关键思想就是对象的复用，避免重复创建、销毁，从而影响性能。

个人觉得它的名字有一定的误导性，因为 Pool 里装的对象可以被无通知地被回收，觉得 sync.Cache 的名字更合适sync.Pool的命名。

sync.Pool首先声明了两个结构体，如下：
```go
// Local per-P Pool appendix.
type poolLocalInternal struct {
  private interface{} // Can be used only by the respective P.
  shared  poolChain   // Local P can pushHead/popHead; any P can popTail.
}

type poolLocal struct {
  poolLocalInternal

  // Prevents false sharing on widespread platforms with
  // 128 mod (cache line size) = 0 .
  pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}
```
为了使得可以在多个goroutine中高效的使用并发，sync.Pool会为每个P(对应CPU，这里有点像GMP模型)都分配一个本地池，当执行Get或者Put操作的时候，会先将goroutine和某个P的对象池关联，再对该池进行操作。

每个P的对象池分为私有对象和共享列表对象，私有对象只能被特定的P访问，共享列表对象可以被任何P访问。因为同一时刻一个P只能执行一个goroutine，所以无需加锁，但是对共享列表对象进行操作时，因为可能有多个goroutine同时操作，即并发操作，所以需要加锁。

需要注意的是 poolLocal 结构体中有个 pad 成员，其目的是为了防止false sharing。cache使用中常见的一个问题是false sharing。当不同的线程同时读写同一个 cache line上不同数据时就可能发生false sharing。false sharing会导致多核处理器上严重的系统性能下降。具体的解释说明这里就不展开赘述了。

## sync.Pool的Put和Get方法

sync.Pool 有两个公开的方法，一个是Get，另一个是Put。

### Put方法

我们先来看一下Put方法的源码，如下：
```go
// Put adds x to the pool.
func (p *Pool) Put(x interface{}) {
  if x == nil {
    return
  }
  if race.Enabled {
    if fastrand()%4 == 0 {
      // Randomly drop x on floor.
      return
    }
    race.ReleaseMerge(poolRaceAddr(x))
    race.Disable()
  }
  l, _ := p.pin()
  if l.private == nil {
    l.private = x
    x = nil
  }
  if x != nil {
    l.shared.pushHead(x)
  }
  runtime_procUnpin()
  if race.Enabled {
    race.Enable()
  }
}
```

阅读以上Put方法的源码可以知道：
- 如果Put放入的值为空，则直接 return 了，不会执行下面的逻辑了；
- 如果不为空，则继续检查当前goroutine的private是否设置对象池私有值，如果没有则将x赋值给该私有成员，并将x设置为nil；
- 如果当前goroutine的private私有值已经被赋值过了，那么将该值追加到共享列表。

### Get方法

我们再来看下Get方法的源码，如下：
```go
func (p *Pool) Get() interface{} {
  if race.Enabled {
    race.Disable()
  }
  l, pid := p.pin()
  x := l.private
  l.private = nil
  if x == nil {
    // Try to pop the head of the local shard. We prefer
    // the head over the tail for temporal locality of
    // reuse.
    x, _ = l.shared.popHead()
    if x == nil {
      x = p.getSlow(pid)
    }
  }
  runtime_procUnpin()
  if race.Enabled {
    race.Enable()
    if x != nil {
      race.Acquire(poolRaceAddr(x))
    }
  }
  if x == nil && p.New != nil {
    x = p.New()
  }
  return x
}
```
阅读以上Get方法的源码，可以知道：
- 首先尝试从本地P对应的那个对象池中获取一个对象值, 并从对象池中删掉该值。
- 如果从本地对象池中获取失败，则从共享列表中获取，并从共享列表中删除该值。
- 如果从共享列表中获取失败，则会从其它P的对象池中“偷”一个过来，并删除共享池中的该值（就是源码中14行的p.getSlow()）。
- 如果还是失败，那么直接通过 New() 分配一个返回值，注意这个分配的值不会被放入对象池中。New()是返回用户注册的New函数的值，如果用户未注册New，那么默认返回nil。

### init函数
最后我们来看一下init函数，如下：
```go
func init() {
  funtime_registerPoolCleanup(poolCleanup)
}
```
可以看到在init的时候注册了一个PoolCleanup函数，他会清除掉sync.Pool中的所有的缓存的对象，这个注册函数会在每次GC的时候运行，所以sync.Pool中的值只在两次GC中间的时段有效。

## sync.Pool使用示例

示例代码：
```go
package main
import (
	"fmt"
	"sync"
)
// 定义一个 Person 结构体，有Name和Age变量
type Person struct {
	Name string
	Age int
}
// 初始化sync.Pool，new函数就是创建Person结构体
func initPool() *sync.Pool {
	return &sync.Pool{
		New: func() interface{} {
			fmt.Println("创建一个 person.")
			return &Person{}
		},
	}
}
// 主函数，入口函数
func main() {
	pool := initPool()
	person := pool.Get().(*Person)
	fmt.Println("首次从sync.Pool中获取person：", person)
	person.Name = "Jack"
	person.Age = 23
	pool.Put(person)
	fmt.Println("设置的对象Name: ", person.Name)
	fmt.Println("设置的对象Age: ", person.Age)
	fmt.Println("Pool 中有一个对象，调用Get方法获取：", pool.Get().(*Person))
	fmt.Println("Pool 中没有对象了，再次调用Get方法：", pool.Get().(*Person))
}
```
运行结果如下所示：
```go
创建一个 person.
首次从sync.Pool中获取person：&{ 0}
设置的对象Name:  Jack
设置的对象Age:  23
Pool 中有一个对象，调用Get方法获取：&{Jack 23}
创建一个 person.
Pool 中没有对象了，再次调用Get方法： &{ 0}
```

## 总结

通过以上的源码及其示例，我们可以知道：
- Get方法并不会对获取到的对象值做任何的保证，因为放入本地对象池中的值有可能会在任何时候被删除，而得不到通知。
- 放入共享池中的值有可能被其他的goroutine拿走，所以对象池比较适合用来存储一些临时切状态无关的数据，但是不适合用来存储数据库连接的实例，因为存入对象池的值有可能会在垃圾回收时被删除掉，这违反了数据库连接池建立的初衷。

由此可知，Golang的对象池严格意义上来说是一个临时的对象池，适用于储存一些会在goroutine间分享的临时对象。主要作用是减少GC，提高性能。在Golang中最常见的使用场景就是fmt包中的输出缓冲区了。


代码Github归档地址：
[sync.Pool使用示例代码](https://github.com/Scoefield/gokeyboardman/blob/main/syncpool/)

推荐阅读：[红黑树详解及其实现](https://zhuanlan.zhihu.com/p/368944960)

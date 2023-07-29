# 如果不控制 Goroutine 的数量会出什么问题？

首先我们都知道 Goroutine 具备以下两个特点：

- 体积轻量（占内存小，一个 2kb 左右）
- 优秀的 GMP 调度（详见：[图解 Golang 的 GMP 原理与调度流程](https://juejin.cn/post/6995091405563494431)）

那么 goroutine 是否真的可以无限制的开启呢？如果做一个服务器或者一些高业务的场景，能否随意的开启 goroutine 并且没有控制控制其生命周期呢？让这些 goroutine 自生自灭，相信大都数人都会这么想：毕竟有强大的 GC 和优秀的调度算法支撑，应该可以无限的开启吧。

我们先看下面一个例子：

`demo1.go`

```go
package main

import (
    "fmt"
    "math"
    "runtime"
)

func main() {
    //模拟用户需求业务的数量
    task_cnt := math.MaxInt64

    for i := 0; i < task_cnt; i++ {
        go func(i int) {
            //... do some busi...

            fmt.Println("go func ", i, " goroutine count = ", runtime.NumGoroutine())
        }(i)
    }
}
```

结果最后被操作系统以 `kill` 信号，强制终结了该进程。

```
signal: killed
```

可以看出，我们如果迅速的开启 goroutine (**不控制并发的 goroutine 数量** )的话，会在短时间内占据操作系统的资源(CPU、内存、文件描述符等)。

- CPU 使用率浮动上涨
- Memory 占用不断上涨
- 主进程崩溃（被杀掉了）

这些资源实际上是所有用户态程序共享的资源，所以大批的 goroutine 开启最终引发的问题不仅仅是自身，还会影响到其他运行的程序。

综上所述，所以我们在平时开发中编写代码的时候，需要注意代码中开启的 goroutine 数量，以及开启的 goroutine 是否可以控制，必须要重视的问题。

# 控制 goroutine 的几种方法

## 方法一：用有 buffer 的 channel 来限制

```go
package main

import (
	"fmt"
	"math"
	"runtime"
)

// 模拟执行业务的 goroutine
func doBusiness(ch chan bool, i int) {
	fmt.Println("go func:", i, "goroutine count:", runtime.NumGoroutine())
	<-ch
}

func main() {
	max_cnt := math.MaxInt64
	// max_cnt := 10
	fmt.Println(max_cnt)

	ch := make(chan bool, 3)

	for i := 0; i < max_cnt; i++ {
		ch <- true
		go doBusiness(ch, i)
	}
}
```

结果：

```
...
go func  352283  goroutine count =  4
go func  352284  goroutine count =  4
go func  352285  goroutine count =  4
go func  352286  goroutine count =  4
go func  352287  goroutine count =  4
go func  352288  goroutine count =  4
go func  352289  goroutine count =  4
go func  352290  goroutine count =  4
go func  352291  goroutine count =  4
go func  352292  goroutine count =  4
go func  352293  goroutine count =  4
go func  352294  goroutine count =  4
go func  352295  goroutine count =  4
go func  352296  goroutine count =  4
go func  352297  goroutine count =  4
go func  352298  goroutine count =  4
go func  352299  goroutine count =  4
go func  352300  goroutine count =  4
go func  352301  goroutine count =  4
go func  352302  goroutine count =  4
...
```

从结果看，程序并没有出现崩溃，而是按部就班的顺序执行，并且 go 的数量控制在了 3，(4 的原因是因为还有一个 main goroutine)那么从数字上看，是不是在跑的 goroutines 有几十万个呢？
![goroutines1.jpeg](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/45ee48b6460a4a819cebbe547e2227b3~tplv-k3u1fbpfcp-watermark.image?)

这里我们用了 buffer 为 3 的 channel, 在代码往 channel 写的过程中，实际上是限制了速度的：

```go
 for i := 0; i < go_cnt; i++ {
        ch <- true
        go busi(ch, i)
}
```

`for` 循环里 `ch<- true` 的速度，因为这个速度决定了 go 的创建速度，而 go 的结束速度取决于  `doBusiness()` 函数的执行速度。这样实际上，我们就能够保证了，同一时间内运行的 goroutine 的数量与 buffer 的数量一致，从而达到了限定效果。

但是这段代码有一个小问题，就是如果我们把 go_cnt 的数量变的小一些，会出现打出的结果不正确。

```go
package main

import (
	"fmt"
	// "math"
	"runtime"
)

func doBusiness(ch chan bool, i int) {
	fmt.Println("go func:", i, "goroutine count:", runtime.NumGoroutine())
	<-ch
}

func main() {
	// max_cnt := math.MaxInt64
	max_cnt := 10

	ch := make(chan bool, 3)

	for i := 0; i < max_cnt; i++ {
		ch <- true
		go doBusiness(ch, i)
	}
}
```

结果：

```
go func  2  goroutine count =  4
go func  3  goroutine count =  4
go func  4  goroutine count =  4
go func  5  goroutine count =  4
go func  6  goroutine count =  4
go func  1  goroutine count =  4
go func  8  goroutine count =  4
```

可以看到有些 goroutine 没有打印出来，是由于 main 把所有 goroutine 开启之后，main 就直接退出了，我们知道 main 进程退出，低下所有的 goroutine 都会结束掉，从而导致有些 goroutine 还没来得及执行就退出了。所以想全部 go 都执行，需要在 main 的最后进行阻塞操作。

## 方法二：使用 sync 同步机制

```
import (
    "fmt"
    "math"
    "sync"
    "runtime"
)

var wg = sync.WaitGroup{}

func doBusiness(i int) {

    fmt.Println("go func ", i, " goroutine count = ", runtime.NumGoroutine())
    wg.Done()
}

func main() {
    //模拟用户需求业务的数量
    task_cnt := math.MaxInt64


    for i := 0; i < task_cnt; i++ {
        wg.Add(1)
        go doBusiness(i)
    }

    wg.Wait()
}
```

很明显，如果单纯的使用 `sync` 也达不到控制 goroutine 的数量，最终结果依然是崩溃。

## 方法三：channel 与 sync 同步组合方式实现控制 goroutine

```
package main

import (
    "fmt"
    "math"
    "sync"
    "runtime"
)

var wg = sync.WaitGroup{}

func doBusiness(ch chan bool, i int) {

    fmt.Println("go func ", i, " goroutine count = ", runtime.NumGoroutine())

    <-ch

    wg.Done()
}

func main() {
    //模拟用户需求go业务的数量
    task_cnt := math.MaxInt64

    ch := make(chan bool, 3)

    for i := 0; i < task_cnt; i++ {
        wg.Add(1)
        ch <- true
        go doBusiness(ch, i)
    }

      wg.Wait()
}
```

结果：

```
//...
go func  228856  goroutine count =  4
go func  228857  goroutine count =  4
go func  228858  goroutine count =  4
go func  228859  goroutine count =  4
go func  228860  goroutine count =  4
go func  228861  goroutine count =  4
go func  228862  goroutine count =  4
go func  228863  goroutine count =  4
go func  228864  goroutine count =  4
go func  228865  goroutine count =  4
go func  228866  goroutine count =  4
go func  228867  goroutine count =  4
//...
```

这样我们程序就不会再造成资源爆炸而崩溃。而且运行 go 的数量控制住了在 buffer 为 3 的这个范围内。

## 方法四：利用无缓冲 channel 与任务发送/执行分离方式

```
package main

import (
    "fmt"
    "math"
    "sync"
    "runtime"
)

var wg = sync.WaitGroup{}

func doBusiness(ch chan int) {

    for t := range ch {
        fmt.Println("go task = ", t, ", goroutine count = ", runtime.NumGoroutine())
        wg.Done()
    }
}

func sendTask(task int, ch chan int) {
    wg.Add(1)
    ch <- task
}

func main() {

    ch := make(chan int)   //无buffer channel

    goCnt := 3              //启动goroutine的数量
    for i := 0; i < goCnt; i++ {
        //启动go
        go doBusiness(ch)
    }

    taskCnt := math.MaxInt64 //模拟用户需求业务的数量
    for t := 0; t < taskCnt; t++ {
        //发送任务
        sendTask(t, ch)
    }

    wg.Wait()
}
```

执行流程大致如下，这里实际上是将任务的发送和执行做了业务上的分离。使得消息出去，输入 SendTask 的频率可设置、执行 Goroutine 的数量也可设置。也就是既控制输入(生产)，又控制输出(消费)。使得可控更加灵活。这也是很多 Go 框架的 Worker 工作池的最初设计思想理念。如下图：
![goroutines2.jpeg](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/399d58b6a2f24f6887d2d5af108ac95b~tplv-k3u1fbpfcp-watermark.image?)

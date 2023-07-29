## Golang “调度器” 的由来？

### 单进程时代没有调度器

我们知道，一切的软件都是跑在操作系统上，真正用来干活 (计算) 的是 CPU。早期的操作系统每个程序就是一个进程，直到一个程序运行完，才能进行下一个进程，这就是“单进程时代”。

![单进程.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2e79d7e365d448898b1792efe4cfdadb~tplv-k3u1fbpfcp-watermark.image)

早期的单进程操作系统，面临 2 个问题：

1.  单一的执行流程，计算机只能一个任务一个任务处理。
1.  进程阻塞所带来的 CPU 时间浪费。

那么能不能有多个进程来宏观一起来执行多个任务呢？

后来操作系统就具有了最早的并发能力：多进程并发，当一个进程阻塞的时候，切换到另外等待执行的进程，这样就能尽量把 CPU 利用起来，CPU 就不浪费了。

### 多进程/多线程时期的调度器

![多进程.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d060f0e3fcf8467aa3d2a08f1e963fd6~tplv-k3u1fbpfcp-watermark.image)

在多进程 / 多线程的操作系统中，就解决了阻塞的问题，因为一个进程阻塞 cpu 可以立刻切换到其他进程中去执行，而且调度 cpu 的算法可以保证在运行的进程都可以被分配到 cpu 的运行时间片。这样从宏观来看，似乎多个进程是在同时被运行。

但新的问题就又出现了，进程拥有太多的资源，进程的创建、切换、销毁，都会占用很长的时间，CPU 虽然利用起来了，但如果进程过多，CPU 有很大的一部分都被用来进行进程调度了。

怎么才能提高 CPU 的利用率呢？

![cpu成本.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7a247e1b1daf40b7aa240b723f4e4a98~tplv-k3u1fbpfcp-watermark.image)

很明显，CPU 调度切换的是进程和线程。尽管线程看起来很美好，但实际上多线程开发设计会变得更加复杂，要考虑很多同步竞争等问题，如锁、竞争冲突等。

### 协程来提高 CPU 利用率

多进程、多线程已经提高了系统的并发能力，但是在当今互联网高并发场景下，为每个任务都创建一个线程是不现实的，因为会消耗大量的内存 (进程虚拟内存会占用 4GB [32 位操作系统], 而线程也要大约 4MB)。

大量的进程 / 线程出现了新的问题

- 高内存占用
- 调度的高消耗 CPU

然后工程师们就发现，其实一个线程分为 “内核态 “线程和” 用户态 “线程。

一个 “用户态线程” 必须要绑定一个 “内核态线程”，但是 CPU 并不知道有 “用户态线程” 的存在，它只知道它运行的是一个 “内核态线程”(Linux 的 PCB 进程控制块)。

这样，我们再去细化去分类一下，内核线程依然叫 “线程 (thread)”，用户线程叫 “协程 (co-routine)”。

#### N:M 的协程与线程的映射关系

![n对m的协程与线程.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a22b3af8462e41f8adb601eb34c22bb6~tplv-k3u1fbpfcp-watermark.image)

协程跟线程是有区别的，线程由 CPU 调度是抢占式的，协程由用户态调度是协作式的，一个协程让出 CPU 后，才执行下一个协程。

### 协程 goroutine

Go 为了提供更容易使用的并发方法，使用了 goroutine 和 channel。goroutine 来自协程的概念，让一组可复用的函数运行在一组线程之上，即使有协程阻塞，该线程的其他协程也可以被 runtime 调度，转移到其他可运行的线程上。最关键的是，程序员看不到这些底层的细节，这就降低了编程的难度，提供了更容易的并发。

Go 中，协程被称为 goroutine，它非常轻量，一个 goroutine 只占几 KB，并且这几 KB 就足够 goroutine 运行完，这就能在有限的内存空间内支持大量 goroutine，支持了更多的并发。虽然一个 goroutine 的栈只占几 KB，但实际是可伸缩的，如果需要更多内容，runtime 会自动为 goroutine 分配。

Goroutine 特点：

- 占用内存更小（几 kb）
- 调度更灵活 (runtime 调度)

### Goroutine 调度器 GMP 模型

在 Go 中，线程是运行 goroutine 的实体，调度器的功能是把可运行的 goroutine 分配到工作线程上。

![gmp.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dc5cebec41054ff7a478736157c9669e~tplv-k3u1fbpfcp-watermark.image)

- 全局队列（Global Queue）：存放等待运行的 G。
- P 的本地队列：同全局队列类似，存放的也是等待运行的 G，存的数量有限，不超过 256 个。新建 G’时，G’优先加入到 P 的本地队列，如果队列满了，则会把本地队列中一半的 G 移动到全局队列。
- P 列表：所有的 P 都在程序启动时创建，并保存在数组中，最多有 GOMAXPROCS(可配置) 个。
- M：线程想运行任务就得获取 P，从 P 的本地队列获取 G，P 队列为空时，M 也会尝试从全局队列拿一批 G 放到 P 的本地队列，或从其他 P 的本地队列偷一半放到自己 P 的本地队列。M 运行 G，G 执行之后，M 会从 P 获取下一个 G，不断重复下去。

### 开启一个 Goroutine 的调度流程

#### go func 创建和执行的流程

![gmp2.jpeg](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/60ea248dd32a41818640007eaa69ced7~tplv-k3u1fbpfcp-watermark.image)

从上图我们可以分析出几个结论：

​ 1、我们通过 go func () 来创建一个 goroutine；

​ 2、有两个存储 G 的队列，一个是局部调度器 P 的本地队列、一个是全局 G 队列。新创建的 G 会先保存在 P 的本地队列中，如果 P 的本地队列已经满了就会保存在全局的队列中；

​ 3、G 只能运行在 M 中，一个 M 必须持有一个 P，M 与 P 是 1：1 的关系。M 会从 P 的本地队列弹出一个可执行状态的 G 来执行，如果 P 的本地队列为空，就会想其他的 MP 组合偷取一个可执行的 G 来执行；

​ 4、一个 M 调度 G 执行的过程是一个循环机制；

​ 5、当 M 执行某一个 G 时候如果发生了 syscall 或则其余阻塞操作，M 会阻塞，如果当前有一些 G 在执行，runtime 会把这个线程 M 从 P 中摘除 (detach)，然后再创建一个新的操作系统的线程 (如果有空闲的线程可用就复用空闲线程) 来服务于这个 P；

​ 6、当 M 系统调用结束时候，这个 G 会尝试获取一个空闲的 P 执行，并放入到这个 P 的本地队列。如果获取不到 P，那么这个线程 M 变成休眠状态， 加入到空闲线程中，然后这个 G 会被放入全局队列中。

## Go 协程调度器各场景调度过程解析

### 场景一：G1 创建 G2

P 拥有 G1，M1 获取 P 后开始运行 G1，G1 使用 go func() 创建了 G2，为了局部性 G2 优先加入到 P1 的本地队列。

![场景一：G1创建G2.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bb3dee189c8349ff869926dfba468ec0~tplv-k3u1fbpfcp-watermark.image)

### 场景二：G1 执行完毕

G1 运行完成后 (函数：goexit)，M 上运行的 goroutine 切换为 G0，G0 负责调度时协程的切换（函数：schedule）。从 P 的本地队列取 G2，从 G0 切换到 G2，并开始运行 G2 (函数：execute)。实现了线程 M1 的复用。

![场景二：G1执行完毕.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7093382a9f7e43fbb113d03de121b72c~tplv-k3u1fbpfcp-watermark.image)

### 场景三：G2 创建过多的 Goroutine

假设每个 P 的本地队列只能存 3 个 G。G2 要创建了 6 个 G，前 3 个 G（G3, G4, G5）已经加入 p1 的本地队列，p1 本地队列满了。

![场景三：G2创建过多的G.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/854d319366ed43958c6ef07084f70d29~tplv-k3u1fbpfcp-watermark.image)

### 场景四：G2 本地队列满再创建 Goroutine

G2 在创建 G7 的时候，发现 P1 的本地队列已满，需要执行负载均衡 (把 P1 中本地队列中前一半的 G，还有新创建 G 转移到全局队列)

> （实现中并不一定是新的 G，如果 G 是 G2 之后就执行的，会被保存在本地队列，利用某个老的 G 替换新 G 加入全局队列）

![场景四：G2本地队列满再创建G.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3ca1b95d62d0487ba7e5f77116e18f92~tplv-k3u1fbpfcp-watermark.image)

这些 G 被转移到全局队列时，会被打乱顺序。所以 G3,G4,G7 被转移到全局队列。

### 场景六：唤醒正在休眠的 M

规定：在创建 G 时，运行的 G 会尝试唤醒其他空闲的 P 和 M 组合去执行。

![场景五：唤醒正在休眠的G.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c498384b568848bd998106c68f07196e~tplv-k3u1fbpfcp-watermark.image)

假定 G2 唤醒了 M2，M2 绑定了 P2，并运行 G0，但 P2 本地队列没有 G，M2 此时为自旋线程（没有 G 但为运行状态的线程，不断寻找 G）。

### 场景七：被唤醒的 M2 从全局队列取 G

M2 尝试从全局队列 (简称 “GQ”) 取一批 G 放到 P2 的本地队列（函数：findrunnable()）。M2 从全局队列取的 G 数量符合下面的公式：

> n = min(len(GQ)/GOMAXPROCS + 1, len(GQ/2))

至少从全局队列取 1 个 g，但每次不要从全局队列移动太多的 g 到 p 本地队列，给其他 p 留点。这是从全局队列到 P 本地队列的负载均衡。

![场景七：唤醒的去全局队列拿.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aef6b72c6096439188f4df430af6a93b~tplv-k3u1fbpfcp-watermark.image)

### 场景八：M2 从 M1 中偷取 G

假设 G2 一直在 M1 上运行，经过 2 轮后，M2 已经把 G7、G4 从全局队列获取到了 P2 的本地队列并完成运行，全局队列和 P2 的本地队列都空了，如场景 7 图的左半部分。

![场景八：M2从M1中偷取.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b85aa2809b2a40f686cea58b4a9bc404~tplv-k3u1fbpfcp-watermark.image)

全局队列已经没有 G，那 m 就要执行 work stealing (偷取)：从其他有 G 的 P 哪里偷取一半 G 过来，放到自己的 P 本地队列。P2 从 P1 的本地队列尾部取一半的 G，本例中一半则只有 1 个 G8，放到 P2 的本地队列并执行。

### 场景九：自旋的最大限制

G1 本地队列 G5、G6 已经被其他 M 偷走并运行完成，当前 M1 和 M2 分别在运行 G2 和 G8，M3 和 M4 没有 goroutine 可以运行，M3 和 M4 处于自旋状态，它们不断寻找 goroutine。

![场景九：自旋的最大限制.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/93ba0b8493184248aa7250b7cb044bbe~tplv-k3u1fbpfcp-watermark.image)

为什么要让 m3 和 m4 自旋，自旋本质是在运行，线程在运行却没有执行 G，就变成了浪费 CPU。

为什么不销毁现场，来节约 CPU 资源。因为创建和销毁 CPU 也会浪费时间，我们希望当有新 goroutine 创建时，立刻能有 M 运行它，如果销毁再新建就增加了时延，降低了效率。当然也考虑了过多的自旋线程是浪费 CPU，所以系统中最多有 GOMAXPROCS 个自旋的线程 (当前例子中的 GOMAXPROCS=4，所以一共 4 个 P)，多余的没事做线程会让他们休眠。

### 场景十：G 发生系统调用/阻塞

假定当前除了 M3 和 M4 为自旋线程，还有 M5 和 M6 为空闲的线程 (没有得到 P 的绑定，注意我们这里最多就只能够存在 4 个 P，所以 P 的数量应该永远是 M>=P, 大部分都是 M 在抢占需要运行的 P)，G8 创建了 G9，G8 进行了阻塞的系统调用，M2 和 P2 立即解绑，P2 会执行以下判断：如果 P2 本地队列有 G、全局队列有 G 或有空闲的 M，P2 都会立马唤醒 1 个 M 和它绑定，否则 P2 则会加入到空闲 P 列表，等待 M 来获取可用的 p。本场景中，P2 本地队列有 G9，可以和其他空闲的线程 M5 绑定。

![场景十：阻塞解绑.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/06864438875d41f99f4a1be858bf2567~tplv-k3u1fbpfcp-watermark.image)

### 场景十一：G 发生系统调用/非阻塞

G8 创建了 G9，假如 G8 进行了非阻塞系统调用。

![场景十一：非阻塞解绑.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/780f5d827b3b44a8a85a54a252bd6540~tplv-k3u1fbpfcp-watermark.image)

M2 和 P2 会解绑，但 M2 会记住 P2，然后 G8 和 M2 进入系统调用状态。当 G8 和 M2 退出系统调用时，会尝试获取 P2，如果无法获取，则获取空闲的 P，如果依然没有，G8 会被记为可运行状态，并加入到全局队列，M2 因为没有 P 的绑定而变成休眠状态 (长时间休眠等待 GC 回收销毁）。

## 总结

Go 调度器很轻量也很简单，足以撑起 goroutine 的调度工作，并且让 Go 具有了原生（强大）并发的能力。Go 调度本质是把大量的 goroutine 分配到少量线程上去执行，并利用多核并行，实现更强大的并发。

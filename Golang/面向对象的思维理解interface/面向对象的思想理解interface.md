---
highlight: a11y-dark
---

# Golang interface 介绍

interface 是 GO 语言的基础特性之一。可以理解为一种类型的规范或者约定。它跟 java，C# 不太一样，不需要显示说明实现了某个接口，它没有继承或子类或“implements”关键字，只是通过约定的形式，隐式的实现 interface 中的方法即可。因此，Golang 中的 interface 让编码更灵活、易扩展。

```
如何理解go 语言中的interface ？ 只需记住以下三点即可：
```

1. interface 是方法声明的集合
2. 任何类型的对象实现了在 interface 接口中声明的全部方法，则表明该类型实现了该接口。
3. interface 可以作为一种数据类型，实现了该接口的任何对象都可以给对应的接口类型变量赋值。

**注意：**

- interface 可以被任意对象实现，一个类型/对象也可以实现多个 interface；
- 方法不能重载，如  `eat(), eat(s string)`  不能同时存在。

```go
package main

import "fmt"

type Phone interface {
    call()
}

type NokiaPhone struct {
}

func (nokiaPhone NokiaPhone) call() {
    fmt.Println("I am Nokia, I can call you!")
}

type ApplePhone struct {
}

func (iPhone ApplePhone) call() {
    fmt.Println("I am Apple Phone, I can call you!")
}

func main() {
    var phone Phone
    phone = new(NokiaPhone)
    phone.call()

    phone = new(ApplePhone)
    phone.call()
}
```

上述中体现了`interface`接口的语法，在`main`函数中，也体现了`多态`的特性。\
同样一个`phone`的抽象接口，分别指向不同的实体对象，调用的 call()方法，打印的效果不同，那么就是体现出了多态的特性。

# 面向对象 - 开闭原则

## 平铺式的模块设计

那么作为`interface`数据类型，他存在的意义在哪呢？ 实际上是为了满足一些面向对象的编程思想。我们知道，软件设计的最高目标就是`高内聚，低耦合`。那么其中有一个设计原则叫`开闭原则`。什么是开闭原则呢，接下来我们看一个例子：

```go
package main

import "fmt"

//我们要写一个类,Banker银行业务员
type Banker struct {
}

//存款业务
func (this *Banker) Save() {
    fmt.Println( "进行了 存款业务...")
}

//转账业务
func (this *Banker) Transfer() {
    fmt.Println( "进行了 转账业务...")
}

//支付业务
func (this *Banker) Pay() {
    fmt.Println( "进行了 支付业务...")
}

func main() {
    banker := &Banker{}

    banker.Save()
    banker.Transfer()
    banker.Pay()
}
```

代码很简单，就是一个银行业务员，他可能拥有很多的业务，比如`Save()`存款、`Transfer()`转账、`Pay()`支付等。那么如果这个业务员模块只有这几个方法还好，但是随着我们的程序写的越来越复杂，银行业务员可能就要增加方法，会导致业务员模块越来越臃肿。

![interface1.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a6425f3b757f487caa66fb6d7fe00ef3~tplv-k3u1fbpfcp-watermark.image?)

这样的设计会导致，当我们去给 Banker 添加新的业务的时候，会直接修改原有的 Banker 代码，那么 Banker 模块的功能会越来越多，出现问题的几率也就越来越大，假如此时 Banker 已经有 99 个业务了，现在我们要添加第 100 个业务，可能由于一次的不小心，导致之前 99 个业务也一起崩溃，因为所有的业务都在一个 Banker 类里，他们的耦合度太高，Banker 的职责也不够单一，代码的维护成本随着业务的复杂正比成倍增大。

## 开闭原则设计

那么，如果我们拥有接口, `interface`这个东西，那么我们就可以抽象一层出来，制作一个抽象的 Banker 模块，然后提供一个抽象的方法。 分别根据这个抽象模块，去实现`支付Banker（实现支付方法）`,`转账Banker（实现转账方法）`，如下：

![interface2.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/62a34ebbc4294487a05d9a90aab48943~tplv-k3u1fbpfcp-watermark.image?)

那么依然可以搞定程序的需求。 然后，当我们想要给 Banker 添加额外功能的时候，之前我们是直接修改 Banker 的内容，现在我们可以单独定义一个`股票Banker(实现股票方法)`，到这个系统中。 而且股票 Banker 的实现成功或者失败都不会影响之前的稳定系统，他很单一，而且独立。

所以以上，当我们给一个系统添加一个功能的时候，不是通过修改代码，而是通过增添代码来完成，那么就是开闭原则的核心思想了。所以要想满足上面的要求，是一定需要 interface 来提供一层抽象的接口的。

golang 代码实现如下:

```go
package main

import "fmt"

//抽象的银行业务员
type AbstractBanker interface{
    DoBusi()    //抽象的处理业务接口
}

//存款的业务员
type SaveBanker struct {
    //AbstractBanker
}

func (sb *SaveBanker) DoBusi() {
    fmt.Println("进行了存款")
}

//转账的业务员
type TransferBanker struct {
    //AbstractBanker
}

func (tb *TransferBanker) DoBusi() {
    fmt.Println("进行了转账")
}

//支付的业务员
type PayBanker struct {
    //AbstractBanker
}

func (pb *PayBanker) DoBusi() {
    fmt.Println("进行了支付")
}


func main() {
    //进行存款
    sb := &SaveBanker{}
    sb.DoBusi()

    //进行转账
    tb := &TransferBanker{}
    tb.DoBusi()

    //进行支付
    pb := &PayBanker{}
    pb.DoBusi()

}
```

当然我们也可以根据`AbstractBanker`设计一个小框架:

```go
//实现架构层(基于抽象层进行业务封装-针对interface接口进行封装)
func BankerBusiness(banker AbstractBanker) {
    //通过接口来向下调用，(多态现象)
    banker.DoBusi()
}
```

那么 main 中可以如下实现业务调用:

```go
func main() {
    //进行存款
    BankerBusiness(&SaveBanker{})

    //进行存款
    BankerBusiness(&TransferBanker{})

    //进行存款
    BankerBusiness(&PayBanker{})
}
```

```
再看开闭原则定义:
开闭原则:一个软件实体如类、模块和函数应该对扩展开放，对修改关闭。
简单的说就是在修改需求的时候，应该尽量通过扩展来实现变化，而不是通过修改已有代码来实现变化。
```

# interface 的意义

好了，现在 interface 已经基本了解，那么接口的意义最终在哪里呢，想必现在你已经有了一个初步的认知，实际上接口的最大的意义就是实现多态的思想，就是我们可以根据 interface 类型来设计 API 接口，那么这种 API 接口的适应能力不仅能适应当下所实现的全部模块，也适应未来实现的模块来进行调用。 `调用未来`可能就是接口的最大意义所在吧，这也是为什么架构师那么值钱，因为良好的架构师是可以针对 interface 设计一套框架，在未来许多年却依然适用。

# 面向对象中的依赖倒转原则

## 耦合度高的设计举例

![interface3.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1e5370d5e28a4574b69fd9c87313a555~tplv-k3u1fbpfcp-watermark.image?)

实现代码如下：

```go
package main

import "fmt"

// === > 奔驰汽车 <===
type Benz struct {

}

func (this *Benz) Run() {
    fmt.Println("Benz is running...")
}

// === > 宝马汽车  <===
type BMW struct {

}

func (this *BMW) Run() {
    fmt.Println("BMW is running ...")
}


//===> 司机张三  <===
type Zhang3 struct {
    //...
}

func (zhang3 *Zhang3) DriveBenZ(benz *Benz) {
    fmt.Println("zhang3 Drive Benz")
    benz.Run()
}

func (zhang3 *Zhang3) DriveBMW(bmw *BMW) {
    fmt.Println("zhang3 drive BMW")
    bmw.Run()
}

//===> 司机李四 <===
type Li4 struct {
    //...
}

func (li4 *Li4) DriveBenZ(benz *Benz) {
    fmt.Println("li4 Drive Benz")
    benz.Run()
}

func (li4 *Li4) DriveBMW(bmw *BMW) {
    fmt.Println("li4 drive BMW")
    bmw.Run()
}

func main() {
    //业务1 张3开奔驰
    benz := &Benz{}
    zhang3 := &Zhang3{}
    zhang3.DriveBenZ(benz)

    //业务2 李四开宝马
    bmw := &BMW{}
    li4 := &Li4{}
    li4.DriveBMW(bmw)
}
```

我们来看上面的代码和图中每个模块之间的依赖关系，实际上并没有用到任何的`interface`接口层的代码，显然最后我们的两个业务  `张三开奔驰`, `李四开宝马`，程序中也都实现了。但是这种设计的问题就在于，小规模没什么问题，但是一旦程序需要扩展，比如我现在要增加一个`丰田汽车`  或者 司机`王五`， 那么模块和模块的依赖关系将成指数级递增，想蜘蛛网一样越来越难维护和捋顺。

## 耦合度低的面向抽象编程设计

![interface4.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1b818e6a68a3479e83a8fe2bad6d0cc0~tplv-k3u1fbpfcp-watermark.image?)

如上图所示，如果我们在设计一个系统的时候，将模块分为 3 个层次，抽象层、实现层、业务逻辑层。那么，我们首先将抽象层的模块和接口定义出来，这里就需要了`interface`接口的设计，然后我们依照抽象层，依次实现每个实现层的模块，在我们写实现层代码的时候，实际上我们只需要参考对应的抽象层实现就好了，实现每个模块，也和其他的实现的模块没有关系，这样也符合了上面介绍的开闭原则。这样实现起来每个模块只依赖对象的接口，而和其他模块没关系，依赖关系单一。系统容易扩展和维护。

我们在指定业务逻辑也是一样，只需要参考抽象层的接口来业务就好了，抽象层暴露出来的接口就是我们业务层可以使用的方法，然后可以通过多态的线下，接口指针指向哪个实现模块，调用了就是具体的实现方法，这样我们业务逻辑层也是依赖抽象成编程。

我们就将这种的设计原则叫做`依赖倒转原则`。

来一起看一下修改的代码：

```go
package main

import "fmt"

// ===== >   抽象层  < ========
type Car interface {
    Run()
}

type Driver interface {
    Drive(car Car)
}

// ===== >   实现层  < ========
type BenZ struct {
    //...
}

func (benz * BenZ) Run() {
    fmt.Println("Benz is running...")
}

type Bmw struct {
    //...
}

func (bmw * Bmw) Run() {
    fmt.Println("Bmw is running...")
}

type Zhang_3 struct {
    //...
}

func (zhang3 *Zhang_3) Drive(car Car) {
    fmt.Println("Zhang3 drive car")
    car.Run()
}

type Li_4 struct {
    //...
}

func (li4 *Li_4) Drive(car Car) {
    fmt.Println("li4 drive car")
    car.Run()
}


// ===== >   业务逻辑层  < ========
func main() {
    //张3 开 宝马
    var bmw Car
    bmw = &Bmw{}

    var zhang3 Driver
    zhang3 = &Zhang_3{}

    zhang3.Drive(bmw)

    //李4 开 奔驰
    var benz Car
    benz = &BenZ{}

    var li4 Driver
    li4 = &Li_4{}

    li4.Drive(benz)
}
```

## 举一反三 - 练习题

```
模拟组装2台电脑:
— 抽象层: 有显卡Card 方法display，有内存Memory 方法storage，有处理器CPU 方法calculate;
— 实现层层: 有 Intel因特尔公司 、产品有(显卡、内存、CPU)，有 Kingston 公司， 产品有(内存3)，有 NVIDIA 公司， 产品有(显卡);
— 逻辑层: 1.组装一台Intel系列的电脑，并运行，2.组装一台 Intel CPU Kingston内存 NVIDIA显卡的电脑，并运行。
```

golang 代码实现如下：

```go
/*
    模拟组装2台电脑
    --- 抽象层 ---
    有显卡Card  方法display
    有内存Memory 方法storage
    有处理器CPU   方法calculate

    --- 实现层层 ---
    有 Intel因特尔公司 、产品有(显卡、内存、CPU)
    有 Kingston 公司， 产品有(内存3)
    有 NVIDIA 公司， 产品有(显卡)

    --- 逻辑层 ---
    1. 组装一台Intel系列的电脑，并运行
    2. 组装一台 Intel CPU  Kingston内存 NVIDIA显卡的电脑，并运行
*/
package main

import "fmt"

//------  抽象层 -----
type Card interface{
    Display()
}

type Memory interface {
    Storage()
}

type CPU interface {
    Calculate()
}

type Computer struct {
    cpu CPU
    mem Memory
    card Card
}

func NewComputer(cpu CPU, mem Memory, card Card) *Computer{
    return &Computer{
        cpu:cpu,
        mem:mem,
        card:card,
    }
}

func (this *Computer) DoWork() {
    this.cpu.Calculate()
    this.mem.Storage()
    this.card.Display()
}

//------  实现层 -----
//intel
type IntelCPU struct {
    CPU
}

func (this *IntelCPU) Calculate() {
    fmt.Println("Intel CPU 开始计算了...")
}

type IntelMemory struct {
    Memory
}

func (this *IntelMemory) Storage() {
    fmt.Println("Intel Memory 开始存储了...")
}

type IntelCard struct {
    Card
}

func (this *IntelCard) Display() {
    fmt.Println("Intel Card 开始显示了...")
}

//kingston
type KingstonMemory struct {
    Memory
}

func (this *KingstonMemory) Storage() {
    fmt.Println("Kingston memory storage...")
}

//nvidia
type NvidiaCard struct {
    Card
}

func (this *NvidiaCard) Display() {
    fmt.Println("Nvidia card display...")
}



//------  业务逻辑层 -----
func main() {
    //intel系列的电脑
    com1 := NewComputer(&IntelCPU{}, &IntelMemory{}, &IntelCard{})
    com1.DoWork()

    //杂牌子
    com2 := NewComputer(&IntelCPU{}, &KingstonMemory{}, &NvidiaCard{})
    com2.DoWork()
}
```

# 小结

以上是用面向对象的思维来理解 Golang 中的 interface，通过设计原则来正反举例该如何正确使用 interface（接口），从而做到“高内聚、低耦合”，让代码写起来更优雅、扩展性更好。

## 设计模式简单介绍

**设计模式**（Design pattern）代表了最佳的实践，通常被有经验的面向对象的软件开发人员所采用。设计模式是软件开发人员在软件开发过程中面临的一般问题的解决方案。这些解决方案是众多软件开发人员经过相当长的一段时间的试验和错误总结出来的。

## 简单工厂模式

我们知道 Go 语言没有像 C++ 那样的构造函数一说。在 Go 语言我们通常会定义像 NewXXX 函数来初始化相关类。

NewXXX 函数返回接口时就是简单工厂模式，也就是说 Go 语言的一般推荐做法就是简单工厂。

在这个 simplefactory 包中只有 API 接口和 NewAPI 函数为包外可见，里面封装了实现的具体细节，简单工厂模式的 demo 如下：

### simple.go

```go
package simplefactory

import "fmt"

// API is interface
type API interface {
    Say(name string) string
}

// NewAPI return Api instance by type
func NewAPI(t int) API {
    if t == 1 {
        return &hiAPI{}
    } else if t == 2 {
        return &helloAPI{}
    }
    return nil
}

// hiAPI is one of API implement
type hiAPI struct{}

// Say hi to name
func (*hiAPI) Say(name string) string {
    return fmt.Sprintf("Hi, %s", name)
}

// HelloAPI is another API implement
type helloAPI struct{}

// Say hello to name
func (*helloAPI) Say(name string) string {
    return fmt.Sprintf("Hello, %s", name)
}
```

### simple_test.go

```
package simplefactory

import "testing"

// TestType1 test get hiapi with factory
func TestType1(t *testing.T) {
    api := NewAPI(1)
    s := api.Say("Tom")
    if s != "Hi, Tom" {
        t.Fatal("Type1 test fail")
    }
}

func TestType2(t *testing.T) {
    api := NewAPI(2)
    s := api.Say("Tom")
    if s != "Hello, Tom" {
        t.Fatal("Type2 test fail")
    }
}
```

## 工厂方法模式

工厂方法模式使用子类的方式延迟生成对象到子类中实现。

Go 中不存在继承 所以使用匿名组合来实现

### factorymethod.go

```
package factorymethod

//Operator 是被封装的实际类接口
type Operator interface {
    SetA(int)
    SetB(int)
    Result() int
}

//OperatorFactory 是工厂接口
type OperatorFactory interface {
    Create() Operator
}

//OperatorBase 是Operator 接口实现的基类，封装公用方法
type OperatorBase struct {
    a, b int
}

//SetA 设置 A
func (o *OperatorBase) SetA(a int) {
    o.a = a
}

//SetB 设置 B
func (o *OperatorBase) SetB(b int) {
    o.b = b
}

//PlusOperatorFactory 是 PlusOperator 的工厂类
type PlusOperatorFactory struct{}

func (PlusOperatorFactory) Create() Operator {
    return &PlusOperator{
        OperatorBase: &OperatorBase{},
    }
}

//PlusOperator Operator 的实际加法实现
type PlusOperator struct {
    *OperatorBase
}

//Result 获取结果
func (o PlusOperator) Result() int {
    return o.a + o.b
}

//MinusOperatorFactory 是 MinusOperator 的工厂类
type MinusOperatorFactory struct{}

func (MinusOperatorFactory) Create() Operator {
    return &MinusOperator{
        OperatorBase: &OperatorBase{},
    }
}

//MinusOperator Operator 的实际减法实现
type MinusOperator struct {
    *OperatorBase
}

//Result 获取结果
func (o MinusOperator) Result() int {
    return o.a - o.b
}
```

### factorymethod_test.go

```
package factorymethod

import "testing"

func compute(factory OperatorFactory, a, b int) int {
    op := factory.Create()
    op.SetA(a)
    op.SetB(b)
    return op.Result()
}

func TestOperator(t *testing.T) {
    var (
        factory OperatorFactory
    )

    factory = PlusOperatorFactory{}
    if compute(factory, 1, 2) != 3 {
        t.Fatal("error with factory method pattern")
    }

    factory = MinusOperatorFactory{}
    if compute(factory, 4, 2) != 2 {
        t.Fatal("error with factory method pattern")
    }
}
```

## 创建者模式

将一个复杂对象的构建分离成多个简单对象的构建组合。

### builder.go

```
package builder

//Builder 是生成器接口
type Builder interface {
    Part1()
    Part2()
    Part3()
}

type Director struct {
    builder Builder
}

// NewDirector ...
func NewDirector(builder Builder) *Director {
    return &Director{
        builder: builder,
    }
}

//Construct Product
func (d *Director) Construct() {
    d.builder.Part1()
    d.builder.Part2()
    d.builder.Part3()
}

type Builder1 struct {
    result string
}

func (b *Builder1) Part1() {
    b.result += "1"
}

func (b *Builder1) Part2() {
    b.result += "2"
}

func (b *Builder1) Part3() {
    b.result += "3"
}

func (b *Builder1) GetResult() string {
    return b.result
}

type Builder2 struct {
    result int
}

func (b *Builder2) Part1() {
    b.result += 1
}

func (b *Builder2) Part2() {
    b.result += 2
}

func (b *Builder2) Part3() {
    b.result += 3
}

func (b *Builder2) GetResult() int {
    return b.result
}
```

### builder_test.go

```
package builder

import "testing"

func TestBuilder1(t *testing.T) {
    builder := &Builder1{}
    director := NewDirector(builder)
    director.Construct()
    res := builder.GetResult()
    if res != "123" {
        t.Fatalf("Builder1 fail expect 123 acture %s", res)
    }
}

func TestBuilder2(t *testing.T) {
    builder := &Builder2{}
    director := NewDirector(builder)
    director.Construct()
    res := builder.GetResult()
    if res != 6 {
        t.Fatalf("Builder2 fail expect 6 acture %d", res)
    }
}
```

## 原型模式

原型模式使对象能复制自身，并且暴露到接口中，使客户端面向接口编程时，不知道接口实际对象的情况下生成新的对象。

原型模式配合原型管理器使用，使得客户端在不知道具体类的情况下，通过接口管理器得到新的实例，并且包含部分预设定配置。

### prototype.go

```
package prototype

//Cloneable 是原型对象需要实现的接口
type Cloneable interface {
    Clone() Cloneable
}

type PrototypeManager struct {
    prototypes map[string]Cloneable
}

func NewPrototypeManager() *PrototypeManager {
    return &PrototypeManager{
        prototypes: make(map[string]Cloneable),
    }
}

func (p *PrototypeManager) Get(name string) Cloneable {
    return p.prototypes[name]
}

func (p *PrototypeManager) Set(name string, prototype Cloneable) {
    p.prototypes[name] = prototype
}
```

### prototype_test.go

```
package prototype

import "testing"

var manager *PrototypeManager

type Type1 struct {
    name string
}

func (t *Type1) Clone() Cloneable {
    tc := *t
    return &tc
}

type Type2 struct {
    name string
}

func (t *Type2) Clone() Cloneable {
    tc := *t
    return &tc
}

func TestClone(t *testing.T) {
    t1 := manager.Get("t1")

    t2 := t1.Clone()

    if t1 == t2 {
        t.Fatal("error! get clone not working")
    }
}

func TestCloneFromManager(t *testing.T) {
    c := manager.Get("t1").Clone()

    t1 := c.(*Type1)
    if t1.name != "type1" {
        t.Fatal("error")
    }

}

func init() {
    manager = NewPrototypeManager()

    t1 := &Type1{
        name: "type1",
    }
    manager.Set("t1", t1)
}
```

## 单例模式

使用懒惰模式的单例模式，使用双重检查加锁保证线程安全。

### singleton.go

```
package singleton

import "sync"

//Singleton 是单例模式类
type Singleton struct{}

var singleton *Singleton
var once sync.Once

//GetInstance 用于获取单例模式对象
func GetInstance() *Singleton {
    once.Do(func() {
        singleton = &Singleton{}
    })

    return singleton
}
```

### singleton_test.go

```
package singleton

import (
    "sync"
    "testing"
)

const parCount = 100

func TestSingleton(t *testing.T) {
    ins1 := GetInstance()
    ins2 := GetInstance()
    if ins1 != ins2 {
        t.Fatal("instance is not equal")
    }
}

func TestParallelSingleton(t *testing.T) {
    wg := sync.WaitGroup{}
    wg.Add(parCount)
    instances := [parCount]*Singleton{}
    for i := 0; i < parCount; i++ {
        go func(index int) {
            instances[index] = GetInstance()
            wg.Done()
        }(i)
    }
    wg.Wait()
    for i := 1; i < parCount; i++ {
        if instances[i] != instances[i-1] {
            t.Fatal("instance is not equal")
        }
    }
}
```

## 小结

以上是设计模式-创建模式中的几种模式，分别为：单例模式、简单工厂模式、抽象工厂模式、创建者模式和原型模式。大家平时在开发中用的比较多的可能是单例模式和工厂模式，但是以上都是需要掌握的。

使用设计模式是为了重用代码、让代码更容易被他人理解、保证代码可靠性。

毫无疑问，设计模式于己于他人于系统都是多赢的，设计模式使代码编制真正工程化，设计模式是软件工程的基石，如同大厦的一块块砖石一样。

项目中合理地运用设计模式可以完美地解决很多问题，每种模式在现实中都有相应的原理来与之对应，每种模式都描述了一个在我们周围不断重复发生的问题，以及该问题的核心解决方案，这也是设计模式能被广泛应用的原因。

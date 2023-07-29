## 基础题目如下

1.下面这段代码输出结果正确吗？

```go
type Foo struct {
    bar string
}
func main() {
    s1 := []Foo{
        {"A"},
        {"B"},
        {"C"},
    }
    s2 := make([]*Foo, len(s1))
    for i, value := range s1 {
        s2[i] = &value
    }
    fmt.Println(s1[0], s1[1], s1[2])
    fmt.Println(s2[0], s2[1], s2[2])
}
```
输出：  
{A} {B} {C}  
&{A} &{B} &{C}

参考答案及解析：s2 的输出结果错误。s2 的输出是 &{C} &{C} &{C}，for range 使用短变量声明(:=)的形式迭代变量时，变量 i、value 在每次循环体中都会被重用，而不是重新声明。
所以 s2 每次填充的都是临时变量 value 的地址，而在最后一次循环中，value 被赋值为{c}。因此，s2 输出的时候显示出了三个 &{c}。
可行的解决办法如下：
```go
for i := range s1 {
    s2[i] = &s1[i]
}
```

2. 下面代码里的 counter 的输出值？

```go
func main() {

    var m = map[string]int{
        "A": 21,
        "B": 22,
        "C": 23,
    }
    counter := 0
    for k, v := range m {
        if counter == 0 {
            delete(m, "A")
        }
        counter++
        fmt.Println(k, v)
    }
    fmt.Println("counter is ", counter)
}
```
A. 2  
B. 3  
C. 2 或 3  

参考答案及解析：C。for range map 是无序的，如果第一次循环到 A，则输出 3；否则输出 2。

3. 关于channel的特性，下面说法正确的是？
- A. 给一个 nil channel 发送数据，造成永远阻塞
- B. 从一个 nil channel 接收数据，造成永远阻塞
- C. 给一个已经关闭的 channel 发送数据，引起 panic
- D. 从一个已经关闭的 channel 接收数据，如果缓冲区中为空，则返回一个零值

参考答案及解析：ABCD。

4. 下面代码有什么问题？

```go
const i = 100
var j = 123

func main() {
    fmt.Println(&j, j)
    fmt.Println(&i, i)
}
```
参考答案及解析：编译报错cannot take the address of i。知识点：常量。常量不同于变量在运行期就分配内存，常量通常会被编译器在预处理阶段直接展开，作为指令数据使用，所以常量无法寻址。

5. 下面代码输出什么？

```go
func main() {
    x := []string{"a", "b", "c"}
    for v := range x {
        fmt.Print(v)
    }
}
```
参考答案及解析：012。注意区别下面代码段：
```go
func main() {
    x := []string{"a", "b", "c"}
    for _, v := range x {
        fmt.Print(v)     //输出 abc
    }
}
```
6. 下面这段代码能否编译通过？如果通过，输出什么？

```go
type User struct{}
type User1 User
type User2 = User

func (i User1) m1() {
    fmt.Println("m1")
}
func (i User) m2() {
    fmt.Println("m2")
}

func main() {
    var i1 User1
    var i2 User2
    i1.m1()
    i2.m2()
}
```
参考答案及解析：能，输出m1 m2，第 2 行代码基于类型 User 创建了新类型 User1，第 3 行代码是创建了 User 的类型别名 User2，注意使用 = 定义类型别名。  
因为 User2 是别名，完全等价于 User，所以 User2 具有 User 所有的方法。但是 i1.m2() 就不行了，i1.m2()是不能执行的，因为 User1 没有定义该方法。

7. 下面代码是否能编译通过？如果通过，输出什么？

```go
func Foo(x interface{}) {
    if x == nil {
        fmt.Println("empty interface")
        return
    }
    fmt.Println("non-empty interface")
}
func main() {
    var x *int = nil
    Foo(x)
}
```
参考答案及解析：non-empty interface。  
考点：interface 的内部结构，我们知道接口除了有静态类型，还有动态类型和动态值，当且仅当动态值和动态类型都为 nil 时，接口类型值才为 nil。  
这里的 x 的动态类型是 *int，所以 x 不为 nil。

8. 关于select机制，下面说法正确的是?

- A. select机制用来处理异步IO问题；
- B. select机制最大的一条限制就是每个case语句里必须是一个IO操作；
- C. golang在语言级别支持select关键字；
- D. select关键字的用法与switch语句非常类似，后面要带判断条件；

参考答案及解析：ABC。

9. 下面的代码有什么问题？

```go
func Stop(stop <-chan bool) {
    close(stop)
}
```
参考答案及解析：有方向的 channel 不可以被关闭。

10. 下面这段代码存在什么问题？

```go
 type Param map[string]interface{}
 
 type Show struct {
     *Param
 }
 
 func main() {
     s := new(Show)
     s.Param["day"] = 2
}
```
参考答案及解析：  
存在两个问题：1.map 需要初始化才能使用；2.指针不支持索引。修复代码如下：

```go
func main() {
    s := new(Show)
    // 修复代码
    p := make(Param)
    p["day"] = 2
    s.Param = &p
    tmp := *s.Param
    fmt.Println(tmp["day"])
}
```

11. 下面这段代码输出什么？

```go
var x = []int{2: 2, 3, 0: 1}

func main() {
    fmt.Println(x)
}
```
参考答案及解析：输出[1 0 2 3]，字面量初始化切片时候，可以指定索引，没有指定索引的元素会在前一个索引基础之上加一，所以输出[1 0 2 3]，而不是[1 3 2]。

12. 请指出下面代码的错误？

```go
 package main
 
 var gvar int 
 
 func main() {  
     var one int   
     two := 2      
     var three int 
     three = 3

    func(unused string) {
        fmt.Println("Unused arg. No compile error")
    }("what?")
}
```
参考答案及解析：变量 one、two 和 three 声明未使用。  
知识点：未使用变量。如果有未使用的变量代码将编译失败。  
但也有例外，函数中声明的变量必须要使用，但可以有未使用的全局变量。函数的参数未使用也是可以的。

13. 下面代码输出什么？

```go
 type ConfigOne struct {
     Daemon string
 }
 
 func (c *ConfigOne) String() string {
     return fmt.Sprintf("print: %v", c)
 }
 
 func main() {
    c := &ConfigOne{}
    c.String()
}
```
参考答案及解析：运行时错误。如果类型实现 String() 方法，当格式化输出时会自动使用 String() 方法。上面这段代码是在该类型的 String() 方法内使用格式化`fmt.Sprintf`输出，导致递归调用，最后抛错。

14. 下面代码输出什么？

```go
 func main() {
     var a = []int{1, 2, 3, 4, 5}
     var r = make([]int, 0)
     for i, v := range a {
         if i == 0 {
             a = append(a, 6, 7)
         }
        r = append(r, v)
    }
    fmt.Println(r)
}
```
参考答案及解析：[1 2 3 4 5]。a 在 for range 过程中增加了两个元素，len 由 5 增加到 7，但 for range 时会使用 a 的副本 a' 参与循环，副本的 len 依旧是 5，因此 for range 只会循环 5 次，也就只获取 a 对应的底层数组的前 5 个元素。

15. 下面代码有几处错误的地方？请说明原因。

```go
func main() {

    var s []int
    s = append(s,1)

    var m map[string]int
    m["one"] = 1 
}
```
参考答案及解析：有 1 出错误，不能对 nil 的 map 直接赋值，需要使用 make() 初始化。但可以使用 append() 函数对为 nil 的 slice 增加元素。
修复代码：

```go
func main() {
    var m map[string]int
    m = make(map[string]int)
    m["one"] = 1
}
```

16. 下面代码能编译通过吗？

```go
 type info struct {
     result int
 }
 
 func work() (int,error) {
     return 13,nil
 }
 
 func main() {
    var data info

    data.result, err := work() 
    fmt.Printf("info: %+v\n",data)
}
```
参考答案及解析：编译失败。
1non-name data.result on left side of :=
不能使用短变量声明设置结构体字段值，修复代码：

```go
 func main() {
     var data info
 
     var err error
     data.result, err = work() //ok
     if err != nil {
         fmt.Println(err)
         return
     }

    fmt.Println(data)   
}
```

17. 下面代码输出什么？

```go
 func test(x byte)  {
     fmt.Println(x)
 }
 
 func main() {
     var a byte = 0x11 
     var b uint8 = a
     var c uint8 = a + b
     test(c)
}
```

参考答案及解析：34。与 rune 是 int32 的别名一样，byte 是 uint8 的别名，别名类型无序转换，可直接转换。

18. 下面代码最后一行输出什么？请说明原因。

```go
func main() {
     x := 1
     fmt.Println(x)
     {
         fmt.Println(x)
         i,x := 2,2
         fmt.Println(i,x)
     }
     fmt.Println(x)  // print ?
}
```
参考答案及解析：输出1。知识点：变量隐藏。使用变量简短声明符号 := 时，如果符号左边有多个变量，只需要保证至少有一个变量是新声明的，并对已定义的变量尽进行赋值操作。但如果出现作用域之后，就会导致变量隐藏的问题，就像这个例子一样。  
这个坑很容易挖，但又很难发现。即使对于经验丰富的 Go 开发者而言，这也是一个非常常见的陷阱。

19. 下面代码输出什么？

```go
 func main() {
     isMatch := func(i int) bool {
         switch(i) {
         case 1:
         case 2:
             return true
         }
         return false
    }

    fmt.Println(isMatch(1))
    fmt.Println(isMatch(2))
}
```

参考答案及解析：false true。Go 语言的 switch 语句虽然没有"break"，但如果 case 完成程序会默认 break，可以在 case 语句后面加上关键字 fallthrough，这样就会接着走下一个 case 语句（不用匹配后续条件表达式）。或者，利用 case 可以匹配多个值的特性。  
修复代码：

```go
 func main() {
     isMatch := func(i int) bool {
         switch(i) {
         case 1:
             fallthrough
         case 2:
             return true
         }
         return false
    }

    fmt.Println(isMatch(1))     // true
    fmt.Println(isMatch(2))     // true

    match := func(i int) bool {
        switch(i) {
        case 1,2:
            return true
        }
        return false
    }

    fmt.Println(match(1))       // true
    fmt.Println(match(2))       // true
}
```


20. 下面的代码能否正确输出？

```go
func main() {
    var fn1 = func() {}
    var fn2 = func() {}

    if fn1 != fn2 {
        println("fn1 not equal fn2")
    }
}
```
参考答案及解析：编译错误.函数只能于nil比较.

21. 下面哪一行代码会 panic，请说明原因？

```go
package main

func main() {
    x := make([]int, 2, 10)
    _ = x[6:10]
    _ = x[6:]
    _ = x[2:]
}
```
参考答案：第 6 行，截取符号 [i:j]，如果 j 省略，默认是原切片或者数组的长度，x 的长度是 2，小于起始下标 6 ，所以 panic。

22. 下面代码会 输出什么，请说明原因？

```go
func main()  {
   data := []int{0,1,2,3}
   m := make(map[int]*int)

   for key, v := range data {
      m[key] = &v
   }
   for k, v := range m {
      fmt.Println(k , "--->", *v)
   }
}
```
输出结果(map输出是无序的):
```
1 ---> 3
2 ---> 3
3 ---> 3
0 ---> 3
```

23. 下面代码会 输出什么，请说明原因？

```go
func demo3()  {
   n := make([]int, 5)
   n = append(n, 1,2,3)
   fmt.Println(n)
}
```
输出结果: `[0 0 0 0 0 1 2 3]`

24. 下面代码会 输出什么，请说明原因？

```go
func demo4()  {
   n := make([]int, 0)
   n = append(n, 1,2,3,4,5)
   fmt.Println(n)
}
```
输出结果: `[1 2 3 4 5]`

25. 下面代码输出什么？

```go
 const (
     azero = iota
     aone  = iota
 )
 const (
     info  = "msg"
     bzero = iota
     bone  = iota
)
const (
   x = iota   // 0
   _        // 1
   y        // 2
   z = "zz"   // "zz"
   k        // "zz"
   p = iota   // 5
)

func main() {
    fmt.Println(azero, aone)
    fmt.Println(bzero, bone)
}
```
参考答案及解析：0 1 1 2。知识点：iota 的使用。  
这道题易错点在 bzero、bone 的值，在一个常量声明代码块中，如果 iota 没出现在第一行，则常量的初始值就是非 0 值。

26. 下面这段代码能通过编译吗？请简要说明。

```go
func main() {
    m := make(map[string]int)
    m["foo"]++
    fmt.Println(m["foo"])
}
```
参考答案及解析：能通过编译。
上面的代码可以理解成：

```go
func main() {
    m := make(map[string]int)
    v := m["foo"]
    v++
    m["foo"] = v
    fmt.Println(m["foo"])
}
```

27. defer巩固

```go
例1：
func f() (result int) {
    defer func() {
        result++
    }()
    return 0
}
output: 1
=====
例2：
func f() (r int) {
    t := 5
    defer func() {
        t = t + 5
    }()
    return t
}
output: 5
=====
例3：
func f() (r int) {
    defer func(r int) {
        r = r + 5
    }(r)
    return 1
}
output：1
--------------------------------
type Person struct {
    age int
}

func main() {
    person := &Person{28}
    // 1. 输出28
    defer fmt.Println(person.age)
    // 2. 输出29， 传的是结构体的指针
    defer func(p *Person) {
        fmt.Println(p.age)
    }(person)  
    // 3. 输出29， 闭包
    defer func() {
        fmt.Println(person.age)
    }()
    person.age = 29
}
```

28. 闭包和协程相关

```go
func demo2() {
   var wg sync.WaitGroup
   wg.Add(12)
   for i := 0; i<6; i++ {
      go func() {
         fmt.Println("T1:", i)  // 输出都是6
         wg.Done()
      }()
   }
   for i := 0; i<6; i++ {
      go func(i int) {
         fmt.Println("T2:", i) // 输出1到6，顺序不确定
         wg.Done()
      }(i)
   }
   wg.Wait()
}
```

29. map取地址相关

```go
func demo3() {
   m := make(map[string]interface{})
   m["name"] = "Jack"
   fmt.Printf("%p\n", m)  //0xc000098000
   fmt.Printf("%p\n", &m) //0xc000092008
}
```

30. 结构体和切片比较

```go
func structCompare() {
   fmt.Println(&Teacher{"Jack"} == &Teacher{"Jack"})  // false
   fmt.Println(Teacher{"Jack"} == Teacher{"Jack"})   // true
}
func sliceCompare() {
   fmt.Println([...]string{"hello"} == [...]string{"hello"})  // true––––
   //fmt.Println([]string{"hello"} == []string{"hello"})  // 编译不通过
}
```

31. 多协程查询切片 

```go
package main

import (
   "context"
   "fmt"
   "time"
)

/*
假设有一个超长的切片，切片的元素类型为int，切片中的元素为乱序排列。限时5秒，
使用多个goroutine查找切片中是否存在给定值，在找到目标值或者超时后立刻结束所有goroutine的执行。
比如切片为：[23, 32, 78, 43, 76, 65, 345, 762, …… 915, 86]，查找的目标值为345，
如果切片中存在目标值程序输出:"Found it!"并且立即取消仍在执行查找任务的goroutine。
如果在超时时间未找到目标值程序输出:"Timeout! Not Found"，同时立即取消仍在执行查找任务的goroutine。
*/

func demo() {
   timer := time.NewTimer(time.Second * 5)
   sliceInt := []int{23,32,78,43,76,65,345,762}
   target := 345

   size := 3
   dataLen := len(sliceInt)

   ctx, cancel := context.WithCancel(context.Background())
   resultCh := make(chan bool)

   for i := 0; i < dataLen; i += size {
      end := i+size
      if end >= dataLen {
         end = dataLen - 1
      }
      go searchTarget(ctx, sliceInt[i:end], target, resultCh)
   }

   select {
   case <-timer.C:
      fmt.Println("Timeout! Not Found")
      cancel()
   case <-resultCh:
      fmt.Println("Found it!")
      cancel()
   }

   //time.Sleep(time.Second*6)
}

func searchTarget(ctx context.Context, data []int, target int, resultCh chan bool) {
   for _, v := range data {
      time.Sleep(time.Second*20)
      select {
      case <-ctx.Done():
         fmt.Println("task cancel!")
         return
      default:
      }

      if target == v {
         resultCh <- true
         return
      }
   }
}

func main() {
   demo()
}
```

32. 结构体比较 

```
1.结构体只能比较是否相等，但是不能比较大小。
2.相同类型的结构体才能够进行比较，结构体是否相同不但与属性类型有关，还与属性顺序相关，sn3 与 sn1 就是不同的结构体；
3.如果 struct 的所有成员都可以比较，则该 struct 就可以通过 == 或 != 进行比较是否相等，比较时逐个项进行比较，如果每一项都相等，则两个结构体才相等，否则不相等；
4.那什么是可比较的呢，常见的有 bool、数值型、字符、指针、数组等，像切片、map、函数等是不能比较的。
```

33. 切片append操作

```go
func addrSlice() {
   a := []int{7, 8, 9}
   fmt.Println(a)    // [7, 8, 9]
   ap(a)
   fmt.Println(a) // [7, 8, 9]
   app(a)
   fmt.Println(a) // [1, 8, 9]

}
func ap(a []int) {
   a = append(a, 10)
}
func app(a []int) {
   a[0] = 1
}
```

34. init说明

```
1.一个包中，可以包含多个init函数
2.程序编译时，先执行依赖包的init函数，再执行main包内的init函数
3.main包中可以有init函数 
```

35. cap适用范围

```
array
slice
channel
```

另外：有兴趣学网络编程的，推荐一个训练营：动手实战学网络编程，可以使用邀请码：AqCJeLyy 有优惠。

更多有关 Golang 技术文章尽在公号：“**Go键盘侠**”。

推荐文章：
[干货！网络编程之 TCP 原理详解](https://juejin.cn/post/6965119742172463141)


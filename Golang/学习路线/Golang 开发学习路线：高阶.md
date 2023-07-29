前面把 Go 语言中阶的学习路线整理出来了：[Golang 开发学习路线：中阶](http://mp.weixin.qq.com/s?__biz=Mzg3NDUxNDg4Mw==&mid=2247484421&idx=1&sn=8c9d6382f32901b0a4409c4813671aff&chksm=ceced4a9f9b95dbff6fda8bfdcaca612008f3a4c3bd929d3f95ba1a6bd6b8f6f28109a5ba4c6&scene=21#wechat_redirect)，今天最后把 Go 语言（高阶）的学习路线也整理完了。

学习路线脑图如下：
`https://gitmind.cn/app/doc/8bc3b64fb1fe8787e6ab7f91c431a89`

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/350d1898b1054277b285deacfe1d62df~tplv-k3u1fbpfcp-zoom-1.image)

另外今天也给大家聊聊关于使用 Go 的一些规范指南，或者说平时写代码需要做的一些 CheckLIst。

## 【必须】切片长度校验

- 在对 slice 进行操作时，必须判断长度是否合法，防止程序 panic

```go
package main

import "fmt"

// bad: 未判断data的长度，可导致 index out of range
func badDecode(data []byte) bool {
	if data[0] == 'F' && data[1] == 'U' && data[2] == 'Z' && data[3] == 'Z' && data[4] == 'E' && data[5] == 'R' {
		fmt.Println("Bad")
		return true
	}
	return false
}

// bad: slice bounds out of range
func foo() {
	slice := []int{0, 1, 2, 3, 4, 5, 6}
	fmt.Println(slice[:10])
}

// good: 使用data前应判断长度是否合法
func goodDecode(data []byte) bool {
	if len(data) == 6 {
		if data[0] == 'F' && data[1] == 'U' && data[2] == 'Z' && data[3] == 'Z' && data[4] == 'E' && data[5] == 'R' {
			fmt.Println("Good")
			return true
		}
	}
	return false
}

func main() {
	data := []byte{'A', 'B', 'C', 'D'}
	fmt.Println(goodDecode(data))
	fmt.Println(badDecode(data))
}
```

## 【必须】nil 指针判断

- 进行指针操作时，必须判断该指针是否为 nil，防止程序 panic，尤其在进行结构体  Unmarshal 时

```go
package main

import (
	"fmt"
	"io"
)

type Packet struct {
	PackeyType uint8
	PackeyVersion uint8
	Data *Data
}
type Data struct {
	Stat uint8
	Len uint8
	Buf [8]byte
}
func (p *Packet) UnmarshalBinary(b []byte) error {
	if len(b) < 2 {
		return io.EOF
	}

	p.PackeyType = b[0]
	p.PackeyVersion = b[1]
	// 若长度等于2，那么不会new Data
	if len(b) > 2 {
		p.Data = new(Data)
		// Unmarshal(b[i:], p.Data)
	}
	return nil
}

// bad: 未判断指针是否为nil
func main() {
	packet := new(Packet)
	data := make([]byte, 2)
	if err := packet.UnmarshalBinary(data); err != nil {
		fmt.Println("Failed to unmarshal packet")
		return
	}
	fmt.Printf("Stat: %v\n", packet.Data.Stat)
}
```

## 【必须】整数安全

在进行数字运算操作时，需要做好长度限制，防止外部输入运算导致异常：

- 确保无符号整数运算时不会反转
- 确保有符号整数运算时不会出现溢出
- 确保整型转换时不会出现截断错误
- 确保整型转换时不会出现符号错误

以下场景必须严格进行长度限制：

- 作为数组索引
- 作为对象的长度或者大小
- 作为数组的边界（如作为循环计数器）

```go
package main

import "fmt"

// bad: 未限制长度，导致整数溢出
func overflowBad(numControlByUser int32) {
	var numInt int32 = 0
	numInt = numControlByUser + 1
	//对长度限制不当，导致整数溢出
	fmt.Printf("%d\n", numInt)
	//使用numInt，可能导致其他错误
}

// good:
func overflowGood(numControlByUser int32) {
	var numInt int32 = 0
	numInt = numControlByUser + 1
	if numInt < 0 {
	fmt.Println("integer overflow")
	return;
	}
	fmt.Println("integer ok")
	}

func main() {
	overflowBad(2147483647)
	overflowGood(2147483647)
}
```

## 【必须】make 分配长度验证

- 在进行 make 分配内存时，需要对外部可控的长度进行校验，防止程序 panic。

```go
package main

import "errors"

// bad
func parseBad(lenControlByUser int, data[] byte) {
	size := lenControlByUser
	//对外部传入的size，进行长度判断以免导致panic
	buffer := make([]byte, size)
	copy(buffer, data)
}

// good
func parseGood(lenControlByUser int, data[] byte) ([]byte, error){
	size := lenControlByUser
	//限制外部可控的长度大小范围
	if size > 64*1024*1024 {
		return nil, errors.New("value too large")
	}
	buffer := make([]byte, size)
	copy(buffer, data)
	return buffer, nil
}

```

## 总结

主要梳理了 Go 语言高阶学习路线，以及整理了一些  Go 的使用规范指南，后续会继续补充这一块的内容。

代码示例已更新到 [github-gochecklist](https://github.com/Scoefield/gokeyboardman/tree/main/gochecklist)。

推荐阅读：
[Golang 开发学习路线：中阶](http://mp.weixin.qq.com/s?__biz=Mzg3NDUxNDg4Mw==&mid=2247484421&idx=1&sn=8c9d6382f32901b0a4409c4813671aff&chksm=ceced4a9f9b95dbff6fda8bfdcaca612008f3a4c3bd929d3f95ba1a6bd6b8f6f28109a5ba4c6&scene=21#wechat_redirect)

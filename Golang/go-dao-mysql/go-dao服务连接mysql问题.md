## 遇到的问题

dao服务访问数据库偶发 `invalid connection` 的错误。

## 本质原因

go底层连接池中的数据库连接 被**mysql**服务器单方面断开，而旧版本的golang mysql驱动(`go-sql-driver`)未提供错误重连的功能。

## go框架中的连接池初始化代码

```go
type MysqlDBPool struct {
	dbOper *sql.DB
	dbUpdateRstsql.Result
	configmysql.Config
	isInitedbool
}

func NewDBPool(user, password, ip, db, charset string, port, timeout int) (c *MysqlDBPool, err error) {
	c = &MysqlDBPool{}
	//.............
	c.dbOper, err = sql.Open("mysql", dnsString)
	if err != nil {
		return nil, err
	}
	c.dbOper.SetMaxIdleConns(10)   // 连接池保有的最大空闲连接数量，默认为2
	c.dbOper.SetMaxOpenConns(500)  // 设置最大打开的连接数，默认值为0，表示不限制
	c.dbOper.SetConnMaxLifetime(600 * time.Second)  // 连接最大的存活时间，默认值为0,表示不限制
	err = c.dbOper.Ping()
	if err != nil {
		return nil, err
	}
	c.isInited = true
	return c, nil
}
```
**其中**：  
- SetMaxIdleConns 为连接池保有的最大空闲连接数量，这里设置为10
- SetMaxOpenConns 为连接池设置最大打开的连接数，这里设置为500
- SetConnMaxLifetime 为连接最大的存活时间（如果在使用期间达到了最大的存活时间，则会等到使用结束之后再关闭）
数据库里的参数 `wait_timeout` 说明：  
wait_timeout 是针对非交互式连接，即在 mysql_real_connect() 函数中未使用 CLIENT_INTERACTIVE 选项的
连接。

## dev环境复现遇到的问题

### 查看 dev 环境 mysql 设置的 wait_timeout

go框架代码(上面示例代码)中设置为600秒，保留10个空闲连接，而 dev 环境 mysql 目前 wait_timeout 设置为500秒(线上的也是)。见下图：

![mysql_wait_timeout.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c09f763538644f7fa01347957b359949~tplv-k3u1fbpfcp-watermark.image)

### 复现过程

1. 写一个 demo，将代码初始化数据库连接池的设置跟线上的一样(见上面示例代码)；
2. 然后 demo 开10个协程查询10次，让连接池缓存有10个连接；
3. 等待大概501秒后(这里为了尽可能的复现问题，相关参数保持与线上的一致)，再开启10个协程继续使用刚刚缓存的10个连接，然后查看结果。

### 测试代码

测试代码如下(关键代码)：
```go
package main

import (
"comm/mysqlclient"
"fmt"
"time"
)

func Query(oDalPoolSet *mysqlclient.DalPoolSet, i int){
	oStorage := oDalPoolSet.GetOperator("t_user_account",0)
	if nil == oStorage{
		fmt.Printf("%d:获取Operator错误\n", i)
		return
	}

	sql := "select * from t_user_account limit 1"
	oRs, err := oStorage.Query(sql)

	if nil != err{
		fmt.Printf("%d:查询错误err：%s\n", i, err.Error())
		return
	}

	oRs.Next()
	oRs.Close()
	fmt.Printf("%d:OK\n", i)
	return
}

func PrintTime(){
	for{
		fmt.Printf("%s\n",time.Now().Format("2006-01-0215:04:05"))
		time.Sleep(1*time.Second)
	}
}

func main() {
	var oDalPoolSet *mysqlclient.DalPoolSet
	oDalPoolSet, err := mysqlclient.InitDBPoolBySet("service_set")
	if err != nil{
		fmt.Printf("初始化连接池错误err：%s", err.Error())
		return
	}

	go PrintTime()
	for i := 1; i <= 10; i++ {
		go Query(oDalPoolSet, i)
	}

	time.Sleep(501*time.Second)

	for i := 1; i <= 10; i++ {
		go Query(oDalPoolSet, i)
	}
	time.Sleep(10*time.Second)

	return
}
```
### 现象及结果如下

执行以上测试代码，结果如下：

![mysqlpooltest1.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/53e7dae9c0bb40f191a642aea83ada23~tplv-k3u1fbpfcp-watermark.image)
![mysqlpooltest2.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f66a993855c14b9ab10661ada12af172~tplv-k3u1fbpfcp-watermark.image)

在调用速度足够快的情况下，10次查询会使用到保留或者说缓存的全部10个空闲连接，但是此时连接在 mysql 服务器端已经超时(500秒)被断开，而 go连接池中并不知道mysql服务器端连接已经断开的这个状态，从而导致全部查询失败：报`invalid connection`的错误。

## 总结（解决方案）

1. 修改数据库 `wait_timeout` 参数大于600秒，最简单;
2. 升级 `go-sql-driver` 版本，能自动重连；
3. 改 go 框架代码，初始化数据库连接池的时候，将 `ConnMaxLifetime` 设置的比数据库的 `wait_timeout` 参数小一些。

具体选择哪个方案来解决以上问题，需要根据业务的具体情况以及模块负责人的意见来权衡决定。

如果大家还有更好的解决方案的话，可以留言评论，互相交流。


---
highlight: a11y-dark
---
## 安装和使用 docker + postgres + tableplus 

### 安装 postgres

- postgres 仓库地址：https://hub.docker.com/_/postgres

拉取 postgres 镜像：

`docker pull pstgres:12-alpine`

运行：

```bash
docker run --name postgres12 -p 5432:5432 -e POSTGRES_USER=root -e POSTGRES_PASSWORD=xxxxxx -d postgres:12-alpine
```

查看运行的容器：

`docker ps`

另外：有兴趣学网络编程的，推荐一个训练营：[动手实战学网络编程](https://www.lanqiao.cn/courses/3384)，可以使用邀请码：**AqCJeLyy** 有优惠。

进入运行的容器 postgres 环境的控制台：

`docker exec -it postgres12 psql -U root`

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/182038e100234ffd93108c80f3068429~tplv-k3u1fbpfcp-watermark.image)

退出控制台：`\q`


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/17555f2ced334c37925af9c619b65d14~tplv-k3u1fbpfcp-watermark.image)

查看 postgres 容器日志：`docker logs postgres12`

### tablepuls

- 网址：https://tableplus.com/

下载：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/23b73740c22f48dfab03361bf7926447~tplv-k3u1fbpfcp-watermark.image)

安装后简单操作：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5053debf4ebe4ab09ba6b6c6c897cad6~tplv-k3u1fbpfcp-watermark.image)

导入 sql 文件，运行 sql 命令创建三个表：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/612f9ae1ec6c4ffebe49bae84666909d~tplv-k3u1fbpfcp-watermark.image)


## 数据库架构迁移

- golang-migrate github 地址：https://github.com/golang-migrate/migrate

MacOS 安装：

`brew install golang-migrate`


进入 postgres12 容器：

`docker exec -it postgres12 /bin/sh`

进入后执行命令，创建数据库，如下所示：

`docker exec -it postgres12 /bin/bash`

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5b5f61b23d8241d59e03084fdf6d2da5~tplv-k3u1fbpfcp-watermark.image)

退出并使用 dropdb 命令删除数据库，如下所示：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4691c1a7405a446b8e20aace39d84943~tplv-k3u1fbpfcp-watermark.image)

运行容器时执行创建数据库的命令：

`docker exec -it postgres12 createdb --username=root --owner=root simple_bank`

运行容器时执行删除数据库的命令：

`docker exec -it postgres12 dropdb simple_bank`

### 编写 Makefile 文件

Makefile 内容和 make 命令如下：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9a1faba594814b51a2d083e1b6cc4dea~tplv-k3u1fbpfcp-watermark.image)

用 TablePlus 打开查看新建的 simple_bank 数据库：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3871150a6ad349dda314edbb07e21f4e~tplv-k3u1fbpfcp-watermark.image)

## 生成 CRUD，比较 db_sql\gorm\sqlx

CRUD 表示“增删改查”操作。

### db_sql

使用标准库数据库工具，比较原始的操作，这种方法的优缺点：

- 在编写代码时运行非常快，性能比较好；
- 需要定义相应的映射字段；
- 函数调用的某些参数错误需要在运行时才会提示；

### gorm

文档地址：https://gorm.io/docs/query.html

特点：

- 简单方便使用，内置封装实现 crud 的操作，高级对象关系映射；
- 当流量很高时，运行会比较慢。

### sqlx

文档地址：https://github.com/jmoiron/sqlx

特点：

- 运行速度几乎与标准库一样快，并且使用起来非常方便；
- 字段映射通过查询文本的方式，而且结构带标签；

### sqlc

参考地址：https://docs.sqlc.dev/en/stable/tutorials/getting-started.html  
Github地址：https://github.com/kyleconroy/sqlc  
使用手册：https://sqlc.dev/

特点：

- 简单，运行速度非常快；
- 自动生成代码；
- 生成代码时，可以知道 sql 的错误；

macOS 下载 sqlc：

1. 首先访问：https://sqlc.dev/
2. 然后点击相应的系统下载

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/84dd13e13d8d4fb6bf05e81dfb07aa8e~tplv-k3u1fbpfcp-watermark.image)

下载解压后，添加环境变量，然后运行 `sqlc version` 查看版本：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/10c769f2823340a4902a1940c5b812af~tplv-k3u1fbpfcp-watermark.image)

初始化 sqlc：`sqlc init`，会生成 sqlc.yaml 配置文件。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0e02b3ea956848a1ae00c2ac3aac0dfe~tplv-k3u1fbpfcp-watermark.image)

### 编写 sqlc.yaml

该文件编写参考：https://docs.sqlc.dev/en/latest/reference/config.html

```yaml
version: "1"
packages:
  - name: "db"
    path: "./db/sqlc"
    queries: "./db/query/"
    schema: "./db/postgresql/"
    engine: "postgresql"
    emit_prepared_queries: false
    emit_interface: false
    emit_exact_table_names: true
    emit_empty_slices: false
    emit_json_tags: true
    json_tags_case_style: "camel"
```

### 完善 Makefile

```makefile
postgres:
	docker run --name postgres12 -p 5432:5432 -e POSTGRES_USER=root -e POSTGRES_PASSWORD=xxxxxx -d postgres:12-alpine

createdb:
	docker exec -it postgres12 createdb --username=root --owner=root simple_bank

dropdb:
	docker exec -it postgres12 dropdb simple_bank

sqlc:
	sqlc generate

.PHONY: postgres createdb dropdb sqlc
```

### query 目录下编写需要创建的数据库表模型文件(*.sql)

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0233ca7f6262467ea3b4779ba0398d8a~tplv-k3u1fbpfcp-watermark.image)

#### account.sql

参考地址（sqlc使用手册）：https://docs.sqlc.dev/en/latest/tutorials/getting-started.html

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/04369b01da54428faf5533e248aa4392~tplv-k3u1fbpfcp-watermark.image)

参考以上手册实现的 account.sql 如下：

```sql
-- name: CreateAccount :one
INSERT INTO accounts (
  owner,
  balance,
  currentcy
) VALUES (
  $1, $2, $3
) RETURNING *;

-- name: GetAccount :one
SELECT * FROM accounts
WHERE id = $1 LIMIT 1;

-- name: ListAccounts :many
SELECT * FROM accounts
ORDER BY id LIMIT $1 OFFSET $2;

-- name: UpdateAccount :exec
UPDATE accounts 
SET balance = $2
WHERE id = $1 
RETURNING *;

-- name: DeleteAccount :exec
DELETE FROM accounts WHERE id = $1;
```

entry.sql 和 transfer.sql 文件实现类似的，这里不再赘述。

### 生成相应的db模型文件

直接运行以上编写好的 Makefile 即可生成相应的模型文件：`make sqlc`，如下：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5d87d85cf8fc4ef8ba42bbbdd6840e67~tplv-k3u1fbpfcp-watermark.image)

生成文件内容大家可以点开看下，其实就是一些数据库连接即 `CRUD` 的相关操作封装方法，后续实现的时候需要用到。

可以看到是非常的方便和简洁，也模块化，比较清晰。

## 用 Go 单元测试刚生成的这些模块方法（test CRUD 操作）

### 获取连接 PostgresSql 的驱动（测试时用到）

Github 地址：https://github.com/lib/pq

`go get github.com/lib/pq`

### golang 单元测试结果判断和检查工具包（测试时用到）

GitHub 地址：https://github.com/stretchr/testify

`go get github.com/stretchr/testify`

### 开始编写单元测试模块

#### 编写单元测试入口函数 main_test.go

```go
package db

import (
	"database/sql"
	_ "github.com/lib/pq"
	"log"
	"os"
	"testing"
)

// 驱动，和连接数据库的url，root：用户名，xxxxxx：密码，后为地址端口和db
const (
	dbDriver = "postgres"
	dbSource = "postgresql://root:xxxxxx@localhost:5432/simple_bank?sslmode=disable"
)

var testQueries *Queries

// 单元测试入口函数
func TestMain(m *testing.M) {
        // 创建连接
	conn, err := sql.Open(dbDriver, dbSource)
	if err != nil {
		log.Fatal("cannot connect to db:", err)
	}
	testQueries = New(conn)
	os.Exit(m.Run())
}
```

然后运行以上文件可以看到入口函数测试通过，如下所示：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1a55111bc57843a5b8572a87a0d0c187~tplv-k3u1fbpfcp-watermark.image)

#### 接下来编写 account_test.sql 测试文件

```go
package db

import (
	"context"
	"testing"
        // 这个导入的是上面下载好的测试检测工具包
	"github.com/stretchr/testify/require"
)

func TestCreateAccount(t *testing.T) {
	arg := CreateAccountParams{
		Owner: "Tom",
		Balance: 100,
		Currentcy: "USD",
	}

	account, err := testQueries.CreateAccount(context.Background(), arg)
	// check the result of testing
	require.NoError(t, err)
	require.NotEmpty(t, account)

	// check input arg whether equal account
	require.Equal(t, arg.Owner, account.Owner)
	require.Equal(t, arg.Balance, account.Balance)
	require.Equal(t, arg.Currentcy, account.Currentcy)

	// check accountId and createAt
	require.NotZero(t, account.ID)
	require.NotZero(t, account.CreatedAt)
}
```
然后运行以上测试文件，结果通过，如下所示：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/db309e6d2be84365a9d07e30ef03cd3e~tplv-k3u1fbpfcp-watermark.image)

上面运行方式只能测试当前函数，如果要测试整个包的覆盖率的话，可以点击运行整个测试包，如下截图的操作：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e69686b68009457986d1f4c89f9082fc~tplv-k3u1fbpfcp-watermark.image)

可以看到测试通过，但是测试的覆盖率为 6.7%，也就是说还有其他函数没有编写到测试案例。如下所示，测试覆盖和未覆盖的函数颜色分别不一样：

实现其他函数的测试用例，大同小异，这里不再赘述。

## 项目地址

Github地址：https://github.com/Scoefield/simplebank

持续更新中......


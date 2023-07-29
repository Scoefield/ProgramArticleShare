---
highlight: a11y-dark
---

# 整体概述及架构

## 概述

使用阿里开源的 `Canal` 数据库同步工具，把 `Mysql` 数据的增删改同步到 `RabbitMQ` 或 `Kafka`，然后从 MQ 中拿消息处理再同步到 `ElasticSearch` 中。

## 整体架构及流程

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6637b1fe94db4196ad160086c57c54fa~tplv-k3u1fbpfcp-watermark.image)

### canal 介绍

**canal** 主要用途是基于 MySQL 数据库增量日志解析，提供增量数据订阅和消费，简单说就是可以对 MySQL 的增量数据进行实时同步。

### canal 工作原理

**canal** 会模拟 MySQL 主库和从库的交互协议，从而伪装成 MySQL 的从库，然后向 MySQL 主库发送 dump 协议，MySQL 主库收到 dump 请求会向 canal 推送 binlog，canal 通过解析 binlog 将数据同步到其他存储中去。

### 使用 canal 和 mq 组件同步 MySql 数据的整体流程

- Mysql --> Canal --> MQ（RabbitMq/Kafka） --> ElasticSearch

其实真正需要写代码实现的是：

- canal 同步数据到 MQ（SyncCanal2Mq）
- 消费 MQ 数据存储到 ElasticSearch（SyncMq2Elastic）

# 环境说明及安装

环境安装基于：Linux-centos7。

安装 docker 教程自行搜索安装。

由于不同版本的 MySQL、Elasticsearch 和 canal 会有兼容性问题，所以我们先对其使用版本做个约定。

| 应用          | 端口  | 版本   |
| ------------- | ----- | ------ |
| MySQL         | 3306  | 5.7    |
| Elasticsearch | 9200  | 7.6.2  |
| Kibanba       | 5601  | 7.6.2  |
| canal-server  | 11111 | 1.1.15 |
| canal-adapter | 8081  | 1.1.15 |
| canal-admin   | 8089  | 1.1.15 |

## 使用 docker 安装 MySql

参考地址: [docker 安装部署 mysql](https://www.cnblogs.com/root0/p/14606674.html)

### 拉取镜像

```bash
docker pull mysql:5.7
docker images
```

### 创建配置文件

```bash
[mysqld]
server_id=101
binlog-ignore-db=mysql
log-bin=mall-mysql-bin
binlog_cache_size=1M
binlog_format=row
expire_logs_days=7
slave_skip_errors=1062

character-set-server=utf8
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock

# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0

# Recommended in standard MySQL setup
sql_mode=NO_ENGINE_SUBSTITUTION

[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid

[mysql]
default-character-set=utf8

[client]
default-character-set=utf8
```

### 创建并部署 mysql 容器

```bash
# 创建宿主机 mysql 容器的数据和配置文件目录
mkdir /root/docker/mysql/{conf,data} -p
cd /root/docker/mysql/

# 创建名字为 mysql5.7 的容器
docker run -p 3306:3306 -v $PWD/data:/var/lib/mysql -v $PWD/conf/my.cnf:/etc/mysql/my.cnf -e MYSQL_ROOT_PASSWORD=your_password --name mysql5.7 -d mysql:5.7

# 查看运行的容器有哪些
docker ps
```

### 通过如下命令查看 binlog 是否启用

`show variables like '%log_bin%'`

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9514802be2aa4adb9d52db2bce358a9e~tplv-k3u1fbpfcp-watermark.image)

### 再查看下 MySQL 的 binlog 模式

`show variables like 'binlog_format%';`

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/886d1046bebb45359549e6e37d153fe8~tplv-k3u1fbpfcp-watermark.image)

### 接下来需要创建一个拥有从库权限的账号，用于订阅 binlog

这里创建的账号为`canal:canal`；

```sql
CREATE USER canal IDENTIFIED BY 'canal';  
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
FLUSH PRIVILEGES;
```

### 创建好测试用的数据库`statement`，之后创建一张表`track`

建表语句如下:

```sql
CREATE TABLE `track`  (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `title` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `sub_title` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
--   `price` decimal(10, 2) NULL DEFAULT NULL,
  `price` DOUBLE NOT NULL,
  `pic` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 2 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;
```

### 插入一些数据，用于后面测试

```sql
INSERT INTO track ( title, sub_title, price, pic ) VALUES ( 'iPhone4', ' 史上最智能手机 6GB+64GB', 4444.88, "http://mycanal/phone/pic.jpeg" );
INSERT INTO track ( title, sub_title, price, pic ) VALUES ( 'iPhone5', ' 史上最智能手机 6GB+64GB', 5555.00, "http://mycanal/phone/pic.jpeg" );
INSERT INTO track ( title, sub_title, price, pic ) VALUES ( 'iPhone6', ' 史上最智能手机 6GB+64GB', 6666.00, "http://mycanal/phone/pic.jpeg" );
```

## 使用 docker 安装 ElasticSearch

参考地址：[docker 安装 ES，记得一定要参考](https://blog.csdn.net/qq_40942490/article/details/111594267)

### 拉取镜像

```bash
docker pull elasticsearch:7.6.2
docker pull mobz/elasticsearch-head:5
docker images
```

### 创建 elasticsearch 和 es-head 容器

```bash
docker run --name myes -d -e  ES_JAVA_OPTS="-Xms512m -Xmx512m" -e "discovery.type=single-node" -p 9200:9200 -p 9300:9300 -v /etc/localtime:/etc/localtime:ro -v /home/es/data:/usr/share/elasticsearch/data  elasticsearch:7.6.2

docker run -d --name es-head -p 9100:9100 mobz/elasticsearch-head:5
```

### 访问浏览器查看 ES 概述信息

`http://192.168.7.130:9100/`

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/df13552fb6694f8383a1270203b4d254~tplv-k3u1fbpfcp-watermark.image)

## docker 安装 kibana（可选项）

参考地址：[使用 docker 安装 kibana](https://blog.csdn.net/shykevin/article/details/108272260)

### 拉取镜像并查看镜像

```bash
docker pull kibana:7.6.2
docker images
```

### 创建 kibana 配置文件及其挂载目录

```bash
mkdir -p /root/mydata/kibana/conf
cd /root/mydata/kibana/conf
```

`vim kibana.yml`

```yml
#
## ** THIS IS AN AUTO-GENERATED FILE **
#

# Default Kibana configuration for docker target
server.name: kibana
server.host: "0"
elasticsearch.hosts: ["http://192.168.7.130:9200"]
xpack.monitoring.ui.container.elasticsearch.enabled: true
```

**注意：请根据实际情况，修改 elasticsearch 地址。**

### 创建 kibana 容器

```bash
docker run -d --name=kibana --restart=always -p 5601:5601 -v $PWD/conf/kibana.yml:/usr/share/kibana/config/kibana.yml kibana:7.6.2
```

### 查看日志

`docker logs -f kibana`，等待 30 秒，如果出现以下信息，说明启动成功了。

```bash
{"type":"log","@timestamp":"2020-08-27T03:00:28Z","tags":["listening","info"],"pid":6,"message":"Server running at http://0:5601"}
{"type":"log","@timestamp":"2020-08-27T03:00:28Z","tags":["info","http","server","Kibana"],"pid":6,"message":"http server running at http://0:5601"}
```

### 访问 kibana 页面

```
http://192.168.31.190:5601/
```

效果如下，这里点击 `Explore on my own`

<div align=center><img src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bed6961963b1446e8fe71121b11ba4e9~tplv-k3u1fbpfcp-watermark.image" width = 500 /></div>

点击左侧的 `dev tools`，就可以通过 restful api 操作 ES 了：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e776bf3acea845798b07a0fddd00b4ec~tplv-k3u1fbpfcp-watermark.image)

## 安装 canal-server

参考地址：[安装 canal-server](https://blog.csdn.net/zhenghongcs/article/details/109476376)

### 组件下载地址

- 需要下载 canal 的各个组件 `canal-server`、`canal-adapter`、`canal-admin`，下载地址：https://github.com/alibaba/canal/releases

其实本次只用到 `canal-server` 组件。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/024cc375e6d143cdbaf2ff10285c6cd3~tplv-k3u1fbpfcp-watermark.image)

### 各组件简单介绍

- canal-server（canal-deploy）：可以直接监听 MySQL 的 binlog，把自己伪装成 MySQL 的从库，只负责接收数据，并不做处理。

- canal-adapter：相当于 canal 的客户端，会从 canal-server 中获取数据，然后对数据进行同步，可以同步到 MySQL、Elasticsearch 和 HBase 等存储中去。

- canal-admin：为 canal 提供整体配置管理、节点运维等面向运维的功能，提供相对友好的 WebUI 操作界面，方便更多用户快速和安全的操作。

### 安装 canal-server

- 将下载好的压缩包 `canal.deployer-1.1.5-SNAPSHOT.tar.gz` 上传到 Linux 服务器，然后解压到指定目录`/mydata/canal-server`，可使用如下命令解压:

  ```bash
  tar -zxvf canal.deployer-1.1.5-SNAPSHOT.tar.gz
  ```

### 修改配置文件 `conf/example/instance.properties`

- 按如下配置即可，主要是修改数据库相关配置

  ```bash
  # 需要同步数据的MySQL地址
  canal.instance.master.address=192.168.7.130:3306
  canal.instance.master.journal.name=
  canal.instance.master.position=
  canal.instance.master.timestamp=
  canal.instance.master.gtid=

  # 用于同步数据的数据库账号和密码以及编码
  canal.instance.dbUsername=canal
  canal.instance.dbPassword=canal
  canal.instance.connectionCharset = UTF-8

  # 需要订阅binlog的表正则表达式
  canal.instance.filter.regex=.*\\..*
  ```

### 启动 canal-server 服务

- 使用 `startup.sh` 脚本启动 `canal-server` 服务

  ```bash
  # 启动服务
  sh bin/startup.sh
  # 停止服务
  sh bin/stop.sh
  # 重启服务
  sh bin/restart.sh
  ```

- 可使用如下命令查看服务日志信息

  ```bash
  tailf logs/canal/canal.log
  ```

- 启动成功后可使用如下命令查看 instance 日志信息

  ```bash
  tailf logs/example/example.log
  ```

  日志内容

  ```bash
  2021-07-25 01:16:21.978 [main] INFO c.a.otter.canal.instance.spring.CanalInstanceWithSpring - start CannalInstance for 1-example
  2021-07-25 01:16:22.044 [main] WARN  c.a.o.canal.parse.inbound.mysql.dbsync.LogEventConvert - --> init table filter : ^.*\..*$
  2021-07-25 01:16:23.537 [main] INFO  c.a.otter.canal.instance.core.AbstractCanalInstance - start successful....
  2021-07-25 01:16:35.092 [destination = example , address = /192.168.7.130:3306 , EventParser] WARN  c.a.o.c.p.inbound.mysql.rds.RdsBinlogEventParserProxy - ---> find start position successfully, EntryPosition[included=false,journalName=mall-mysql-bin.000002,position=91467,serverId=101,gtid=,timestamp=1627139329000] cost : 11666ms , the next step is binlog dump
  ```

## docker 安装 rabbitmq

参考地址：[docker 安装 rabbitmq](https://www.cnblogs.com/sentangle/p/13201127.html)

### 拉取镜像、创建挂载目录 和 创建 rabbitmq 容器

```bash
# 拉取和查看镜像
docker pull rabbitmq
docker images

# 创建挂载目录
mkdir -p /root/docker/rabbitmq/data
cd /root/docker/rabbitmq/

# 创建 rabbitmq 容器
docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 -v $PWD/data:/var/lib/rabbitmq --hostname myRabbit -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=admin rabbitmq
```

说明：

- -d 后台运行容器；
- --name 指定容器名；
- -p 指定服务运行的端口（5672：应用访问端口；15672：控制台 Web 端口号）；
- -v 映射目录或文件（宿主机和容器的映射）；
- --hostname 主机名（RabbitMQ 的一个重要注意事项是它根据所谓的 “节点名称” 存储数据，默认为主机名）；
- -e 指定环境变量；（RABBITMQ_DEFAULT_VHOST：默认虚拟机名；RABBITMQ_DEFAULT_USER：默认的用户名；RABBITMQ_DEFAULT_PASS：默认用户名的密码）

### 启动 rabbitmq_management（web UI）

```bash
# 其中 rabbitmq 为上面的容器名
docker exec -it rabbitmq rabbitmq-plugins enable rabbitmq_management
```

- 浏览器打开 web 管理端：[http://ip:15672](http://ip:15672/)

登录的账号密码为上面创建容器时的账号密码（默认：admin/admin）

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/821f449b9bca4367992894c9fd7582de~tplv-k3u1fbpfcp-watermark.image)

## 实现数据同步

- Mysql -> canal，上面修改 mysql 和 canal 的配置文件，启动服务即可；
- canal 同步数据到 MQ（SyncCanal2Mq），需要实现 SyncCanal2Mq 的代码逻辑，后续继续补充和优化；
- 消费 MQ 数据存储到 ElasticSearch（SyncMq2Elastic），需要实现 SyncMq2Elastic 的代码逻辑，后续继续补充和优化。

代码见 GitHub 地址：https://github.com/Scoefield/syncMySqlToES

Gitee 地址：https://gitee.com/scoefield/syncMySqlToES

## 获取镜像

之前我们提到过 Docker 官方提供了一个公共的镜像仓库：Docker Hub，我们就可以从这上面获取镜像，获取镜像的命令：docker pull，格式为：

```
➜ docker pull [选项] [Docker Registry 地址[:端口]/]仓库名[:标签]
```

**镜像仓库地址**：地址的格式一般是 <域名/IP>[:端口号]，默认地址是 Docker Hub。

**仓库名**：这里的仓库名是两段式名称，即 <用户名>/<软件名>。对于 Docker Hub，如果不给出用户名，则默认为 library，也就是官方镜像。比如：

```
➜ docker pull routeman/user-api:v1
v1: Pulling from routeman/user-api
59bf1c3509f3: Pull complete
858b8406d0e4: Pull complete
3779a8cd7b07: Pull complete
8451c22ff927: Pull complete
0354926bb12f: Pull complete
Digest: sha256:925a1560e41009c60f8110398b068ebef45c0c61ef6728f87533521e8bc49529
Status: Downloaded newer image for routeman/user-api:v1
docker.io/routeman/user-api:v1
```

上面的命令中没有给出 Docker 镜像仓库地址，因此会默认从 **Docker Hub** 官方仓库获取镜像。而镜像名称是 routeman/user-api:v1，因此将会获取官方镜像 routeman/user-api 仓库中标签为 v1 的镜像。

从下载过程中可以看到我们之前提及的分层存储的概念，镜像是由多层存储所构成。下载也是一层层的去下载，并非单一文件。下载过程中给出了每一层的 ID 的前 12 位。并且下载结束后，给出该镜像完整的 sha256 的摘要，以确保下载一致性。

## 运行容器

有了镜像后，我们就能够以这个镜像为基础启动并运行一个容器。以上面的 ubuntu:16.04 为例，如果我们打算启动里面的 bash 并且进行交互式操作的话，可以执行下面的命令。

```
➜ docker run --rm -it -p 8888:8888 --name=user-api routeman/user-api:v1
Starting server at 0.0.0.0:8888...
```

可参考我的镜像仓库说明：[user-api:v1 镜像运行说明](https://hub.docker.com/repository/docker/routeman/user-api)

- docker run 就是运行容器的命令，具体格式我们会在后面的课程中进行详细讲解，我们这里简要的说明一下上面用到的参数。
- it：这是两个参数，一个是 -i：交互式操作，一个是 -t 终端。我们这里打算进入 bash 执行一些命令并查看返回结果，因此我们需要交互式终端。
- rm：这个参数是说容器退出后随之将其删除。默认情况下，为了排障需求，退出的容器并不会立即删除，除非手动 docker rm。我们这里只是随便执行个命令，看看结果，不需要排障和保留结果，因此使用 --rm 可以避免浪费空间。
- p：容器对外暴露端口，这里映射的端口为 8888。
- name：定义运行后容器的名称。
- routeman/user-api:v1 ：指用 routeman/user-api:v1 镜像为基础来启动容器。

**需要注意的是：**

当利用 docker run 来创建容器时，Docker 在后台运行的标准操作包括：

- 检查本地是否存在指定的镜像，不存在就从公有仓库下载
- 利用镜像创建并启动一个容器
- 分配一个文件系统，并在只读的镜像层外面挂载一层可读写层
- 从宿主主机配置的网桥接口中桥接一个虚拟接口到容器中去
- 从地址池配置一个 ip 地址给容器
- 执行用户指定的应用程序
- 执行完毕后容器被终止

## 查看有哪些镜像

```
➜ docker images
```

列表包含了仓库名、标签、镜像 ID、创建时间以及所占用的空间。镜像 ID 则是镜像的唯一标识，一个镜像可以对应多个标签。

## 后台运行容器

更多的时候，需要让 Docker 在后台运行而不是直接把执行命令的结果输出在当前宿主机下。此时，可以通过添加-d 参数来实现。下面举两个例子来说明一下。

```
➜ docker run --rm -it -d -p 8888:8888 --name=user-api routeman/user-api:v1
2e2bf5e8b16e724f5315045247024db7215971d5d77cc1c12aaed64995eaa866
➜
➜ docker ps |grep user-api
2e2bf5e8b16e   routeman/user-api:v1  "./user -f etc/user-…"  16 seconds ago  Up 15 seconds    0.0.0.0:8888->8888/tcp   user-api
```

使用-d 参数启动后会返回一个唯一的 id，也可以通过 docker container ls 命令来查看容器信息。

## 进入容器

在使用-d 参数时，容器启动后会进入后台。某些时候需要进入容器进行操作：exec 命令 -i -t 参数。

只用-i 参数时，由于没有分配伪终端，界面没有我们熟悉的 Linux 命令提示符，但命令执行结果仍然可以返回。 当-i -t 参数一起使用时，则可以看到我们熟悉的 Linux 命令提示符。

```
➜ docker exec -it 2e2bf5e8b16e /bin/sh
/app #
/app # ls
etc   user
```

如果从这个 stdin 中 exit，不会导致容器的停止。这就是为什么推荐大家使用 docker exec 的原因。

> 更多参数说明请使用 docker exec --help 查看。

## 删除容器

可以使用 docker container rm 来删除一个处于终止状态的容器。例如:

```
➜ docker container rm user-api
```

也可用使用 docker rm 容器名来删除，如果要删除一个运行中的容器，可以添加-f 参数。Docker 会发送 SIGKILL 信号给容器。

用 docker container ls -a (或者 docker ps -a)命令可以查看所有已经创建的包括终止状态的容器，如果数量太多要一个个删除可能会很麻烦，用下面的命令可以清理掉所有处于终止状态的容器。

```
➜ docker container prune
```

## 删除本地镜像

如果要删除本地的镜像，可以使用`docker image rm·命令，其格式为：

```
➜ docker image rm [选项] <镜像1> [<镜像2> ...]
```

或者

```
➜ docker rmi
```

或者用 ID、镜像名、摘要删除镜像 其中，<镜像> 可以是 镜像短 ID、镜像长 ID、镜像名 或者 镜像摘要。 比如我们有这么一些镜像：

```
➜ docker images
REPOSITORY                               TAG                 IMAGE ID            CREATED             SIZE
routeman/user-api                        v1                  8d8edc6be0e2        2 days ago          19.7MB
postgres                                 12-alpine           a58cf5527d36        8 months ago        158MB
mysql                                    5.7                 2c9028880e58        8 months ago        447MB
ubuntu                                   16.04               b9409899fe86        2 years ago         122MB
```

我们可以用镜像的完整 ID，也称为 长 ID，来删除镜像。使用脚本的时候可能会用长 ID，但是人工输入就太累了，所以更多的时候是用 短 ID 来删除镜像。docker image ls 默认列出的就已经是短 ID 了，一般取前 3 个字符以上，只要足够区分于别的镜像就可以了。

比如这里，如果我们要删除 mysql:5.7 镜像，可以执行：

```
➜ docker rmi 2c9
```

我们也可以用镜像名，也就是 <仓库名>:<标签>，来删除镜像。

```
➜ docker rmi mysql:5.7
```

......

> docker 的更多基本操作，可以参考官方文档。

下面再来讲讲如何通过编写 **Dockerfile** 文件来定制镜像。

## Dockerfile 定制镜像

镜像的定制实际上就是定制每一层所添加的配置、文件等信息，但是命令毕竟只是命令，每次定制都得去重复执行这个命令，而且还不够直观，如果我们可以把每一层修改、安装、构建、操作的命令都写入一个脚本，用这个脚本来构建、定制镜像，那么这些问题不就都可以解决了吗？对的，这个脚本就是我们说的 Dockerfile。

### Dockerfile 简单介绍

Dockerfile 是一个文本文件，其内包含了一条条的指令(Instruction)，每一条指令构建一层，因此每一条指令的内容，就是描述该层应当如何构建。

以上面我自己的一个镜像 `user-api` 为例，这次我们使用 Dockerfile 来定制。目录结构如下：

```
routeman
├── go.mod
├── go.sum
└── service
    └── user-api
        ├── Dockerfile
        ├── etc
        │   └── user-api.yaml
        ├── user-api.go
        └── cmd
            ├── config
            │   └── config.go
            ├── handler
            │   ├── user-api-handler.go
            │   └── routes.go
            ├── logic
            │   └── user-api-logic.go
            ├── svc
            │   └── servicecontext.go
            └── types
                └── types.go
```

Dockerfile 内容为：

```
FROM golang:alpine AS builder
LABEL stage=gobuilder
ENV CGO_ENABLED 0
ENV GOOS linux
ENV GOPROXY https://goproxy.cn,direct
WORKDIR /build/routeman
ADD go.mod .
ADD go.sum .
RUN go mod download
COPY . .
COPY service/user-api/etc /app/etc
RUN go build -ldflags="-s -w" -o /app/user-api service/user-api/user-api.go

WORKDIR /app
COPY --from=builder /app/user-api /app/user-api
COPY --from=builder /app/etc /app/etc
CMD ["./user-api", "-f", "etc/user-api.yaml"]
```

> Dockerfile 命令详细说明参考官方文档：[Dockerfile reference](https://docs.docker.com/engine/reference/builder/)

### 通过编写好的 Dockerfile 构建镜像

首先进到 routeman 目录下，即 go.mod 和 go.sum 目录下执行：

```
➜ docker built -t routeman/user-api:v1 -f service/user-api/Dockerfile .
```

这里我们使用了 docker build 命令进行镜像构建。其格式为：

```
➜ docker built [选项] <上下文路径/URL/>
```

可以重新定义标签：

```
# 格式
➜ docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]
# 修改标签命令
➜ docker tag routeman/user-api:v1 routeman/user-api:v2
```

### 迁移镜像

Docker 还提供了 docker load 和 docker save 命令，用以将镜像保存为一个 tar 文件，然后传输到另一个位置上，再加载进来。这是在没有 Docker Registry 时的做法，现在已经不推荐，镜像迁移应该直接使用 Docker Registry，无论是直接使用 Docker Hub 还是使用内网私有 Registry 都可以。

```
# 打包镜像，拷贝到其它机器
docker save -o user-api.tar routeman/user-api:v1
scp user-api.tar username@ip:/dir/...

# 加载镜像包
docker load -i user-api.tar
```

## 小结

以上是关于 Docker 镜像和容器的一些基本操作说明，以及如何通过编写 **Dockerfile** 文件来构建镜像。主要以自己写的一个 **user-api** 镜像为例来进行讲解。更高级的命令及操作可查看官方文档。

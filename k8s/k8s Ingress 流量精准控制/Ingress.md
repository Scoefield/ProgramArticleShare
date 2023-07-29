## 前言

在 Kubernetes 中，Ingress 是一个非常重要的概念。它可以将外部流量路由到 Kubernetes 集群内的不同服务。

Ingress 可以让你更加方便地管理 HTTP 和 HTTPS 流量，并且可以配置负载均衡、SSL 证书等功能。本文将会介绍 Ingress 的定义、类型、更新、以及相关的控制器和类别。

## 什么是 Ingress

![](https://files.mdnice.com/user/24277/a60e0479-ae8c-4515-8ce6-f1061e75a904.png)

通常情况下，service 和 pod 的 IP 仅可在集群内部访问。集群外部的请求需要通过负载均衡转发到 service 在 Node 上暴露的 NodePort 上，然后再由 kube-proxy 通过边缘路由器 (edge router) 将其转发给相关的 Pod 或者丢弃。

Ingress 是 Kubernetes 的一个 API 对象，它定义了如何将外部流量路由到 Kubernetes 集群内的不同服务。它可以通过 HTTP 和 HTTPS 协议进行流量路由，并且支持域名和路径的匹配。

Ingress 是一个非常强大的功能，可以让你更加方便地管理流量，并且可以轻松实现负载均衡、SSL 证书等功能。

## Ingress 的定义格式

Ingress 的定义格式如下：

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ingress-name
  annotations:
    key: value
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /path
            backend:
              serviceName: service-name
              servicePort: service-port
```

其中，`apiVersion`字段表示 Ingress 对象的 API 版本，`kind`字段表示对象的类型，`metadata`字段包含了 Ingress 对象的元数据，例如对象名称和标签等，`spec`字段则包含了 Ingress 对象的配置信息，例如规则和后端服务等。

## Ingress 的类型有哪几种？

Kubernetes 支持多种不同类型的 Ingress，每种类型都有自己的特点和用途。下面是一些常见的 Ingress 类型：

### 1. Simple fanout

简单的 Fanout Ingress 会将流量路由到指定的多个服务上。它可以通过域名和路径进行匹配，并且支持负载均衡功能。

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: simple-fanout-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /service1
        backend:
          serviceName: service1
          servicePort: 80
      - path: /service2
        backend:
          serviceName: service2
          servicePort: 80

```

### 2. Name-based virtual hosting

![](https://files.mdnice.com/user/24277/f4bb2aab-6919-4972-9c4b-464d26f11897.png)

基于名称的虚拟主机 Ingress 会将流量路由到指定的服务上，具体的服务由请求的 Host 头部决定。它可以通过域名进行匹配，并且支持负载均衡和 SSL 证书等功能。

```
foo.example.com --|                 |-> foo.example.com s1:80
                  | 168.92.133.131  |
bar.example.com --|                 |-> bar.example.com s2:80
```

下面是一个基于 Host header 路由请求的 Ingress：

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: name-based-virtual-hosting-ingress
spec:
  rules:
  - host: foo.example.com
    http:
      paths:
      - backend:
          serviceName: service1
          servicePort: 80
  - host: bar.example.com
    http:
      paths:
      - backend:
          serviceName: service2
          servicePort: 80

```

### 3. Path-based routing

基于路径的路由 Ingress 会将流量路由到指定的服务上，具体的服务由请求的 URL 路径决定。它可以通过路径进行匹配，并且支持负载均衡和 SSL 证书等功能。

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: path-based-routing-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /service1
        backend:
          serviceName: service1
          servicePort: 80
      - path: /service2
        backend:
          serviceName: service2
          servicePort: 80

```

## 该如何实现更新 Ingress

更新 Ingress 非常简单，只需要修改 Ingress 对象的定义文件，然后执行`kubectl apply`命令即可。例如，假设我们想要将`simple-fanout-ingress`的路径`/service1`修改为`/service3`，则可以编辑 Ingress 对象的定义文件，然后执行以下命令：

```
$ kubectl apply -f ingress.yaml

```

其中，`ingress.yaml`是 Ingress 对象的定义文件。

更新后：

```
$ kubectl get ing
NAME      RULE          BACKEND   ADDRESS
test      -                       168.92.133.131
          foo.example.com
          /foo          s1:80
          bar.example.com
          /bar          s2:80
```

## Ingress Controller

Ingress Controller 是一个运行在 Kubernetes 集群内的服务，它可以监听 Kubernetes API 服务器上的 Ingress 对象，并将外部流量路由到 Kubernetes 集群内的不同服务。

每种类型的 Ingress 都需要特定的 Ingress Controller 来处理。例如，简单的 Fanout Ingress 需要使用 Nginx Ingress Controller，而基于名称的虚拟主机 Ingress 则需要使用 Traefik Ingress Controller。

## Ingress Class

Ingress Class 是一个可选的字段，它可以让你更加精细地控制 Ingress 对象的路由方式。

每个 Ingress 对象都可以指定一个 Ingress Class，这个 Ingress Class 可以对应不同的 Ingress Controller，并且可以让你更加方便地控制路由方式。

## 总结

Ingress 是 Kubernetes 中非常重要的一个资源对象，它可以将外部流量路由到 Kubernetes 集群内的不同服务，并且支持多种不同类型的路由方式。

换言之，Ingress 就是为进入集群的请求提供路由规则的集合，可以给 service 提供集群外部访问的 URL、负载均衡、SSL 终止、HTTP 路由等功能。

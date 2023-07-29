本文将介绍 Kubernetes 的资源对象 Service，内容包括 Service 介绍、Service 的四种类型及使用方式、不指定 Selectors 的服务、Headless 服务、Service 工作原理及原理图。同时也会讲解 Ingress 和集群外部如何访问服务。

## 前言

在容器编排系统中，如 Kubernetes，Pod 是最小的部署单元。而一组 Pod 通常对外提供某种服务。在 Kubernetes 中，Service 就是用来对外暴露一组 Pod 的服务的资源对象。Service 可以通过 IP 地址和端口号访问，从而对外提供服务。

## Service 介绍

Service 是 Kubernetes 中一个非常重要的概念，它可以将一组 Pod 封装成一个逻辑服务单元。

Service 可以通过定义的 Label Selector，将一组 Pod 绑定到一起，形成一个 Service。通过 Service，用户可以方便地访问这个服务，不需要关心 Pod 的具体 IP 地址和端口号，也不需要担心 Pod 的数量变化会影响服务的访问。

## Service 的四种类型及使用方式

Kubernetes 中的 Service 一共有四种类型，分别是 ClusterIP、NodePort、LoadBalancer 和 ExternalName。

- ClusterIP：是 Service 的默认类型，它会为 Service 创建一个 Cluster IP，这个 IP 只能在集群内部访问。通过 ClusterIP，用户可以访问该 Service 关联的 Pod。
- NodePort：在每个节点上绑定一个端口，从而将 Service 暴露到集群外部。用户可以通过任意一个节点的 IP 地址和该端口号来访问 Service。
- LoadBalancer：在云厂商提供的负载均衡器上创建一个 VIP，从而将 Service 暴露到集群外部。用户可以通过该 VIP 地址来访问 Service。
- ExternalName：可以将 Service 映射到集群外部的一个 DNS 名称上，从而将 Service 暴露到集群外部。

另外，也可以将已有的服务以 Service 的形式加入到 Kubernetes 集群中来，只需要在创建 Service 的时候不指定 Label selector，而是在 Service 创建好后手动为其添加 endpoint。

## Service 的定义和使用

Service 也是可以通过 yaml 来定义的。

以下是带有 selector 和 type 的 YAML 文件定义一个名为 nginx 的 Service，将服务的 80 端口转发到 default namespace 中带有标签 run=nginx 的 Pod 的 80 端口：

```
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    run: nginx
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

其中，type 为 Service 的类型，可以取值为 ClusterIP、NodePort、LoadBalancer 和 ExternalName。

本例中的 type 为 ClusterIP，表示该 Service 的类型为 ClusterIP。

以下是通过命令创建及查看创建的服务情况的操作步骤和预期的展示内容。

### 通过命令创建服务

1. 使用 `kubectl create` 命令创建一个名为 `nginx` 的服务，并将服务的 80 端口转发到 default namespace 中带有标签 `run=nginx` 的 Pod 的 80 端口。运行以下命令：

   ```bash
   kubectl create service clusterip nginx --tcp=80:80 --dry-run=client -o yaml > nginx-service.yaml
   ```

   这个命令将在当前目录下生成一个名为 `nginx-service.yaml` 的 YAML 文件，其中包含了创建 Service 所需的配置信息。

   预期展示内容：

   ```
   service/nginx created (dry run)
   ```

2. 使用 `kubectl apply` 命令创建 Service。运行以下命令：

   ```
   kubectl apply -f nginx-service.yaml
   ```

   这个命令将根据 YAML 文件中的配置信息创建一个名为 `nginx` 的 Service。

   预期展示内容：

   ```
   service/nginx created
   ```

### 查看创建的服务情况

1. 使用 `kubectl get` 命令查看已经创建的 Service。运行以下命令：

   ```
   kubectl get services
   ```

   这个命令将显示所有已经创建的 Service，包括它们的名称、类型、Cluster IP、端口等信息。

   预期展示内容：

   ```
   NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
   kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP    4h19m
   nginx        ClusterIP   10.96.58.173   <none>        80/TCP     1m
   ```

2. 使用 `kubectl describe` 命令查看指定 Service 的详细信息。运行以下命令：

   ```
   kubectl describe service nginx
   ```

   这个命令将显示名为 `nginx` 的 Service 的详细信息，包括它的类型、Cluster IP、端口、Selector 等信息。

   预期展示内容：

   ```
   Name:              nginx
   Namespace:         default
   Labels:            run=nginx
   Annotations:       <none>
   Selector:          run=nginx
   Type:              ClusterIP
   IP:                10.96.58.173
   Port:              <unset>  80/TCP
   TargetPort:        80/TCP
   Endpoints:         10.244.0.7:80
   Session Affinity:  None
   Events:            <none>
   ```

## 不指定 Selectors 的服务

当用户创建 Service 的时候，可以不指定 Label Selector，这种 Service 被称为无选择器服务（no selector service）。无选择器服务不能和 Pod 绑定，而只是提供一个固定的 IP 和端口，用于访问后端服务。这种服务通常用于代理到外部的服务，如数据库、消息队列等。

## Headless 服务

Kubernetes 中的 Service 还有一个特殊的类型，叫做 Headless 服务。Headless 服务的 ClusterIP 为 None，它不会为 Service 创建 Cluster IP。通过 Headless 服务，用户可以直接访问该 Service 关联的 Pod，而不需要通过 Service 进行访问。

## Service 工作原理及原理图

Service 的工作原理是通过代理模式实现的，即 kube-proxy 负责将 service 负载均衡到后端 Pod 中。

当用户通过 Service 的 IP 和端口访问 Service 时，请求会先到达 Service 代理，然后由代理将请求转发给后端的 Pod。

当 Pod 发生变化时，Service 会自动更新 Endpoint，从而保证请求能够正确地到达后端 Pod。

以下是 Service 工作原理的原理图：

![](https://files.mdnice.com/user/24277/dc4a2a12-3ec0-455e-8b23-9aae4b551cb7.jpg)


## Ingress 讲解

Ingress 是 Kubernetes 中另一个重要的资源对象，它用于将集群外部的 HTTP(S) 流量路由到集群内部的 Service。通过 Ingress，用户可以在集群外部定义一个域名，然后将该域名路由到 Service 中。Ingress 可以实现灰度发布、负载均衡、SSL 终止等功能。

## 集群外部如何访问服务

当用户需要从集群外部访问 Kubernetes 中的 Service 时，可以通过 NodePort、LoadBalancer 或 Ingress 来实现。具体方式如下：

- NodePort

用户可以通过任意一个节点的 IP 地址和该节点上绑定的端口号来访问 Service。例如，如果 NodePort 的端口为 30080，节点 IP 地址为 192.168.0.10，则用户可以通过 [http://192.168.0.10:30080](http://192.168.0.10:30080/) 访问该 Service。

- LoadBalancer

当 Service 的类型为 LoadBalancer 时，云厂商会在其提供的负载均衡器上为 Service 创建一个 VIP，用户可以通过该 VIP 地址来访问 Service。

- Ingress

通过 Ingress，用户可以在集群外部定义一个域名，并将该域名路由到 Service 中。用户可以通过该域名来访问 Service。

## 总结

以上是关于 Kubernetes 中的 Service 的介绍，包括 Service 介绍和定义、Service 的四种类型 Service 工作原理、Ingress 讲解以及集群外部如何访问服务。

在使用 Service 时，用户不需要关心 Pod 的具体 IP 地址和端口号，也不需要担心 Pod 的数量变化会影响服务的访问。

通过 Ingress，用户可以在集群外部定义一个域名，然后将该域名路由到 Service 中，从而实现灰度发布、负载均衡、SSL 终止等功能。

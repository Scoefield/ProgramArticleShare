## 前言

随着微服务的流行，越来越多的云服务平台需要大量模块之间的网络调用。Kubernetes 在 1.3 引入了 Network Policy，Network Policy 提供了基于策略的网络控制，用于隔离应用并减少攻击面。它使用标签选择器模拟传统的分段网络，并通过策略控制它们之间的流量以及来自外部的流量。

## 网络策略

### Namespace 隔离

默认情况下，所有 Pod 之间是全通的。每个 Namespace 可以配置独立的网络策略，来隔离 Pod 之间的流量。
通过创建匹配所有 Pod 的 Network Policy 来作为默认的网络策略，比如默认拒绝所有 Pod 之间 Ingress 通信：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

默认拒绝所有 Pod 之间 Egress 通信的策略为：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
    - Egress
```

甚至是默认拒绝所有 Pod 之间 Ingress 和 Egress 通信的策略为：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

## Pod 隔离

Pod 有两种隔离: 出口的隔离和入口的隔离。它们涉及到可以建立哪些连接。 这里的“隔离”不是绝对的，而是意味着“有一些限制”。

通过使用标签选择器（包括 namespaceSelector 和 podSelector）来控制 Pod 之间的流量。比如下面的 Network Policy

- 允许 default namespace 中带有 role=frontend 标签的 Pod 访问 default namespace 中带有 role=db 标签 Pod 的 6379 端口
- 允许带有 project=myprojects 标签的 namespace 中所有 Pod 访问 default namespace 中带有 role=db 标签 Pod 的 6379 端口

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              project: myproject
        - podSelector:
            matchLabels:
              role: frontend
      ports:
        - protocol: tcp
          port: 6379
```

另外一个同时开启 Ingress 和 Egress 通信的策略为:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - ipBlock:
            cidr: 172.17.0.0/16
            except:
              - 172.17.1.0/24
        - namespaceSelector:
            matchLabels:
              project: myproject
        - podSelector:
            matchLabels:
              role: frontend
      ports:
        - protocol: TCP
          port: 6379
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.0.0/24
      ports:
        - protocol: TCP
          port: 5978
```

它用来隔离 default namespace 中带有 role=db 标签的 Pod：

- 允许 default namespace 中带有 role=frontend 标签的 Pod 访问 default namespace 中带有 role=db 标签 Pod 的 6379 端口
- 允许带有 project=myprojects 标签的 namespace 中所有 Pod 访问 default namespace 中带有 role=db 标签 Pod 的 6379 端口
- 允许 default namespace 中带有 role=db 标签的 Pod 访问 10.0.0.0/24 网段的 TCP 5987 端口

## 简单示例

以 calico 为例看一下 Network Policy 的具体用法。
首先配置 kubelet 使用 CNI 网络插件：

```
kubelet --network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin ...
```

安装 calio 网络插件

```
# 注意修改 CIDR，需要跟 k8s pod-network-cidr 一致，默认为 192.168.0.0/16
kubectl apply -f https://docs.projectcalico.org/v3.0/getting-started/kubernetes/installation/hosted/kubeadm/1.7/calico.yaml
```

首先部署一个 nginx 服务

```
$ kubectl run nginx --image=nginx --replicas=2
deployment "nginx" created
$ kubectl expose deployment nginx --port=80
service "nginx" exposed
```

此时，通过其他 Pod 是可以访问 nginx 服务的:

```
$ kubectl get svc,pod
NAME                        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
svc/kubernetes              10.101.0.1    <none>        443/TCP    46m
svc/nginx                   10.101.0.16   <none>        80/TCP     33s
​
NAME                        READY         STATUS        RESTARTS   AGE
po/nginx-601339713-e0qfq    1/1           Running       0          35s
po/nginx-601339713-o00ef    1/1           Running       0
​
$ kubectl run busybox --rm -ti --image=busybox /bin/sh
Waiting for pod default/busybox-472357175-y0m47 to be running, status is Pending, pod ready: false
​
Hit enter for command prompt
​
/ # wget --spider --timeout=1 nginx
Connecting to nginx (10.101.0.16:80)
/ #
```

开启 default namespace 的 DefaultDeny Network Policy 后，其他 Pod（包括 namespace 外部）不能访问 nginx 了：

```
$ cat default-deny.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
​
$ kubectl create -f default-deny.yaml
​
$ kubectl run busybox --rm -ti --image=busybox /bin/sh
Waiting for pod default/busybox-472357175-y0m47 to be running, status is Pending, pod ready: false
​
Hit enter for command prompt
​
/ # wget --spider --timeout=1 nginx
Connecting to nginx (10.101.0.16:80)
wget: download timed out
/ #
```

最后再创建一个运行带有 access=true 的 Pod 访问的网络策略:

```
$ cat nginx-policy.yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: access-nginx
spec:
  podSelector:
    matchLabels:
      run: nginx
  ingress:
  - from:
    - podSelector:
        matchLabels:
          access: "true"
​
$ kubectl create -f nginx-policy.yaml
networkpolicy "access-nginx" created
​
# 不带 access=true 标签的 Pod 还是无法访问 nginx 服务
$ kubectl run busybox --rm -ti --image=busybox /bin/sh
Waiting for pod default/busybox-472357175-y0m47 to be running, status is Pending, pod ready: false
​
Hit enter for command prompt
​
/ # wget --spider --timeout=1 nginx
Connecting to nginx (10.101.0.16:80)
wget: download timed out
/ #
​
​
# 而带有 access=true 标签的 Pod 可以访问 nginx 服务
$ kubectl run busybox --rm -ti --labels="access=true" --image=busybox /bin/sh
Waiting for pod default/busybox-472357175-y0m47 to be running, status is Pending, pod ready: false
​
Hit enter for command prompt
​
/ # wget --spider --timeout=1 nginx
Connecting to nginx (10.101.0.16:80)
/ #
```

最后开启 nginx 服务的外部访问：

```
$ cat nginx-external-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: front-end-access
  namespace: sock-shop
spec:
  podSelector:
    matchLabels:
      run: nginx
  ingress:
    - ports:
        - protocol: TCP
          port: 80
​
$ kubectl create -f nginx-external-policy.yaml
```

## 针对某个端口范围

在编写 NetworkPolicy 时，你可以针对一个端口范围而不是某个固定端口。

这一目的可以通过使用 endPort 字段来实现，如下例所示：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: multi-port-egress
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.0.0/24
      ports:
        - protocol: TCP
          port: 32000
          endPort: 32768
```

上面的规则允许名字空间 default 中所有带有标签 role=db 的 Pod 使用 TCP 协议与 10.0.0.0/24 范围内的 IP 通信，只要目标端口介于 32000 和 32768 之间就可以。

使用此字段时存在以下限制：

- endPort 字段必须等于或者大于 port 字段的值。
- 只有在定义了 port 时才能定义 endPort。
- 两个字段的设置值都只能是数字。

## 使用场景

### 禁止访问指定服务

```
kubectl run web --image=nginx --labels app=web,env=prod --expose --port 80
```

网络策略:

```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: web-deny-all
spec:
  podSelector:
    matchLabels:
      app: web
      env: prod
```

### 只允许指定 Pod 访问服务

```
kubectl run apiserver --image=nginx --labels app=bookstore,role=api --expose --port 80
```

网络策略:

```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: api-allow
spec:
  podSelector:
    matchLabels:
      app: bookstore
      role: api
  ingress:
  - from:
      - podSelector:
          matchLabels:
            app: bookstore
```

### 禁止 namespace 中所有 Pod 之间的相互访问

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: default
spec:
  podSelector: {}
```

### 禁止其他 namespace 访问服务

```
kubectl create namespace secondary
kubectl run web --namespace secondary --image=nginx \
    --labels=app=web --expose --port 80
```

网络策略

```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: secondary
  name: web-deny-other-namespaces
spec:
  podSelector:
    matchLabels:
  ingress:
  - from:
    - podSelector: {}
```

### 只允许指定 namespace 访问服务

```
kubectl run web --image=nginx \
    --labels=app=web --expose --port 80
```

网络策略

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: web-allow-prod
spec:
  podSelector:
    matchLabels:
      app: web
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              purpose: production
```

### 允许外网访问服务

```
kubectl run web --image=nginx --labels=app=web --port 80
kubectl expose deployment/web --type=LoadBalancer
```

网络策略

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: web-allow-external
spec:
  podSelector:
    matchLabels:
      app: web
  ingress:
    - ports:
        - port: 80
      from: []
```

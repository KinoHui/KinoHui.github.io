---
title: "K8s 节点排障（三）：kube-proxy、iptables 与 Service 转发"
date: 2026-06-14T8:00:00+08:00
draft: false
tags: ["Kubernetes", "Linux", "排障", "kube-proxy", "iptables", "Service", "NodePort", "conntrack"]
categories: ["云原生", "Linux 排障"]
series: ["K8s 节点排障实战"]
summary: "深入理解 kube-proxy 工作原理、iptables 转发规则、DNAT/SNAT/MASQUERADE 机制，以及集群内外请求的生命周期"
weight: 3
---

## 1. 背景

当前集群里的 Service 示例：

```text
NAMESPACE     NAME         TYPE        CLUSTER-IP    PORT(S)
default       kubernetes   ClusterIP   10.96.0.1     443/TCP
default       nginx-svc    NodePort    10.98.59.52   80:31233/TCP
kube-system   kube-dns     ClusterIP   10.96.0.10    53/UDP,53/TCP,9153/TCP
```

`nginx-svc` 的信息：

```text
Type: NodePort
ClusterIP: 10.98.59.52
Service Port: 80
NodePort: 31233
```

Endpoints：

```text
10.244.169.177:80  node2
10.244.169.179:80  node2
10.244.235.242:80  master
```

Node IP：

```text
k8s-master: 192.168.59.134
k8s-node1:  192.168.59.135
k8s-node2:  192.168.59.136
```

Pod 网段：

```text
10.244.0.0/16
```

---

## 2. Service 的 PORT(S) 怎么看

### ClusterIP Service

```text
kubernetes   ClusterIP   10.96.0.1   443/TCP
```

表示：

```text
Service IP: 10.96.0.1
Service Port: 443
Protocol: TCP
```

访问方式：

```bash
curl -k https://10.96.0.1:443
```

### NodePort Service

```text
nginx-svc   NodePort   10.98.59.52   80:31233/TCP
```

`80:31233/TCP` 的含义：

```text
80:
Service 的 port，也就是 ClusterIP 上的端口。

31233:
NodePort，也就是每个 Node IP 上暴露的端口。

TCP:
协议。
```

集群内访问 ClusterIP：

```bash
curl http://10.98.59.52:80
```

Pod 内访问 Service 名：

```bash
wget -qO- http://nginx-svc:80
```

从节点或集群外访问 NodePort：

```bash
curl http://192.168.59.134:31233
curl http://192.168.59.135:31233
curl http://192.168.59.136:31233
```

### 多端口 Service

```text
kube-dns   ClusterIP   10.96.0.10   53/UDP,53/TCP,9153/TCP
```

表示：

```text
53/UDP:
DNS 查询常用端口。

53/TCP:
DNS 也支持 TCP，常用于大响应或重试等场景。

9153/TCP:
CoreDNS metrics 端口。
```

字段对应关系：

```yaml
ports:
- port: 80        # Service ClusterIP 的端口
  targetPort: 80  # 后端 Pod 容器端口
  nodePort: 31233 # NodePort 端口，只有 NodePort/LoadBalancer 类型有
  protocol: TCP
```

---

## 3. NodePort 是不是只有有 Pod 的节点才能访问

不是。

准确说：

```text
NodePort 默认会在 Service 所在集群的每个 Node 上暴露端口。
```

例如 `nginx-svc` 的 NodePort 是 `31233`，默认情况下即使 `k8s-node1` 上没有 nginx Pod，也通常可以访问：

```bash
curl http://192.168.59.135:31233
```

kube-proxy 会把请求转发到真正的后端 Pod：

```text
10.244.169.177:80
10.244.169.179:80
10.244.235.242:80
```

重要例外是 `externalTrafficPolicy`。

默认：

```text
externalTrafficPolicy: Cluster
```

特点：

```text
任意 NodeIP:NodePort 都可以访问，哪怕该 Node 没有本地后端 Pod。
流量可以跨节点转发到其他 Node 上的 Endpoint。
通常会 SNAT，后端 Pod 可能看不到真实客户端源 IP。
```

如果设置为：

```text
externalTrafficPolicy: Local
```

特点：

```text
只有本节点有 ready 后端 Pod 的 Node 才转发。
不会跨节点转发到其他 Node 的 Pod。
通常可以保留真实客户端源 IP。
访问没有本地 Endpoint 的 NodePort 可能失败。
```

---

## 4. kube-proxy 的工作原理

一句话：

```text
kube-proxy watch Service 和 EndpointSlice，在每个 Node 上生成转发规则，让访问 ClusterIP/NodePort 的流量被内核转发到真实 Pod IP。
```

### Service 本身不是一个进程

例如：

```text
nginx-svc
ClusterIP: 10.98.59.52
Port: 80
Endpoints:
10.244.169.177:80
10.244.169.179:80
10.244.235.242:80
```

`10.98.59.52` 不是某个 Pod，也不是某张网卡上的真实 IP。

通常执行：

```bash
ip addr | grep 10.98.59.52
```

查不到。

它是一个虚拟入口，由 kube-proxy 写入 Linux 转发规则来实现。

### kube-proxy 监听 apiserver

每个 Node 上都有 kube-proxy。它会 watch：

```text
Service
EndpointSlice / Endpoints
```

当 Service 或 Endpoint 变化时，kube-proxy 会把最新信息同步成本机的 iptables 或 IPVS 规则。

### kube-proxy 不是每包代理

名字叫 proxy，但在 iptables/IPVS 模式下，它不是传统意义上的用户态代理。

不是：

```text
客户端 -> kube-proxy 进程 -> 后端 Pod
```

而是：

```text
kube-proxy 负责写规则。
真正转发发生在 Linux 内核网络栈。
```

所以更准确地说：

```text
kube-proxy 是 Service 规则控制器，不是每个包都经过的用户态代理进程。
```

### 常见模式

```text
userspace:
老模式，基本不用了。真正由 kube-proxy 进程代理流量。

iptables:
常见模式。kube-proxy 写 iptables 规则，由内核 DNAT 转发。

ipvs:
使用 Linux IPVS 负载均衡能力，适合大规模 Service/Endpoint 场景。
```

当前集群是 iptables 模式。

证据：

```bash
sudo iptables-save | grep 10.98.59.52
```

能看到：

```text
KUBE-SERVICES
KUBE-SVC-...
```

而：

```bash
sudo ipvsadm -Ln
```

没有输出。

---

## 5. iptables 规则基础：`-s`、`-d`、`!`、`-j`

示例规则：

```bash
-A KUBE-SVC-HL5LMXD5JFHQZ6LN \
  ! -s 10.244.0.0/16 \
  -d 10.98.59.52/32 \
  -p tcp \
  -j KUBE-MARK-MASQ
```

拆解：

```text
-A KUBE-SVC-HL5LMXD5JFHQZ6LN
往 KUBE-SVC-HL5LMXD5JFHQZ6LN 这条链里追加一条规则。

! -s 10.244.0.0/16
来源 IP 不是 10.244.0.0/16。

-d 10.98.59.52/32
目标 IP 是 nginx-svc 的 ClusterIP。

-p tcp
协议是 TCP。

-j KUBE-MARK-MASQ
匹配后跳到 KUBE-MARK-MASQ 链，给这个包打一个后续需要 SNAT/MASQUERADE 的标记。
```

重点：

```text
-s 表示 source，即来源地址。
-d 表示 destination，即目标地址。
! 表示取反。
-j 表示 jump，即匹配后跳到某个链或执行某个动作。
```

所以：

```text
-s 10.244.0.0/16
来源是 Pod 网段。

! -s 10.244.0.0/16
来源不是 Pod 网段。
```

---

## 6. 什么叫"不是 Pod 网段里的流量访问 Service"

Pod 访问 Service：

```text
源 IP: 10.244.169.177
目标 IP: 10.98.59.52
```

源 IP 属于 Pod 网段：

```text
10.244.0.0/16
```

因此这条规则不匹配：

```text
! -s 10.244.0.0/16
```

Node 宿主机访问 Service：

```bash
curl http://10.98.59.52
```

请求可能是：

```text
源 IP: 192.168.59.136
目标 IP: 10.98.59.52
```

源 IP `192.168.59.136` 不属于 `10.244.0.0/16`，所以它就是：

```text
不是 Pod 网段里的流量来访问这个 Service。
```

这时规则匹配，会打 MASQ 标记，后续做 SNAT/MASQUERADE。

---

## 7. DNAT、SNAT、MASQUERADE、conntrack

### DNAT

DNAT 是改目标地址。

Service 转发时常见：

```text
原目标: 10.98.59.52:80
改成:   10.244.169.177:80
```

也就是：

```text
Service IP -> Pod IP
```

### SNAT / MASQUERADE

SNAT 是改源地址。

MASQUERADE 是 SNAT 的一种常见形式，通常用于源地址可能动态变化的场景。

Node 或集群外来源访问 Service 时，kube-proxy 可能会做 SNAT/MASQUERADE，让回包路径稳定。

### KUBE-MARK-MASQ

`KUBE-MARK-MASQ` 不是立刻改源地址，而是：

```text
先给包打标记，告诉后面的规则：这个包之后需要做 SNAT/MASQUERADE。
```

### conntrack

conntrack 负责记录连接和 NAT 映射。

例如它记住：

```text
客户端访问 NodeIP:31233
被 DNAT 到 10.244.169.177:80
同时做了 SNAT/MASQUERADE
```

回包回来时，conntrack 根据记录把响应包还原成客户端期望看到的连接。

---

## 8. 集群内请求生命周期：Pod 访问 nginx-svc

假设源 Pod：

```text
nginx-demo-db78b9d74-hnkd2
IP: 10.244.169.177
Node: k8s-node2
```

它执行：

```bash
wget http://nginx-svc.default.svc
```

### 8.1 应用发起请求

应用请求：

```text
http://nginx-svc.default.svc:80
```

此时只有域名，没有目标 IP。

### 8.2 Pod 读取 DNS 配置

Pod 的 `/etc/resolv.conf` 类似：

```text
search default.svc.cluster.local svc.cluster.local cluster.local localdomain
nameserver 10.96.0.10
options ndots:5
```

`10.96.0.10` 是 kube-dns/CoreDNS Service 的 ClusterIP。

### 8.3 DNS 查询发给 CoreDNS Service

Pod 向：

```text
10.96.0.10:53
```

发送 DNS 查询。

`10.96.0.10` 也是 Service IP，不是网卡上的真实 IP，所以也会先经过 kube-proxy 规则。

流程：

```text
Pod -> 10.96.0.10:53
  -> node2 上 iptables KUBE-SERVICES
  -> kube-dns Service 链
  -> DNAT 到某个 CoreDNS Pod
```

CoreDNS Pod 在 master：

```text
10.244.235.241
10.244.235.243
```

DNS 请求可能被 DNAT 到：

```text
10.244.235.241:53
```

### 8.4 Calico 跨节点发送到 CoreDNS

源 Pod 在 node2，CoreDNS Pod 在 master。

node2 路由类似：

```text
10.244.235.192/26 via 192.168.59.134 dev tunl0
```

于是流量通过 Calico IPIP：

```text
node2 Pod
  -> node2 tunl0
  -> 封装到 Node IP 192.168.59.134
  -> master tunl0
  -> CoreDNS Pod 10.244.235.241
```

### 8.5 CoreDNS 返回 Service IP

CoreDNS 返回：

```text
nginx-svc.default.svc.cluster.local -> 10.98.59.52
```

现在应用得到目标 IP：

```text
10.98.59.52
```

### 8.6 应用连接 nginx-svc ClusterIP

`wget` 发起 TCP 连接：

```text
源: 10.244.169.177:随机端口
目标: 10.98.59.52:80
```

### 8.7 命中 kube-proxy Service 规则

node2 上 iptables 命中：

```text
KUBE-SERVICES
  -d 10.98.59.52/32 -p tcp
  -> KUBE-SVC-HL5LMXD5JFHQZ6LN
```

### 8.8 选择 Endpoint 并做 DNAT

Service 有 3 个 endpoints：

```text
10.244.169.177:80
10.244.169.179:80
10.244.235.242:80
```

iptables 按概率选择一个 endpoint。

假设选中：

```text
10.244.235.242:80
```

iptables 做 DNAT：

```text
原目标: 10.98.59.52:80
改成:   10.244.235.242:80
```

### 8.9 Calico 路由到后端 Pod

目标现在是：

```text
10.244.235.242
```

node2 查路由：

```text
10.244.235.192/26 via 192.168.59.134 dev tunl0
```

于是流量通过 Calico IPIP 到 master，再进入目标 Pod。

### 8.10 nginx 返回响应

后端 nginx Pod 收到请求：

```text
源: 10.244.169.177:随机端口
目标: 10.244.235.242:80
```

nginx 返回响应。

### 8.11 conntrack 还原连接

node2 conntrack 记得之前做过：

```text
10.98.59.52:80 -> 10.244.235.242:80
```

响应回到源 Pod 时，应用仍然认为自己访问的是：

```text
10.98.59.52:80
```

应用不会感知真实后端 Pod 是哪个。

### 8.12 集群内请求压缩版

```text
wget http://nginx-svc.default.svc
  -> DNS 查询 10.96.0.10
  -> kube-proxy 把 kube-dns Service 转到 CoreDNS Pod
  -> CoreDNS 返回 nginx-svc ClusterIP 10.98.59.52
  -> wget 连接 10.98.59.52:80
  -> node2 iptables 命中 KUBE-SERVICES
  -> 跳到 nginx-svc 的 KUBE-SVC 链
  -> 选择一个 KUBE-SEP endpoint
  -> DNAT 到真实 Pod IP:80
  -> Calico 路由/隧道送到目标 Pod
  -> nginx 返回响应
  -> conntrack 保证回包和连接还原
  -> wget 收到页面
```

---

## 9. 集群外请求生命周期：外部访问 NodePort

假设在 WSL2 或 Windows 宿主机上访问：

```bash
curl http://192.168.59.135:31233
```

这里访问的是 node1：

```text
Node IP: 192.168.59.135
NodePort: 31233
```

注意：node1 上没有 nginx Pod。

### 9.1 外部客户端发起请求

客户端发出 TCP 请求：

```text
源: WSL2/Windows IP:随机端口
目标: 192.168.59.135:31233
```

### 9.2 请求到达 node1

node1 Linux 网络栈收到包。

因为 kube-proxy 在每个 Node 上都写了 NodePort 规则，所以即使 node1 没有 nginx Pod，也能处理这个请求。

### 9.3 命中 NodePort 规则

包命中 kube-proxy 写的 iptables 规则：

```text
KUBE-NODEPORTS
  目标端口 31233
  -> nginx-svc 对应的 KUBE-SVC 链
```

NodePort 可以理解为：

```text
NodeIP:31233 是额外入口。
进入后还是走 nginx-svc 的 Service 转发逻辑。
```

### 9.4 外部来源通常需要 MASQUERADE/SNAT

这个请求的源 IP 不是 Pod 网段：

```text
源: WSL2/Windows IP
不是: 10.244.0.0/16
```

kube-proxy 会给它打 MASQ 标记：

```text
KUBE-MARK-MASQ
```

后续做 SNAT/MASQUERADE。

原因：后端 Pod 可能在另一个节点上。假设选中的 Endpoint 是：

```text
10.244.169.177:80
```

它在 node2。

如果不 SNAT，后端 Pod 看到的源 IP 是外部客户端 IP，回包可能绕开 node1，导致 conntrack 对不上。

SNAT 后，后端 Pod 看到的源 IP 通常变成 node1 地址：

```text
192.168.59.135
```

这样响应会先回 node1，再由 node1 做 NAT 还原后发给外部客户端。

### 9.5 选择 Endpoint 并做 DNAT

nginx-svc 后端：

```text
10.244.169.177:80
10.244.169.179:80
10.244.235.242:80
```

假设选中：

```text
10.244.169.177:80
```

iptables 做 DNAT：

```text
原目标: 192.168.59.135:31233
改成:   10.244.169.177:80
```

也就是：

```text
NodePort -> 后端 Pod IP
```

### 9.6 node1 把包发往 node2 上的 Pod

目标现在是：

```text
10.244.169.177
```

node1 查路由，发现这是 node2 的 Pod CIDR：

```text
10.244.169.128/26 via 192.168.59.136 dev tunl0
```

Calico 通过 IPIP 隧道把包发到 node2：

```text
node1
  -> tunl0
  -> 外层目标 Node IP: 192.168.59.136
  -> node2
  -> cali* 接口
  -> Pod 10.244.169.177
```

### 9.7 nginx Pod 收到请求

由于默认 `externalTrafficPolicy: Cluster` 通常会做 SNAT，后端 Pod 看到的可能是：

```text
源: 192.168.59.135:某端口
目标: 10.244.169.177:80
```

而不是原始 WSL2/Windows IP。

这就是默认 Cluster 策略下可能丢失真实客户端源 IP 的原因。

### 9.8 nginx 返回响应

nginx 返回：

```text
源: 10.244.169.177:80
目标: 192.168.59.135:某端口
```

响应先回到 node1。

### 9.9 node1 做 NAT 还原

node1 的 conntrack 记得前面做过：

```text
DNAT:
192.168.59.135:31233 -> 10.244.169.177:80

SNAT/MASQUERADE:
外部客户端 IP -> 192.168.59.135
```

响应回来时，node1 会还原成客户端期望看到的连接：

```text
源: 192.168.59.135:31233
目标: WSL2/Windows IP:随机端口
```

客户端认为自己一直在和：

```text
192.168.59.135:31233
```

通信。

### 9.10 集群外请求压缩版

```text
curl http://192.168.59.135:31233
  -> 包到 node1
  -> 命中 kube-proxy NodePort 规则
  -> 跳到 nginx-svc 的 KUBE-SVC 链
  -> 外部来源，打 MASQ 标记
  -> 选择一个 Endpoint
  -> DNAT 到 10.244.169.177:80
  -> Calico 把包从 node1 送到 node2 的 Pod
  -> nginx 处理请求
  -> 响应回 node1
  -> conntrack 做 NAT 还原
  -> 客户端收到响应
```

---

## 10. 集群内访问和集群外访问的区别

集群内 Pod 访问 Service：

```text
源 IP: 10.244.x.x
目标: 10.98.59.52:80
入口: ClusterIP
通常不需要对 Pod 网段流量做 MASQ。
```

集群外访问 NodePort：

```text
源 IP: 外部客户端 IP
目标: NodeIP:31233
入口: NodePort
通常需要 MASQ/SNAT。
```

默认 `externalTrafficPolicy: Cluster`：

```text
任意 NodeIP:NodePort 都能访问。
流量可以转发到任意节点上的 Endpoint。
通常会 SNAT，后端 Pod 可能看不到真实客户端 IP。
```

`externalTrafficPolicy: Local`：

```text
只有有本地 Endpoint 的 Node 才转发。
不会跨节点转发到其他 Node 的 Pod。
通常可以保留真实客户端源 IP。
访问没有本地 Endpoint 的 NodePort 可能失败。
```

---

## 11. 分工总结

```text
CoreDNS:
服务名 -> Service IP

kube-proxy:
Service IP / NodePort -> Endpoint Pod IP

iptables:
在内核网络栈里执行匹配、跳转、DNAT、打 MASQ 标记等规则。

Calico:
Pod IP -> Pod IP 的跨节点路由。

conntrack:
记住 NAT 映射，让回包能正确还原。

后端 Pod:
真正处理业务请求。
```

---

## 12. conntrack 简介

`conntrack` 可以理解成 Linux 内核里的"连接记账本"。

它负责记住：

```text
谁和谁建立了连接
这个连接现在是什么状态
这个连接有没有做过 NAT
回包应该怎么还原
```

### 12.1 为什么需要 conntrack

比如外部客户端访问 NodePort：

```text
客户端 -> 192.168.59.135:31233
```

kube-proxy 的 iptables 规则可能把目标改成：

```text
10.244.169.177:80
```

这个动作叫 DNAT。

问题是后端 Pod 回包时，源地址会是：

```text
10.244.169.177:80
```

但客户端原本访问的是：

```text
192.168.59.135:31233
```

如果没有人记住"这两个其实是同一条连接"，客户端会觉得回来的包不对。

conntrack 会记住映射：

```text
原始连接:
客户端IP:随机端口 -> 192.168.59.135:31233

NAT 后连接:
客户端IP:随机端口 -> 10.244.169.177:80
```

回包时，conntrack 帮忙把响应还原成客户端期待的样子：

```text
源: 192.168.59.135:31233
目标: 客户端IP:随机端口
```

所以客户端不会感知后端真实 Pod IP。

### 12.2 conntrack 记录什么

一条 TCP 连接通常可以用五元组描述：

```text
源 IP
源端口
目标 IP
目标端口
协议
```

例如：

```text
10.244.169.177:45678 -> 10.98.59.52:80 TCP
```

如果发生 NAT，conntrack 还会记录 NAT 前后的对应关系。

### 12.3 K8s 里 conntrack 为什么重要

Service 转发：

```text
ClusterIP / NodePort 通过 iptables DNAT 到 Pod，回包需要 conntrack 还原。
```

SNAT / MASQUERADE：

```text
外部流量进来或 Pod 出集群时，源地址可能被改写，也需要 conntrack 记住。
```

Service 后端变更：

```text
Endpoint 变化后，旧连接可能仍然按旧 conntrack 记录走一段时间。
```

DNS / 短连接多：

```text
大量短连接会制造很多 conntrack entry，表满会导致新连接异常。
```

### 12.4 conntrack 出问题的表现

如果 conntrack 表满，可能出现：

```text
连接超时
Service 间歇性不通
DNS 查询失败
新连接建立失败
节点网络抖动
```

内核日志可能看到：

```text
nf_conntrack: table full, dropping packet
```

### 12.5 常用观察命令

看当前 conntrack 数量：

```bash
sudo sysctl net.netfilter.nf_conntrack_count
```

看最大容量：

```bash
sudo sysctl net.netfilter.nf_conntrack_max
```

判断是否接近上限：

```text
nf_conntrack_count / nf_conntrack_max
```

如果比例接近 80%、90%，就要关注。

如果安装了 `conntrack` 工具，可以看具体连接：

```bash
sudo conntrack -L | head
sudo conntrack -S
```

看某个 Service IP 相关连接：

```bash
sudo conntrack -L | grep 10.98.59.52
```

很多节点默认没装 `conntrack` 命令，但内核 conntrack 功能通常是有的。

### 12.6 conntrack 和 iptables 的关系

iptables 负责规则：

```text
这个包要不要 DNAT？
要不要打 MASQ 标记？
要不要 SNAT？
```

conntrack 负责状态：

```text
这是不是一条已有连接？
这条连接之前做过什么 NAT？
回包该怎么还原？
```

可以这样记：

```text
iptables 决定怎么改包。
conntrack 记住改过什么。
```

一句话总结：

```text
conntrack 是 Linux 内核的连接状态表。K8s Service 的 DNAT/SNAT 能正常工作，很大程度依赖它记住连接和 NAT 映射，让回包能正确还原。
```

---

## 13. 必须记牢的短句

```text
ClusterIP 不是网卡 IP，而是 kube-proxy 规则实现的虚拟入口。
NodePort 默认在每个 Node 上暴露同一个端口。
kube-proxy watch Service 和 EndpointSlice，然后维护 iptables/IPVS 规则。
iptables 模式下，真正转发发生在 Linux 内核里，不是 kube-proxy 进程逐包代理。
DNAT 改目标地址：Service IP -> Pod IP。
SNAT/MASQUERADE 改源地址：让回包路径稳定。
KUBE-MARK-MASQ 是先打标记，后续统一做 MASQUERADE。
iptables 决定怎么改包，conntrack 记住改过什么。
externalTrafficPolicy=Cluster 时，NodePort 可跨节点转发，但可能丢失真实客户端源 IP。
externalTrafficPolicy=Local 时，通常保留源 IP，但只有有本地 Endpoint 的 Node 才转发。
```

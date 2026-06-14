---
title: "K8s 节点排障（四）：Linux 网络、DNS 与 Service 排障"
date: 2026-06-14T9:00:00+08:00
draft: false
tags: ["Kubernetes", "Linux", "排障", "网络", "DNS", "Service", "Calico", "iptables"]
categories: ["云原生", "Linux 排障"]
series: ["K8s 节点排障实战"]
summary: "系统学习 Kubernetes 网络分层排障方法，涵盖 Node IP、Pod IP、Service IP 的性质区分，Calico 路由与隧道，DNS 解析链路，以及 iptables 转发规则观察"
weight: 4
---

## 0. 本课目标

本课目标：

```text
看到 Pod 不通、Service 不通、端口不通、DNS 不通时，
能先从 Linux 网络层分流：
IP 有没有、路由对不对、端口有没有监听、连接卡在哪一段。
```

当前集群特征：

```text
Node IP 网段：192.168.59.0/24
Pod IP 网段：10.244.0.0/16
Service IP 示例：
  kubernetes: 10.96.0.1
  kube-dns: 10.96.0.10
  nginx-svc: 10.98.59.52

网络插件：Calico
Service 实现：kube-proxy iptables 模式
Calico 跨节点模式：BGP + IPIP tunnel
```

---

## 1. Node IP、Pod IP、Service IP

三类 IP 的性质不同。

### Node IP

Node IP 是 Kubernetes 节点的真实主机 IP，绑定在节点网卡上。

实验中 node2：

```bash
ip -br addr
```

输出：

```text
ens33 UP 192.168.59.136/24
```

说明：

```text
k8s-node2 的 Node IP 是 192.168.59.136
它绑定在 ens33 网卡上
```

### Pod IP

Pod IP 是 Pod 网络中的 IP，由 CNI 插件管理。

实验中 nginx Pod：

```text
nginx-demo-db78b9d74-hnkd2  10.244.169.177  k8s-node2
nginx-demo-db78b9d74-nvcb5  10.244.169.179  k8s-node2
nginx-demo-db78b9d74-zzq8k  10.244.235.242  k8s-master
```

Calico 会在宿主机上创建 `cali*` veth host 端接口，把本节点普通 Pod 接入网络。

### Service IP

Service IP，也就是 ClusterIP，是 Kubernetes Service 的虚拟 IP。

例子：

```text
nginx-svc  ClusterIP  10.98.59.52
kubernetes ClusterIP  10.96.0.1
kube-dns   ClusterIP  10.96.0.10
```

重要修正：

```text
ClusterIP 主要用于集群内部访问，不是专门用于外部访问。
NodePort / LoadBalancer / Ingress 才更偏向外部访问入口。
```

ClusterIP 通常不会绑定到某张普通网卡上，而是由 kube-proxy 通过 iptables 或 IPVS 实现转发。

所以：

```bash
ip addr | grep 10.98.59.52
```

查不到是正常的。

---

## 2. node2 网卡观察

命令：

```bash
ip -br addr
```

node2 输出：

```text
lo               UNKNOWN  127.0.0.1/8 ::1/128
ens33            UP       192.168.59.136/24
cali0b629a1a617@if2 UP    fe80::ecee:eeff:feee:eeee/64
calif9779232691@if2 UP    fe80::ecee:eeff:feee:eeee/64
tunl0@NONE       UNKNOWN  10.244.169.128/32
```

解释：

```text
lo:
本机回环接口。

ens33:
节点主网卡，承载 Node IP 192.168.59.136。

cali*:
Calico 为本节点普通 Pod 创建的 veth host 端接口。

tunl0:
Calico IPIP tunnel 接口，用于跨节点 Pod 流量封装。
```

node2 上有两个 `cali*` 接口，对应 node2 上两个普通 nginx Pod。

---

## 3. node1 为什么没有 cali* 接口

node1 网卡：

```text
lo      UNKNOWN 127.0.0.1/8
ens33   UP      192.168.59.135/24
tunl0   UNKNOWN 10.244.36.64/32
```

没有 `cali*`。

查询 node1 上的 Pod：

```bash
kubectl get pod -A -o wide --field-selector spec.nodeName=k8s-node1
```

输出：

```text
gpu-platform/fake-gpu-device-plugin-wtm8w  IP 192.168.59.135
kube-system/calico-node-2qbs9              IP 192.168.59.135
kube-system/kube-proxy-hhc2j               IP 192.168.59.135
```

继续看 hostNetwork：

```bash
kubectl get pod -A -o jsonpath='{range .items[?(@.spec.nodeName=="k8s-node1")]}{.metadata.namespace}/{.metadata.name}{" hostNetwork="}{.spec.hostNetwork}{" podIP="}{.status.podIP}{"\n"}{end}'
```

输出：

```text
gpu-platform/fake-gpu-device-plugin-wtm8w hostNetwork=true podIP=192.168.59.135
kube-system/calico-node-2qbs9 hostNetwork=true podIP=192.168.59.135
kube-system/kube-proxy-hhc2j hostNetwork=true podIP=192.168.59.135
```

结论：

```text
node1 当前没有普通 Pod。
node1 上的 Pod 都是 hostNetwork=true。
hostNetwork Pod 直接使用宿主机网络 namespace，不会创建普通 Pod veth。
所以 node1 没有 cali* 接口。
```

node1 仍然有 `tunl0`，说明 Calico 仍为它分配了 Pod CIDR，并维护跨节点路由。

---

## 4. Calico 路由观察

node2 路由：

```bash
ip route
```

输出：

```text
default via 192.168.59.2 dev ens33 proto dhcp src 192.168.59.136 metric 100
10.244.36.64/26 via 192.168.59.135 dev tunl0 proto bird onlink
blackhole 10.244.169.128/26 proto bird
10.244.169.177 dev cali0b629a1a617 scope link
10.244.169.179 dev calif9779232691 scope link
10.244.235.192/26 via 192.168.59.134 dev tunl0 proto bird onlink
192.168.59.0/24 dev ens33 proto kernel scope link src 192.168.59.136 metric 100
```

节点网段：

```text
192.168.59.0/24 dev ens33
```

默认路由：

```text
default via 192.168.59.2 dev ens33
```

跨节点 Pod CIDR：

```text
10.244.36.64/26 via 192.168.59.135 dev tunl0
10.244.235.192/26 via 192.168.59.134 dev tunl0
```

本节点 Pod 路由：

```text
10.244.169.177 dev cali0b629a1a617
10.244.169.179 dev calif9779232691
```

`proto bird` 表示这些路由由 Calico 的 BIRD 组件写入内核路由表。

---

## 5. tunl0 的作用

`tunl0` 是 Calico IPIP tunnel 接口。

在当前集群里，它主要用于跨节点 Pod 流量。

例子：

```bash
ip route get 10.244.235.242
```

输出：

```text
10.244.235.242 via 192.168.59.134 dev tunl0 src 10.244.169.128
```

说明：

```text
目标 Pod IP 10.244.235.242 在 master 上。
node2 要通过 tunl0，把 Pod 流量封装到 IPIP 隧道中。
外层包发往对端 Node IP 192.168.59.134。
```

对比本节点 Pod：

```bash
ip route get 10.244.169.177
```

输出：

```text
10.244.169.177 dev cali0b629a1a617 src 192.168.59.136
```

总结：

```text
本节点 Pod IP：dev cali*
跨节点 Pod IP：via 对端 Node IP dev tunl0
```

---

## 6. blackhole 路由

node2 路由中有：

```text
blackhole 10.244.169.128/26 proto bird
```

这表示 node2 自己负责的 Pod CIDR 是：

```text
10.244.169.128/26
```

作用：

```text
如果目标 IP 属于本节点 Pod CIDR，
但没有更具体的 /32 本地 Pod 路由，
就直接丢弃。
```

Linux 路由按最长前缀匹配：

```text
访问 10.244.169.177：
匹配更具体的 /32 路由 -> 走 cali0b629a1a617。

访问 10.244.169.180：
如果没有对应 /32 Pod 路由，只匹配 /26 blackhole -> 丢弃。
```

修正一个说法：

```text
blackhole 不是为了防止"网段外"的访问。
它主要处理"本节点 Pod CIDR 内，但并不存在的 Pod IP"的流量，避免这类流量被错误转发。
```

---

## 7. 端口监听观察

命令：

```bash
sudo ss -lntup | head -40
```

node2 输出节选：

```text
127.0.0.1:10248 kubelet
127.0.0.1:10249 kube-proxy
127.0.0.1:9099  calico-node
0.0.0.0:22      sshd
0.0.0.0:179     bird
*:10250         kubelet
*:10256         kube-proxy
127.0.0.53:53   systemd-resolve
```

解释：

```text
10248:
kubelet healthz，通常只监听本地。

10250:
kubelet HTTPS API。apiserver 可能访问它做 logs/exec/metrics 等。

10249:
kube-proxy metrics，通常本地监听。

10256:
kube-proxy healthz。

179:
BGP 端口，Calico BIRD 使用。

9099:
calico-node 本地健康/状态相关端口。

127.0.0.53:53:
systemd-resolved 的本地 DNS stub，不是 CoreDNS。
```

---

## 8. TCP 连接状态观察

命令：

```bash
ss -ant | awk 'NR==1 || /ESTAB|SYN-SENT|SYN-RECV|TIME-WAIT|CLOSE-WAIT/' | head -40
```

输出中看到：

```text
192.168.59.136:50192 -> 192.168.59.134:6443 ESTAB
192.168.59.136:38520 -> 192.168.59.134:6443 ESTAB
```

说明 node2 上组件正在连接 master 上的 apiserver。

还看到：

```text
192.168.59.136:52363 -> 192.168.59.134:179 ESTAB
192.168.59.136:44855 -> 192.168.59.135:179 ESTAB
```

说明 Calico BIRD 与其他节点建立了 BGP session。

还看到：

```text
192.168.59.136:60836 -> 10.96.0.1:443 ESTAB
```

`10.96.0.1:443` 是 `kubernetes` Service 的 ClusterIP。

这说明节点上的某些组件访问的是 Service IP，底层由 kube-proxy 转发到真实 apiserver endpoint。

TCP 状态简表：

```text
LISTEN:
服务正在监听。

SYN-SENT:
本机发起连接，等待对端响应。常见于目标不可达、防火墙丢包、路由问题。

SYN-RECV:
本机收到 SYN，已回 SYN+ACK，等待对方 ACK。

ESTAB:
连接已建立。

TIME-WAIT:
主动关闭连接后等待一段时间，正常现象。特别多时要结合短连接压力看。

CLOSE-WAIT:
对方关闭连接，本机应用还没 close。大量 CLOSE-WAIT 常见于应用没有正确关闭 socket。
```

---

## 9. Pod 到 Pod 连通性

源 Pod：

```text
nginx-demo-db78b9d74-hnkd2
IP: 10.244.169.177
Node: k8s-node2
```

目标 Pod：

```text
nginx-demo-db78b9d74-zzq8k
IP: 10.244.235.242
Node: k8s-master
```

命令：

```bash
kubectl exec -it nginx-demo-db78b9d74-hnkd2 -- wget -qO- --timeout=2 http://10.244.235.242
```

结果返回 nginx welcome page。

结论：

```text
Pod -> 跨节点 Pod 网络正常。
node2 -> tunl0/IPIP -> master -> cali* -> 目标 Pod 这条链路正常。
```

---

## 10. Pod DNS 配置

命令：

```bash
kubectl exec -it nginx-demo-db78b9d74-hnkd2 -- cat /etc/resolv.conf
```

输出：

```text
search default.svc.cluster.local svc.cluster.local cluster.local localdomain
nameserver 10.96.0.10
options ndots:5
```

解释：

```text
nameserver 10.96.0.10:
Pod DNS 请求发给 kube-dns/CoreDNS Service。

search default.svc.cluster.local svc.cluster.local cluster.local localdomain:
查询非绝对域名时，会尝试拼接这些搜索后缀。

ndots:5:
如果查询名里的点数少于 5 个，resolver 会优先尝试 search domain 补全。
```

例如 `kubernetes.default.svc` 点数少于 5，resolver 可能尝试：

```text
kubernetes.default.svc.default.svc.cluster.local
kubernetes.default.svc.svc.cluster.local
kubernetes.default.svc.cluster.local
kubernetes.default.svc.localdomain
kubernetes.default.svc
```

真正存在的是：

```text
kubernetes.default.svc.cluster.local
```

---

## 11. Kubernetes Service DNS 现象

命令：

```bash
kubectl exec -it nginx-demo-db78b9d74-hnkd2 -- getent hosts kubernetes.default.svc
```

输出：

```text
10.96.0.1 kubernetes.default.svc.cluster.local ... kubernetes.default.svc
```

说明系统 resolver 通过 search domain 补全成功。

命令：

```bash
kubectl exec -it nginx-demo-db78b9d74-hnkd2 -- nslookup kubernetes.default.svc.cluster.local
```

输出：

```text
Name: kubernetes.default.svc.cluster.local
Address: 10.96.0.1
```

说明 CoreDNS 对完整 FQDN 工作正常。

命令：

```bash
kubectl exec -it nginx-demo-db78b9d74-hnkd2 -- nslookup kubernetes.default.svc.
```

输出：

```text
NXDOMAIN
```

原因：

```text
末尾加点表示绝对域名，不再拼 search domain。
Kubernetes Service 的完整域名是 kubernetes.default.svc.cluster.local。
kubernetes.default.svc. 这个绝对域名本身不存在，所以 NXDOMAIN 正常。
```

访问：

```bash
kubectl exec -it nginx-demo-db78b9d74-hnkd2 -- wget -qO- --timeout=2 https://kubernetes.default.svc
```

输出：

```text
certificate verify failed
Connection reset by peer
```

这不是网络不通，而是说明：

```text
DNS 解析成功。
Service 10.96.0.1:443 转发成功。
TCP/TLS 已开始握手。
失败点在证书校验。
```

---

## 12. nginx-svc DNS、Service 与 Endpoint

Service：

```bash
kubectl get svc
```

输出：

```text
nginx-svc NodePort 10.98.59.52 <none> 80:31233/TCP
```

DNS 解析：

```bash
kubectl exec -it nginx-demo-db78b9d74-hnkd2 -- nslookup nginx-svc.default.svc.cluster.local
```

输出：

```text
Name: nginx-svc.default.svc.cluster.local
Address: 10.98.59.52
```

访问 Service IP：

```bash
kubectl exec -it nginx-demo-db78b9d74-hnkd2 -- wget -qO- --timeout=2 http://10.98.59.52
```

返回 nginx welcome page。

访问短名：

```bash
kubectl exec -it nginx-demo-db78b9d74-hnkd2 -- wget -qO- --timeout=2 http://nginx-svc
kubectl exec -it nginx-demo-db78b9d74-hnkd2 -- wget -qO- --timeout=2 http://nginx-svc.default.svc
```

均成功。

Endpoint：

```bash
kubectl get endpoints nginx-svc -o wide
```

输出：

```text
10.244.169.177:80,10.244.169.179:80,10.244.235.242:80
```

EndpointSlice：

```bash
kubectl get endpointslice -l kubernetes.io/service-name=nginx-svc -o wide
```

输出：

```text
10.244.169.177,10.244.169.179,10.244.235.242
```

结论：

```text
CoreDNS 正常。
nginx-svc DNS 记录正常。
Service IP 正常。
Endpoint 正常。
Pod -> Service IP -> 后端 nginx Pod 这条链路正常。
```

---

## 13. nslookup 成功但 wget/getent 异常时如何判断

实验中曾出现：

```text
nslookup nginx-svc.default.svc.cluster.local 成功
wget http://nginx-svc.default.svc.cluster.local 失败 bad address
getent hosts nginx-svc.default.svc.cluster.local 失败
getent hosts nginx-svc 成功
wget http://nginx-svc 成功
wget http://nginx-svc.default.svc 成功
```

这说明不能只靠一个工具判断 DNS 故障。

原因可能包括：

```text
镜像中的 resolver/libc/NSS 行为差异
nslookup 与应用实际 resolver 路径不同
search domain / ndots 影响
IPv4 / IPv6 查询差异
工具自身实现限制
证书问题或应用层问题
```

排障时要综合判断：

```text
nslookup FQDN 是否成功
getent 短名是否成功
wget/curl Service IP 是否成功
wget/curl Service DNS 是否成功
Service endpoints 是否存在
CoreDNS Pod 是否 Running
```

本次实验中：

```text
nslookup FQDN 成功
wget Service IP 成功
wget 短名成功
endpoints 正常
CoreDNS Running
```

足以判断 K8s DNS/Service 主链路正常。

---

## 14. CoreDNS 组件观察

查看 kube-dns Service：

```bash
kubectl get svc -n kube-system kube-dns -o wide
```

输出：

```text
kube-dns ClusterIP 10.96.0.10 53/UDP,53/TCP,9153/TCP
```

查看 CoreDNS Pod：

```bash
kubectl get pod -n kube-system -l k8s-app=kube-dns -o wide
```

输出：

```text
coredns-855c4dd65d-7jz6h  Running  10.244.235.241  k8s-master
coredns-855c4dd65d-sspfp  Running  10.244.235.243  k8s-master
```

说明：

```text
Pod 的 /etc/resolv.conf nameserver 是 10.96.0.10。
10.96.0.10 是 kube-dns Service IP。
kube-dns Service 后端是 CoreDNS Pod。
```

---

## 15. kube-proxy 模式判断

查看 kube-proxy ConfigMap：

```bash
kubectl get cm -n kube-system kube-proxy -o yaml | grep -i mode
```

输出：

```text
detectLocalMode: ""
mode: ""
```

`mode: ""` 表示没有显式指定，kube-proxy 使用默认模式。

节点上验证：

```bash
sudo iptables-save | grep 10.98.59.52 | head -20
sudo ipvsadm -Ln 2>/dev/null | head -40
```

结果：

```text
iptables-save 能看到 KUBE-SERVICES / KUBE-SVC 规则。
ipvsadm 没有输出。
```

结论：

```text
当前 kube-proxy 实际使用 iptables 模式。
```

---

## 16. 为什么 ip addr 查不到 ClusterIP

命令：

```bash
ip addr | grep -E '10.98.59.52|10.96.0.1'
```

无输出。

这是正常的。

原因：

```text
ClusterIP 不是绑定在某张网卡上的真实 IP。
它是 kube-proxy 写入 iptables 的匹配目标。
内核转发路径中命中 KUBE-SERVICES 规则后，会被 DNAT 到真实 Endpoint Pod IP。
```

所以：

```text
ip addr 查不到 10.98.59.52
但 Pod 仍然可以访问 10.98.59.52
```

并不矛盾。

---

## 17. nginx-svc 的 iptables 转发链

查询：

```bash
sudo iptables-save | grep KUBE-SVC-HL5LMXD5JFHQZ6LN
```

Service 链入口：

```text
-A KUBE-SERVICES -d 10.98.59.52/32 -p tcp \
  --comment "default/nginx-svc cluster IP" \
  -j KUBE-SVC-HL5LMXD5JFHQZ6LN
```

含义：

```text
访问 10.98.59.52:80 的 TCP 流量，
跳到 nginx-svc 对应的 KUBE-SVC-* 链。
```

Service 选择 Endpoint：

```text
-A KUBE-SVC-HL5LMXD5JFHQZ6LN \
  --comment "default/nginx-svc -> 10.244.169.177:80" \
  -m statistic --mode random --probability 0.33333333349 \
  -j KUBE-SEP-EJMK4UCYIRT4LIJQ

-A KUBE-SVC-HL5LMXD5JFHQZ6LN \
  --comment "default/nginx-svc -> 10.244.169.179:80" \
  -m statistic --mode random --probability 0.50000000000 \
  -j KUBE-SEP-762TSIBB364NDZNY

-A KUBE-SVC-HL5LMXD5JFHQZ6LN \
  --comment "default/nginx-svc -> 10.244.235.242:80" \
  -j KUBE-SEP-L7O4Y5XZIFZPR2LC
```

为什么概率看起来不是全都 1/3：

```text
iptables 按顺序匹配。

第一条：整体 1/3 概率命中。
第二条：在剩余 2/3 流量里，用 1/2 概率命中，整体仍是 1/3。
第三条：剩下的 1/3 兜底命中。
```

所以三个 endpoint 整体大约均分。

Endpoint 链：

```text
-A KUBE-SEP-EJMK4UCYIRT4LIJQ -p tcp -j DNAT --to-destination 10.244.169.177:80
-A KUBE-SEP-762TSIBB364NDZNY -p tcp -j DNAT --to-destination 10.244.169.179:80
-A KUBE-SEP-L7O4Y5XZIFZPR2LC -p tcp -j DNAT --to-destination 10.244.235.242:80
```

核心动作：

```text
DNAT
```

即把目标地址从：

```text
10.98.59.52:80
```

改写为某个真实后端：

```text
10.244.169.177:80
10.244.169.179:80
10.244.235.242:80
```

补充：

```text
SNAT/MASQUERADE 在某些来源、回包路径、NodePort/externalTrafficPolicy 等场景中也会出现。
但 ClusterIP 转发到 Endpoint 的关键动作是 DNAT。
```

---

## 18. KUBE-SVC 与 KUBE-SEP

`KUBE-SVC-*`：

```text
Service 级别链。
负责接收访问某个 Service ClusterIP/Port 的流量，
并按规则选择某个后端 Endpoint。
```

`KUBE-SEP-*`：

```text
Service Endpoint 级别链。
通常对应某个具体 Endpoint PodIP:Port。
负责执行 DNAT 到真实后端。
```

Service 转发链：

```text
访问 ClusterIP:Port
  -> KUBE-SERVICES
  -> KUBE-SVC-*
  -> KUBE-SEP-*
  -> DNAT 到 Endpoint PodIP:Port
```

---

## 19. NodePort 规则

Service：

```text
nginx-svc NodePort 80:31233/TCP
```

iptables 中看到：

```text
-A KUBE-NODEPORTS -p tcp --comment "default/nginx-svc" -j KUBE-EXT-HL5LMXD5JFHQZ6LN
-A KUBE-EXT-HL5LMXD5JFHQZ6LN -j KUBE-SVC-HL5LMXD5JFHQZ6LN
```

NodePort 流量大致路径：

```text
访问任意 NodeIP:31233
  -> KUBE-NODEPORTS
  -> KUBE-EXT-*
  -> KUBE-SVC-*
  -> KUBE-SEP-*
  -> DNAT 到 Endpoint PodIP:80
```

因此，即使某个节点上没有 nginx Pod，访问该节点的 NodePort 也可以转发到其他节点上的后端 Pod。

---

## 20. K8s 网络排障分层

### 20.1 Pod 是否有 IP

```bash
kubectl get pod -A -o wide
```

看：

```text
STATUS
IP
NODE
```

### 20.2 节点网卡和路由

```bash
ip -br addr
ip route
ip route get <target-ip>
```

看：

```text
Node IP 在哪张网卡
是否有 cali*
是否有 tunl0
本节点 Pod IP 是否走 cali*
跨节点 Pod IP 是否走 tunl0 / 对端 Node IP
```

### 20.3 端口是否监听

```bash
sudo ss -lntup
sudo ss -lntup | grep <port>
```

### 20.4 TCP 连接状态

```bash
ss -ant
ss -ant state established
ss -ant state time-wait
```

关注：

```text
SYN-SENT
SYN-RECV
ESTAB
TIME-WAIT
CLOSE-WAIT
```

### 20.5 DNS

```bash
cat /etc/resolv.conf
nslookup <service>.<namespace>.svc.cluster.local
getent hosts <service>
```

### 20.6 Service 与 Endpoint

```bash
kubectl get svc
kubectl get endpoints <svc> -o wide
kubectl get endpointslice -l kubernetes.io/service-name=<svc> -o wide
```

### 20.7 kube-proxy 规则

iptables 模式：

```bash
sudo iptables-save | grep <service-ip>
sudo iptables-save | grep <endpoint-ip>
```

IPVS 模式：

```bash
sudo ipvsadm -Ln
```

---

## 21. 本课小测点评

问题 1：Node IP、Pod IP、Service IP 分别是什么性质？

你的回答：

```text
Node IP 为节点 ip，ens33 这种地址。
pod ip 为 pod 在集群中的 ip。通过网络插件如 calico 设置如 caliXXX 的网卡。
service ip 为服务 ip，做外部访问用，没有实际网卡，通过 iptables 等规则来进行控制转发。
```

点评：

```text
Node IP、Pod IP 部分正确。
Service IP 部分需要修正：ClusterIP 主要用于集群内部访问，不是外部访问用。
它通常不绑定真实网卡，由 kube-proxy 的 iptables/IPVS 规则实现转发。
```

问题 2：为什么 node1 没有 `cali*` 接口？

你的回答：

```text
因为此时 node1 上没有普通 pod，pod 的 hostNetwork=true。
```

点评：正确。

问题 3：`tunl0` 在你的 Calico 集群里主要用于什么？

你的回答：

```text
tunl0 主要用于跨节点 pod 隧道访问。
```

点评：正确。

补充：当前集群中是 Calico IPIP tunnel，用于跨节点 Pod 流量封装。

问题 4：`blackhole 10.244.169.128/26` 的作用是什么？

你的回答：

```text
防止网段外的无效访问。
```

点评：方向接近，但需要更准确。

修正：

```text
blackhole 用于丢弃"属于本节点 Pod CIDR，但没有具体 /32 Pod 路由"的目标。
它不是处理网段外访问，而是处理本节点 Pod CIDR 内不存在的 Pod IP。
```

问题 5：为什么 `ip addr` 查不到 `10.98.59.52`，但 Pod 仍然能访问这个 Service IP？

你的回答：

```text
10.98.59.52 为 service ip，做外部访问用，没有实际网卡，通过 iptables 等规则来进行控制转发。
```

点评：

```text
核心机制正确：不是实际网卡 IP，而是 iptables 规则实现。
但"外部访问用"需要改成"集群内部虚拟 IP"。
```

问题 6：kube-proxy iptables 模式下，`KUBE-SVC-*` 和 `KUBE-SEP-*` 大概分别代表什么？

你的回答：

```text
KUBE-SVC-* 代表某个 svc，KUBE-SEP-* 代表 svc 背后的实际端点。
```

点评：正确。

问题 7：Service 转发到某个后端 Pod 的关键 iptables 动作是什么？

你的回答：

```text
DNAT，SNAT
```

点评：

```text
关键动作是 DNAT。
SNAT/MASQUERADE 在某些场景会参与，但 Service ClusterIP 转发到 Endpoint 的核心是 DNAT。
```

问题 8：`nslookup FQDN 成功，但 wget 某个域名失败` 时，为什么不能立刻判定 CoreDNS 故障？

你的回答：

```text
可能是该 pod 镜像本身工具链的问题。
```

点评：正确。

补充：

```text
还可能涉及 search domain、ndots、libc/NSS、IPv4/IPv6、应用 resolver、证书校验、代理配置等。
```

---

## 22. 必须记牢的短句

```text
Node IP 是节点真实网卡 IP。
Pod IP 是 CNI 管理的集群内 Pod 网络 IP。
ClusterIP 是集群内部虚拟 IP，不一定出现在 ip addr 中。
Calico 本节点 Pod 通常走 cali*。
Calico 跨节点 Pod 在当前集群里走 tunl0/IPIP。
blackhole 路由用于丢弃本节点 Pod CIDR 内不存在的 Pod IP。
hostNetwork Pod 不会创建普通 Pod veth，因此不会产生 cali*。
CoreDNS Service IP 是 10.96.0.10，Pod resolv.conf 指向它。
DNS 排障不能只看一个工具，nslookup/getent/wget 可能走不同解析路径。
kube-proxy iptables 模式中，KUBE-SVC 是 Service 链，KUBE-SEP 是 Endpoint 链。
ClusterIP 转发到 Pod Endpoint 的核心动作是 DNAT。
NodePort 流量也会进入 KUBE-SVC/KUBE-SEP 这套链路。
```

---

## 23. 下一课预告：日志与系统事件

下一课进入阶段 1 的最后一块：

```text
journalctl
dmesg
kubelet 日志
containerd 日志
K8s event 与 Linux 系统事件对齐
```

目标：

```text
看到 Pod 重启、OOMKilled、节点 NotReady、containerd/kubelet 异常时，
能从 Kubernetes event 追到节点系统日志。
```

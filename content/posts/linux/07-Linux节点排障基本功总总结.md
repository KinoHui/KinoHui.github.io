---
title: "K8s 节点排障（七）：阶段 1 总总结"
date: 2026-06-14T11:00:00+08:00
draft: false
tags: ["Kubernetes", "Linux", "排障", "总结", "namespace", "cgroup", "网络", "日志"]
categories: ["云原生", "Linux 排障"]
series: ["K8s 节点排障实战"]
summary: "Linux 节点排障基本功阶段 1 总总结，涵盖进程与 /proc、CPU 与负载、内存与 OOMKilled、磁盘与 inode、网络与 Service、日志与事件等核心能力梳理"
weight: 7
---

## 1. 阶段目标

完成本阶段后，应该能做到：

```text
看到 Pod 异常，能找到它所在节点和宿主机进程。
看到 CPU 高、load 高，能区分 CPU 压力、IO 阻塞、cgroup throttling。
看到 OOMKilled，能从 K8s Event 追到内核 cgroup OOM 证据。
看到磁盘满，能区分块空间、inode、容器日志、containerd、journal。
看到 Service/DNS/Pod 网络不通，能按 DNS、Service、Endpoint、路由、iptables 分层排查。
看到 ImagePullBackOff，能从 Event 对齐 kubelet 日志并分类错误原因。
```

## 2. 已完成课程

```text
第 1 课：进程与 /proc
第 2 课：CPU 与负载排障
第 3 课：内存、page cache 与 OOMKilled
第 4 课：磁盘、inode、文件句柄与容器日志
第 5 课：Linux 网络、Calico、DNS 与 Service
第 6 课：日志与系统事件
```

## 3. 实验环境

```text
Windows 宿主机
WSL2 使用 kubectl 访问集群
三节点 Kubernetes 集群
Kubernetes v1.31.14
Ubuntu 22.04.5 LTS
Linux kernel 5.15.0-177-generic
containerd 1.7.23
Calico 网络插件
```

节点：

```text
k8s-master  192.168.59.134
k8s-node1   192.168.59.135
k8s-node2   192.168.59.136
```

## 4. 核心认知

### 容器与 Linux 进程

```text
容器不是虚拟机，本质是宿主机上的 Linux 进程。
容器内 PID 1 和宿主机 PID 是同一个进程在不同 PID namespace 中的不同视图。
```

定位链路：

```text
kubectl get pod -o wide
  -> 找 Node
ssh 到 Node
  -> crictl ps / inspect
  -> 找宿主机 PID
  -> /proc/<pid>
  -> /sys/fs/cgroup
```

### namespace 与 cgroup

```text
namespace 负责"看见什么"。
cgroup 负责"能用多少"。
```

namespace 示例：

```text
PID namespace: 容器里看到 PID 1
Network namespace: Pod 有自己的 IP、网卡、路由
Mount namespace: 容器看到自己的 rootfs
```

cgroup 示例：

```text
CPU limit
memory limit
pids limit
资源统计
OOM kill
```

### K8s QoS 与 cgroup 层级

典型路径：

```text
/kubepods.slice
  /kubepods-burstable.slice
    /kubepods-burstable-pod<uid>.slice
      /cri-containerd-<container-id>.scope
```

对应：

```text
K8s Pod 总入口
QoS 层级
Pod cgroup
Container cgroup
Linux 进程
```

## 5. CPU 与 load

核心概念：

```text
load average 不是 CPU 使用率。
load average 统计 R 状态和 D 状态任务数平均值。
```

判断：

```text
R 多：CPU 调度压力。
D 多：IO 或内核路径阻塞。
```

CPU limit：

```text
K8s CPU limit -> cgroup v2 cpu.max
cpu.max 20000 100000 = 200m = 0.2 core
```

throttling 证据：

```text
cpu.stat:
nr_periods
nr_throttled
throttled_usec
```

如果 `nr_throttled` 和 `throttled_usec` 持续增长，说明容器被 CPU limit 限流。

## 6. 内存与 OOMKilled

核心概念：

```text
free 少不等于内存不足。
available 更适合判断节点可用内存。
page cache 是 Linux 正常缓存机制。
```

进程视角：

```text
VmRSS: 单进程驻留物理内存。
RssAnon: 匿名页，常见于堆、栈、malloc。
RssFile: 文件映射页，例如动态库、可执行文件、mmap 文件。
```

cgroup 视角：

```text
memory.max: memory limit
memory.current: 当前 cgroup 内存使用
memory.events: oom / oom_kill 等事件
```

OOMKilled 链路：

```text
Pod resources.limits.memory
  -> kubelet/containerd/runc 创建 cgroup
  -> memory.max 写入限制值
  -> 进程分配内存，memory.current 增长
  -> 超过 memory.max
  -> 内核 memory cgroup OOM
  -> dmesg: constraint=CONSTRAINT_MEMCG
  -> kill 进程
  -> kubelet 上报 OOMKilled / Exit Code 137
```

## 7. 磁盘、inode、fd

磁盘两类满：

```text
df -h: 块空间满。
df -i: inode 满。
```

常见目录：

```text
/var/log/journal: systemd journal
/var/log/pods: 容器真实日志
/var/log/containers: 指向 /var/log/pods 的软链接
/var/lib/containerd: 镜像、快照、容器层数据
/var/lib/kubelet: Pod volume、plugin、kubelet 管理数据
```

文件句柄：

```text
Max open files 太小或 fd 泄漏会导致 too many open files。
```

删除大文件但空间不释放：

```text
文件目录项删除了，但仍有进程 fd 持有 inode。
空间要等 fd 关闭后才释放。
```

## 8. 网络、DNS、Service

当前集群网络：

```text
Node IP: 192.168.59.x
Pod IP: 10.244.x.x
Service IP: 10.96.x.x / 10.98.x.x
Calico: BGP + IPIP tunnel
```

node2 路由特征：

```text
本节点 Pod: dev cali*
跨节点 Pod: via 对端 Node IP dev tunl0
blackhole: 丢弃本节点 Pod CIDR 内不存在的 Pod IP
```

DNS：

```text
Pod /etc/resolv.conf nameserver 指向 kube-dns Service IP 10.96.0.10。
CoreDNS Pod 是 kube-dns Service 的后端。
```

Service：

```text
ClusterIP 是集群内部虚拟 IP，不一定绑定在网卡上。
kube-proxy iptables 模式通过 KUBE-SERVICES / KUBE-SVC / KUBE-SEP 实现转发。
```

iptables 链路：

```text
访问 ClusterIP:Port
  -> KUBE-SERVICES
  -> KUBE-SVC-*
  -> KUBE-SEP-*
  -> DNAT 到 Endpoint PodIP:Port
```

## 9. 日志与事件

三层日志：

```text
kubectl describe / events:
K8s 认为发生了什么。

journalctl -u kubelet:
kubelet 在节点上做了什么。

journalctl -u containerd:
containerd 创建、启动、停止容器时发生了什么。

dmesg / journalctl -k:
内核发生了什么。
```

ImagePullBackOff：

```text
ErrImagePull: 一次拉取失败。
ImagePullBackOff: 拉取失败后的退避重试状态。
```

镜像拉取错误分类：

```text
not found: 镜像名或 tag 不存在。
authentication required / unauthorized: imagePullSecrets 或仓库权限。
i/o timeout / TLS handshake timeout: 网络或仓库连通性。
x509 certificate signed by unknown authority: 证书问题。
no such host: DNS 问题。
connection refused: 仓库服务或代理问题。
```

## 10. 阶段总测试结果

**A. 概念题**

1. 容器里的进程和宿主机 Linux 进程是什么关系？为什么容器里看到 PID 1，宿主机上却是另一个 PID？
2. namespace 和 cgroup 的核心区别是什么？
3. `load average` 是 CPU 使用率吗？它统计的是什么？
4. `free -h` 里 `free` 很低，但 `available` 很高，通常说明什么？
5. CPU limit 和 memory limit 超限后的结果有什么不同？
6. ClusterIP 为什么 `ip addr` 查不到，但 Pod 仍然能访问？

**B. 命令题**

1. 已知 Pod 名叫 `app-xxx`，你想找到它运行在哪个节点、Pod IP 是多少，用什么命令？
2. 登录节点后，如何找出当前 CPU 占用最高的前 10 个进程？
3. 已知宿主机 PID 是 `12345`，如何查看它的状态、内存、cgroup、文件句柄数量？
4. 如何判断一个容器是否发生 CPU throttling？需要看哪些 cgroup v2 文件和字段？
5. 如何判断一个 Pod 的 OOMKilled 是 cgroup OOM？K8s 层和节点层分别看什么？
6. 如何判断节点磁盘是空间满还是 inode 满？
7. 如何看一个 Service 是否有后端 Endpoint？

**C. 场景排障题**

1. 线上某 Pod 状态是 `ImagePullBackOff`，你会按什么顺序排查？不同错误信息分别代表什么方向？
2. 某服务延迟升高，但节点 CPU 没满，Pod 也没重启。你怀疑 CPU limit 太低，怎么验证？
3. 某节点 `df -h` 显示空间很紧张，你如何定位到底是容器日志、containerd、kubelet、journal 还是别的目录占用？
4. 某应用报 `too many open files`，你如何从 Linux 侧确认？
5. 某 Pod 访问 `nginx-svc` 不通，你如何分层排查 DNS、Service、Endpoint、Pod 网络？
6. 某节点上没有 `cali*` 接口，这一定是 Calico 异常吗？为什么？

**D. 深一点的链路题**

1. 请描述一次 `Pod -> Service ClusterIP -> 后端 Pod` 在 kube-proxy iptables 模式下的大致转发链路。
2. 请描述一次 Pod memory limit 触发 OOMKilled 的完整链路，从 K8s resources 到 Linux cgroup，再到 kubelet 上报状态。
3. 请描述一次 kubelet 拉取不存在镜像 tag 失败时，从 kubelet、CRI/containerd 到 K8s Event 的链路。

我的答案：

```text
1. 容器进程本质上是宿主机上的一个进程，只不过通过 namespace 进行隔离，
   因此在容器里看 pid 是 1，宿主机上是本质的 pid，实际上都是同一个进程
2. namespace 做隔离，cgroup 做资源限制
3. load average 不是 cpu 使用率，是统计一段时间内处于 R 或 D 状态的进程数量
4. free 表示实际物理空闲的内存大小，available 表示可以给下一个进程分配的内存大小，
   包括了用于 page cache 等内存部分，可以随时进行分配，available 更能代表可以使用的内存大小
5. CPU limit 超限一般会发生限速，进程运行变慢。
   memory limit 超限一般会触发 oom，进程可能直接被 kill
6. cluster ip 是一个逻辑 ip，实际由 kube proxy 在每个节点编写转发规则来控制请求访问
7. kubectl get pod app-xxx -o wide
8. top | grep "cpu%" head -10
9. 忘了
10. 拿到对应进程 id，看进程状态，运行周期，被限制周期，非主动切换上下文次数。/proc/<pid>/cgroup
11. 有个 xxx=cgroup 和 none 的区别。k8s 看 pod 日志上次状态和原因，节点看内核日志
12. df -h 和 df -i 看磁盘空间和 inode 分配数的情况
13. kubectl get endpoints
14. 通常是镜像拉取时有问题，先看 k8s 的 event reason，是什么类型的错误
15. 先看 pod 设置的 limit，再看对应进程状态的 cpu 使用情况是否达到 limit
16. 忘了
17. 查看对应节点的 fd 使用情况
18. 先看该 pod 的 dns 解析是否正确，再直接请求 service ip 看 service 是否工作，
    检查 service 背后是否有 endpoint
19. 不一定，可能该节点没有普通 pod，或 network=host
20. wget http://nginx-svc.default.svc
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
21. 忘了
22. kubelet -> CRI ImageService.PullImage -> containerd -> registry
                                          <- NotFound
    kubelet 记录日志并上报 Event
    Pod 状态变成 ImagePullBackOff
```

测试结果：

```text
A 概念题：5.5 / 6
B 命令题：4 / 7
C 场景题：4.5 / 6
D 链路题：2 / 3

总评：16 / 22
阶段 1 通过。
```

强项：

```text
容器进程、namespace/cgroup、load、available、CPU/memory limit 等概念基本清楚。
ImagePullBackOff 分类掌握较好。
Service 链路掌握较好，能串起 DNS、CoreDNS、ClusterIP、iptables、KUBE-SVC、KUBE-SEP、DNAT、Calico、conntrack。
```

薄弱点：

```text
/proc/<pid> 常用命令需要更熟。
cgroup v2 cpu/memory 文件名和字段需要背熟。
磁盘 du 定位路径需要更熟。
OOMKilled 完整链路需要再复述几遍。
```

## 11. 查漏补缺清单

### /proc/\<pid\>

```bash
sudo cat /proc/<pid>/status | grep -E 'Name|State|Pid|PPid|Threads|VmRSS|VmSize|ctxt'
sudo cat /proc/<pid>/cgroup
sudo cat /proc/<pid>/limits | grep 'Max open files'
sudo ls /proc/<pid>/fd | wc -l
```

### CPU throttling

```bash
sudo cat /sys/fs/cgroup/<cgroup-path>/cpu.max
sudo cat /sys/fs/cgroup/<cgroup-path>/cpu.stat
```

重点：

```text
nr_periods
nr_throttled
throttled_usec
```

### memory OOM

```bash
sudo cat /sys/fs/cgroup/<cgroup-path>/memory.max
sudo cat /sys/fs/cgroup/<cgroup-path>/memory.current
sudo cat /sys/fs/cgroup/<cgroup-path>/memory.events
sudo dmesg -T | grep -i -E 'oom|killed process|memory cgroup'
```

### 磁盘定位

```bash
df -h
df -i
sudo du -xh --max-depth=1 /var | sort -h
sudo du -xh --max-depth=1 /var/log | sort -h
sudo du -xh --max-depth=1 /var/lib | sort -h
sudo du -sh /var/log/containers /var/log/pods 2>/dev/null
sudo du -sh /var/lib/containerd /var/lib/kubelet 2>/dev/null
```

## 12. 下一阶段建议

阶段 2 建议进入：容器底层。

主题：

```text
namespace 深入
cgroup v2 深入
overlayfs
containerd / runc / OCI
CRI
kubelet 创建 Pod 调用链
pause 容器 / Pod sandbox
```

阶段 2 目标：

```text
能解释一个 Pod 是如何从 kubelet syncPod，
经过 CRI/containerd/runc，
最终变成 Linux namespace、cgroup 和进程的。
```

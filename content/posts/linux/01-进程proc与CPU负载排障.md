---
title: "K8s 节点排障（一）：进程、/proc、cgroup 与 CPU 负载"
date: 2026-06-12T08:00:00+08:00
draft: false
tags: ["Kubernetes", "Linux", "排障", "cgroup", "namespace", "CPU"]
categories: ["云原生", "Linux 排障"]
series: ["K8s 节点排障实战"]
summary: "深入理解 Pod 与宿主机进程的关系，掌握 /proc、namespace、cgroup 的核心概念，以及 CPU 与负载排障方法"
weight: 1
---

## 0. 当前学习背景

**目标岗位**：滴滴 K8s 开发岗位。

**已有基础**：
- Go 较熟悉
- 已阅读过部分 Kubernetes apiserver、informer、scheduler 源码
- 当前补强重点不是普通 Linux 命令使用，而是 K8s 节点侧开发、排障、kubelet/containerd/CNI/cgroup 相关的 Linux 底层能力

**实验环境**：
- Windows 宿主机
- WSL2 中使用 `kubectl` 访问集群
- 三节点 K8s 集群，节点均为 Ubuntu 22.04.5 LTS
- Kubernetes 版本：v1.31.14
- 容器运行时：containerd 1.7.23
- 内核版本：5.15.0-177-generic
- 网络插件：Calico

**节点信息**：

```text
k8s-master  192.168.59.134  control-plane
k8s-node1   192.168.59.135  worker
k8s-node2   192.168.59.136  worker
```

---

## 1. 阶段 1 总目标

**阶段 1：Linux 节点排障基本功。**

目标：登录任意 K8s Node 后，能快速判断问题大致属于以下哪类：

- 进程问题
- CPU 问题
- load 问题
- 内存问题
- 磁盘问题
- 网络问题
- kubelet/containerd 节点组件问题
- cgroup 资源限制问题

目前已经完成：
1. 进程与 `/proc`
2. namespace 与 cgroup 基础
3. CPU 与负载排障
4. CPU limit 与 cgroup v2 throttling 实验

---

## 2. 核心认知：Pod 里的进程就是宿主机上的 Linux 进程

Kubernetes Pod 不是虚拟机。Pod 里的容器进程，本质上就是宿主机上的普通 Linux 进程，只是被 namespace 隔离、被 cgroup 限制，并由 kubelet/containerd/runc 管理。

**上层排障入口**：

```bash
kubectl get pod
kubectl describe pod
kubectl logs
kubectl get pod -A -o wide
```

**底层 Linux 证据入口**：

```bash
ps
top
/proc/<pid>
/sys/fs/cgroup
journalctl
dmesg
```

**容器创建的大致链路**：

```text
kubelet -> CRI -> containerd -> containerd-shim -> runc -> Linux 进程
```

说明：
- kubelet 负责节点上 Pod 生命周期管理
- containerd 负责容器生命周期管理
- containerd-shim 负责在 containerd 与容器进程之间做守护和托管
- runc 负责最终调用 Linux 内核能力创建容器进程
- runc 多数时候是短生命周期进程，创建完容器后退出
- containerd-shim 和容器主进程通常长期存在

---

## 3. 从 Pod 追到宿主机 PID

**实验 Pod**：

```text
default/nginx-demo-db78b9d74-ntdbj
Node: k8s-master
container id: d09fe2169a3dc
host pid: 3972
```

先从 WSL2 控制端确认 Pod 所在节点：

```bash
kubectl get pod -A -o wide
```

关键字段：

```text
NAME                         IP               NODE
nginx-demo-db78b9d74-ntdbj   10.244.235.222   k8s-master
```

说明该 Pod 的真实 Linux 进程在 `k8s-master` 上，而不是 WSL2 里。

登录节点：

```bash
ssh <user>@192.168.59.134
```

查看 containerd 中的容器：

```bash
sudo crictl ps | grep nginx-demo
```

实验输出中出现：

```text
d09fe2169a3dc ... Running nginx ... nginx-demo-db78b9d74-ntdbj
```

查看容器详情并找 PID：

```bash
sudo crictl inspect d09fe2169a3dc | grep -E '"pid"|"podSandboxId"|"name"'
```

关键输出：

```text
"pid": 1
"name": "nginx"
"pid": 3972
"name": "nginx"
```

这里有两个 PID：

```text
pid: 1     容器 PID namespace 内看到的 PID
pid: 3972  宿主机上看到的真实 PID
```

同一个进程，在不同 PID namespace 中看到的 PID 可以不同。

---

## 4. 用 ps 与 pstree 验证进程链路

查看宿主机 PID：

```bash
ps -fp 3972
```

实验输出：

```text
UID   PID   PPID  CMD
root  3972  3917  nginx: master process nginx -g daemon off;
```

说明：
- PID 3972 是 nginx master process
- PPID 3917 是它的父进程

查看父进程：

```bash
ps -fp 3917
```

实验输出：

```text
/usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id 93a059de0aa96... -address /run/containerd/containerd.sock
```

说明 PID 3917 是 `containerd-shim-runc-v2`。

查看进程树：

```bash
pstree -aps 3972
```

实验输出：

```text
systemd,1
  └─containerd-shim,3917 -namespace k8s.io -id 93a059de0aa96... -address /run/containerd/containerd.sock
      └─nginx,3972
          ├─nginx,4008
          └─nginx,4009
```

对应关系：

```text
systemd -> containerd-shim -> nginx master -> nginx workers
```

nginx 进程角色：
- master 进程：管理 worker、加载配置、接收信号
- worker 进程：真正处理请求

---

## 5. 容器内 PID 与宿主机 PID 的差异

进入容器查看进程：

```bash
kubectl exec -it nginx-demo-db78b9d74-ntdbj -- sh -c 'ps -ef'
```

实验输出：

```text
PID   USER     TIME  COMMAND
1     root     0:00  nginx: master process nginx -g daemon off;
30    nginx    0:00  nginx: worker process
31    nginx    0:00  nginx: worker process
32    root     0:00  ps -ef
```

和宿主机 `pstree` 对比：

```text
宿主机 PID 3972 <=> 容器内 PID 1
宿主机 PID 4008 <=> 容器内 PID 30
宿主机 PID 4009 <=> 容器内 PID 31
```

**结论**：
- 容器里的进程不是不存在于宿主机
- 它们存在于宿主机进程表中，只是容器通过 PID namespace 看到另一套 PID 视图

---

## 6. /proc/<pid> 是进程的体检报告

查看进程状态：

```bash
sudo cat /proc/3972/status | grep -E 'Name|State|Pid|PPid|Threads|VmRSS|VmSize|ctxt'
```

实验输出：

```text
Name:   nginx
State:  S (sleeping)
Pid:    3972
PPid:   3917
VmSize: 10348 kB
VmRSS:  5816 kB
Threads: 1
voluntary_ctxt_switches: 18
nonvoluntary_ctxt_switches: 85
```

**字段解释**：

| 字段 | 含义 |
|------|------|
| Name | 进程名 |
| State | 进程状态 |
| Pid | 宿主机当前 PID |
| PPid | 父进程 PID |
| VmSize | 虚拟地址空间大小 |
| VmRSS | 实际驻留物理内存大小 |
| Threads | 线程数 |
| voluntary_ctxt_switches | 主动上下文切换次数 |
| nonvoluntary_ctxt_switches | 被动上下文切换次数 |

**进程状态常见值**：

| 状态 | 含义 |
|------|------|
| R | running/runnable，正在运行或等待 CPU |
| S | interruptible sleep，可中断睡眠，正常等待事件 |
| D | uninterruptible sleep，不可中断睡眠，常见于 IO 卡住 |
| Z | zombie，僵尸进程 |
| T | stopped，被暂停 |
| I | idle，内核线程空闲状态常见 |

nginx 的 `S (sleeping)` 不代表异常。nginx 大多数时间都在等待网络事件，所以 sleeping 很正常。

查看进程启动命令：

```bash
sudo tr '\0' ' ' < /proc/3972/cmdline
```

实验输出：

```text
nginx: master process nginx -g daemon off;
```

查看文件描述符：

```bash
sudo ls -l /proc/3972/fd | head
```

查看资源限制：

```bash
sudo cat /proc/3972/limits | head -20
```

实验关键输出：

```text
Max open files  1024  524288  files
```

含义：
- **Soft Limit**：当前生效的软限制，这里是 1024
- **Hard Limit**：进程最多能调高到的硬限制，这里是 524288

`Max open files` 限制的是文件描述符数量。Linux 中很多东西都是 fd：
- 普通文件
- socket
- pipe
- eventfd
- epoll fd

如果高并发服务连接数很多，soft limit 太小，可能出现：

```text
too many open files
```

**排查命令**：

```bash
cat /proc/<pid>/limits
ls /proc/<pid>/fd | wc -l
```

---

## 7. namespace 与 cgroup 基础

**一句话记忆**：

```text
namespace 负责"看见什么"
cgroup 负责"能用多少"
```

### namespace

namespace 是 Linux 内核提供的隔离机制，让不同进程看到不同的系统视图。

常见 namespace：

| Namespace | 作用 |
|-----------|------|
| PID namespace | 隔离进程号视图 |
| Network namespace | 隔离网卡、IP、路由表、端口 |
| Mount namespace | 隔离挂载点视图 |
| UTS namespace | 隔离 hostname |
| IPC namespace | 隔离进程间通信资源 |
| User namespace | 隔离用户和 UID 映射 |
| Cgroup namespace | 隔离 cgroup 路径视图 |

例子：

```text
PID namespace:
容器里 nginx 是 PID 1，宿主机上是 PID 3972。

Network namespace:
Pod 有自己的 eth0、IP、路由表。

Mount namespace:
容器看到的是镜像 rootfs，不是宿主机的 /。

UTS namespace:
容器内 hostname 可以和宿主机不同。
```

### cgroup

cgroup，全称 control group，是 Linux 内核提供的资源管理机制。

它负责：
- 限制资源
- 统计资源
- 隔离资源压力
- 控制进程组

常见资源：
- CPU
- memory
- pids
- blkio/io
- cpuset
- hugetlb

例子：

```text
限制一个容器最多用 512Mi 内存
限制一个容器最多用 0.5 个 CPU
限制一个容器最多创建 100 个进程
统计某个 Pod 当前用了多少内存
```

K8s 不是自己实现 namespace 和 cgroup，而是通过 kubelet、CRI、containerd、runc 最终调用 Linux 内核能力。

**链路**：

```text
kubelet
  -> containerd
    -> runc
      -> Linux namespace
      -> Linux cgroup
      -> 容器进程
```

---

## 8. K8s QoS、Pod cgroup、Container cgroup 的层级关系

实验中 nginx 的 cgroup：

```bash
sudo cat /proc/3972/cgroup
```

输出：

```text
0::/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-pode8375fe9_85c8_494e_bc57_c549aba3a625.slice/cri-containerd-d09fe2169a3dc...scope
```

`0::` 说明节点使用 cgroup v2 unified hierarchy。

这个路径不是多个互相独立的 cgroup，而是一棵父子层级树：

```text
kubepods.slice
  └─ kubepods-besteffort.slice
      └─ pod<PodUID>.slice
          └─ cri-containerd-<ContainerID>.scope
              └─ nginx 进程
```

**对应关系**：

```text
kubepods.slice
=> K8s 管理的 Pod 总入口

kubepods-besteffort.slice
=> BestEffort QoS 这一类 Pod

kubepods-besteffort-pod<uid>.slice
=> 某一个具体 Pod

cri-containerd-<container-id>.scope
=> 这个 Pod 里的某一个容器
```

**为什么既有 Pod cgroup 又有 Container cgroup？**

因为一个 Pod 可以包含多个容器：
- pause/sandbox 容器
- init container
- app 容器
- sidecar 容器

Kubernetes 需要同时表达：
- 这个 Pod 整体属于哪个 QoS 类别
- 这个 Pod 整体如何归组和统计资源
- 这个 Pod 内每个容器分别用了多少资源、受到哪些限制

所以可以理解为：

```text
QoS 级别 cgroup：区分 Guaranteed / Burstable / BestEffort
Pod 级别 cgroup：管理或统计整个 Pod
Container 级别 cgroup：管理 Pod 内某个具体容器
Linux 进程：真正被调度和运行的执行体
```

实际资源限制落在哪一层、哪些 controller 启用，取决于：
- kubelet 配置
- cgroup driver
- cgroup v1/v2
- container runtime 实现
- Pod/container resources 配置

当前环境是：

```text
systemd + cgroup v2 + containerd
```

---

## 9. Pod QoS 分类

K8s Pod QoS 常见三类：

| QoS | 条件 |
|-----|------|
| Guaranteed | 每个容器都设置 CPU 和 memory request/limit，并且 request == limit |
| Burstable | 至少设置了部分 request/limit，但不满足 Guaranteed |
| BestEffort | 所有容器都没有设置 CPU 和 memory request/limit |

实验中的 nginx Pod：

```text
kubepods-besteffort.slice
```

说明它是 BestEffort，通常意味着 Pod 内所有容器都没有设置 CPU 和 memory request/limit。

实验中的 `cpu-limited` Pod：

```text
kubepods-burstable.slice
```

原因是创建时设置了 CPU request/limit，但没有设置 memory request/limit，因此不满足 Guaranteed。

---

## 10. CPU 与 load average 基础

CPU 使用率、load average、进程状态不是同一个概念。

| 概念 | 含义 |
|------|------|
| CPU 使用率 | CPU 时间花在了哪里，例如 user、system、iowait、idle 等 |
| load average | 一段时间内处于 R 状态和 D 状态的任务数量平均值 |
| 进程状态 | 单个进程当前是在运行、睡眠、等待 IO、僵尸，还是停止 |

**它不是 CPU 百分比。**

判断 load 高不高，需要结合 CPU 核数。

例如 4 核机器：

```text
load 1: 通常很轻
load 4: 接近满载
load 8: 可能有明显排队
```

但 load 高不一定是 CPU 算力不足，因为 D 状态任务也会计入 load。

**关键判断**：

```text
load 高时，要先区分是 R 多，还是 D 多。

R 多：可能 CPU 不够或大量任务等待调度。
D 多：优先怀疑 IO、块设备、网络文件系统或内核路径卡住。
```

---

## 11. top CPU 字段解释

观察命令：

```bash
top
# 或批处理输出
top -b -n 1 | head -20
```

**CPU 字段**：

| 字段 | 含义 |
|------|------|
| us | user，用户态 CPU，业务代码、Go 服务、nginx 等 |
| sy | system，内核态 CPU，系统调用、网络协议栈、文件系统等 |
| ni | nice，调整过优先级的用户态任务 |
| id | idle，空闲 |
| wa | iowait，等待块设备 IO |
| hi | hardware interrupt，硬中断 |
| si | software interrupt，软中断，网络包处理常见 |
| st | steal time，虚拟机里 CPU 被宿主机抢走的时间 |

**K8s 节点常见判断**：

```text
us 高：业务进程吃 CPU
sy 高：系统调用、网络、内核路径压力大
wa 高：磁盘或块设备 IO 慢
si 高：网络包处理压力大
st 高：虚拟化宿主机资源争抢
```

因为当前 K8s 节点是虚拟机，所以 `st` 值值得关注。若节点卡顿但 CPU 不高，`st` 高可能说明 Windows 宿主机或虚拟化层资源紧张。

---

## 12. 当前 master 节点健康状态观察

观察命令：

```bash
uptime
top -b -n 1 | head -20
ps -eo pid,ppid,stat,comm,%cpu,%mem --sort=-%cpu | head -15
ps -eo pid,ppid,stat,comm,wchan:32 | awk '$3 ~ /D|R/ {print}' | head -30
```

实验输出摘要：

```text
load average: 0.31, 0.36, 0.47
Tasks: 280 total, 1 running, 279 sleeping, 0 stopped, 0 zombie
%Cpu(s): 28.6 us, 28.6 sy, 42.9 id, 0.0 wa, 0.0 si, 0.0 st
```

进程 CPU 排名：

```text
kube-apiserver   3.4%
kubelet          2.0%
etcd             1.9%
kube-controller  1.4%
calico-node      1.2%
containerd       0.7%
```

R/D 状态观察：

```text
53685 7104 R+ ps -
```

只有 `ps` 自己处于 R，这是正常的。

**结论**：

```text
节点 load 很低。
没有明显 CPU 压力。
没有 iowait、softirq、steal 问题。
没有异常 R/D/Z 状态进程。
控制面组件轻微占用 CPU，属于正常现象。
```

注意：`top -b -n 1` 是瞬时采样，可能波动。更稳定的观察方式：

```bash
top -b -d 1 -n 3 | tail -40
mpstat 1 5
```

---

## 13. 找高 CPU 进程的常用命令

按 CPU 排序：

```bash
ps -eo pid,ppid,stat,comm,%cpu,%mem --sort=-%cpu | head -15
```

找 R 或 D 状态任务：

```bash
ps -eo pid,ppid,stat,comm,wchan:32 | awk '$3 ~ /D|R/ {print}' | head -30
```

查看某个进程的线程：

```bash
top -H -p <pid>
ps -L -p <pid> -o pid,tid,stat,comm,%cpu
```

对 Go/Java/nginx 等多线程或多进程服务，线程维度很重要。

---

## 14. CPU 压力实验：没有 CPU limit 的 BestEffort Pod

创建 CPU burning Pod：

```bash
kubectl run cpu-burn --image=busybox --restart=Never -- sh -c 'while true; do :; done'
```

确认调度位置：

```bash
kubectl get pod cpu-burn -o wide
```

实验中调度到了 `k8s-node2`。

在 node2 上查看 CPU：

```bash
ps -eo pid,ppid,stat,comm,%cpu,%mem --sort=-%cpu | head -10
```

实验输出：

```text
PID    PPID  STAT  COMMAND  %CPU
33291  32726 Rs    sh       98.4
33291  32726 Rs    sh       100
```

说明该 busy loop 进程几乎吃满一个 CPU core。

通过 crictl 找容器和 PID：

```bash
sudo crictl ps | grep cpu-burn
sudo crictl inspect ff33127c28915 | grep -E '"pid"|"name"'
```

输出：

```text
"pid": 1
"name": "cpu-burn"
"pid": 33291
"name": "cpu-burn"
```

查看宿主机进程：

```bash
ps -fp 33291
```

输出：

```text
sh -c while true; do :; done
```

查看 `/proc`：

```bash
sudo cat /proc/33291/status | grep -E 'Name|State|Pid|PPid|Threads|ctxt'
```

输出：

```text
Name: sh
State: R (running)
Pid: 33291
PPid: 32726
Threads: 1
voluntary_ctxt_switches: 10
nonvoluntary_ctxt_switches: 4127
```

**解读**：
- `State: R`：进程正在运行或等待 CPU
- `Threads: 1`：单线程
- `nonvoluntary_ctxt_switches` 很高：进程不主动让出 CPU，时间片到了以后经常被内核强制切走

查看 cgroup：

```bash
sudo cat /proc/33291/cgroup
```

输出：

```text
0::/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-pod4d93d451_4163_4f05_9f3b_f3f319493b65.slice/cri-containerd-ff33127c2891...scope
```

说明：

```text
cpu-burn 是 BestEffort Pod，没有设置 CPU/memory request 和 limit。
```

**结论**：

```text
没有 CPU limit 的 CPU 密集型 Pod 可以尽量吃 CPU。
Linux 调度器仍然会调度其他进程，但该 Pod 会尽可能拿到可用 CPU 时间。
```

清理：

```bash
kubectl delete pod cpu-burn
```

---

## 15. CPU limit 与 cgroup v2 throttling 实验

创建带 CPU request/limit 的 Pod：

```bash
kubectl run cpu-limited --image=busybox --restart=Never \
  --limits='cpu=200m' \
  --requests='cpu=200m' \
  -- sh -c 'while true; do :; done'
```

实验中调度到了 `k8s-node1`。

查容器：

```bash
sudo crictl ps | grep cpu-limited
```

输出：

```text
8938e2201b070 ... Running cpu-limited ... cpu-limited
```

查 PID：

```bash
sudo crictl inspect 8938e2201b070 | grep -E '"pid"|"name"'
```

输出：

```text
"pid": 1
"name": "cpu-limited"
"pid": 39557
"name": "cpu-limited"
```

查看进程：

```bash
ps -fp 39557
ps -eo pid,ppid,stat,comm,%cpu,%mem --sort=-%cpu | head -10
```

输出：

```text
39557 39264 Rs sh 20.2
```

该进程仍然是死循环，但只能吃到约 20% CPU。

原因：Pod 设置了 `cpu=200m`，等价于 0.2 core。

查看 cgroup：

```bash
sudo cat /proc/39557/cgroup
```

输出：

```text
0::/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-podf8782d68_1a43_4aa6_91c2_073802c18816.slice/cri-containerd-8938e2201b070...scope
```

说明该 Pod 是 Burstable。

原因：设置了 CPU request/limit，但没有设置 memory request/limit，不满足 Guaranteed。

---

## 16. cgroup v2 cpu.max

cgroup v2 的 CPU quota 文件：

```bash
sudo cat /sys/fs/cgroup/<cgroup-path>/cpu.max
```

实验输出：

```text
20000 100000
```

格式：

```text
quota period
```

含义：

```text
每 100000 微秒，也就是 100ms
最多只能运行 20000 微秒，也就是 20ms
```

计算：

```text
20000 / 100000 = 20%
```

对应 K8s：

```text
200m = 0.2 core = 单核 20% CPU 时间
```

**注意**：这里是 `200m`，不是 `200M`。

- `m` 表示 milliCPU，千分之一 CPU core
- `M` 常被理解为 Mega，不是 Kubernetes CPU 单位

**Kubernetes CPU 单位**：

```text
1 CPU = 1 core
500m = 0.5 core
200m = 0.2 core
1000m = 1 core
```

---

## 17. cgroup v2 cpu.stat 与 throttling

查看：

```bash
sudo cat /sys/fs/cgroup/<cgroup-path>/cpu.stat
```

实验输出：

```text
usage_usec 26784103
user_usec 26727641
system_usec 56462
nr_periods 1338
nr_throttled 1001
throttled_usec 79136954
```

**字段解释**：

| 字段 | 含义 |
|------|------|
| usage_usec | 总 CPU 使用时间，微秒 |
| user_usec | 用户态 CPU 使用时间，微秒 |
| system_usec | 内核态 CPU 使用时间，微秒 |
| nr_periods | 经历了多少个 CPU 控制周期 |
| nr_throttled | 有多少个周期发生过限流 |
| throttled_usec | 累计被限流了多少微秒 |

实验中：

```text
nr_periods = 1338
nr_throttled = 1001
```

限流周期比例：

```text
1001 / 1338 ≈ 74.8%
```

说明大约 75% 的 CPU 控制周期里，该容器都因为用完 quota 被限流。

`throttled_usec`：

```text
79136954 微秒 ≈ 79 秒
```

说明累计被限流约 79 秒。

**线上常见现象**：

```text
Pod 是 Running。
节点 CPU 不一定满。
服务延迟升高或变慢。
业务看起来没有明显错误。
底层原因可能是容器 CPU limit throttling。
```

**排查判断**：

```text
cpu.max 不是 max，说明设置了 CPU quota。
nr_throttled 持续增长，说明发生限流。
throttled_usec 持续快速增长，说明限流严重。
```

清理：

```bash
kubectl delete pod cpu-limited
```

---

## 18. CPU request 与 limit 的关系

Kubernetes resources 示例：

```yaml
resources:
  requests:
    cpu: "500m"
  limits:
    cpu: "1"
```

含义：

```text
request:
主要影响 scheduler 调度和 CPU 权重。
告诉 Kubernetes 这个 Pod 期望或保证需要多少 CPU 资源。

limit:
通过 cgroup CPU quota 限制运行时最多能使用多少 CPU。
```

关键点：

```text
CPU request 主要影响调度和权重。
CPU limit 会造成 cgroup throttling。
```

**线上非常重要的判断**：

```text
节点 CPU 没满，不代表 Pod 没有 CPU 问题。
如果 Pod 设置了较低 CPU limit，它可能已经被 cgroup 限流。
```

---

## 19. crictl warning 的处理

实验中 `crictl` 多次出现 warning：

```text
runtime connect using default endpoints ... deprecated
image connect using default endpoints ... deprecated
```

原因：没有显式配置 CRI runtime endpoint 和 image endpoint。

可以在节点上配置：

```bash
sudo crictl config runtime-endpoint unix:///run/containerd/containerd.sock
sudo crictl config image-endpoint unix:///run/containerd/containerd.sock
```

之后 `crictl ps` 输出会更干净。

---

## 20. 排障命令链总结

### 20.1 从 Pod 找到节点

```bash
kubectl get pod -A -o wide
kubectl get pod <pod> -o wide
```

看 `NODE` 字段。

### 20.2 在节点上找高 CPU 进程

```bash
ps -eo pid,ppid,stat,comm,%cpu,%mem --sort=-%cpu | head -15
```

### 20.3 用 crictl 对上容器和 PID

```bash
sudo crictl ps | grep <pod-name>
sudo crictl inspect <container-id> | grep -E '"pid"|"name"'
```

### 20.4 查看进程状态

```bash
ps -fp <pid>
sudo cat /proc/<pid>/status | grep -E 'Name|State|Pid|PPid|Threads|VmRSS|VmSize|ctxt'
```

### 20.5 查看启动命令

```bash
sudo tr '\0' ' ' < /proc/<pid>/cmdline
```

### 20.6 查看资源限制

```bash
sudo cat /proc/<pid>/limits
```

### 20.7 查看 cgroup 路径

```bash
sudo cat /proc/<pid>/cgroup
```

### 20.8 查看 cgroup v2 CPU 限制和限流

```bash
sudo cat /sys/fs/cgroup/<cgroup-path>/cpu.max
sudo cat /sys/fs/cgroup/<cgroup-path>/cpu.stat
```

### 20.9 找 R/D 状态进程

```bash
ps -eo pid,ppid,stat,comm,wchan:32 | awk '$3 ~ /D|R/ {print}' | head -30
```

### 20.10 查看线程

```bash
top -H -p <pid>
ps -L -p <pid> -o pid,tid,stat,comm,%cpu
```

---

## 21. 必须记牢的短句

```text
容器不是虚拟机，本质是宿主机进程。
namespace 负责"看见什么"。
cgroup 负责"能用多少"。
Pod 主要是网络等 namespace 的共享单位。
Container 是资源限制的重要执行单位。
load average 不是 CPU 使用率，而是 R/D 任务数平均值。
CPU limit 会变成 cgroup 的 quota。
节点 CPU 没满，不代表 Pod 没有被 CPU throttling。
/proc/<pid> 是进程体检报告。
/sys/fs/cgroup 是资源限制和统计证据入口。
```

---

## 系列导航

- 下一篇：[K8s 节点排障（二）：内存、磁盘与 OOMKilled]({{< relref "02-内存磁盘与OOM排障.md" >}})

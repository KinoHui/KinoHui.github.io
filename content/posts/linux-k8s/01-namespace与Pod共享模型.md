---
title: "容器底层（一）：namespace 与 Pod 共享模型"
date: 2026-06-23T10:00:00+08:00
draft: false
tags: ["Kubernetes", "Linux", "namespace", "容器", "Pod", "nsenter"]
categories: ["云原生", "容器底层"]
series: ["容器底层原理"]
summary: "深入理解 Linux namespace 隔离机制与 Kubernetes Pod 共享模型，通过实验掌握 PID/Network/Mount namespace 的判断方法、pause 容器的作用以及 nsenter 排障技巧。"
weight: 1
---

## 0. 本课目标

阶段 2 目标是理解容器底层：

```text
一个 Pod/容器到底是怎么被 Linux 创建出来的。
```

第 1 课聚焦 namespace：

```text
namespace 解决"进程看到什么"。
cgroup 解决"进程能用多少资源"。
```

本课目标：

```text
能用 /proc/<pid>/ns 判断容器之间是否共享 namespace。
理解 pause/sandbox 容器如何持有 Pod network namespace。
理解同 Pod 容器默认共享 net namespace，但不共享 pid/mount namespace。
理解 shareProcessNamespace: true 对 PID namespace 的影响。
学会用 nsenter 进入目标进程 network namespace 排障。
```

---

## 1. namespace 类型

常见 namespace：

```text
PID namespace:
隔离进程号视图。容器内主进程通常看到自己是 PID 1。

Network namespace:
隔离网卡、IP、路由表、端口空间。

Mount namespace:
隔离文件系统挂载视图。容器看到自己的 rootfs。

UTS namespace:
隔离 hostname/domainname。

IPC namespace:
隔离 System V IPC、POSIX message queue 等 IPC 资源。

User namespace:
隔离 UID/GID 映射。容器内 root 可以映射成宿主机非 root。

Cgroup namespace:
隔离进程看到的 cgroup 路径视图。
```

一句话：

```text
namespace 让进程看到一个被隔离过的世界。
```

---

## 2. 判断两个进程是否共享 namespace

查看：

```bash
sudo ls -l /proc/<pid>/ns
sudo readlink /proc/<pid>/ns/net
sudo readlink /proc/<pid>/ns/pid
sudo readlink /proc/<pid>/ns/mnt
```

输出示例：

```text
net -> net:[4026532656]
pid -> pid:[4026532739]
mnt -> mnt:[4026532738]
```

判断标准：

```text
两个进程某个 namespace 的 inode ID 相同，就说明它们在同一个 namespace。
例如 net:[4026532656] 相同，说明共享 network namespace。
```

---

## 3. nginx Pod 与 pause/sandbox 容器实验

实验 Pod：

```text
nginx-demo-db78b9d74-hnkd2
Node: k8s-node2
Pod IP: 10.244.169.184
```

业务容器：

```bash
sudo crictl inspect 8d0304b86ff44 | grep -E '"pid"|"name"'
```

输出：

```text
name: nginx
container pid: 1
host pid: 2661
```

Pod sandbox：

```bash
sudo crictl inspectp eec3ae7375269 | grep -E '"pid"|"name"|"namespace"|"ip"'
```

输出：

```text
name: nginx-demo-db78b9d74-hnkd2
host pid: 2613
ip: 10.244.169.184
```

判断：

```text
PID 2613 是该 Pod sandbox / pause 容器的宿主机 PID。
PID 2661 是 nginx 业务容器的宿主机 PID。
```

---

## 4. nginx 与 pause 的 namespace 对比

nginx：

```text
cgroup -> cgroup:[4026532804]
ipc     -> ipc:[4026532800]
mnt     -> mnt:[4026532802]
net     -> net:[4026532734]
pid     -> pid:[4026532803]
uts     -> uts:[4026532799]
```

pause：

```text
cgroup -> cgroup:[4026531835]
ipc     -> ipc:[4026532800]
mnt     -> mnt:[4026532798]
net     -> net:[4026532734]
pid     -> pid:[4026532801]
uts     -> uts:[4026532799]
```

宿主机 PID 1：

```text
net -> net:[4026531840]
pid -> pid:[4026531836]
mnt -> mnt:[4026531841]
ipc -> ipc:[4026531839]
uts -> uts:[4026531838]
```

结论：

```text
nginx 和 pause 共享：
net namespace
ipc namespace
uts namespace

nginx 和 pause 不共享：
pid namespace
mnt namespace
cgroup namespace
```

核心：

```text
Pod IP 属于 Pod 共享 network namespace。
pause/sandbox 容器持有这个 network namespace。
业务容器加入这个 network namespace。
```

---

## 5. nsenter 进入 Pod network namespace

进入 nginx 的 network namespace：

```bash
sudo nsenter -t 2661 -n ip addr
sudo nsenter -t 2661 -n ip route
```

进入 pause 的 network namespace：

```bash
sudo nsenter -t 2613 -n ip addr
sudo nsenter -t 2613 -n ip route
```

两者输出一致：

```text
1: lo
3: eth0@if7
   inet 10.244.169.184/32

default via 169.254.1.1 dev eth0
169.254.1.1 dev eth0 scope link
```

结论：

```text
nginx 和 pause 进入的是同一个 network namespace。
```

Pod netns 里的 Calico 路由特征：

```text
Pod IP 是 /32。
默认路由指向 169.254.1.1。
流量通过 eth0 进入 veth 对端，由宿主机 cali* 接口和 Calico 路由接管。
```

注意：

```text
Pod netns 中可能看到 tunl0 DOWN。
排障时主要关注 eth0 和 default route。
宿主机 netns 中的 tunl0 才是跨节点 IPIP 转发重点。
```

---

## 6. 双容器 Pod：共享 netns，不共享 pid/mnt

Pod 定义：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:
  containers:
  - name: web
    image: nginx
  - name: sidecar
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3600"]
```

Pod 状态：

```text
two-containers 2/2 Running
Pod IP: 10.244.36.116
Node: k8s-node1
```

在 sidecar 中访问 localhost：

```bash
kubectl exec -it two-containers -c sidecar -- wget -qO- --timeout=2 http://127.0.0.1
```

返回 nginx welcome page。

结论：

```text
sidecar 容器里的 127.0.0.1 能访问 web 容器里的 nginx。
同 Pod 内容器共享 network namespace、localhost 和端口空间。
```

在 sidecar 中查看进程：

```bash
kubectl exec -it two-containers -c sidecar -- ps
```

输出：

```text
PID   USER     TIME  COMMAND
1     root     0:00  sh -c sleep 3600
12    root     0:00  ps
```

看不到 nginx。

结论：

```text
默认情况下，同 Pod 内容器不共享 PID namespace。
```

web 容器中执行 `ps` 失败：

```text
exec: "ps": executable file not found in $PATH
```

解释：

```text
这是 nginx 镜像里没有 ps 命令，不是 PID namespace 问题。
精简镜像经常缺少 procps/busybox 等排障工具。
```

---

## 7. two-containers 的宿主机 namespace 证据

容器 PID：

```text
web host pid: 10206
sidecar host pid: 10326
pause host pid: 9997
```

network namespace：

```text
web     net:[4026532656]
sidecar net:[4026532656]
pause   net:[4026532656]
```

三者相同，说明：

```text
web、sidecar、pause 共享同一个 network namespace。
```

PID namespace：

```text
web     pid:[4026532739]
sidecar pid:[4026532742]
pause   pid:[4026532737]
```

三者不同，说明：

```text
默认情况下，同 Pod 内容器不共享 PID namespace。
```

mount namespace：

```text
web     mnt:[4026532738]
sidecar mnt:[4026532741]
pause   mnt:[4026532734]
```

三者不同，说明：

```text
每个容器有自己的 mount namespace。
每个容器看到自己的 rootfs。
```

进入 pause netns：

```bash
sudo nsenter -t 9997 -n ip addr
```

输出：

```text
eth0@if6
inet 10.244.36.116/32
```

进入 web netns 查看监听：

```bash
sudo nsenter -t 10206 -n ss -lntp
```

输出：

```text
LISTEN 0 511 0.0.0.0:80 users:(("nginx",pid=10245),("nginx",pid=10244),("nginx",pid=10206))
LISTEN 0 511 [::]:80    users:(("nginx",pid=10245),("nginx",pid=10244),("nginx",pid=10206))
```

说明：

```text
nginx 正在 Pod network namespace 中监听 80 端口。
```

---

## 8. nsenter 的排障价值

命令：

```bash
sudo nsenter -t <pid> -n ip addr
sudo nsenter -t <pid> -n ip route
sudo nsenter -t <pid> -n ss -lntp
```

含义：

```text
-t <pid>: 以某个目标进程作为 namespace 参照。
-n: 进入目标进程的 network namespace。
```

特点：

```text
执行的 ip/ss 程序来自宿主机。
但看到的是目标进程 network namespace 里的网络视图。
```

适合排查：

```text
Pod netns 中有没有 eth0 和 Pod IP。
Pod netns 路由是否正确。
容器进程是否真的监听某端口。
localhost 为什么能或不能通。
精简镜像缺少 ip/ss/ps 等工具时，仍可从宿主机进入 netns 排查。
```

---

## 9. shareProcessNamespace 实验

Pod 定义：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: share-pid
spec:
  shareProcessNamespace: true
  containers:
  - name: web
    image: nginx
  - name: sidecar
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3600"]
```

容器 PID：

```text
web host pid: 17513
sidecar host pid: 17619
pause host pid: 17370
Pod IP: 10.244.169.191
```

PID namespace：

```text
web     pid:[4026532867]
sidecar pid:[4026532867]
pause   pid:[4026532867]
```

三者相同，说明：

```text
shareProcessNamespace: true 让 Pod 内容器共享 PID namespace。
```

network namespace：

```text
web     net:[4026532806]
sidecar net:[4026532806]
pause   net:[4026532806]
```

三者相同，和普通 Pod 一样共享 network namespace。

mount namespace：

```text
web     mnt:[4026532868]
sidecar mnt:[4026532870]
pause   mnt:[4026532864]
```

三者仍不同，说明：

```text
shareProcessNamespace: true 不会让容器共享 mount namespace。
web 仍是 nginx rootfs。
sidecar 仍是 busybox rootfs。
pause 仍是 pause rootfs。
```

---

## 10. CRI inspect 中 pid namespace 模式

普通 Pod 中曾看到：

```text
"pid": "CONTAINER"
```

shareProcessNamespace Pod 中看到：

```text
"pid": "POD"
```

含义：

```text
CONTAINER:
每个容器有自己的 PID namespace。

POD:
Pod 内容器共享 Pod 级 PID namespace。
```

这和 Kubernetes 的 `shareProcessNamespace: true` 对应。

---

## 11. 默认 Pod 与 shareProcessNamespace Pod 对比

默认 Pod：

```text
network namespace：共享
PID namespace：不共享
mount namespace：不共享
```

开启 `shareProcessNamespace: true`：

```text
network namespace：共享
PID namespace：共享
mount namespace：仍不共享
```

更完整的 Pod namespace 模型：

```text
pause/sandbox 容器先创建 Pod sandbox。
CNI 配置 pause/sandbox 的 network namespace。
业务容器加入 sandbox 的 network namespace。
默认每个业务容器有自己的 PID namespace 和 mount namespace。
如果 shareProcessNamespace=true，业务容器加入同一个 Pod PID namespace。
每个容器仍然有自己的 rootfs/mount namespace。
```

---

## 12. 小测点评

问题 1：怎么判断两个进程是否在同一个 namespace？

你的回答：

```text
看 /proc/$PID/ns/pid 看 namespace 是否相同。
```

点评：

```text
正确，但要按具体 namespace 看。
判断 PID namespace 看 /proc/<pid>/ns/pid。
判断 network namespace 看 /proc/<pid>/ns/net。
判断 mount namespace 看 /proc/<pid>/ns/mnt。
inode ID 相同就表示共享该 namespace。
```

问题 2：为什么同一个 Pod 里的 sidecar 可以通过 `127.0.0.1` 访问 web 容器？

你的回答：

```text
默认同一个 pod 里的容器在同一个网络命名空间。
```

点评：正确。

问题 3：默认情况下，同 Pod 内容器是否共享 PID namespace？你的实验依据是什么？

你的回答：

```text
默认情况下，同 Pod 内容器不共享 PID namespace。默认情况下的 /proc/$XXX_PID/ns/pid 不同。
```

点评：正确。

问题 4：`shareProcessNamespace: true` 改变了什么？没有改变什么？

你的回答：

```text
shareProcessNamespace: true 可以让同一 pod 里的容器进程在同一 pid 命名空间中。不会改变 mnt 命名空间。
```

点评：正确。

补充：

```text
network namespace 本来就共享，开启 shareProcessNamespace 后仍共享。
mount namespace 仍不共享。
```

问题 5：为什么同 Pod 内容器通常不共享 mount namespace？

你的回答：

```text
它们可以共享 volume，但不是共享整个 /。
```

点评：正确。

补充：

```text
每个容器有自己的镜像 rootfs 和挂载视图。
共享 volume 是把同一块 volume 挂到各自 mount namespace 中，不等于共享整个根文件系统。
```

问题 6：`pause`/sandbox 容器在 Pod namespace 模型里主要起什么作用？

你的回答：

```text
pause/sandbox 容器主要在 pod 创建时申请命名空间，业务容器直接按配置加入命名空间即可。
```

点评：基本正确。

更精确地说：

```text
pause/sandbox 容器持有 Pod sandbox，尤其是 Pod network namespace。
CNI 通常为 sandbox netns 配置 eth0、Pod IP、路由。
业务容器随后加入这个 sandbox 的 network namespace。
```

问题 7：`nsenter -t <pid> -n ss -lntp` 这类命令为什么对排障有用？

你的回答：

```text
ss 程序在宿主机内，可以通过命名空间的方式来查看。
```

点评：正确。

补充：

```text
它使用宿主机上的 ss/ip 工具，但进入目标进程的 network namespace。
因此即使容器镜像里没有 ss/ip，也能查看 Pod netns 的监听端口、IP、路由。
```

---

## 13. 必须记牢的短句

```text
判断是否共享 namespace，看 /proc/<pid>/ns/<type> 的 inode ID。
同 Pod 内容器默认共享 network namespace。
同 Pod 内容器默认不共享 PID namespace。
同 Pod 内容器默认不共享 mount namespace。
shareProcessNamespace: true 会让同 Pod 内容器共享 PID namespace。
shareProcessNamespace: true 不会让容器共享 rootfs/mount namespace。
pause/sandbox 容器持有 Pod sandbox，尤其是 Pod network namespace。
CNI 为 sandbox network namespace 配置 Pod IP 和路由。
业务容器加入 sandbox 的 network namespace。
nsenter 可以用宿主机工具进入目标进程 namespace 排障。
```

---

## 14. 下一课预告

下一课建议进入：

```text
cgroup v2 深入
```

重点：

```text
Pod cgroup 与 container cgroup
cpu.max / cpu.weight / cpu.stat
memory.max / memory.current / memory.events
pids.max / pids.current
K8s request/limit 如何落到 cgroup
```

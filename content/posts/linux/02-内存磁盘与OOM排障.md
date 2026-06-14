---
title: "K8s 节点排障（二）：内存、磁盘与 OOMKilled"
date: 2026-06-12T09:00:00+08:00
draft: false
tags: ["Kubernetes", "Linux", "排障", "内存", "磁盘", "OOMKilled"]
categories: ["云原生", "Linux 排障"]
series: ["K8s 节点排障实战"]
summary: "深入理解 Linux 内存机制、page cache、OOMKilled 原理，以及磁盘空间与 inode 排障方法"
weight: 2
---

## 0. 本阶段上下文

**实验环境**：
- Windows 宿主机
- WSL2 使用 `kubectl` 访问集群
- 三节点 Kubernetes 集群
- Kubernetes v1.31.14
- Ubuntu 22.04.5 LTS
- Linux kernel 5.15.0-177-generic
- containerd 1.7.23
- Calico 网络插件

**节点**：

```text
k8s-master  192.168.59.134
k8s-node1   192.168.59.135
k8s-node2   192.168.59.136
```

**本笔记覆盖**：
1. Linux 内存观察
2. page cache
3. RSS / VSZ / RssAnon / RssFile
4. K8s memory limit 与 cgroup v2
5. OOMKilled 实验
6. 磁盘空间与 inode
7. 容器日志路径
8. containerd/kubelet 数据目录
9. 文件句柄与 deleted file
10. 小测点评

---

## 1. Linux 内存观察：free、available 与 page cache

观察命令：

```bash
free -h
cat /proc/meminfo | head -30
ps -eo pid,ppid,stat,comm,rss,vsz,%mem --sort=-rss | head -15
```

实验输出：

```text
Mem:  total 3.8Gi  used 1.0Gi  free 526Mi  buff/cache 2.3Gi  available 2.5Gi
Swap: 0B
```

**关键判断**：

```text
free 只有 526Mi，但 available 有 2.5Gi。
节点整体内存并不紧张。
```

### free 和 available 的区别

| 字段 | 含义 |
|------|------|
| free | 完全没有被使用的内存 |
| available | 系统估算在不明显影响系统运行的情况下，还能分配给新进程的内存。它可能包含一部分可回收 page cache、可回收 slab 等 |

**判断 Linux 节点内存压力时，不能只看 `free`，更应该重点看 `available`。**

---

## 2. /proc/meminfo 关键字段

实验输出节选：

```text
MemTotal:        3968828 kB
MemFree:          539340 kB
MemAvailable:    2607696 kB
Buffers:          116608 kB
Cached:          2081708 kB
AnonPages:        745216 kB
SwapTotal:             0 kB
SwapFree:              0 kB
Slab:             271632 kB
SReclaimable:     171556 kB
SUnreclaim:       100076 kB
```

**字段解释**：

| 字段 | 含义 |
|------|------|
| MemTotal | 总物理内存 |
| MemFree | 完全空闲内存 |
| MemAvailable | 估算可用内存，更适合判断系统是否缺内存 |
| Buffers | 块设备相关缓存 |
| Cached | 文件页缓存，很多情况下可以回收 |
| AnonPages | 匿名页，通常更接近应用 malloc/new、堆、栈等占用，不能像文件缓存那样轻易丢弃 |
| SwapTotal / SwapFree | swap 总量和剩余量。当前节点 swap 为 0 |
| Slab | 内核对象缓存 |
| SReclaimable | 可回收 slab |
| SUnreclaim | 不可回收 slab |

**当前节点状态**：

```text
内存健康。
free 少但 available 充足。
大量内存用于 page cache，不是坏事。
```

---

## 3. page cache

page cache 是 Linux 用内存缓存文件内容的机制，用于提高文件 IO 性能。

**重要认知**：

```text
Linux 会尽量利用空闲内存做缓存。
因此 buff/cache 高不一定是问题。
page cache 不等于内存泄漏。
```

当应用需要内存时，很多 page cache 可以被回收。

**常见误区**：

```text
看到 free 很少就认为内存不足。
```

**正确方式**：

```text
结合 available、AnonPages、进程 RSS、cgroup memory.current 判断。
```

---

## 4. 进程内存：RSS、VSZ、RssAnon、RssFile

观察命令：

```bash
ps -eo pid,ppid,stat,comm,rss,vsz,%mem --sort=-rss | head -15
```

实验输出节选：

```text
PID   COMMAND          RSS     VSZ      %MEM
1461  kube-apiserver   307796  1520572  7.7
1444  kube-controller  107360  1312904  2.7
1159  kubelet           98136  2119296  2.4
1498  etcd              92660 11739516  2.3
3558  calico-node       73228  2163656  1.8
889   containerd        63660  2445920  1.6
```

控制面节点上，`kube-apiserver`、`etcd`、`kubelet`、`controller-manager` 常驻内存较高，属于正常现象。

查看 nginx 进程内存：

```bash
sudo cat /proc/<pid>/status | grep -E 'VmRSS|VmSize|RssAnon|RssFile|RssShmem'
```

实验输出：

```text
VmSize:    10368 kB
VmRSS:      6092 kB
RssAnon:    1520 kB
RssFile:    4568 kB
RssShmem:      4 kB
```

**字段解释**：

| 字段 | 含义 |
|------|------|
| VmSize / VSZ | 进程虚拟地址空间大小，不等于真实物理内存占用 |
| VmRSS / RSS | 进程当前驻留在物理内存中的大小，更接近真实物理内存占用 |
| RssAnon | 匿名页，通常包括堆、栈、malloc/new 出来的内存 |
| RssFile | 文件映射页，例如可执行文件、动态库、mmap 文件、文件缓存页等。其中一部分可能被多个进程共享，也更容易回收 |
| RssShmem | 共享内存 |

**关系**：

```text
VmRSS ≈ RssAnon + RssFile + RssShmem
6092 ≈ 1520 + 4568 + 4
```

---

## 5. CPU limit 与 memory limit 的核心区别

| 场景 | 结果 |
|------|------|
| CPU 超过 limit | 通常不会杀进程。内核通过 cgroup CPU quota 进行 throttling。表现为服务变慢、延迟升高、吞吐下降 |
| Memory 超过 limit | 通常触发 cgroup OOM。内核 kill cgroup 内的进程。容器退出，K8s 显示 OOMKilled |

**一句话**：

```text
CPU 超 limit：变慢。
Memory 超 limit：被杀。
```

---

## 6. OOMKilled 实验

创建会 OOM 的 Pod：

```bash
kubectl run mem-oom --image=python:3.12-alpine --restart=Never \
  --limits='memory=64Mi' \
  --requests='memory=64Mi' \
  -- python -c "x=[]; import time; [x.append('x'*1024*1024) or time.sleep(0.05) for _ in range(200)]"
```

观察：

```bash
kubectl get pod mem-oom -w
```

实验输出：

```text
mem-oom   0/1   ContainerCreating   0   4s
mem-oom   1/1   Running             0   13s
mem-oom   0/1   OOMKilled           0   17s
```

查看 Pod 所在节点：

```bash
kubectl get pods -o wide
```

实验输出：

```text
mem-oom  0/1  OOMKilled  0  61s  10.244.36.109  k8s-node1
```

说明 `mem-oom` 被调度到了 `k8s-node1`。

清理：

```bash
kubectl delete pod mem-oom
```

---

## 7. OOMKilled 的内核日志证据

在 Pod 所在节点查看内核日志：

```bash
sudo dmesg -T | grep -i -E 'oom|killed process' | tail -20
```

实验输出关键行：

```text
python invoked oom-killer: gfp_mask=0xcc0(GFP_KERNEL), order=0, oom_score_adj=984
```

说明触发 OOM 的进程是 `python`。

**关键证据**：

```text
oom-kill:constraint=CONSTRAINT_MEMCG
```

说明这是 memory cgroup OOM，不是节点级全局 OOM。

继续看：

```text
oom_memcg=/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-poda7bb27f8_473b_4470_a151_c646f92307b5.slice

task_memcg=/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-poda7bb27f8_473b_4470_a151_c646f92307b5.slice/cri-containerd-cfadd8a0...scope
```

说明 OOM 发生在：

```text
kubepods-burstable.slice
  -> Pod cgroup
    -> Container cgroup
      -> python 进程
```

**最关键日志**：

```text
Memory cgroup out of memory: Killed process 51979 (python) total-vm:72916kB, anon-rss:64036kB, file-rss:5468kB, shmem-rss:0kB
```

**字段解释**：

| 字段 | 含义 |
|------|------|
| Killed process 51979 (python) | 被杀的是宿主机 PID 51979 的 python 进程 |
| total-vm:72916kB | 虚拟地址空间约 71Mi |
| anon-rss:64036kB | 匿名常驻内存约 62.5Mi |
| file-rss:5468kB | 文件页常驻内存约 5.3Mi |

因为 Pod memory limit 是 64Mi，Python 匿名内存已经接近 64Mi，再加上文件页、页表、运行时开销，最终触发 cgroup OOM。

**核心结论**：

```text
节点 available 很充足，不代表容器不会 OOMKilled。
容器是否 OOMKilled，首先看它自己的 cgroup memory limit。
```

---

## 8. cgroup v2 memory 观察实验

创建不立即 OOM、便于观察的 Pod：

```bash
kubectl run mem-watch --image=python:3.12-alpine --restart=Never \
  --limits='memory=128Mi' \
  --requests='memory=128Mi' \
  -- python -c "x=[]; import time; [x.append('x'*1024*1024) or time.sleep(0.2) for _ in range(80)]; time.sleep(3600)"
```

查看 Pod：

```bash
kubectl get pod mem-watch -o wide
```

实验输出：

```text
mem-watch  1/1  Running  0  2m  10.244.36.111  k8s-node1
```

在节点上查看容器：

```bash
sudo crictl ps | grep mem-watch
sudo crictl inspect 7b2a8e64b13f8 | grep -E '"pid"|"name"'
```

实验输出：

```text
container id: 7b2a8e64b13f8
container pid: 1
host pid: 3151
```

查看 cgroup：

```bash
sudo cat /proc/3151/cgroup
```

输出：

```text
0::/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod2d20e0c0_e945_4676_86b7_118cf85abe53.slice/cri-containerd-7b2a8e64b13f8...scope
```

---

## 9. cgroup v2 memory.max、memory.current、memory.events

查看 memory limit：

```bash
sudo cat /sys/fs/cgroup/<cgroup-path>/memory.max
```

实验输出：

```text
134217728
```

换算：

```text
134217728 bytes / 1024 / 1024 = 128 MiB
```

对应 K8s：

```text
--limits='memory=128Mi'
```

查看当前 cgroup 内存使用：

```bash
sudo cat /sys/fs/cgroup/<cgroup-path>/memory.current
```

实验输出：

```text
96346112
```

换算：

```text
96346112 bytes / 1024 / 1024 ≈ 91.9 MiB
```

说明该容器 cgroup 当前使用约 92MiB。

**注意**：`memory.current` 不是简单等同于某个进程的 RSS。它是 cgroup 视角，可能包含：
- 匿名内存
- 文件页缓存
- 共享内存
- 部分内核相关内存
- 该 cgroup 内所有进程的内存

**对比**：

| 视角 | 说明 |
|------|------|
| `/proc/<pid>/status` | 单进程视角 |
| /sys/fs/cgroup/.../memory.current | 容器/cgroup 视角 |

查看 memory events：

```bash
sudo cat /sys/fs/cgroup/<cgroup-path>/memory.events
```

实验输出：

```text
low 0
high 0
max 0
oom 0
oom_kill 0
```

**字段解释**：

| 字段 | 含义 |
|------|------|
| low | 低保护相关事件，当前阶段先不重点关注 |
| high | 超过 memory.high 的次数。memory.high 是软限制/压力控制 |
| max | 达到或尝试超过 memory.max 的事件次数 |
| oom | cgroup 内发生 OOM 的次数 |
| oom_kill | 因为 cgroup OOM 触发 kill 的次数 |

**当前状态**：

```text
memory.max = 128Mi
memory.current ≈ 92Mi
oom = 0
oom_kill = 0
```

说明容器接近 limit，但没有超过，也没有触发 OOM。

---

## 10. K8s memory limit 到 OOMKilled 的完整链路

```text
kubectl run --limits=memory=128Mi
  -> kubelet 处理 Pod spec
  -> containerd/runc 创建容器 cgroup
  -> cgroup v2 memory.max = 134217728
  -> 进程运行时 memory.current 增长
  -> 超过 memory.max 后触发 cgroup OOM
  -> memory.events 中 oom/oom_kill 增长
  -> 内核 kill 进程
  -> kubelet 观测容器退出
  -> Pod 显示 OOMKilled
```

**线上 OOMKilled 排查命令链**：

```bash
kubectl get pod <pod> -o wide
kubectl describe pod <pod>
```

重点看：

```text
Last State: Terminated
Reason: OOMKilled
Exit Code: 137
```

到节点上：

```bash
sudo dmesg -T | grep -i -E 'oom|killed process' | tail -50
sudo journalctl -k | grep -i -E 'oom|killed process' | tail -50
```

如果看到：

```text
constraint=CONSTRAINT_MEMCG
Memory cgroup out of memory
oom_memcg=...
```

基本可以判断是容器 cgroup OOM。

如果是节点级全局 OOM，常见可能出现：

```text
constraint=CONSTRAINT_NONE
```

但实际仍需结合上下文判断，例如节点 MemAvailable、是否有 `Memory cgroup out of memory`、被杀进程、kubelet eviction 事件等。

---

## 11. 磁盘排障：df -h 与 df -i

观察命令：

```bash
df -h
df -i
```

实验输出：

```text
/dev/mapper/ubuntu--vg-ubuntu--lv  9.8G  6.8G  2.5G  73% /
```

根分区状态：

```text
总空间 9.8G
已用 6.8G
可用 2.5G
使用率 73%
```

**结论**：

```text
当前不是故障，但 K8s 节点根分区偏小，使用率 73% 需要留意。
```

### K8s 节点上常见消耗 / 的目录

```text
/var/log
/var/lib/containerd
/var/lib/kubelet
镜像层
容器可写层
temporary files
emptyDir
journal 日志
```

**经验阈值**：

| 使用率 | 状态 |
|--------|------|
| > 80% | 需要关注 |
| > 85% | 可能触发告警或 kubelet image GC |
| > 90% | 比较危险 |
| > 95% | 容易出现 Pod 创建失败、日志写入失败、节点 DiskPressure |

### inode 输出

```text
Filesystem                        Inodes  IUsed   IFree  IUse%
/dev/mapper/ubuntu--vg-ubuntu--lv 655360  130091  525269 20%
```

**结论**：

```text
inode 使用率只有 20%，很健康。
```

### df -h 和 df -i 的区别

| 命令 | 作用 |
|------|------|
| df -h | 看块空间是否用完 |
| df -i | 看 inode 是否用完 |

**两种"磁盘满"**：

| 类型 | 现象 |
|------|------|
| 块空间满 | 文件数据没地方写 |
| inode 满 | 不能创建新文件，即使 df -h 看起来还有空间 |

---

## 12. /var 目录占用分析

观察命令：

```bash
sudo du -xh --max-depth=1 /var | sort -h
sudo du -xh --max-depth=1 /var/log | sort -h
sudo du -xh --max-depth=1 /var/lib | sort -h
```

### /var 输出节选

```text
147M  /var/cache
1.1G  /var/log
1.8G  /var/lib
3.0G  /var
```

### /var/log 输出节选

```text
40K   /var/log/containers
548K  /var/log/pods
21M   /var/log/calico
925M  /var/log/journal
1.1G  /var/log
```

**结论**：

```text
/var/log 主要由 systemd journal 日志占用，不是容器日志。
```

### /var/lib 输出节选

```text
208K  /var/lib/kubelet
300M  /var/lib/apt
460M  /var/lib/snapd
999M  /var/lib/containerd
1.8G  /var/lib
```

**结论**：

```text
/var/lib 主要由 containerd 数据目录占用。
```

### K8s 节点上 /var/lib/containerd 通常存放

```text
镜像内容
镜像元数据
快照 snapshot
容器可写层
content store
运行时相关数据
```

### /var/lib/kubelet 通常和以下内容相关

```text
Pod volume
plugin 数据
kubelet 管理的 Pod 目录
某些 emptyDir/volume 相关数据
```

当前节点 `/var/lib/kubelet` 很小，说明该节点 kubelet 本地数据压力不大。

---

## 13. 容器日志路径：/var/log/containers 与 /var/log/pods

观察命令：

```bash
sudo ls -lh /var/log/containers | head
sudo du -sh /var/log/containers /var/log/pods 2>/dev/null
```

实验输出：

```text
/var/log/containers 40K
/var/log/pods       548K
```

说明容器日志当前很小。

`/var/log/containers` 中看到的文件是软链接：

```text
calico-node-xxx.log -> /var/log/pods/kube-system_calico-node-xxx_<uid>/calico-node/7.log
```

### 路径关系

```text
/var/log/containers/*.log
  是软链接，方便按 pod/container 名称查找。

/var/log/pods/<namespace>_<pod>_<uid>/<container>/<restart-count>.log
  是真实日志文件。
```

### 示例

```text
/var/log/containers/log-demo_default_log-demo-64f93a....log
  -> /var/log/pods/default_log-demo_abc96835-b48f-48af-b8ee-c0ce67aa088c/log-demo/0.log
```

**含义**：

| 部分 | 含义 |
|------|------|
| default | namespace |
| log-demo | pod name |
| abc96835-b48f-48af-b8ee-c0ce67aa088c | pod uid |
| log-demo | container name |
| 0.log | 容器 restart count 为 0 时的日志文件 |

如果容器发生重启，可能出现：

```text
0.log
1.log
2.log
```

`kubectl logs` 背后也是通过 kubelet 读取节点上的容器日志。

---

## 14. 容器日志实验与 shell glob 细节

创建日志 Pod：

```bash
kubectl run log-demo --image=busybox --restart=Never -- sh -c 'i=0; while [ $i -lt 20000 ]; do echo "log-demo line $i $(date)"; i=$((i+1)); done; sleep 3600'
```

查看软链接：

```bash
sudo ls -lh /var/log/containers | grep log-demo
```

实验输出：

```text
log-demo_default_log-demo-64f93ae9...log -> /var/log/pods/default_log-demo_abc96835-b48f-48af-b8ee-c0ce67aa088c/log-demo/0.log
```

Pod 删除后再查：

```bash
LOG=$(sudo readlink -f /var/log/containers/*log-demo*.log)
echo "$LOG"
sudo ls -lh "$LOG"
```

输出：

```text
/var/log/containers/*log-demo*.log
ls: cannot access '/var/log/containers/*log-demo*.log': No such file or directory
```

**原因**：

```text
Pod 清理后，kubelet 可能清理 /var/log/containers 下的软链接和 /var/log/pods 下的目录。
```

同时还有一个 shell glob 细节：

```text
当 bash 的通配符没有匹配到任何文件时，默认会保留原始字符串。
```

所以：

```bash
/var/log/containers/*log-demo*.log
```

没有匹配时，会原样传给 `readlink`、`ls`、`du`、`tail`，于是报 `No such file or directory`。

---

## 15. inode 实验

实验命令：

```bash
df -i /
mkdir -p /tmp/inode-demo
for i in $(seq 1 5000); do : > /tmp/inode-demo/file-$i; done
du -sh /tmp/inode-demo
find /tmp/inode-demo -type f | wc -l
df -i /
rm -rf /tmp/inode-demo
df -i /
```

**实验前**：

```text
IUsed 130091
IUse% 20%
```

**创建 5000 个小文件后**：

```text
du -sh /tmp/inode-demo = 136K
find ... | wc -l = 5000
IUsed 135092
IUse% 21%
```

**删除后**：

```text
IUsed 回到 130091
IUse% 回到 20%
```

**结论**：

```text
大量小文件不一定占很多磁盘空间，但会消耗大量 inode。
```

**线上遇到**：

```text
No space left on device
```

**不能只看**：

```bash
df -h
```

**还必须看**：

```bash
df -i
```

### 典型 inode 耗尽场景

```text
日志切分出海量小文件
临时目录堆积小文件
应用缓存碎文件
容器运行时残留大量元数据
```

---

## 16. 文件句柄 fd 与 Max open files

观察 kubelet：

```bash
pid=$(pidof kubelet)
echo $pid
sudo cat /proc/$pid/limits | grep 'Max open files'
sudo ls /proc/$pid/fd | wc -l
```

实验输出：

```text
PID: 877
Max open files 1000000 1000000 files
当前 fd 数量 25
```

**结论**：

```text
kubelet 当前 fd 很健康，距离 limit 很远。
```

### Linux 中很多对象都占用文件描述符

```text
普通文件
socket
pipe
eventfd
epoll fd
日志文件
设备文件
```

如果 `Max open files` 太小或 fd 泄漏，应用可能报：

```text
too many open files
```

### 排查命令

```bash
cat /proc/<pid>/limits | grep 'Max open files'
ls /proc/<pid>/fd | wc -l
ls -l /proc/<pid>/fd | head
```

### Go 服务常见 fd 泄漏原因

```text
HTTP response body 没有 Close
文件打开后没有 Close
连接池配置失控
大量长连接未释放
日志文件句柄泄漏
```

---

## 17. 删除大文件后 df 不下降

**经典现象**：

```text
删除了一个大日志文件，但 df -h 还是没有下降。
```

**原因**：

```text
文件名从目录中删除了，但如果还有进程打开着这个文件，inode 和数据块仍然被进程 fd 引用。
磁盘空间要等最后一个 fd 关闭后才释放。
```

### 排查

```bash
sudo lsof | grep deleted
```

如果没有 `lsof`，可以看：

```bash
sudo ls -l /proc/<pid>/fd | grep deleted
```

### 处理思路

```text
找到持有 deleted 文件的进程。
让进程重新打开日志文件、重启进程，或通过安全方式释放 fd。
```

**注意**：线上不能随便 kill 关键进程，要结合业务影响和组件类型判断。

---

## 18. 磁盘问题分流判断框架

### 18.1 是否块空间满

```bash
df -h
```

关注 `/`、`/var`、容器运行时所在分区。

### 18.2 是否 inode 满

```bash
df -i
```

如果 inode 使用率 100%，即使 `df -h` 还有空间，也可能创建不了文件。

### 18.3 哪个目录大

```bash
sudo du -xh --max-depth=1 /var | sort -h
sudo du -xh --max-depth=1 /var/log | sort -h
sudo du -xh --max-depth=1 /var/lib | sort -h
```

### 18.4 是否容器日志大

```bash
sudo du -sh /var/log/containers /var/log/pods 2>/dev/null
sudo ls -lh /var/log/containers | head
```

### 18.5 是否 containerd/kubelet 数据大

```bash
sudo du -sh /var/lib/containerd /var/lib/kubelet 2>/dev/null
```

### 18.6 是否文件删了但空间没释放

```bash
sudo lsof | grep deleted
```

或：

```bash
sudo ls -l /proc/<pid>/fd | grep deleted
```

### 18.7 是否 fd 快耗尽

```bash
cat /proc/<pid>/limits | grep 'Max open files'
ls /proc/<pid>/fd | wc -l
```

---

## 19. 必须记牢的短句

```text
free 少不等于内存不足，available 更关键。
page cache 是 Linux 正常缓存机制，不是天然泄漏。
VmRSS 是单进程物理内存视角。
memory.current 是 cgroup/container 视角。
CPU 超 limit 通常变慢，memory 超 limit 通常被杀。
constraint=CONSTRAINT_MEMCG 指向 cgroup OOM。
df -h 看空间，df -i 看 inode。
大量小文件可能先耗尽 inode。
/var/log/containers 是软链接入口，/var/log/pods 是真实容器日志位置。
/var/lib/containerd 常见存镜像、快照、容器层数据。
too many open files 要看 /proc/<pid>/limits 和 /proc/<pid>/fd。
删除大文件后 df 不降，优先怀疑仍有进程持有 deleted 文件 fd。
```

---

## 系列导航

- 上一篇：[K8s 节点排障（一）：进程、/proc、cgroup 与 CPU 负载]({{< relref "01-进程proc与CPU负载排障.md" >}})
- 下一篇：敬请期待

---
title: "容器底层（二）：cgroup v2 资源限制"
date: 2026-06-23T10:30:00+08:00
draft: false
tags: ["Kubernetes", "Linux", "cgroup", "资源限制", "CPU", "Memory", "QoS"]
categories: ["云原生", "容器底层"]
series: ["容器底层原理"]
summary: "深入理解 cgroup v2 资源限制机制，掌握 cpu.max、cpu.weight、memory.max、pids.max 等核心文件含义，以及 Kubernetes request/limit 如何映射到 cgroup 层级。"
weight: 2
---

## 0. 本课目标

上一课关注 namespace：

```text
namespace：进程看到什么
```

本课关注 cgroup：

```text
cgroup：进程能用多少，以及用了多少
```

目标是看懂这条链：

```text
K8s resources request/limit
  -> kubelet / containerd / runc
  -> systemd cgroup 路径
  -> cpu.max / cpu.weight / memory.max / pids.max
  -> Linux 内核实际限制和统计
```

---

## 1. cgroup v2 常用文件

当前节点使用 cgroup v2，证据：

```text
/proc/<pid>/cgroup 中出现：
0::/kubepods.slice/...
```

常用文件：

```text
cpu.max:
CPU 硬限制，也就是 CPU limit。

cpu.weight:
CPU 权重，通常和 CPU request 相关。

cpu.stat:
CPU 使用和 throttling 统计。

memory.max:
内存硬限制，也就是 memory limit。

memory.current:
当前 cgroup 内存使用量。

memory.events:
OOM、oom_kill 等事件统计。

pids.max:
最大进程/线程数量限制。

pids.current:
当前进程/线程数量。
```

---

## 2. K8s request/limit 与 cgroup 文件

实战对应关系：

```text
CPU limit
  -> cpu.max

CPU request
  -> cpu.weight，并影响调度

CPU throttling
  -> cpu.stat: nr_throttled / throttled_usec

Memory limit
  -> memory.max

Memory 当前用量
  -> memory.current

Memory OOM
  -> memory.events: oom / oom_kill

进程数限制
  -> pids.max

当前进程/线程数
  -> pids.current
```

重要区别：

```text
limit 决定硬上限。
request 影响调度和权重，不是硬限制。
```

---

## 3. CPU 与 Memory 的资源性质

CPU 是可压缩资源：

```text
超过 CPU limit 后，内核通常通过 throttling 限流。
表现是服务变慢、延迟升高、吞吐下降。
进程通常不会因为 CPU 超限而被杀。
```

Memory 是不可压缩资源：

```text
超过 memory limit 后，通常触发 cgroup OOM。
内核 kill 进程。
K8s 看到容器 OOMKilled，Exit Code 通常是 137。
```

一句话：

```text
CPU 超限：变慢。
Memory 超限：被杀。
```

---

## 4. 实验 Pod：BestEffort、Burstable、Guaranteed

创建 3 个 Pod 对比：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cg-besteffort
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3600"]
---
apiVersion: v1
kind: Pod
metadata:
  name: cg-burstable
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "500m"
        memory: "128Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: cg-guaranteed
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "500m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "128Mi"
```

查询：

```bash
kubectl get pod cg-besteffort cg-burstable cg-guaranteed -o jsonpath='{range .items[*]}{.metadata.name}{" qos="}{.status.qosClass}{" node="}{.spec.nodeName}{" ip="}{.status.podIP}{"\n"}{end}'
```

输出：

```text
cg-besteffort qos=BestEffort node=k8s-node1 ip=10.244.36.114
cg-burstable qos=Burstable node=k8s-node1 ip=10.244.36.113
cg-guaranteed qos=Guaranteed node=k8s-node1 ip=10.244.36.115
```

---

## 5. cgroup 路径与 QoS

BestEffort：

```text
/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-pod...slice/cri-containerd-....scope
```

Burstable：

```text
/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod...slice/cri-containerd-....scope
```

Guaranteed：

```text
/kubepods.slice/kubepods-pod...slice/cri-containerd-....scope
```

注意：

```text
Guaranteed 在当前 systemd cgroup driver 形态下，不一定出现 kubepods-guaranteed.slice。
它可能直接挂在 kubepods.slice 下。
不要死记路径名字，要结合实际 cgroup 路径判断。
```

---

## 6. BestEffort cgroup 文件

Pod：

```text
cg-besteffort
PID: 33681
QoS: BestEffort
```

关键文件：

```text
cpu.max = max 100000
cpu.weight = 1

cpu.stat:
usage_usec 26832
nr_periods 0
nr_throttled 0
throttled_usec 0

memory.max = max
memory.current = 393216

memory.events:
oom 0
oom_kill 0

pids.max = 4510
pids.current = 1
```

解读：

```text
没有 CPU quota 硬限制。
CPU 权重最低，为 1。
没有 memory limit。
当前内存使用很低。
没有发生 OOM。
存在 pids 上限 4510。
```

---

## 7. Burstable cgroup 文件

Pod：

```text
cg-burstable
PID: 33671
QoS: Burstable
request.cpu = 100m
limit.cpu = 500m
request.memory = 64Mi
limit.memory = 128Mi
```

关键文件：

```text
cpu.max = 50000 100000
cpu.weight = 4

cpu.stat:
usage_usec 20708
nr_periods 3
nr_throttled 0
throttled_usec 0

memory.max = 134217728
memory.current = 385024

memory.events:
oom 0
oom_kill 0

pids.max = 4510
pids.current = 1
```

解读：

```text
CPU limit 是 500m。
CPU request 100m 对应 cpu.weight 4。
Memory limit 是 128Mi。
当前没有 CPU throttling。
当前没有 OOM。
```

---

## 8. Guaranteed cgroup 文件

Pod：

```text
cg-guaranteed
PID: 33661
QoS: Guaranteed
request.cpu = 500m
limit.cpu = 500m
request.memory = 128Mi
limit.memory = 128Mi
```

关键文件：

```text
cpu.max = 50000 100000
cpu.weight = 20

cpu.stat:
usage_usec 26497
nr_periods 3
nr_throttled 0
throttled_usec 0

memory.max = 134217728
memory.current = 3510272

memory.events:
oom 0
oom_kill 0

pids.max = 4510
pids.current = 1
```

解读：

```text
CPU limit 是 500m。
CPU request 500m 对应 cpu.weight 20。
Memory limit 是 128Mi。
当前没有 CPU throttling。
当前没有 OOM。
```

---

## 9. cpu.max 换算

`cpu.max` 格式：

```text
quota period
```

例子：

```text
50000 100000
```

含义：

```text
每 100000 微秒周期内，最多运行 50000 微秒。
```

换算：

```text
50000 / 100000 = 0.5 core = 500m
```

BestEffort：

```text
max 100000
```

表示：

```text
没有 CPU quota 硬限制。
```

---

## 10. cpu.weight 与 CPU request

实验中：

```text
BestEffort: cpu.weight = 1
Burstable 100m request: cpu.weight = 4
Guaranteed 500m request: cpu.weight = 20
```

可以看到：

```text
CPU request 越高，cpu.weight 越高。
```

重要：

```text
cpu.weight 不是硬限制。
它在 CPU 竞争时影响分配权重。
```

对比：

```text
cpu.max:
硬限制，超过会 throttling。

cpu.weight:
竞争权重，影响 CPU 分配比例。
```

---

## 11. cpu.stat 与 throttling

关键字段：

```text
nr_periods:
经历了多少个 CPU 控制周期。

nr_throttled:
有多少个周期发生过 throttling。

throttled_usec:
累计被 throttling 的时间，单位微秒。
```

判断：

```text
nr_throttled 持续增长
throttled_usec 持续增长
=> 容器被 CPU limit 限流
```

本次 3 个 Pod 都只是 `sleep 3600`：

```text
nr_throttled 0
throttled_usec 0
```

所以没有 throttling。

---

## 12. memory.max 与 memory.current

BestEffort：

```text
memory.max = max
```

表示：

```text
没有 memory limit。
```

Burstable / Guaranteed：

```text
memory.max = 134217728
```

换算：

```text
134217728 bytes / 1024 / 1024 = 128Mi
```

对应：

```text
limits.memory = 128Mi
```

`memory.current`：

```text
当前 cgroup 内存使用量。
```

注意：

```text
memory.current 是 cgroup 视角。
/proc/<pid>/status 里的 VmRSS 是单进程视角。
```

一个 cgroup 里可能有多个进程，也可能包含匿名页、文件页、共享内存、部分内核内存等。

---

## 13. memory.events

字段：

```text
low
high
max
oom
oom_kill
```

重点：

```text
oom:
该 cgroup 内发生 OOM 的次数。

oom_kill:
该 cgroup 内因为 OOM 触发 kill 的次数。
```

注意修正：

```text
oom_kill 不是"整个机器触发 OOM kill 的次数"。
它仍然是当前 cgroup 视角的 OOM kill 计数。
```

节点级全局 OOM 需要结合内核日志判断，例如：

```bash
sudo dmesg -T | grep -i -E 'oom|killed process|memory cgroup'
sudo journalctl -k --since "1 hour ago" --no-pager | grep -i -E 'oom|killed process|memory cgroup'
```

---

## 14. pids.max 与 pids.current

实验中三个 Pod 都是：

```text
pids.max = 4510
pids.current = 1
```

含义：

```text
pids.max:
当前 cgroup 中允许的最大进程/线程数量。

pids.current:
当前 cgroup 中已有进程/线程数量。
```

`pids.max` 不是 `max`，说明当前节点对容器设置了进程数上限。

进入容器 exec 后：

```bash
kubectl exec -it cg-burstable -- sh
```

容器内看到：

```text
PID 1  sh -c sleep 3600
PID 7  sh
PID 12 ps
```

如果保持 exec shell 不退出，再读 `pids.current`，它会比 1 更高。

---

## 15. 本课核心表格

```text
BestEffort:
  QoS = BestEffort
  cpu.max = max 100000
  cpu.weight = 1
  memory.max = max
  pids.max = 4510

Burstable:
  QoS = Burstable
  cpu.max = 50000 100000
  cpu.weight = 4
  memory.max = 134217728
  pids.max = 4510

Guaranteed:
  QoS = Guaranteed
  cpu.max = 50000 100000
  cpu.weight = 20
  memory.max = 134217728
  pids.max = 4510
```

核心结论：

```text
CPU limit 落到 cpu.max。
CPU request 影响 cpu.weight。
Memory limit 落到 memory.max。
Memory request 不等于 memory.max。
QoS 会体现在 cgroup 层级和资源管理策略中。
pids.max 控制进程/线程数量上限。
```

---

## 16. 小测点评

问题 1：`cpu.max = 50000 100000` 表示什么意思？对应 K8s CPU 多少？

你的回答：

```text
每 100000 微秒能够最大运行 50000 微秒 cpu。对应 500m cpu。
```

点评：正确。

问题 2：`cpu.weight` 和 CPU request 有什么关系？它是硬限制吗？

你的回答：

```text
cpu.weight 对应 CPU request。不是硬限制，但是会影响调度和权重。
```

点评：基本正确。

补充：

```text
CPU request 会影响调度；在节点 cgroup 层面通常体现为 cpu.weight。
cpu.weight 影响 CPU 竞争时的分配权重，不是硬限制。
```

问题 3：如何从 cgroup v2 判断容器是否被 CPU throttling？

你的回答：

```text
查看 cpu.stat 文件的 nr_throttled / throttled_usec。
```

点评：正确。

问题 4：`memory.max = max` 和 `memory.max = 134217728` 分别代表什么？

你的回答：

```text
memory.max = max 代表无限制， memory.max = 134217728 代表内存最大使用限制为 134217728 bytes。
```

点评：正确。

问题 5：`memory.current` 和 `/proc/<pid>/status` 里的 `VmRSS` 有什么区别？

你的回答：

```text
memory.current 为当前 cgroup 的当前使用量，根据 cgroup 来确定，可能包含多个进程的使用量。
/proc/<pid>/status 的 VmRSS 为具体某个进程的实际使用量。
```

点评：正确。

问题 6：`memory.events` 里的 `oom`、`oom_kill` 代表什么？

你的回答：

```text
oom 代表触发 cgroup 上限导致的 oom。
oom_kill 为整个机器触发 oom 导致 kill 的次数。
```

点评：一半正确，需要修正。

正确理解：

```text
oom:
当前 cgroup 内发生 OOM 的次数。

oom_kill:
当前 cgroup 内因为 OOM 触发 kill 的次数。
```

`oom_kill` 不是整机 OOM kill 次数。

问题 7：`pids.max` 和 `pids.current` 分别看什么？

你的回答：

```text
pids.max 为当前 cgroup 最大进程数量，pids.current 是当前进程数量。
```

点评：正确。

补充：

```text
这里的统计更准确地说是 tasks 数量，通常可理解为进程/线程数量。
```

问题 8：为什么说 CPU 是可压缩资源，memory 是不可压缩资源？这对 Pod 超限结果有什么影响？

你的回答：

```text
cpu 超过 limit 会被 throttling，通常变慢。内存超过 limit 会 OOM kill，进程死亡。
```

点评：正确。

---

## 17. 必须记牢的短句

```text
cgroup v2 中，CPU limit 看 cpu.max。
cgroup v2 中，CPU request 通常影响 cpu.weight。
cpu.weight 不是硬限制。
CPU throttling 看 cpu.stat 的 nr_throttled 和 throttled_usec。
memory limit 看 memory.max。
当前 cgroup 内存看 memory.current。
cgroup OOM 事件看 memory.events。
oom_kill 是当前 cgroup 内 OOM kill 计数，不是整机计数。
pids.max 控制进程/线程数量上限。
CPU 超限通常变慢，memory 超限通常被杀。
```

---

## 18. 下一课预告

下一课建议进入：

```text
rootfs 与 overlayfs
```

重点问题：

```text
容器为什么看到自己的 /？
镜像层和容器可写层是什么关系？
overlayfs 的 lowerdir / upperdir / workdir / merged 分别是什么？
containerd snapshotter 如何把镜像层挂成容器 rootfs？
```

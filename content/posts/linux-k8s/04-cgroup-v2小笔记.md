---
title: "容器底层（四）：cgroup v2 小笔记"
date: 2026-06-23T11:30:00+08:00
draft: false
tags: ["Kubernetes", "Linux", "cgroup", "笔记", "QoS", "jsonpath"]
categories: ["云原生", "容器底层"]
series: ["容器底层原理"]
summary: "cgroup v2 补充笔记，涵盖 cut 命令用法、Kubernetes QoS 分类、kubectl jsonpath 语法、memory.events 详解、cgroup 与 PID 互查等实用知识点。"
weight: 4
---

## 1. `cut -d: -f3` 是什么

`cut` 用来按分隔符截取字段：

```bash
cut -d: -f3
```

含义：

```text
-d:   使用冒号 : 作为分隔符
-f3   取第 3 个字段
```

cgroup v2 下：

```bash
cat /proc/<pid>/cgroup
```

常见输出：

```text
0::/kubepods.slice/kubepods-burstable.slice/podxxx.slice/cri-containerd-xxx.scope
```

用 `:` 分割：

```text
第 1 段：0
第 2 段：空
第 3 段：/kubepods.slice/...
```

所以：

```bash
cat /proc/<pid>/cgroup | cut -d: -f3
```

可以取出 cgroup 路径。

---

## 2. Kubernetes QoS：BestEffort、Burstable、Guaranteed

Kubernetes 会根据 Pod 里容器的 `resources.requests` 和 `resources.limits` 自动给 Pod 分 QoS Class。

### BestEffort

所有容器都没有设置 CPU/memory request 和 limit。

```yaml
containers:
- name: app
  image: busybox
```

特点：

```text
资源声明最弱
节点资源紧张时最容易被驱逐
cgroup 路径常见 kubepods-besteffort.slice
```

### Burstable

设置了一部分 request/limit，或者 request 和 limit 不相等。

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "64Mi"
  limits:
    cpu: "500m"
    memory: "128Mi"
```

特点：

```text
资源保障居中
节点资源紧张时驱逐优先级居中
cgroup 路径常见 kubepods-burstable.slice
```

### Guaranteed

每个容器都同时设置 CPU 和 memory 的 request/limit，并且：

```text
cpu request == cpu limit
memory request == memory limit
```

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "128Mi"
```

特点：

```text
资源保障最强
节点资源紧张时最不容易被驱逐
```

简单记：

```text
BestEffort：啥都不写
Burstable：写了一些，或 request != limit
Guaranteed：CPU/memory 都写全，且 request == limit
```

---

## 3. `kubectl -o jsonpath` 怎么用

`jsonpath` 用来从 Kubernetes 对象 JSON 中提取字段。

例子：

```bash
kubectl get pod cg-besteffort cg-burstable cg-guaranteed \
  -o jsonpath='{range .items[*]}{.metadata.name}{" qos="}{.status.qosClass}{" node="}{.spec.nodeName}{" ip="}{.status.podIP}{"\n"}{end}'
```

含义：

```text
{range .items[*]}   遍历 items 数组
{.metadata.name}    输出 Pod 名
{" qos="}           输出普通字符串
{.status.qosClass}  输出 QoS
{" node="}          输出普通字符串
{.spec.nodeName}    输出节点名
{" ip="}            输出普通字符串
{.status.podIP}     输出 Pod IP
{"\n"}              输出换行
{end}               结束遍历
```

常用写法：

```bash
# 单字段
kubectl get pod nginx -o jsonpath='{.metadata.name}'

# 所有 Pod 名
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# 遍历并换行
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'

# 第一个容器名
kubectl get pod nginx -o jsonpath='{.spec.containers[0].name}'

# 所有容器名
kubectl get pod nginx -o jsonpath='{.spec.containers[*].name}'
```

简单记：

```text
. 表示当前对象
字段用 .metadata.name
数组用 [0] 或 [*]
range ... end 用来循环
{"xxx"} 输出普通文本
```

---

## 4. `memory.events` 里的 `oom` 和 `oom_kill`

查看：

```bash
cat /sys/fs/cgroup/<cgroup-path>/memory.events
```

示例：

```text
low 0
high 0
max 12
oom 3
oom_kill 2
```

含义：

```text
oom：
该 cgroup 发生内存不足，进入 OOM 处理流程的次数。

oom_kill：
因为该 cgroup OOM，内核实际 kill 进程的次数。
```

区别：

```text
oom：触发 OOM 处理
oom_kill：真的杀了进程
```

可能出现：

```text
oom > oom_kill
```

因为不是每次 OOM 事件都一定对应一次新的 kill。

---

## 5. `oom_kill` 杀的是谁

`oom_kill` 是针对当前 cgroup 统计的。

如果你看的是容器 cgroup：

```text
/sys/fs/cgroup/.../cri-containerd-xxx.scope/memory.events
```

那里面的 `oom_kill` 通常表示：

```text
这个容器 cgroup 内因为 OOM 被 kill 的进程次数
```

如果你看的是 Pod 级 cgroup：

```text
/sys/fs/cgroup/.../podxxx.slice/memory.events
```

可能包含子 cgroup 的事件统计。

cgroup v2 还有：

```text
memory.events
memory.events.local
```

区别：

```text
memory.events：
包含当前 cgroup 以及子 cgroup 的事件统计。

memory.events.local：
只统计当前 cgroup 本身，不包含子 cgroup。
```

---

## 6. cgroup 是不是每个进程一个

不是。

cgroup 是"进程组"，一个 cgroup 里可以有：

```text
0 个进程
1 个进程
多个进程
多个线程
```

K8s/containerd 常见结构：

```text
kubepods.slice
  -> kubepods-burstable.slice
    -> pod<uid>.slice
      -> cri-containerd-<container-id>.scope
        -> 容器内进程
```

也就是说：

```text
一个 Pod 通常有 Pod 级 cgroup
一个容器通常有容器级 cgroup
一个容器 cgroup 里可以有多个进程
```

比如 nginx 容器：

```text
nginx master
nginx worker 1
nginx worker 2
```

它们通常都在同一个容器 cgroup 里。

---

## 7. PID 查 cgroup

```bash
cat /proc/<pid>/cgroup
```

例子：

```text
0::/kubepods.slice/kubepods-burstable.slice/podxxx.slice/cri-containerd-xxx.scope
```

取路径：

```bash
CG=$(cat /proc/<pid>/cgroup | cut -d: -f3)
```

拼成真实目录：

```text
/sys/fs/cgroup$CG
```

---

## 8. cgroup 查 PID

看当前 cgroup 里有哪些进程：

```bash
cat /sys/fs/cgroup/<cgroup-path>/cgroup.procs
```

例子：

```bash
CG=$(cat /proc/12345/cgroup | cut -d: -f3)
cat /sys/fs/cgroup$CG/cgroup.procs
```

可能输出：

```text
12345
12380
12381
```

这些就是该 cgroup 里的进程 PID。

注意：

```text
cgroup.procs 通常只列当前 cgroup 直接包含的进程
不一定递归包含子 cgroup
```

递归查看某个 Pod cgroup 下所有进程：

```bash
find /sys/fs/cgroup/<pod-cgroup-path> -name cgroup.procs -exec sh -c 'echo === $1; cat "$1"' sh {} \;
```

---

## 9. `cgroup.procs` 和 `cgroup.threads`

```text
cgroup.procs：
进程 ID 视角，列出该 cgroup 中的进程。

cgroup.threads：
线程 ID 视角，列出该 cgroup 中的线程。
```

K8s 排障里通常优先看：

```bash
cat /sys/fs/cgroup/<container-cgroup>/cgroup.procs
```

---

## 10. 简单速记

```text
PID -> cgroup：
cat /proc/<pid>/cgroup

cgroup -> PID：
cat /sys/fs/cgroup/<cgroup-path>/cgroup.procs

QoS：
BestEffort：啥都不写
Burstable：写了一些，或 request != limit
Guaranteed：CPU/memory 都写全，且 request == limit

memory.events：
oom：触发 OOM
oom_kill：真的杀进程

memory.events：
包含子 cgroup

memory.events.local：
只看当前 cgroup

jsonpath：
从 kubectl 返回的 JSON 中提取字段
```

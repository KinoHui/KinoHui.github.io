---
title: "容器底层（五）：containerd / runc / OCI"
date: 2026-06-24T10:00:00+08:00
draft: false
tags: ["Kubernetes", "Linux", "containerd", "runc", "OCI", "shim", "config.json", "容器"]
categories: ["云原生", "容器底层"]
series: ["容器底层原理"]
summary: "理清 containerd、containerd-shim、runc 的分工，理解 OCI runtime bundle 与 config.json，能从运行中容器找到 runtime bundle，并把 config.json 中的 namespaces、mounts、resources、rootfs 与前面课程对应起来。"
weight: 5
---

## 0. 本课目标

前面三课分别讲了：

```text
namespace：容器看到什么
cgroup：容器能用多少
overlayfs/rootfs：容器的 / 从哪来
```

本课把它们串起来：

```text
kubelet
  -> CRI
  -> containerd
  -> containerd-shim-runc-v2
  -> runc
  -> Linux kernel
  -> 容器进程
```

目标：

```text
理解 containerd、containerd-shim、runc 的分工。
理解 OCI runtime bundle 和 config.json。
能从运行中容器找到 runtime bundle。
能把 config.json 中的 namespaces、mounts、resources、rootfs 与前面课程对应起来。
```

---

## 1. 角色分工

### kubelet

```text
Kubernetes 节点 agent。
负责 Pod 生命周期。
通过 CRI 调用容器运行时。
```

### CRI

```text
Container Runtime Interface。
Kubernetes 定义的容器运行时接口。
包含 RunPodSandbox、CreateContainer、StartContainer、StopContainer 等。
```

### containerd

```text
高级容器运行时。
负责镜像管理、snapshot 管理、容器元数据、CRI plugin、runtime bundle 准备、调用 runtime。
```

### containerd-shim-runc-v2

```text
长期存在的 shim 进程。
负责托管容器进程。
收集容器退出状态。
保持 stdio。
让容器进程不依赖 containerd 主进程生命周期。
```

### runc

```text
低级 OCI runtime。
读取 OCI bundle/config.json。
根据配置创建 namespace、mount、cgroup。
最终启动容器进程。
```

### Linux kernel

```text
真正提供 namespace、cgroup、mount、进程调度、网络栈等能力。
```

一句话：

```text
containerd 管生命周期和资源准备。
runc 按 OCI spec 真正创建容器。
shim 长期托管容器进程。
```

---

## 2. 为什么通常看不到长期存在的 runc

`runc` 通常是短生命周期进程。

它的主要工作：

```text
读取 config.json。
创建 namespace、cgroup、mount。
启动容器 init 进程。
```

创建完成后：

```text
runc 退出。
containerd-shim 继续托管容器进程。
```

所以节点上长期能看到：

```text
containerd
containerd-shim-runc-v2
容器进程
```

但通常看不到长期存在的：

```text
runc
```

---

## 3. OCI runtime bundle

OCI，全称 Open Container Initiative。

本课关心：

```text
OCI Runtime Spec
```

OCI runtime bundle 通常包含：

```text
config.json
rootfs/
```

`config.json` 描述：

```text
进程启动命令
环境变量
工作目录
rootfs 路径
mounts
namespaces
cgroups/resources
capabilities
seccomp
rlimits
用户
```

`runc` 可以理解为：

```text
读取 bundle/config.json
按 spec 创建容器
```

---

## 4. 实验对象

使用一个 nginx Pod。

进程树：

```text
systemd,1
  └─containerd-shim,2618 -namespace k8s.io -id 993dfefc...
      └─nginx,2674
          ├─nginx,2711
          └─nginx,2712
```

结论：

```text
containerd-shim-runc-v2 是长期托管进程。
nginx 是容器主进程。
runc 创建完容器后已经退出。
```

业务容器 ID：

```text
CID=9f68030acc8b2ea32c10b85ce29ec7b3aaa6a0054c66647c5e6ced8b0b2aaaae
```

Pod sandbox ID：

```text
SID=993dfefcc3b6fa499a5f89c928d5d2c55c1aa449c4022d0b730eb1ba700363b6
```

runtime bundle 路径：

```text
/run/containerd/io.containerd.runtime.v2.task/k8s.io/<container-id>/
```

---

## 5. 业务容器 config.json：process 与 root

查看：

```bash
sudo jq '.process.args, .root, .linux.namespaces, .mounts[0:5], .linux.resources' \
  /run/containerd/io.containerd.runtime.v2.task/k8s.io/$CID/config.json
```

业务容器启动命令：

```json
[
  "/docker-entrypoint.sh",
  "nginx",
  "-g",
  "daemon off;"
]
```

这对应容器内 nginx 主进程。

root：

```json
{
  "path": "rootfs"
}
```

含义：

```text
bundle 目录下的 rootfs 是该容器的根文件系统路径。
runc 使用这个 rootfs 启动容器进程。
```

和上一课 overlayfs/rootfs 的关系：

```text
root.path = "rootfs"
指向运行时准备好的容器 rootfs。
这个 rootfs 背后通常来自 containerd snapshotter 准备的 overlayfs 合并视图。
```

---

## 6. 业务容器 config.json：namespaces

业务容器 namespaces：

```json
[
  {
    "type": "pid"
  },
  {
    "type": "ipc",
    "path": "/proc/2641/ns/ipc"
  },
  {
    "type": "uts",
    "path": "/proc/2641/ns/uts"
  },
  {
    "type": "mount"
  },
  {
    "type": "network",
    "path": "/proc/2641/ns/net"
  },
  {
    "type": "cgroup"
  }
]
```

没有 `path` 的 namespace：

```json
{ "type": "pid" }
{ "type": "mount" }
{ "type": "cgroup" }
```

含义：

```text
runc 为这个业务容器创建新的 PID namespace。
runc 为这个业务容器创建新的 mount namespace。
runc 为这个业务容器创建新的 cgroup namespace。
```

带 `path` 的 namespace：

```json
{ "type": "ipc", "path": "/proc/2641/ns/ipc" }
{ "type": "uts", "path": "/proc/2641/ns/uts" }
{ "type": "network", "path": "/proc/2641/ns/net" }
```

含义：

```text
业务容器加入已有进程 2641 的 IPC namespace。
业务容器加入已有进程 2641 的 UTS namespace。
业务容器加入已有进程 2641 的 network namespace。
```

这个 `2641` 通常就是 pause/sandbox 进程 PID。

这和前面的 namespace 实验对应：

```text
业务容器共享 pause 的 net/ipc/uts namespace。
业务容器有自己的 pid/mount/cgroup namespace。
```

---

## 7. sandbox config.json

sandbox 启动命令：

```json
[
  "/pause"
]
```

root：

```json
{
  "path": "rootfs",
  "readonly": true
}
```

含义：

```text
pause 容器运行 /pause。
pause rootfs 是只读的。
pause 主要用于持有 Pod sandbox，不需要写自己的 rootfs。
```

sandbox namespaces：

```json
[
  {
    "type": "pid"
  },
  {
    "type": "ipc"
  },
  {
    "type": "uts"
  },
  {
    "type": "mount"
  },
  {
    "type": "network",
    "path": "/var/run/netns/cni-8fda8269-d492-30e1-527c-a4208b992a0e"
  }
]
```

含义：

```text
pause 自己创建 pid/ipc/uts/mount namespace。
network namespace 使用 CNI 创建/准备好的 netns 路径。
```

Pod network namespace 链路：

```text
CNI 创建并配置 Pod netns
  -> netns 路径类似 /var/run/netns/cni-...
  -> pause 加入并持有这个 netns
  -> 业务容器通过 /proc/<pause-pid>/ns/net 加入 pause 的 network namespace
```

---

## 8. mounts 配置

业务容器和 sandbox 中都能看到类似 mounts：

```json
[
  {
    "destination": "/proc",
    "type": "proc",
    "source": "proc",
    "options": ["nosuid", "noexec", "nodev"]
  },
  {
    "destination": "/dev",
    "type": "tmpfs",
    "source": "tmpfs",
    "options": ["nosuid", "strictatime", "mode=755", "size=65536k"]
  },
  {
    "destination": "/dev/pts",
    "type": "devpts",
    "source": "devpts"
  },
  {
    "destination": "/dev/mqueue",
    "type": "mqueue",
    "source": "mqueue"
  },
  {
    "destination": "/sys",
    "type": "sysfs",
    "source": "sysfs",
    "options": ["nosuid", "noexec", "nodev", "ro"]
  }
]
```

含义：

```text
runc 根据 config.json 中 mounts 配置，
在容器 mount namespace 中挂载 /proc、/dev、/dev/pts、/dev/mqueue、/sys 等。
```

这和 mount namespace 课程对应：

```text
每个容器有自己的 mount namespace。
runc 按 mounts 准备容器内基础文件系统视图。
```

---

## 9. linux.resources

业务容器：

```json
{
  "devices": [
    {
      "allow": false,
      "access": "rwm"
    }
  ],
  "memory": {},
  "cpu": {
    "shares": 2,
    "period": 100000
  },
  "unified": {
    "memory.oom.group": "1",
    "memory.swap.max": "0"
  }
}
```

解读：

```text
memory 为空，说明没有显式 memory limit。
cpu 有 shares/period 等默认值。
unified 是 cgroup v2 相关配置。
```

`memory.oom.group=1`：

```text
cgroup OOM 时按组处理。
```

`memory.swap.max=0`：

```text
禁用或限制 swap。
```

sandbox resources：

```json
{
  "devices": [
    {
      "allow": false,
      "access": "rwm"
    }
  ],
  "cpu": {
    "shares": 2
  }
}
```

说明：

```text
pause 基本不消耗资源，资源配置更少。
```

这和 cgroup 课程对应：

```text
config.json linux.resources
  -> runc 按配置设置 cgroup
  -> 节点上看到 cpu.max、cpu.weight、memory.max、memory.events 等文件
```

---

## 10. 前三课如何汇合到 OCI config.json

namespace 课：

```text
config.json linux.namespaces
  -> /proc/<pid>/ns/*
```

cgroup 课：

```text
config.json linux.resources
  -> /sys/fs/cgroup/.../cpu.max
  -> /sys/fs/cgroup/.../memory.max
  -> /sys/fs/cgroup/.../pids.max
```

overlayfs/rootfs 课：

```text
config.json root.path = "rootfs"
mounts 描述 /proc、/dev、/sys 等
  -> runc 准备容器 mount namespace
  -> 容器看到自己的 /
```

containerd/runc 本课：

```text
containerd 准备镜像、snapshot、metadata、OCI bundle。
runc 读取 config.json 创建容器。
containerd-shim 长期托管容器进程。
```

---

## 11. 关键验证命令

看进程树：

```bash
pstree -aps <container-pid>
```

看 bundle：

```bash
sudo ls -la /run/containerd/io.containerd.runtime.v2.task/k8s.io/<container-id>
sudo ls -la /run/containerd/io.containerd.runtime.v2.task/k8s.io/<sandbox-id>
```

看 config.json：

```bash
sudo jq '.process.args, .root, .linux.namespaces, .mounts[0:5], .linux.resources' \
  /run/containerd/io.containerd.runtime.v2.task/k8s.io/<container-id>/config.json
```

验证 namespace：

```bash
ps -fp <pause-pid>
sudo readlink /proc/<pause-pid>/ns/net
sudo readlink /proc/<container-pid>/ns/net
```

看 task init pid：

```bash
sudo cat /run/containerd/io.containerd.runtime.v2.task/k8s.io/<container-id>/init.pid 2>/dev/null
sudo cat /run/containerd/io.containerd.runtime.v2.task/k8s.io/<sandbox-id>/init.pid 2>/dev/null
```

---

## 12. 小测点评

问题 1：containerd、containerd-shim、runc 分别负责什么？

你的回答：

```text
containerd 负责管理镜像，管理 snapshot，metadata，OCI bundle。
containerd-shim 负责托管容器进程，与 containerd 解耦。
runc 负责实际根据 config.json 创建 namespace、mount、cgroup 并启动进程。
```

点评：正确。

问题 2：为什么通常看不到长期存在的 runc 进程？

你的回答：

```text
runc 创建完通常退出。
```

点评：正确。

问题 3：OCI runtime bundle 里的 `config.json` 大概描述哪些内容？

你的回答：

```text
容器启动命令，root 位置，ns，mount，resource 等等。
```

点评：正确。

补充：

```text
还可能包括 env、cwd、user、capabilities、seccomp、rlimits 等。
```

问题 4：业务容器 config 中 namespace 有的带 `path`，有的不带 `path`，分别表示什么？

你的回答：

```text
业务容器 config 中 namespace 有的带 path 表示是原来就有的 ns。
有的不带 path 表示是新创建的 ns。
```

点评：正确。

更准确：

```text
带 path 表示加入指定路径对应的已有 namespace。
不带 path 表示由 runtime 创建新的 namespace。
```

问题 5：为什么业务容器的 network namespace path 指向 pause 进程的 `/proc/<pid>/ns/net`？

你的回答：

```text
pause 进程的 /proc/<pid>/ns/net 通常是加入 CNI 创建的 ns，业务容器再加入 pause 容器的 netns。
```

点评：正确。

细微修正：

```text
不是“假如”，是“加入”。
CNI 先创建/配置 Pod netns，pause 加入并持有它，业务容器再加入 pause 的 netns。
```

问题 6：`root.path = "rootfs"` 和上一课 overlayfs/rootfs 有什么关系？

你的回答：

```text
root.path = "rootfs" 就是 overlayfs 合并后视图的根目录。
```

点评：基本正确。

更严谨：

```text
root.path = "rootfs" 指向 bundle 内的容器 rootfs。
这个 rootfs 通常由 containerd snapshotter 基于 overlayfs snapshot 准备。
```

问题 7：`containerd-shim` 为什么重要？如果没有 shim，containerd 重启或退出会有什么问题？

你的回答：

```text
没有 shim，containerd 重启或退出所有容器进程也会退出。
shim 能够让容器进程和 containerd 进程解耦。
```

点评：正确。

补充：

```text
shim 还负责收集退出状态、维护 stdio、向 containerd 汇报任务状态。
```

---

## 13. 必须记牢的短句

```text
containerd 是高级运行时，负责镜像、snapshot、metadata、CRI、bundle 准备。
runc 是低级 OCI runtime，按 config.json 创建容器。
runc 通常创建完容器就退出。
containerd-shim 长期托管容器进程，让容器和 containerd 主进程解耦。
OCI config.json 是 runc 创建容器的说明书。
namespace 带 path 表示加入已有 namespace，不带 path 表示创建新 namespace。
业务容器通过 /proc/<pause-pid>/ns/net 加入 Pod network namespace。
root.path = rootfs 指向 bundle 内容器 rootfs。
mounts 描述容器内 /proc、/dev、/sys 等挂载。
linux.resources 描述 cgroup 资源限制。
```

---

## 14. 下一课预告

下一课建议进入：

```text
kubelet 创建 Pod 调用链
```

重点：

```text
kubelet syncPod
RunPodSandbox
CreateContainer
StartContainer
pause 容器 / Pod sandbox
CRI 到 containerd
从 Kubernetes PodSpec 到 Linux namespace/cgroup/rootfs 的完整链路
```

---
title: "容器底层（七）：kubelet 创建 Pod 调用链"
date: 2026-06-24T11:00:00+08:00
draft: false
tags: ["Kubernetes", "kubelet", "CRI", "containerd", "runc", "Pod", "源码"]
categories: ["云原生", "容器底层"]
series: ["容器底层原理"]
summary: "梳理 Pod 从 kubelet SyncPod 到 CRI RunPodSandbox/PullImage/CreateContainer/StartContainer 的完整调用链，结合 chain-demo 实验与 OCI config 证据，把源码函数和节点上的 Linux 现象对起来。"
weight: 7
---

## 0. 本课目标

前面已经学习：

```text
namespace
cgroup
overlayfs/rootfs
containerd
containerd-shim
runc
OCI config.json
```

本课把这些底层部件接到 Kubernetes 源码主线上：

```text
PodSpec
  -> kubelet
  -> CRI
  -> containerd
  -> shim/runc
  -> namespace/cgroup/rootfs/process
```

目标：

```text
能讲清楚一个 Pod 从 apiserver 上的期望状态，
如何被 kubelet 同步到节点上，
再变成 pause 容器、业务容器、Linux 进程。
```

---

## 1. 高层创建链路

整体流程：

```text
apiserver 中出现 Pod
  -> kubelet watch 到 Pod
  -> pod worker 处理该 Pod
  -> kubelet SyncPod
  -> kubeGenericRuntimeManager.SyncPod
  -> RunPodSandbox
  -> PullImage
  -> CreateContainer
  -> StartContainer
  -> containerd
  -> shim/runc
  -> Linux namespace/cgroup/rootfs/process
```

更细一点：

```text
1. scheduler 把 Pod 绑定到某个 Node。
2. 该 Node 上的 kubelet watch 到这个 Pod。
3. kubelet 把 Pod 交给 podWorkers。
4. podWorker 串行处理单个 Pod 的同步。
5. kubelet SyncPod 判断当前状态和期望状态差异。
6. runtime manager 判断 sandbox 是否存在、是否需要重建。
7. 调用 CRI RunPodSandbox 创建 Pod sandbox / pause。
8. CNI 为 sandbox network namespace 配置网络。
9. 拉取业务容器镜像。
10. 调用 CRI CreateContainer。
11. 调用 CRI StartContainer。
12. containerd 准备 snapshot、OCI bundle。
13. runc 创建 namespace/cgroup/mount 并启动进程。
14. shim 托管容器进程。
15. kubelet 更新 PodStatus。
```

---

## 2. 源码入口

重点文件：

```text
pkg/kubelet/kubelet.go
pkg/kubelet/pod_workers.go
pkg/kubelet/kuberuntime/kuberuntime_manager.go
pkg/kubelet/kuberuntime/kuberuntime_sandbox.go
pkg/kubelet/kuberuntime/kuberuntime_container.go
pkg/kubelet/kuberuntime/kuberuntime_image.go
staging/src/k8s.io/cri-api/pkg/apis/runtime/v1/api.proto
```

建议搜索：

```bash
rg "func .*SyncPod" pkg/kubelet
rg "RunPodSandbox" pkg/kubelet
rg "CreateContainer" pkg/kubelet/kuberuntime
rg "StartContainer" pkg/kubelet/kuberuntime
rg "PullImage" pkg/kubelet/kuberuntime
```

关键模块：

```text
pod_workers.go：
保证同一个 Pod 的同步按 worker 语义串行处理，避免同一 Pod 多个同步并发乱跑。

kubelet.go SyncPod：
kubelet 同步单个 Pod 的主入口之一，负责 volume、secret、configmap、调用 runtime manager 等。

kuberuntime_manager.go SyncPod：
容器运行时层面的 Pod 同步，判断 sandbox、init container、app container 的创建/启动/停止。

kuberuntime_sandbox.go：
Pod sandbox / pause 相关逻辑。

kuberuntime_container.go：
业务容器 CreateContainer / StartContainer 等逻辑。

kuberuntime_image.go：
镜像 PullImage 相关逻辑。
```

---

## 3. Kubernetes Event 与调用链

正常 Pod 的 Event：

```text
Scheduled
Pulling
Pulled
Created
Started
```

对应关系：

```text
Scheduled：
scheduler 已经把 Pod 绑定到 Node。

Pulling：
kubelet 准备拉取镜像，调用 CRI ImageService.PullImage。

Pulled：
镜像拉取完成，或本地已有镜像。

Created：
CRI CreateContainer 成功。
containerd 已经准备容器 metadata、rootfs、OCI bundle 等。

Started：
CRI StartContainer 成功。
containerd-shim/runc 启动了容器进程。
```

常见失败事件：

```text
FailedCreatePodSandBox：
RunPodSandbox 失败，常见 CNI、sandbox、runtime 问题。

ErrImagePull / ImagePullBackOff：
PullImage 失败。

CreateContainerError：
CreateContainer 阶段失败。

RunContainerError：
StartContainer 或 runtime 启动阶段失败。

CrashLoopBackOff：
容器成功启动，但进程很快退出，kubelet 退避重启。
```

---

## 4. chain-demo 实验

创建：

```bash
kubectl run chain-demo --image=nginx --restart=Never
kubectl describe pod chain-demo
kubectl get pod chain-demo -o wide
```

Pod 信息：

```text
Name: chain-demo
Node: k8s-node1/192.168.59.135
Status: Running
IP: 10.244.36.119
Container ID: containerd://0093b75cfaf66730bc80fa75d868f955b50dfcf7985587d9d288706b4ef5faa7
QoS Class: BestEffort
```

Events：

```text
Scheduled  Successfully assigned default/chain-demo to k8s-node1
Pulling    Pulling image "nginx"
Pulled     Successfully pulled image "nginx"
Created    Created container: chain-demo
Started    Started container chain-demo
```

时间线：

```text
14:56:21 Scheduled
14:56:22 Pulling
14:56:24 Pulled
14:56:24 Created
14:56:24 Started
```

---

## 5. 进程树证据

业务容器 PID：

```text
96066
```

进程树：

```text
systemd,1
  └─containerd-shim,95981 -namespace k8s.io -id cf626152...
      └─nginx,96066
          ├─nginx,96102
          └─nginx,96103
```

结论：

```text
containerd-shim 长期托管容器进程。
nginx master 是容器主进程。
runc 已经创建完容器并退出。
shim 的 -id 是 sandbox ID。
```

---

## 6. runtime bundle 目录

业务容器 ID：

```text
CID=0093b75cfaf66730bc80fa75d868f955b50dfcf7985587d9d288706b4ef5faa7
```

sandbox ID：

```text
SID=cf626152bee6ed73bc229efeea4e6e562b3b51a08a6a398ebbc4f65c4c40ba8d
```

业务容器 bundle：

```bash
sudo ls -la /run/containerd/io.containerd.runtime.v2.task/k8s.io/$CID
```

输出包含：

```text
address
config.json
init.pid
log
log.json
options.json
rootfs
runtime
shim-binary-path
work
```

sandbox bundle：

```bash
sudo ls -la /run/containerd/io.containerd.runtime.v2.task/k8s.io/$SID
```

也包含：

```text
config.json
rootfs
init.pid
...
```

说明：

```text
业务容器和 Pod sandbox 都有自己的 runtime bundle。
config.json 是 runc 创建容器的说明书。
rootfs 是该 task 的容器根文件系统路径。
init.pid 通常记录该 task 的宿主机主进程 PID。
```

---

## 7. 业务容器 OCI config

查看：

```bash
sudo jq '.process.args, .root, .linux.namespaces, .linux.resources' \
  /run/containerd/io.containerd.runtime.v2.task/k8s.io/$CID/config.json
```

启动命令：

```json
[
  "/docker-entrypoint.sh",
  "nginx",
  "-g",
  "daemon off;"
]
```

root：

```json
{
  "path": "rootfs"
}
```

说明：

```text
runc 以 bundle/rootfs 作为容器根文件系统。
```

namespaces：

```json
[
  {
    "type": "pid"
  },
  {
    "type": "ipc",
    "path": "/proc/96002/ns/ipc"
  },
  {
    "type": "uts",
    "path": "/proc/96002/ns/uts"
  },
  {
    "type": "mount"
  },
  {
    "type": "network",
    "path": "/proc/96002/ns/net"
  },
  {
    "type": "cgroup"
  }
]
```

解释：

```text
96002 是 pause/sandbox 进程 PID。

pid：
不带 path，说明业务容器创建新的 PID namespace。

mount：
不带 path，说明业务容器创建新的 mount namespace。

cgroup：
不带 path，说明业务容器创建新的 cgroup namespace。

ipc / uts / network：
带 path，说明业务容器加入 pause 进程对应的已有 namespace。
```

这和 namespace 实验对齐：

```text
业务容器默认共享 pause 的 net/ipc/uts namespace。
业务容器默认有自己的 pid/mount/cgroup namespace。
```

resources：

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

因为 `chain-demo` 是 BestEffort，没有显式 request/limit，所以资源配置很少。

---

## 8. kubelet 日志观察

查询：

```bash
sudo journalctl -u kubelet --since "10 minutes ago" --no-pager \
  | grep -i -E 'chain-demo|sandbox|pull|create|start|container' \
  | tail -120
```

实际只抓到：

```text
VerifyControllerAttachedVolume started ...
pod="default/chain-demo"
```

解释：

```text
这不代表 kubelet 没做 Pull/Create/Start。
只是当前 kubelet 日志级别和 grep 关键字不一定能抓到所有步骤。
kubectl describe 中的 Pulling/Pulled/Created/Started Events 已经是 kubelet 上报的结果。
```

如需进一步查，可以扩大时间和关键词：

```bash
sudo journalctl -u kubelet --since "2026-06-24 06:56:00" --until "2026-06-24 06:57:00" --no-pager \
  | grep -i -E 'd283c07d|chain-demo|0093b75|cf626|pull|created|started|syncpod|sandbox|container'
```

---

## 9. 和源码函数对齐

### Scheduled

```text
scheduler 完成绑定。
kubelet watch 到被分配给本节点的 Pod。
```

### pod worker

```text
kubelet 把该 Pod 交给 podWorkers。
同一个 Pod 的同步按 worker 语义串行执行。
```

### kubelet SyncPod

```text
准备 volume / secret / configmap。
调用 runtime manager SyncPod。
```

实验日志中看到：

```text
VerifyControllerAttachedVolume started ...
```

属于 volume 准备相关。

### kubeGenericRuntimeManager.SyncPod

```text
判断 Pod sandbox 是否存在，是否需要重建。
判断 init container / app container 应该启动或停止。
```

### RunPodSandbox

```text
创建 pause / sandbox。
CNI 为 sandbox network namespace 配置 Pod IP。
```

实验证据：

```text
Pod annotation：
cni.projectcalico.org/containerID: cf626152...
cni.projectcalico.org/podIP: 10.244.36.119/32

sandbox ID：
cf626152...

pause PID：
96002
```

精确边界：

```text
RunPodSandbox 创建 Pod sandbox。
具体 netns 创建/配置由 runtime 和 CNI 协作完成。
CNI 负责给 Pod netns 配置 IP、路由、veth 等。
```

### PullImage

```text
kubelet 调用 CRI ImageService.PullImage。
containerd 拉取 nginx 镜像。
```

实验事件：

```text
Pulling
Pulled
```

### CreateContainer

```text
kubelet 调用 CRI CreateContainer。
containerd 准备容器 metadata、rootfs、OCI bundle。
```

实验事件：

```text
Created
```

实验文件：

```text
/run/containerd/io.containerd.runtime.v2.task/k8s.io/$CID/config.json
/run/containerd/io.containerd.runtime.v2.task/k8s.io/$CID/rootfs
```

### StartContainer

```text
kubelet 调用 CRI StartContainer。
containerd-shim / runc 启动 nginx 进程。
runc 创建 namespace/cgroup/mount。
shim 托管容器进程。
```

实验事件：

```text
Started
```

实验进程：

```text
containerd-shim -> nginx
```

---

## 10. Created 与 Started 的区别

`Created`：

```text
CreateContainer 成功。
容器 metadata、rootfs、OCI bundle 等启动所需资源已经准备好。
但容器主进程不一定已经运行。
```

`Started`：

```text
StartContainer 成功。
容器进程已经被启动。
节点上能看到对应业务进程 PID。
```

所以：

```text
Created 不等于进程已经运行。
Started 才对应容器进程真正起来。
```

---

## 11. StartContainer 成功后的 Linux 证据

可以看到：

```text
宿主机业务容器 PID
containerd-shim 进程
pstree 中 shim -> nginx
/proc/<pid>/ns/*
/proc/<pid>/cgroup
/sys/fs/cgroup/...
/run/containerd/.../$CID/config.json
/run/containerd/.../$CID/rootfs
/var/lib/containerd/.../overlayfs/snapshots/...
```

这些分别对应：

```text
进程存在
shim 托管
namespace
cgroup
OCI config
rootfs
overlayfs snapshot
```

---

## 12. 完整链路总结

```text
PodSpec
  -> scheduler bind 到 Node
  -> kubelet watch 到 Pod
  -> podWorkers 处理 Pod
  -> kubelet SyncPod
  -> runtime manager SyncPod
  -> RunPodSandbox
      -> pause/sandbox
      -> CNI 配置 Pod IP 和 netns
  -> PullImage
      -> containerd 拉镜像
  -> CreateContainer
      -> containerd 准备 metadata / snapshot / rootfs / OCI bundle
  -> StartContainer
      -> runc 按 config.json 创建 namespace/cgroup/mount
      -> 启动容器进程
      -> containerd-shim 托管
  -> kubelet 更新 PodStatus
```

最终落到 Linux：

```text
Pod IP: 10.244.36.119
sandbox ID: cf626152...
container ID: 0093b75...
pause PID: 96002
nginx PID: 96066
shim PID: 95981
OCI config.json
rootfs
namespace
cgroup
overlayfs snapshot
```

---

## 13. 小测点评

问题 1：`Scheduled / Pulling / Pulled / Created / Started` 分别大致对应哪一步？

你的回答：

```text
Scheduled：scheduler 把 Pod 绑定到 k8s-node1。
Pulling：kubelet 调用 CRI ImageService.PullImage 拉 nginx 镜像。
Pulled：镜像拉取完成，containerd 本地已有镜像内容和 snapshot 所需层。
Created：kubelet 调用 CRI CreateContainer 成功。containerd 已经准备容器 metadata、rootfs、OCI bundle。
Started：kubelet 调用 CRI StartContainer 成功。containerd-shim/runc 启动了 nginx 进程。
```

点评：正确。

问题 2：`RunPodSandbox` 主要创建什么？和 pause/CNI/netns 有什么关系？

你的回答：

```text
RunPodSandbox 创建 pause 容器。调用 CNI 分配 pod ip，创建 netns。
```

点评：基本正确。

更严谨：

```text
RunPodSandbox 创建 Pod sandbox / pause。
runtime 与 CNI 协作创建和配置 Pod network namespace。
CNI 负责配置 Pod IP、路由、veth 等内容。
pause 持有这个 Pod netns。
```

问题 3：`CreateContainer` 和 `StartContainer` 的区别是什么？

你的回答：

```text
CreateContainer 准备启动容器进程的环境，如 containerd 已经准备容器 metadata、rootfs、OCI bundle。
StartContainer 启动容器进程，containerd-shim/runc 启动了 nginx 进程。
```

点评：正确。

问题 4：`chain-demo` 的业务容器 config 里为什么 network namespace 指向 `/proc/96002/ns/net`？

你的回答：

```text
96002 为该 pod 的 pause 容器进程 id，业务进程的 netns 通常直接加入 pause 容器进程的 netns。
```

点评：正确。

问题 5：为什么 `Created` 事件出现时，容器进程不一定已经在运行？

你的回答：

```text
Created 事件只是准备好了启动容器进程需要的相关资源，实际容器进程可能还没启动，Started 才是容器进程启动运行中。
```

点评：正确。

问题 6：`containerd` 在 CreateContainer 前后大概准备了哪些东西？

你的回答：

```text
准备容器 metadata、rootfs、OCI bundle。
```

点评：正确。

补充：

```text
也包括 snapshot、运行时 task 目录、config.json 等。
```

问题 7：`StartContainer` 成功后，节点上能看到哪些 Linux 证据？

你的回答：

```text
有相应业务容器进程 pid，并且有对应 ns 路径。
```

点评：正确，但可以更完整。

补充：

```text
还可以看到 shim 进程、/proc/<pid>/cgroup、/sys/fs/cgroup、runtime bundle、OCI config、rootfs、overlayfs snapshot。
```

问题 8：你怎么把 kubelet 源码里的 `SyncPod` 和节点上的 crictl/containerd/runc 现象对起来？

你的回答：

```text
SyncPod 中 RunPodSandbox 创建 pause，配置 ip，分配 pid。
PullImage 容器运行时拉取镜像。
CreateContainer 准备 bundle，相关挂载等等。
StartContainer runc 创建 namespace/cgroup/mount。
```

点评：正确。

更完整：

```text
kubelet SyncPod 调 runtime manager SyncPod。
runtime manager 再调用 CRI RunPodSandbox / PullImage / CreateContainer / StartContainer。
这些 CRI 调用落到 containerd。
containerd 准备 snapshot 和 OCI bundle。
runc 按 config.json 创建 namespace/cgroup/mount 并启动进程。
shim 托管容器进程。
```

---

## 14. 必须记牢的短句

```text
Scheduled 是调度绑定完成。
Pulling/Pulled 对应 PullImage。
Created 对应 CreateContainer 成功。
Started 对应 StartContainer 成功。
RunPodSandbox 创建 Pod sandbox / pause。
CNI 为 Pod netns 配置 IP、路由、veth。
pause 持有 Pod network namespace。
CreateContainer 准备 metadata、snapshot、rootfs、OCI bundle。
StartContainer 通过 shim/runc 真正启动容器进程。
Created 不等于进程已运行，Started 才表示进程启动。
业务容器通过 /proc/<pause-pid>/ns/net 加入 Pod network namespace。
StartContainer 后可以从 /proc、cgroup、runtime bundle、pstree 看到 Linux 证据。
```

---

## 15. 下一步建议

阶段 2 已完成：

```text
namespace 与 Pod 共享模型
cgroup v2 资源限制
rootfs 与 overlayfs
containerd / runc / OCI
kubelet 创建 Pod 调用链
```

下一步建议：

```text
做阶段 2 总测试。
整理阶段 2 总结笔记。
之后进入阶段 3：K8s 节点网络深入，重点看 CNI、Calico、kube-proxy、conntrack、iptables/IPVS。
```

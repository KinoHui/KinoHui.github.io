---
title: "容器底层（三）：rootfs 与 overlayfs"
date: 2026-06-23T11:00:00+08:00
draft: false
tags: ["Kubernetes", "Linux", "rootfs", "overlayfs", "容器", "镜像", "containerd"]
categories: ["云原生", "容器底层"]
series: ["容器底层原理"]
summary: "深入理解容器 rootfs 构成与 overlayfs 工作原理，掌握 lowerdir/upperdir/workdir/merged 核心概念，以及 containerd snapshotter 在镜像层与容器可写层中的角色。"
weight: 3
---

## 0. 本课目标

前两课分别讲了：

```text
namespace：进程看到什么
cgroup：进程能用多少
```

本课讲：

```text
rootfs：容器看到的 / 从哪来
overlayfs：镜像层和容器可写层怎么拼成一个文件系统视图
```

目标：

```text
理解容器不是虚拟机，容器共享宿主机内核。
理解容器里的 / 是由运行时准备的 rootfs。
理解 overlayfs lowerdir / upperdir / workdir / merged。
理解新建文件、修改文件、删除 lowerdir 文件时 overlayfs 的行为。
理解 containerd snapshotter 在 rootfs 准备中的角色。
```

---

## 1. 容器里的 / 是怎么来的

容器没有独立内核。

容器里的 `/` 不是凭空来的，而是容器运行时准备好的 root filesystem。

然后运行时通过 mount namespace 让容器进程看到这个 rootfs。

一句话：

```text
容器共享宿主机内核，但在自己的 mount namespace 中看到运行时准备的 rootfs。
```

容器文件系统粗略模型：

```text
只读镜像层 lowerdir
  + 容器可写层 upperdir
  + overlayfs merged 视图
  = 容器看到的 /
```

---

## 2. overlayfs 基本概念

`lowerdir`：

```text
只读层。
通常来自镜像层。
可以是一层，也可以是多层。
```

`upperdir`：

```text
容器可写层。
容器运行后新增、修改、删除产生的变化主要记录在这里。
```

`workdir`：

```text
overlayfs 内部工作目录。
用于支持 copy-up、rename 等内部操作。
```

merged 视图：

```text
overlayfs 合并 lowerdir 和 upperdir 后呈现出来的视图。
容器进程在 mount namespace 里看到的 / 就是这个合并视图。
```

---

## 3. containerd 与 snapshotter

当前集群使用 containerd。

containerd 常见默认 snapshotter 是 overlayfs。

大致链路：

```text
镜像 pull 下来
  -> 解压成多个只读 layer
  -> containerd snapshotter 准备 active snapshot
  -> overlayfs 挂载 lowerdir + upperdir + workdir
  -> runc 把 merged 作为容器 rootfs
  -> 容器进程在 mount namespace 中看到自己的 /
```

containerd snapshotter 的角色：

```text
管理镜像层和容器可写层。
为容器准备可挂载的 snapshot。
组织 overlayfs 需要的 lowerdir、upperdir、workdir。
```

---

## 4. 实验 Pod

创建：

```bash
kubectl run rootfs-demo --image=busybox:1.36 --restart=Never -- sh -c 'sleep 3600'
kubectl get pod rootfs-demo -o wide
```

找到 PID：

```bash
sudo crictl ps | grep rootfs-demo
sudo crictl inspect <container-id> | grep -E '"pid"|"name"'
```

设：

```bash
PID=<host-pid>
```

---

## 5. mount namespace 中的 overlay 挂载

尝试：

```bash
sudo nsenter -t $PID -m mount | head -30
sudo nsenter -t $PID -m findmnt /
```

遇到：

```text
mount: no /proc/mounts
nsenter: failed to execute findmnt: No such file or directory
```

解释：

```text
只进入目标进程的 mount namespace 后，执行命令时看到的是容器的 mount 视图。
busybox rootfs 中没有 findmnt。
mount 工具也可能因为 /proc/mounts 视图或工具实现问题报错。
```

更可靠的方式：

```bash
sudo grep ' / ' /proc/$PID/mountinfo | head -5
sudo grep overlay /proc/$PID/mountinfo | head -5
```

实验输出：

```text
1149 1069 0:115 / / rw,relatime master:426 - overlay overlay rw,
lowerdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/15246/fs,
upperdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/15273/fs,
workdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/15273/work,
nouserxattr
```

结论：

```text
容器的 / 是 overlay 挂载。
lowerdir = snapshots/15246/fs
upperdir = snapshots/15273/fs
workdir = snapshots/15273/work
```

---

## 6. 新建文件：写入 upperdir

容器内执行：

```bash
kubectl exec -it rootfs-demo -- sh -c 'echo hello-overlay > /tmp/from-container.txt && cat /tmp/from-container.txt'
```

节点上查 upperdir：

```bash
sudo find /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/15273/fs -name from-container.txt -ls
sudo cat /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/15273/fs/tmp/from-container.txt
```

输出：

```text
/var/lib/containerd/.../snapshots/15273/fs/tmp/from-container.txt
hello-overlay
```

结论：

```text
容器中新建文件直接写入 upperdir。
```

---

## 7. /etc/profile 实验修正

容器内执行：

```bash
kubectl exec -it rootfs-demo -- sh -c 'echo changed > /etc/profile && tail -1 /etc/profile'
```

upperdir 中看到：

```text
/var/lib/containerd/.../snapshots/15273/fs/etc/profile
changed
```

一开始可能会以为这是 copy-up。

但进一步检查 lowerdir：

```bash
sudo ls -l /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/15246/fs/etc/profile
```

结果：

```text
No such file or directory
```

说明：

```text
lowerdir 中没有 /etc/profile。
因此这个实验更可能是"容器中新建 /etc/profile -> 写入 upperdir"，而不是修改 lowerdir 既有文件。
```

重要经验：

```text
证据不支持原假设时，要及时修正判断。
不要硬把所有 upperdir 文件都解释成 copy-up。
```

删除 `/etc/profile` 后：

```bash
sudo find /var/lib/containerd/.../snapshots/15273/fs/etc -maxdepth 1 -name '*profile*' -ls
```

无输出。

原因：

```text
/etc/profile 只存在于 upperdir。
删除 upperdir 文件时，直接从 upperdir 删除。
不需要 whiteout。
```

---

## 8. lowerdir 文件删除与 whiteout

为了观察 whiteout，选择 lowerdir 中确定存在的小文件：

```text
/etc/nsswitch.conf
```

确认 lowerdir 存在：

```bash
sudo ls -l /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/15246/fs/etc/nsswitch.conf
```

输出：

```text
-rw-r--r-- 1 root root 494 May 18 2023 /var/lib/containerd/.../snapshots/15246/fs/etc/nsswitch.conf
```

容器内删除：

```bash
kubectl exec -it rootfs-demo -- sh -c 'rm /etc/nsswitch.conf && ls -l /etc/nsswitch.conf || echo deleted-in-container'
```

查看 upperdir：

```bash
sudo ls -la /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/15273/fs/etc | grep nsswitch
sudo find /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/15273/fs/etc -maxdepth 1 -name '*nsswitch*' -ls
```

输出：

```text
c--------- 2 root root 0, 0 Jun 23 12:27 nsswitch.conf
```

这是 overlayfs whiteout 标记。

解释：

```text
lowerdir 里仍然有 /etc/nsswitch.conf。
但 upperdir 里放了同名 whiteout。
overlayfs 合并 merged 视图时看到 whiteout，
就隐藏 lowerdir 中的 /etc/nsswitch.conf。
所以容器里看起来该文件被删除了。
```

`c--------- 0,0`：

```text
表示 character device 0,0。
这是 overlayfs 表达 whiteout 的方式之一。
```

---

## 9. overlayfs 三种核心行为

### 新建文件

```text
容器内 echo > /tmp/from-container.txt
  -> upperdir/tmp/from-container.txt
```

### 新建或修改 upperdir 文件

```text
lowerdir 不存在 /etc/profile。
容器内 echo > /etc/profile。
  -> upperdir/etc/profile

删除后：
  -> 直接从 upperdir 消失。
```

### 删除 lowerdir 文件

```text
lowerdir/etc/nsswitch.conf 存在。
容器内 rm /etc/nsswitch.conf。
  -> upperdir/etc/nsswitch.conf 变成 c 0,0 whiteout。
  -> merged 视图中该文件消失。
```

---

## 10. BusyBox 硬链接现象

容器内：

```bash
kubectl exec -it rootfs-demo -- sh -c 'ls -l /bin | head -20'
```

输出中很多命令：

```text
-rwxr-xr-x 405 root root 1013320 ... awk
-rwxr-xr-x 405 root root 1013320 ... base64
-rwxr-xr-x 405 root root 1013320 ... sh
```

解释：

```text
BusyBox 常见设计是一个 busybox 多调用二进制，
通过不同 argv[0] 表现成不同命令。
/bin/ls、/bin/cat、/bin/sh 等很多可能是同一个文件的硬链接。
```

因此实验时不要随便删除 `/bin` 下命令。

更适合作为 whiteout 实验对象的是：

```text
/etc/nsswitch.conf
```

这类普通小配置文件。

---

## 11. 特殊文件：hostname / hosts / resolv.conf

upperdir 中看到：

```text
hostname
hosts
resolv.conf
```

且可能是 0 字节文件。

解释：

```text
/etc/hostname
/etc/hosts
/etc/resolv.conf
```

在容器中经常由 runtime 单独处理，可能涉及 bind mount 或运行时管理。

它们不适合作为普通镜像层 copy-up / whiteout 观察对象。

---

## 12. 和 Kubernetes / containerd / runc 的关系

容器 rootfs 链路：

```text
kubelet 请求创建容器
  -> containerd 准备 snapshot
  -> overlayfs 组合 lowerdir + upperdir + workdir
  -> runc 把这个 rootfs 放进容器 mount namespace
  -> 容器进程看到自己的 /
```

所以容器里写文件：

```bash
echo hello > /tmp/a.txt
```

不是写进镜像层，而是写进该容器自己的 upperdir。

容器删除后：

```text
该容器的 writable snapshot 通常会被清理。
```

---

## 13. 小测点评

问题 1：容器里的 `/` 是怎么来的？它和宿主机内核是什么关系？

你的回答：

```text
容器里的 / 通过 overlayfs 合并 merged 视图看到的。共享宿主机的内核。
```

点评：正确。

补充：

```text
容器共享宿主机内核，但在自己的 mount namespace 中看到运行时准备的 rootfs。
```

问题 2：overlayfs 里的 `lowerdir`、`upperdir`、`workdir`、merged 视图分别是什么？

你的回答：

```text
lowerdir 是只读层、upperdir 是可写层、workdir 是 overlay 工作层、merged 视图是容器看到的目录视图。
```

点评：正确。

问题 3：容器中新建文件为什么会出现在 `upperdir`？

你的回答：

```text
新建的只会出现在可写层。
```

点评：正确。

问题 4：修改 lowerdir 已有文件时，overlayfs 通常会发生什么？

你的回答：

```text
修改 lowerdir 已有文件时，overlayfs 通常会在 upperdir 中创建一份修改后的文件。叫 copy-up。
```

点评：正确。

问题 5：删除 lowerdir 已有文件时，为什么需要 whiteout？

你的回答：

```text
删除 lowerdir 已有文件时，whiteout 是因为 lowerdir 是只读层，不能修改，通过 whiteout 标记，在 merged 视图中就不会显示。
```

点评：正确。

问题 6：你这次看到的 `c--------- root root 0,0 nsswitch.conf` 说明什么？

你的回答：

```text
说明 nsswitch.conf 是在 lowerdir 已有的文件在容器中被删除的。
```

点评：正确。

补充：

```text
c 0,0 是 overlayfs whiteout 标记。
```

问题 7：containerd snapshotter 在这个链路里大概负责什么？

你的回答：

```text
containerd snapshotter 在这个链路里主要准备分层镜像。
```

点评：基本正确。

更准确：

```text
snapshotter 管理镜像层和容器可写层，
为容器准备 overlayfs 所需的 snapshot，
组织 lowerdir、upperdir、workdir，
供 runc 作为容器 rootfs 使用。
```

---

## 14. 必须记牢的短句

```text
容器共享宿主机内核。
容器里的 / 来自运行时准备的 rootfs。
overlayfs 把 lowerdir 和 upperdir 合并成容器看到的视图。
lowerdir 是只读镜像层。
upperdir 是容器可写层。
workdir 是 overlayfs 工作目录。
新建文件写入 upperdir。
修改 lowerdir 文件会 copy-up 到 upperdir。
删除 lowerdir 文件会在 upperdir 创建 whiteout。
c 0,0 可以表示 overlayfs whiteout。
containerd snapshotter 负责管理镜像层和容器可写层，准备 rootfs snapshot。
runc 最终使用这个 rootfs 启动容器进程。
```

---

## 15. 清理实验资源

如果实验 Pod 不再使用：

```bash
kubectl delete pod rootfs-demo
```

---

## 16. 下一课预告

下一课建议进入：

```text
containerd / runc / OCI
```

重点问题：

```text
containerd、containerd-shim、runc 分别负责什么？
OCI runtime spec 是什么？
config.json 里如何描述 namespaces、mounts、cgroups、rootfs？
如何从一个运行中容器找到对应的 runtime bundle？
```

---
title: "容器底层（六）：rootfs 小笔记"
date: 2026-06-24T10:30:00+08:00
draft: false
tags: ["Kubernetes", "Linux", "rootfs", "overlayfs", "容器", "mount", "笔记"]
categories: ["云原生", "容器底层"]
series: ["容器底层原理"]
summary: "containerd bundle 中 rootfs 与 work 的区别，overlayfs 挂载与 bundle/rootfs 的关系，以及文件系统、挂载、overlayfs、overlay 等概念梳理。"
weight: 6
---

## 1. bundle 中的 `rootfs` 与 `work` 不是一回事

在 runtime bundle 目录：

```text
/run/containerd/io.containerd.runtime.v2.task/k8s.io/<container-id>/
```

中会看到：

```text
rootfs
work -> /var/lib/containerd/io.containerd.runtime.v2.task/k8s.io/<container-id>
```

它们不是一回事。

```text
rootfs：
OCI bundle 里的容器根文件系统路径。
config.json 里 root.path = "rootfs"。
runc 会把它作为容器里的 /。

work：
containerd runtime task 的工作目录软链接。
它是 containerd/shim/runc 管理 task 用的，不是容器里的 /。
```

同时还要区分 overlayfs 的 `workdir`：

```text
overlayfs workdir：
overlayfs 内部工作目录，通常位于
/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/<id>/work
```

所以三者是：

```text
OCI rootfs        -> 容器看到的 /
runtime work      -> containerd task 工作目录
overlayfs workdir -> overlayfs 内部工作目录
```

---

## 2. overlayfs 合并结果通常挂载到 bundle/rootfs

关系是：

```text
lowerdir + upperdir + workdir
  -> overlayfs mount
  -> 挂载到 /run/containerd/.../<container-id>/rootfs
  -> config.json root.path = "rootfs"
  -> runc 把这个 rootfs 作为容器里的 /
```

验证命令：

```bash
findmnt "$BUNDLE/rootfs"
mount | grep "$BUNDLE/rootfs"
sudo stat -f -c %T "$BUNDLE/rootfs"
sudo df -T "$BUNDLE/rootfs"
```

输出会证明：

```text
$BUNDLE/rootfs 是 overlay 类型的挂载点。
```

尤其类似这条：

```text
overlay on /run/containerd/.../<container-id>/rootfs type overlay (
  lowerdir=...,
  upperdir=...,
  workdir=...
)
```

说明：

```text
多个 lowerdir：镜像只读层
upperdir：容器可写层
workdir：overlayfs 工作目录
挂载点：bundle/rootfs
```

---

## 3. 文件系统、挂载、mount、overlayfs、overlay 的关系

核心关系：

```text
文件系统：
组织和访问文件的一套机制，比如 ext4、xfs、tmpfs、procfs、sysfs、overlayfs。

挂载 / mount：
把某个文件系统接到 Linux 唯一目录树上的某个目录。

挂载点：
文件系统接入的位置，比如 /data 或 containerd 的 bundle/rootfs。

mount 命令：
查看或执行挂载操作的命令。

overlayfs：
Linux 内核提供的一种特殊文件系统，用来把 lowerdir 和 upperdir 合并成一个视图。

overlay：
mount 里显示/使用的 overlayfs 文件系统类型名。
```

一句话：

```text
文件系统是一种机制；
mount 是把文件系统接到目录树上的动作；
overlayfs 是一种文件系统；
overlay 是它在挂载信息里的类型名；
容器用 overlayfs 把镜像层和可写层合并后挂载到 rootfs。
```

---

## 4. workdir 是 overlayfs 自己用的，不是容器直接用的

```text
workdir 是 overlayfs 内部工作目录，
不是给容器进程直接使用的目录。
```

容器看到的是：

```text
merged 视图，也就是 bundle/rootfs
```

容器写文件时，表面是在写：

```text
/tmp/a.txt
/etc/profile
```

实际变化主要落到：

```text
upperdir
```

而：

```text
workdir
```

用于 overlayfs 内部操作，比如：

```text
copy-up
rename
原子操作过程中的临时状态
维护 overlayfs 自己的工作数据
```

例如：

```text
upperdir=/var/lib/containerd/.../snapshots/15284/fs
workdir=/var/lib/containerd/.../snapshots/15284/work
```

含义是：

```text
fs：
容器可写层

work：
overlayfs 内部工作目录
```

---

## 5. 最终模型

```text
镜像只读层 lowerdir
        +
容器可写层 upperdir
        +
overlayfs 内部 workdir
        |
        v
overlayfs merged 视图
        |
        v
挂载到 OCI bundle/rootfs
        |
        v
runc 根据 config.json root.path="rootfs"
把它作为容器里的 /
```

最短记忆版：

```text
容器使用 rootfs。
rootfs 是 overlayfs merged 视图的挂载点。
容器写入落到 upperdir。
workdir 是 overlayfs 内部干活用的。
lowerdir 是镜像只读层。
```

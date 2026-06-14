---
title: "K8s 节点排障（六）：Linux 节点排障速查表"
date: 2026-06-14T10:30:00+08:00
draft: false
tags: ["Kubernetes", "Linux", "排障", "速查表", "journalctl", "cgroup", "iptables"]
categories: ["云原生", "Linux 排障"]
series: ["K8s 节点排障实战"]
summary: "Linux 节点排障速查表，覆盖 Pod 定位、/proc 体检、CPU、内存、磁盘、网络、DNS、Service、日志等常用命令与判断要点"
weight: 6
---

## 1. Pod 基础定位

```bash
kubectl get pod -A -o wide
kubectl describe pod <pod>
kubectl get events -A --sort-by=.lastTimestamp | tail -30
```

看：

```text
Pod 在哪个 Node
Pod IP 是多少
状态是 Running / Pending / OOMKilled / ImagePullBackOff
Events 中 kubelet/scheduler 报了什么
```

## 2. Pod 到宿主机 PID

在 Pod 所在节点：

```bash
sudo crictl ps | grep <pod-name>
sudo crictl inspect <container-id> | grep -E '"pid"|"name"'
ps -fp <pid>
pstree -aps <pid>
```

记忆：

```text
容器内 PID 1
宿主机 PID 是 crictl inspect 看到的真实 pid
```

## 3. /proc/\<pid\> 进程体检

```bash
sudo cat /proc/<pid>/status | grep -E 'Name|State|Pid|PPid|Threads|VmRSS|VmSize|RssAnon|RssFile|ctxt'
sudo tr '\0' ' ' < /proc/<pid>/cmdline
sudo cat /proc/<pid>/cgroup
sudo cat /proc/<pid>/limits | grep 'Max open files'
sudo ls /proc/<pid>/fd | wc -l
```

常见状态：

```text
R: 正在运行或等待 CPU
S: 正常睡眠等待事件
D: 不可中断睡眠，优先怀疑 IO/内核路径
Z: 僵尸进程
```

## 4. CPU 与 Load

```bash
uptime
top
ps -eo pid,ppid,stat,comm,%cpu,%mem --sort=-%cpu | head -15
ps -eo pid,ppid,stat,comm,wchan:32 | awk '$3 ~ /D|R/ {print}' | head -30
top -H -p <pid>
ps -L -p <pid> -o pid,tid,stat,comm,%cpu
```

判断：

```text
load average 不是 CPU 使用率
load = R 状态 + D 状态任务数平均值
R 多：CPU 调度压力
D 多：IO/内核路径阻塞
```

## 5. CPU Throttling

```bash
sudo cat /proc/<pid>/cgroup
sudo cat /sys/fs/cgroup/<cgroup-path>/cpu.max
sudo cat /sys/fs/cgroup/<cgroup-path>/cpu.stat
```

重点字段：

```text
cpu.max:
quota period，例如 20000 100000 = 200m

cpu.stat:
nr_periods
nr_throttled
throttled_usec
```

判断：

```text
nr_throttled / throttled_usec 持续增长，说明容器被 CPU limit 限流。
节点 CPU 没满，不代表 Pod 没有被 throttling。
```

## 6. 内存观察

```bash
free -h
cat /proc/meminfo | head -30
ps -eo pid,ppid,stat,comm,rss,vsz,%mem --sort=-rss | head -15
```

记忆：

```text
free 少不等于内存不足
available 更适合判断可用内存
page cache 是正常缓存，不是天然泄漏
VmRSS 是单进程物理内存视角
memory.current 是 cgroup/container 视角
```

## 7. Memory cgroup 与 OOMKilled

```bash
kubectl describe pod <pod>
sudo dmesg -T | grep -i -E 'oom|killed process|memory cgroup' | tail -50
sudo journalctl -k --since "1 hour ago" --no-pager | grep -i -E 'oom|killed process|memory cgroup'
```

如果容器还在：

```bash
sudo cat /proc/<pid>/cgroup
sudo cat /sys/fs/cgroup/<cgroup-path>/memory.max
sudo cat /sys/fs/cgroup/<cgroup-path>/memory.current
sudo cat /sys/fs/cgroup/<cgroup-path>/memory.events
```

判断：

```text
constraint=CONSTRAINT_MEMCG
Memory cgroup out of memory
=> cgroup OOM

memory.events:
oom
oom_kill
```

链路：

```text
memory limit -> memory.max
memory.current 增长 -> 超过 memory.max
内核 cgroup OOM -> kill 进程
kubelet 上报 OOMKilled / Exit Code 137
```

## 8. 磁盘与 inode

```bash
df -h
df -i
sudo du -xh --max-depth=1 /var | sort -h
sudo du -xh --max-depth=1 /var/log | sort -h
sudo du -xh --max-depth=1 /var/lib | sort -h
sudo du -sh /var/log/containers /var/log/pods 2>/dev/null
sudo du -sh /var/lib/containerd /var/lib/kubelet 2>/dev/null
```

判断：

```text
df -h: 块空间
df -i: inode
/var/log/journal: systemd journal
/var/log/pods: 容器真实日志
/var/log/containers: 指向 /var/log/pods 的软链接
/var/lib/containerd: 镜像、快照、容器层数据
```

删除大文件后空间不释放：

```bash
sudo lsof | grep deleted
sudo ls -l /proc/<pid>/fd | grep deleted
```

## 9. 文件句柄 fd

```bash
sudo cat /proc/<pid>/limits | grep 'Max open files'
sudo ls /proc/<pid>/fd | wc -l
sudo ls -l /proc/<pid>/fd | head
```

典型报错：

```text
too many open files
```

常见原因：

```text
socket 未关闭
HTTP response body 未 close
文件未 close
连接池失控
日志 fd 泄漏
```

## 10. 节点网络

```bash
ip -br addr
ip route
ip route get <target-ip>
sudo ss -lntup
ss -ant
```

Calico 当前集群：

```text
Node IP: 192.168.59.x，绑定 ens33
Pod IP: 10.244.x.x
本节点 Pod: dev cali*
跨节点 Pod: dev tunl0 via 对端 Node IP
BGP: TCP 179
```

## 11. DNS

Pod 内：

```bash
cat /etc/resolv.conf
nslookup <svc>.<namespace>.svc.cluster.local
getent hosts <svc>
```

CoreDNS：

```bash
kubectl get svc -n kube-system kube-dns -o wide
kubectl get pod -n kube-system -l k8s-app=kube-dns -o wide
```

记忆：

```text
Pod nameserver 通常是 kube-dns Service IP
DNS 排障不能只靠一个工具
nslookup / getent / wget 解析路径可能不同
```

## 12. Service / Endpoint / kube-proxy

```bash
kubectl get svc -A -o wide
kubectl get endpoints <svc> -o wide
kubectl get endpointslice -l kubernetes.io/service-name=<svc> -o wide
```

iptables 模式：

```bash
sudo iptables-save | grep <service-ip>
sudo iptables-save | grep <endpoint-ip>
```

链路：

```text
ClusterIP:Port
  -> KUBE-SERVICES
  -> KUBE-SVC-*
  -> KUBE-SEP-*
  -> DNAT 到 PodIP:Port
```

记忆：

```text
ClusterIP 不一定出现在 ip addr
它是 kube-proxy 规则里的虚拟 IP
Service 到 Endpoint 的关键动作是 DNAT
```

## 13. 日志与事件

```bash
kubectl describe pod <pod>
kubectl get events -A --sort-by=.lastTimestamp | tail -30
sudo journalctl -u kubelet --since "1 hour ago" --no-pager | tail -100
sudo journalctl -u containerd --since "1 hour ago" --no-pager | tail -100
sudo journalctl -k --since "1 hour ago" --no-pager | tail -100
```

过滤：

```bash
sudo journalctl -u kubelet --since "1 hour ago" --no-pager | grep -i -E 'error|fail|oom|kill|evict|image|pull|container'
sudo journalctl -k --since "1 hour ago" --no-pager | grep -i -E 'oom|killed process|memory cgroup|disk|ext4|xfs|io error'
```

## 14. ImagePullBackOff

```bash
kubectl describe pod <pod>
sudo journalctl -u kubelet --since "10 minutes ago" --no-pager | grep -i -E 'image|pull|failed|<pod>|<image>'
```

分类：

```text
not found: 镜像名或 tag 不存在
authentication required / unauthorized: imagePullSecrets 或仓库权限
i/o timeout / TLS handshake timeout: 网络或仓库连通性
x509 certificate signed by unknown authority: 证书问题
no such host: DNS 问题
connection refused: 仓库服务或代理问题
```

## 15. 第一阶段总链路

```text
K8s 现象
  -> kubectl describe / events
  -> 找 Node
  -> kubelet/containerd 日志
  -> /proc/<pid>
  -> /sys/fs/cgroup
  -> dmesg / journalctl -k
  -> Linux 证据闭环
```

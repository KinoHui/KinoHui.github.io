---
title: "GPU 与 K8s 实操环境搭建笔记"
date: 2026-06-15T17:30:00+08:00
draft: false
tags: ["GPU", "Kubernetes", "WSL2", "Docker", "minikube", "CUDA", "NVIDIA"]
categories: ["云原生", "实战教程"]
series: ["GPU K8s 环境搭建"]
summary: "在一台 Windows 10 + WSL2 + RTX 3060 笔记本上，打通从本机 GPU、WSL、Docker GPU 容器到 minikube 单节点 K8s GPU Pod 的完整链路，记录每一步的实操命令、输出和排障思路。"
weight: 1
---

> 目标：在一台 Windows 10 + WSL2 + RTX 3060 笔记本上，打通从本机 GPU、WSL、Docker GPU 容器，到 minikube 单节点 K8s GPU Pod 的完整链路。

## 1. 当前实验环境

本次实操环境：

```text
宿主系统：Windows 10
Linux 环境：WSL2 Ubuntu 24.04
GPU：NVIDIA GeForce RTX 3060 Laptop GPU
显存：6GB
Windows NVIDIA Driver：581.95
Driver 支持 CUDA：13.0
WSL Kernel：6.18.33.1-microsoft-standard-WSL2
Python：3.12
pip：24.0
Docker Engine：29.4.1
NVIDIA Container Toolkit：1.19.1
minikube：v1.38.1
Kubernetes：v1.35.1
```

本次最终打通的链路：

```text
Windows RTX 3060
  ↓
WSL2
  ↓
Docker + NVIDIA Container Toolkit
  ↓
minikube 节点容器
  ↓
NVIDIA Device Plugin
  ↓
Kubernetes nvidia.com/gpu
  ↓
GPU Pod
```

## 2. Windows 下验证 GPU

在 Windows PowerShell 中执行：

```powershell
nvidia-smi
```

实际输出中的关键信息：

```text
NVIDIA-SMI 581.95
Driver Version: 581.95
CUDA Version: 13.0
GPU: NVIDIA GeForce RTX 3060 Laptop GPU
Memory-Usage: 1086MiB / 6144MiB
GPU-Util: 2%
Driver-Model: WDDM
MIG M.: N/A
ECC: N/A
```

结论：

```text
Windows 宿主机可以识别 RTX 3060
NVIDIA 驱动正常
GPU 当前基本空闲
```

注意点：

```text
CUDA Version: 13.0
```

这里表示当前 NVIDIA Driver 最高支持 CUDA 13.0 运行时兼容能力，不代表系统一定安装了完整 CUDA Toolkit。

`Memory-Usage` 有 1GB 左右占用是正常现象，Windows 桌面、浏览器、壁纸、Codex、Steam 等图形程序都会占用显存。

## 3. WSL2 下验证 GPU

进入 WSL：

```powershell
wsl
```

在 WSL Ubuntu 中执行：

```bash
nvidia-smi
```

实际输出中的关键信息：

```text
NVIDIA-SMI 580.112
Driver Version: 581.95
CUDA Version: 13.0
GPU: NVIDIA GeForce RTX 3060 Laptop GPU
Memory-Usage: 866MiB / 6144MiB
GPU-Util: 1%
Process: /Xwayland
```

结论：

```text
WSL2 可以访问 Windows NVIDIA Driver 提供的 GPU 能力
WSL2 GPU 透传正常
```

说明：

```text
WSL 中 NVIDIA-SMI 工具版本和 Windows 中显示的版本可能不同
底层 Driver Version 仍然来自 Windows 宿主机
```

`/Xwayland` 出现在进程列表中是正常现象，说明 WSL 图形子系统正在使用 GPU。

## 4. 准备 Python 虚拟环境

检查 Python 和 pip：

```bash
python3 --version
pip3 --version
python3 -m pip --version
```

实际 pip 输出：

```text
pip 24.0 from /usr/lib/python3/dist-packages/pip (python 3.12)
```

安装 venv 支持：

```bash
sudo apt install -y python3-venv
```

创建 GPU 实验虚拟环境：

```bash
python3 -m venv ~/venvs/gpu-lab
source ~/venvs/gpu-lab/bin/activate
```

激活成功后，shell 前面会出现：

```text
(gpu-lab)
```

升级虚拟环境中的 pip：

```bash
python -m pip install --upgrade pip
```

## 5. 安装并验证 PyTorch CUDA

安装 PyTorch CUDA 12.8 版：

```bash
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu128
```

说明：

```text
当前 Driver 支持 CUDA 13.0
PyTorch wheel 使用 CUDA Runtime 12.8
NVIDIA Driver 通常向下兼容较低版本 CUDA Runtime
```

编写测试文件 `gpu-test.py`：

```python
import torch

print("torch_version:", torch.__version__)
print("cuda_available:", torch.cuda.is_available())
print("cuda_runtime:", torch.version.cuda)
print("device_count:", torch.cuda.device_count())

if torch.cuda.is_available():
    print("device_name:", torch.cuda.get_device_name(0))
    props = torch.cuda.get_device_properties(0)
    print("memory_gb:", round(props.total_memory / 1024**3, 2))
    print("compute_capability:", f"{props.major}.{props.minor}")

    x = torch.randn(4096, 4096, device="cuda")
    y = x @ x
    torch.cuda.synchronize()
    print("matmul_ok:", tuple(y.shape))
```

运行：

```bash
python gpu-test.py
```

实际输出：

```text
torch_version: 2.11.0+cu128
cuda_available: True
cuda_runtime: 12.8
device_count: 1
device_name: NVIDIA GeForce RTX 3060 Laptop GPU
memory_gb: 6.0
compute_capability: 8.6
matmul_ok: (4096, 4096)
```

结论：

```text
PyTorch 可以识别 GPU
CUDA Runtime 12.8 可用
RTX 3060 可以真实执行 CUDA 计算
```

观察 GPU 使用率：

```bash
watch -n 1 nvidia-smi
```

运行 PyTorch 矩阵乘法时，可以看到：

```text
显存上涨
GPU-Util 上涨
功率上涨
```

这说明计算真实发生在 GPU 上。

核心判断：

```text
torch.cuda.is_available() = 框架能识别 GPU
显存上涨 = 数据或模型进入 GPU
GPU-Util 上涨 = GPU 正在计算
功率上涨 = GPU 负载确实变重
```

`compute_capability: 8.6` 对应 RTX 3060 的 Ampere 架构，很多 CUDA 库中会写成 `sm_86`。

## 6. 检查 Docker 环境

虚拟环境不用退出也可以执行 Docker 命令。`(gpu-lab)` 只影响当前 shell 的 `python`/`pip`，不影响 Docker。

检查 Docker：

```bash
docker --version
docker info
```

实际输出中的关键信息：

```text
Docker version 29.4.1
Server Version: 29.4.1
Storage Driver: overlayfs
Cgroup Driver: systemd
Cgroup Version: 2
Operating System: Ubuntu 24.04.4 LTS
Kernel Version: 6.18.33.1-microsoft-standard-WSL2
Architecture: x86_64
Docker Root Dir: /var/lib/docker
```

一开始 Docker runtime 信息中没有看到 `nvidia`：

```text
Runtimes: io.containerd.runc.v2 runc
Default Runtime: runc
CDI spec directories:
  /etc/cdi
  /var/run/cdi
```

这不一定代表 GPU 不能用，但需要实测。

## 7. 首次运行 Docker GPU 容器失败

执行：

```bash
docker run --rm --gpus all nvidia/cuda:12.8.0-base-ubuntu24.04 nvidia-smi
```

镜像可以正常拉取：

```text
Unable to find image 'nvidia/cuda:12.8.0-base-ubuntu24.04' locally
Pull complete
Status: Downloaded newer image for nvidia/cuda:12.8.0-base-ubuntu24.04
```

但容器启动失败：

```text
docker: Error response from daemon: failed to discover GPU vendor from CDI: no known GPU vendor found
```

排查：

```bash
which nvidia-ctk
nvidia-ctk --version
ls -l /etc/cdi
ls -l /var/run/cdi
```

当时输出：

```text
nvidia-ctk: command not found
ls: cannot access '/etc/cdi': No such file or directory
ls: cannot access '/var/run/cdi': No such file or directory
```

结论：

```text
WSL 能看到 GPU
Docker 能运行
CUDA 镜像能拉取
但 Docker 缺少 NVIDIA Container Toolkit / GPU 注入配置
```

关键知识：

```text
nvidia-smi 在 WSL 可用
≠
Docker 容器自动可用 GPU

容器要用 GPU，需要 NVIDIA Container Toolkit 把 GPU 设备和驱动库注入容器
```

## 8. 安装 NVIDIA Container Toolkit

建议安装系统包前可以退出 Python 虚拟环境，避免混淆：

```bash
deactivate
```

安装基础工具：

```bash
sudo apt-get update
sudo apt-get install -y --no-install-recommends ca-certificates curl gnupg2
```

配置 NVIDIA Container Toolkit apt 源：

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

安装：

```bash
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
```

> **若 `nvidia-container-toolkit` 安装失败**
>
> 情况：
>
> ```text
> hui@G15-r0b:~/venvs/gpu-lab$ sudo apt-get install -y nvidia-container-toolkit
> Reading package lists... Done
> Building dependency tree... Done
> Reading state information... Done
> E: Unable to locate package nvidia-container-toolkit
> ```
>
> 按照以下步骤配置：
>
> ```bash
> rm -f /tmp/nvidia-gpgkey
> curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey -o /tmp/nvidia-gpgkey
> ls -lh /tmp/nvidia-gpgkey
>
> sudo gpg --dearmor --yes \
>   -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
>   /tmp/nvidia-gpgkey
>
> curl -fsSL https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
>   -o /tmp/nvidia-container-toolkit.list
> cat /tmp/nvidia-container-toolkit.list
>
> sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
>   /tmp/nvidia-container-toolkit.list \
>   | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
>
> sudo apt-get update
> apt-cache policy nvidia-container-toolkit
> sudo apt-get install -y nvidia-container-toolkit
> nvidia-container-cli --version
> ```

配置 Docker runtime：

```bash
sudo nvidia-ctk runtime configure --runtime=docker
```

实际输出：

```text
INFO[0000] Loading config from /etc/docker/daemon.json
INFO[0000] Wrote updated config to /etc/docker/daemon.json
INFO[0000] It is recommended that docker daemon be restarted.
```

重启 Docker：

```bash
sudo systemctl restart docker
```

如果 WSL 环境中 `systemctl` 不可用，可改用：

```bash
sudo service docker restart
```

验证：

```bash
which nvidia-ctk
nvidia-ctk --version
```

实际输出：

```text
/usr/bin/nvidia-ctk
NVIDIA Container Toolkit CLI version 1.19.1
commit: 09ceee5dde66ba9ce25c7cc69b1ebd5e6e3266fa
```

## 9. Docker 中验证 nvidia-smi

再次执行：

```bash
docker run --rm --gpus all nvidia/cuda:12.8.0-base-ubuntu24.04 nvidia-smi
```

实际输出中的关键信息：

```text
NVIDIA-SMI 580.112
Driver Version: 581.95
CUDA Version: 13.0
GPU: NVIDIA GeForce RTX 3060 Laptop GPU
Memory-Usage: 946MiB / 6144MiB
GPU-Util: 8%
```

结论：

```text
Docker 容器可以访问 GPU
NVIDIA Container Toolkit 配置成功
```

此时链路：

```text
Windows NVIDIA Driver
  ↓
WSL2
  ↓
Docker Engine
  ↓
NVIDIA Container Toolkit
  ↓
GPU 容器
```

## 10. Docker 中验证 PyTorch CUDA 计算

执行 PyTorch GPU 容器测试：

```bash
docker run --rm --gpus all --ipc=host \
  pytorch/pytorch:2.6.0-cuda12.4-cudnn9-runtime \
  python -c "import torch; print(torch.__version__); print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0)); x=torch.randn(4096,4096,device='cuda'); y=x@x; torch.cuda.synchronize(); print(y.shape)"
```

首次运行会拉取镜像：

```text
Unable to find image 'pytorch/pytorch:2.6.0-cuda12.4-cudnn9-runtime' locally
Pull complete
Status: Downloaded newer image for pytorch/pytorch:2.6.0-cuda12.4-cudnn9-runtime
```

实际输出：

```text
2.6.0+cu124
True
NVIDIA GeForce RTX 3060 Laptop GPU
torch.Size([4096, 4096])
```

结论：

```text
容器不只是能看到 GPU
容器里的 PyTorch 也能真实使用 GPU 做 CUDA 计算
```

排障判断：

```text
nvidia-smi 成功，但 torch.cuda.is_available() 是 False
=> 容器能看到 GPU，但框架/CUDA/PyTorch 镜像可能有问题

nvidia-smi 都失败
=> Docker GPU runtime / NVIDIA Container Toolkit / 驱动注入层有问题
```

## 11. Kubernetes 工具检查

检查 `kubectl`、`kind`、`minikube`：

```bash
kubectl version --client
kind version
minikube version
```

实际输出：

```text
kubectl Client Version: v1.34.5+k3s1
Kustomize Version: v5.7.1
kind v0.31.0 go1.25.3 linux/amd64
minikube: command not found
```

检查 kubectl 上下文：

```bash
kubectl config get-contexts
kubectl get nodes -o wide
kind get clusters
```

实际上下文：

```text
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
          devk8s                        devk8s       devk8s
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin
```

当时 `kubectl get nodes` 连的是 VMware 集群：

```text
https://192.168.59.134:6443
context deadline exceeded
```

kind 集群：

```text
No kind clusters found.
```

结论：

```text
kubectl 当前指向 VMware 中的集群，但它当时不可达
为了避免混淆，后续新建独立的 minikube gpu-lab 集群
```

## 12. 安装 minikube

下载并安装 minikube：

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube version
```

实际输出：

```text
minikube version: v1.38.1
commit: c93a4cb9311efc66b90d33ea03f75f2c4120e9b0
```

## 13. 启动 minikube GPU 实验集群

使用 Docker driver，并开启 GPU：

```bash
minikube start \
  --driver=docker \
  --container-runtime=docker \
  --gpus=all \
  --memory=4096 \
  --cpus=4 \
  --profile=gpu-lab
```

查看节点：

```bash
kubectl get nodes -o wide
```

实际输出：

```text
NAME      STATUS   ROLES           AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION                      CONTAINER-RUNTIME
gpu-lab   Ready    control-plane   32s   v1.35.1   192.168.49.2   <none>        Debian GNU/Linux 12 (bookworm)   6.18.33.1-microsoft-standard-WSL2   docker://29.2.1
```

结论：

```text
minikube 单节点 K8s 集群启动成功
节点名为 gpu-lab
容器运行时为 Docker
```

## 14. 验证 minikube 节点容器能访问 GPU

执行：

```bash
minikube ssh -p gpu-lab -- nvidia-smi
```

实际输出中的关键信息：

```text
NVIDIA-SMI 580.112
Driver Version: 581.95
CUDA Version: 13.0
GPU: NVIDIA GeForce RTX 3060 Laptop GPU
Memory-Usage: 910MiB / 6144MiB
```

结论：

```text
GPU 已经进入 minikube 节点容器
```

此时链路：

```text
Windows GPU
  ↓
WSL2
  ↓
Docker
  ↓
minikube 节点容器
```

注意：

```text
Docker GPU 容器成功
≠
K8s Pod 自动能用 GPU

还需要 NVIDIA Device Plugin 把 GPU 注册给 kubelet
```

## 15. 检查 Kubernetes GPU 资源注册

查看节点 Capacity 和 Allocatable：

```bash
kubectl describe node gpu-lab | grep -A8 -E "Capacity|Allocatable"
```

实际输出：

```text
Capacity:
  cpu:                16
  ephemeral-storage:  1055762868Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             8052368Ki
  nvidia.com/gpu:     1
  pods:               110
Allocatable:
  cpu:                16
  ephemeral-storage:  1055762868Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             8052368Ki
  nvidia.com/gpu:     1
  pods:               110
```

结论：

```text
Kubernetes 节点已经注册 nvidia.com/gpu: 1
```

说明：

```text
nvidia.com/gpu 是 NVIDIA Device Plugin 注册给 Kubernetes 的扩展资源
K8s 原生并不会自动识别 GPU
```

## 16. NVIDIA Device Plugin 状态

尝试安装 NVIDIA Device Plugin：

```bash
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.17.1/deployments/static/nvidia-device-plugin.yml
```

实际输出：

```text
Error from server (AlreadyExists): error when creating "...": daemonsets.apps "nvidia-device-plugin-daemonset" already exists
```

说明：

```text
Device Plugin 已经存在，不需要重复安装
```

查看插件 Pod：

```bash
kubectl -n kube-system get pods | grep nvidia
```

实际输出：

```text
nvidia-device-plugin-daemonset-g645n   1/1     Running   0   7m19s
```

查看日志：

```bash
kubectl -n kube-system logs -l name=nvidia-device-plugin-ds
```

关键日志：

```text
I0615 07:01:06.740087       1 server.go:197] Starting GRPC server for 'nvidia.com/gpu'
I0615 07:01:06.741328       1 server.go:141] Starting to serve 'nvidia.com/gpu' on /var/lib/kubelet/device-plugins/nvidia-gpu.sock
I0615 07:01:06.743723       1 server.go:148] Registered device plugin for 'nvidia.com/gpu' with Kubelet
```

结论：

```text
NVIDIA Device Plugin 已经通过 kubelet device plugin 机制注册 GPU
K8s 节点出现 nvidia.com/gpu 是这个插件工作的结果
```

## 17. 创建 K8s GPU 测试 Pod

创建 `gpu-test.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-test
spec:
  restartPolicy: Never
  containers:
    - name: cuda
      image: nvidia/cuda:12.8.0-base-ubuntu24.04
      command: ["nvidia-smi"]
      resources:
        limits:
          nvidia.com/gpu: 1
```

应用：

```bash
kubectl apply -f gpu-test.yaml
```

实际输出：

```text
pod/gpu-test created
```

查看 Pod：

```bash
kubectl get pod gpu-test -o wide
```

实际过程：

```text
gpu-test   0/1   ContainerCreating   0   53s    <none>      gpu-lab
gpu-test   0/1   Completed           0   98s    10.244.0.5  gpu-lab
```

说明：

```text
这个 Pod 只执行 nvidia-smi
命令执行完成后 Pod 状态变成 Completed 是正常的
```

查看日志：

```bash
kubectl logs gpu-test
```

实际输出中的关键信息：

```text
NVIDIA-SMI 580.112
Driver Version: 581.95
CUDA Version: 13.0
GPU: NVIDIA GeForce RTX 3060 Laptop GPU
Memory-Usage: 921MiB / 6144MiB
GPU-Util: 1%
```

结论：

```text
Kubernetes 成功调度 GPU Pod
Pod 内部可以访问 RTX 3060
```

## 18. 本次排障分层总结

GPU/K8s 环境可以按下面顺序分层排查：

```text
1. Windows nvidia-smi 能否看到 GPU？
2. WSL nvidia-smi 能否看到 GPU？
3. Docker --gpus all 能否看到 GPU？
4. minikube 节点中 nvidia-smi 是否成功？
5. K8s 节点是否注册 nvidia.com/gpu？
6. Pod 申请 nvidia.com/gpu 后是否能运行？
7. Pod 日志中是否能看到 nvidia-smi 或框架 CUDA 计算成功？
```

本次状态：

```text
[完成] Windows nvidia-smi
[完成] WSL2 nvidia-smi
[完成] WSL2 PyTorch CUDA 计算
[完成] Docker GPU 容器 nvidia-smi
[完成] Docker PyTorch CUDA 计算
[完成] minikube 节点访问 GPU
[完成] K8s 节点注册 nvidia.com/gpu
[完成] GPU Pod 运行 nvidia-smi
```

## 19. 关键组件理解

### NVIDIA Driver

安装在 Windows 宿主机上，WSL2 通过它获得 GPU 能力。

```text
Driver Version: 581.95
CUDA Version: 13.0
```

`CUDA Version` 是驱动支持的最高 CUDA 运行时能力，不等于已经安装完整 CUDA Toolkit。

### CUDA Runtime

应用或框架实际使用的 CUDA 运行时。

例如：

```text
PyTorch: 2.11.0+cu128 -> CUDA Runtime 12.8
PyTorch Docker: 2.6.0+cu124 -> CUDA Runtime 12.4
```

### NVIDIA Container Toolkit

负责把 GPU 设备和驱动库注入 Docker 容器。

如果缺少它，常见报错：

```text
failed to discover GPU vendor from CDI: no known GPU vendor found
```

### NVIDIA Device Plugin

负责把 GPU 注册给 Kubernetes kubelet。

成功后节点上出现：

```text
nvidia.com/gpu: 1
```

### nvidia.com/gpu

Kubernetes 中的 GPU 扩展资源。

Pod 申请 GPU 时需要写：

```yaml
resources:
  limits:
    nvidia.com/gpu: 1
```

GPU 资源通常按整数卡分配，不像 CPU/内存那样天然支持小数细分。

## 20. 常见问题速查

### WSL 中 nvidia-smi 不可用

可能原因：

```text
Windows NVIDIA Driver 不支持 WSL GPU
WSL2 版本过旧
不是 WSL2
NVIDIA 驱动安装异常
```

先查：

```powershell
nvidia-smi
wsl --status
```

### Docker 中 `--gpus all` 报错

典型报错：

```text
failed to discover GPU vendor from CDI: no known GPU vendor found
```

排查：

```bash
which nvidia-ctk
nvidia-ctk --version
ls -l /etc/cdi
ls -l /var/run/cdi
```

修复方向：

```bash
sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

### nvidia-smi 成功但 PyTorch 不可用

现象：

```text
nvidia-smi 成功
torch.cuda.is_available() = False
```

可能原因：

```text
装了 CPU 版 PyTorch
PyTorch CUDA 版本不合适
镜像或 Python 环境不对
CUDA 库依赖缺失
```

检查：

```bash
python -c "import torch; print(torch.__version__); print(torch.version.cuda); print(torch.cuda.is_available())"
```

### K8s 节点没有 `nvidia.com/gpu`

排查：

```bash
kubectl describe node gpu-lab | grep -A8 -E "Capacity|Allocatable"
kubectl -n kube-system get pods | grep nvidia
kubectl -n kube-system logs -l name=nvidia-device-plugin-ds
```

可能原因：

```text
NVIDIA Device Plugin 没有安装
Device Plugin Pod 没有 Running
minikube 节点容器看不到 GPU
container runtime 没有正确配置 GPU
```

### GPU Pod 执行完变成 Completed

如果 Pod 只执行：

```yaml
command: ["nvidia-smi"]
```

那么命令跑完后变成：

```text
Completed
```

这是正常状态，不是异常。

## 21. 下一节建议

下一节可以做 GPU 独占调度实验：

```text
1. 创建一个长期占用 GPU 的 Pod
2. 再创建第二个同样申请 nvidia.com/gpu: 1 的 Pod
3. 观察第二个 Pod Pending
4. 使用 kubectl describe pod 查看调度失败原因
```

这个实验对应生产环境中非常常见的问题：

```text
为什么 GPU Pod 一直 Pending？
```

它能帮助理解：

```text
GPU 是扩展设备资源
GPU 默认是整数卡分配
单张 GPU 被一个 Pod 占用后，第二个 Pod 无法继续调度
Kubernetes 调度器依赖 nvidia.com/gpu 的 Allocatable 资源做决策
```

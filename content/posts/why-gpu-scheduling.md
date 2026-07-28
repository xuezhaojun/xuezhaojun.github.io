---
title: "K8S GPU多租户 101"
date: 2026-07-22
draft: false
tags: ["ai-infra"]
collections: ["GPU Scheduling"]
weight: 1
summary: "用最基本的case讲明白K8S下的GPU多租户共享问题"
ShowToc: true
TocOpen: true
ShowReadingTime: true
---

本文以内存为例，从调度和运行时两个阶段来讲清楚 K8S 多租户隔离机制，然后看 GPU 显存为什么做不到同样的事，以及 MIG 和 HAMI 两种解决方案。

### 如何实现多租户共享内存

首先,我们来看内存是如何实现多租户共享的。假设我们有一个内存需求为 1 GiB 的 Pod 要调度到某个 node 上,在 Pod spec 里我们会这样声明:

```yaml
resources:
  requests:
    memory: 1Gi
  limits:
    memory: 1Gi
```

这里的 `requests` 和 `limits` 分别服务于两个阶段：

**阶段一：调度——node 上还有没有空闲资源？**

kube-scheduler 做调度决策时看的是 `requests`。每个 node 有一个 `allocatable` 内存总量，减去该 node 上所有 Pod 的 `requests` 之和，就是可分配的内存。如果 `requests.memory: 1Gi`，调度器就会找一个剩余内存 ≥ 1Gi 的 node 来放这个 Pod。

**阶段二：运行时——容器真的不能超限**

调度完成后，kubelet 通过容器运行时将 `limits` 写入 node 内核的 cgroup 配置：

```
$ cat /sys/fs/cgroup/.../memory.max
1073741824  # 1 GiB
```

每个容器在 node 上对应一个 cgroup，`memory.max` 定义了该容器的内存硬上限。容器进程的内存使用超过这个上限时，内核的 cgroup memory controller 会直接 OOM kill。

这样一来，多租户场景下的公平性就有了保障：`limits` + cgroup 确保：**用户 A 的容器配置了 1 GiB 上限，就不可能实际占用 10 GiB 进而挤占同一 node 上其他容器的内存。**

### 为什么这一机制对于 GPU 显存无效

如果有一个 Pod 需要 10 GiB 的 GPU 显存，我们能否像内存那样做类似的限制，并预期容器超过 10 GiB 显存时出现 OOM Kill 呢？

```yaml
resources:
  requests:
    gpu.memory: 10Gi
  limits:
    gpu.memory: 10Gi
```

我们分别看下**调度**和**运行时**：

**阶段一：调度——node 上还有多少 GPU 显存？**

我们先假设扩展出了一个 `gpu.memory` 资源，device plugin 上报 node 的 GPU 显存总量（比如 100 GiB），调度器像内存一样做 `allocatable - requests` 的计算。那么一个 `requests.gpu.memory: 10Gi` 的 Pod，调度器就会找一个剩余显存 ≥ 10 GiB 的 node 来放置它。

**阶段二：运行时——GPU 显存能不能做硬限制？**

内存的分配（`malloc` / `mmap`）必须走内核，cgroup memory controller 可以**拦截**每一次分配，超了就当场拒绝或 kill。

但 GPU 显存的分配（`cudaMalloc`）是进程通过 CUDA driver 直接和 GPU 硬件交互，内核根本不在这个路径上。没有 cgroup 这层拦截点，自然就没有"超了上限就 OOM kill"这种事。

那为什么 Linux 内核不把 GPU 显存也纳入 cgroup 管理？要回答这个问题，需要回溯到计算机架构的源头 -- 冯·诺依曼架构。

冯·诺依曼架构定义了计算机的基本构成：**CPU + 内存 + 总线 + I/O**。CPU 通过内存总线直接控制物理内存，内核跑在 CPU 上，拥有所有页表和控制权。

GPU 不在这张图纸里。GPU 是一台挂在 PCIe 总线上的独立设备——它有自己的计算单元、内存控制器和显存。从 CPU 的视角看，GPU 和网卡、硬盘是同一类东西：外设。

这不是 Linux 设计上的遗漏,是物理拓扑决定的:CPU 的内存控制器只能管插在主板插槽上的内存,管不到 PCIe 另一端一个独立设备上的显存。

这意味着：即使调度层算清楚了（100 GiB 总量，调度了 80 GiB 的 requests），运行时层的限制是缺失的。一个容器声明 `limits.gpu.memory: 10Gi`，`cudaMalloc` 照样可以申请 20 GiB，OS 无法干预。

### 方案一:MIG--硬件隔离

MIG（Multi-Instance GPU）是 GPU 厂商在硬件层面提供的能力，可以将一块物理 GPU 拆分成多个独立的 GPU 实例。每个实例拥有固定的显存和算力，彼此之间硬件隔离——一个实例里的进程无法访问另一个实例的显存。

```
┌──────────────────────────────────────┐
│            一块 A100 (80GB)            │
├──────────┬──────────┬────────────────┤
│ 10 GiB   │ 20 GiB   │     50 GiB     │
│ 实例 1   │ 实例 2   │     实例 3     │
└──────────┴──────────┴────────────────┘
```

这正是我们前面一直在找的东西：GPU 显存的硬隔离。有了 MIG，`nvidia.com/mig-1g.10gb: 1` 的 Pod 只能用 10 GiB 显存，不可能多占。Kubernetes 调度器也能按 MIG 实例数量做精确的调度决策。

但 MIG 有两个代价：

**成本：拆开不如直接买小的。** 一张 80 GiB 的 A100 切成 4 个 20 GiB 实例，和直接买 4 张 20 GiB 的卡相比，前者通常更贵。因为大卡的 HBM 带宽、张量核心等资源在拆分后并不线性对应——你在为一张完整大卡付钱，但只用到了它拆分后的一小部分。MIG 的价值是当你**已经有一张大卡**，想在上面跑多个小任务时才有意义，而不是作为一个省钱的采购策略。

**故障隔离：软件隔离不了硬件故障。** MIG 实例之间的隔离只在显存和算力层面。如果这张 GPU 本身出现硬件故障（掉卡、HBM ECC 报错、电源问题），上面的所有 MIG 实例一起挂。相比之下，多张独立的小卡天然提供物理级别的故障隔离。

### 方案二：HAMI——软件拦截

如果不靠硬件，能不能用软件做到 GPU 显存的硬隔离？

回顾内存 cgroup 的核心做法：**在 `malloc` 的路径上拦截，超限就拒绝**。HAMI（Heterogeneous AI Computing Virtualization Middleware）做的就是这个——在 GPU 的调用路径上拦截。

```
  容器进程
    │
    │  cudaMalloc(...)
    ▼
┌──────────────┐
│  HAMI 拦截层  │  ← 在这里检查:当前容器的 GPU 显存使用量 + 本次申请量 > limit?
│              │     是 → 拒绝
│              │     否 → 放行
└──────┬───────┘
       │
       ▼
  CUDA driver
       │
       ▼
     GPU
```

HAMI 在 GPU 驱动 API 层（CUDA 运行时）插入拦截逻辑：每收到一次 `cudaMalloc` 调用，先检查当前容器的 GPU 显存使用量是否超过了声明的限制。超了就返回错误，不超过就放行。

这和内存 cgroup 的思路一致：**在分配路径上设闸门**。区别在于——内存 cgroup 的闸门在内核，HAMI 的闸门在用户态。

代价也是对应的：每次 GPU 显存分配都要经过一次软件拦截和判断。但和"每次回内核"不同，HAMI 的拦截发生在用户态，不跨 PCIe、不进入内核上下文。对于 GPU 训练/推理这种场景（显存分配集中在启动阶段，之后以计算为主），这个开销通常可以接受。

HAMI 的核心价值在于：**不需要改硬件、不需要改内核，只要在 GPU 驱动层做拦截，就能实现 GPU 显存的限额控制。**

### 总结

我们从内存这一个维度切入，用最基础的 case 展示了 Kubernetes GPU 多租户面临的核心问题：**调度层可扩展，运行时层没有硬隔离。** MIG 和 HAMI 分别从硬件和软件两个方向给出了答案，各有代价。

如果你想看更完整的内容——从 MIG、MPS、时间切片到 DRA、安全隔离、多租户参考架构——推荐继续阅读：

- 中文版：《[GPU 多租户平台](https://jimmysong.io/zh/book/gpu-multi-tenancy-platforms/)》（Jimmy Song）
- 英文版：[GPU-enabled Platforms on Kubernetes](https://www.vcluster.com/gpu-enabled-platforms-on-kubernetes)（vCluster）

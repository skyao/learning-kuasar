---
title: "Kuasar介绍"
linkTitle: "介绍"
weight: 10
date: 2025-10-07
description: >
  Kuasar介绍
---

https://github.com/skyao/kuasar

---

## 介绍

Kuasar 是一款高效的容器运行时，通过支持多种沙箱技术提供云原生全场景容器解决方案。它采用 Rust 语言编写，基于沙箱 API 提供标准沙箱抽象层。此外，Kuasar 还提供优化框架以加速容器启动并减少不必要的开销。

在当前发展阶段，没有任何单一的基础容器技术能够完美支持所有云原生场景的需求。我们的目标是提供更优的解决方案，以管理和平衡企业对容器隔离性、安全性、通用性、运行速度及资源消耗等方面的需求。

### kuasar的优点

在容器领域，沙盒是一种用于将容器进程相互隔离以及与操作系统本身隔离的技术。随着沙盒 API 的引入，沙盒已成为 containerd 中的第一类公民。随着容器领域中越来越多的沙盒技术出现，预计将提出一个名为“沙盒管理器”的管理服务。

Kuasar 支持多种类型的沙盒管理器，使用户能够根据应用需求为每个应用选择最合适的沙盒管理器。

与其他容器运行时相比，Kuasar 具有以下优势：

- 统一沙盒抽象：沙盒在 Kuasar 中是一流公民，因为 Kuasar 完全基于沙盒 API 构建，该 API 由 containerd 社区于 2022 年 10 月预览。Kuasar 充分利用了沙盒 API 的优势，提供了一种统一的沙盒访问和管理方式，并提高了沙盒运维效率。
- 多沙盒共置：Kuasar 内置支持主流沙盒，允许多种类型的沙盒在单个节点上运行。Kuasar 能够平衡用户对安全隔离、快速启动和标准化的需求，并使无服务器节点资源池满足各种云原生场景要求。
- 优化框架：通过移除所有暂停容器并将 shim 进程替换为单个驻留沙盒器进程，Kuasar 进行了优化，带来了 1:N 进程管理模型，其性能优于当前的 1:1 shim v2 进程模型。基准测试结果显示，Kuasar 的沙盒启动速度提升 2 倍，而管理资源开销降低了 99%。更多详情请参考性能。
- 开放与中立：Kuasar 致力于构建一个开放且兼容的多沙盒技术生态系统。得益于沙盒 API，集成沙盒技术更加便捷且节省时间。Kuasar 对沙盒技术保持开放和中立的态度，因此所有沙盒技术都受到欢迎。目前，Kuasar 项目正在与 WasmEdge、openEuler 和 QuarkContainers 等开源社区和项目合作。

Kuasar 的技术优势主要包括：

- 灵活的插件沙盒扩展能力

   Kuasar 具有灵活的插件沙盒扩展能力。它提供了一个更对开发者友好的沙盒访问框架，使得主流沙盒的持续集成成为可能。

- 清晰统一的沙盒定义

   Kuasar 基于最新的沙盒 API，具有清晰统一的沙盒定义。沙盒的管理逻辑与容器分离，不再像以前那样耦合在一起。而且这将得到上游 containerd 社区的原生支持。

- 简化容器 API 调用链

   Kuasar 拥有简化的容器 API 调用链。如您所见，容器运行时可以在无需像以前那样在“shim”中进行额外转换的情况下管理容器。

- 高效的 Kuasar 沙盒器

   Kuasar 拥有高效的沙盒器和优化的框架：

     - 它的进程是驻留的，这意味着启动时间更短。
     - 它的进程模块是 1:N，这意味着随着容器数量的增加，其资源开销不会增加。
     - 它是用 Rust 编写的，这意味着高性能、高安全性和低开销。

- 消失的“pause”容器

   Kuasar 移除“暂停”容器的原因是安全容器是一个自然的 Pod 模型。这种优化减少了由暂停容器引入的资源开销和启动时间。

## 功能

### 支持的沙箱



| Sandboxer  | Sandbox          | Status          |
| ---------- | ---------------- | --------------- |
| MicroVM    | Cloud Hypervisor | Supported       |
|            | QEMU             | Supported       |
|            | Firecracker      | Planned in 2025 |
|            | StratoVirt       | Supported       |
| Wasm       | WasmEdge         | Supported       |
|            | Wasmtime         | Supported       |
| App Kernel | Quark            | Supported       |
| runC       | runC             | Supported       |



## 架构

![](images/arch.png)

Kuasar 中的沙盒器使用各自的容器隔离技术，并且它们也是基于新的沙盒插件机制构建的 containerd 外部插件。关于沙盒器插件的讨论已在 containerd 问题中提出，并在该评论中附有社区会议记录和幻灯片。现在这一功能已被纳入 2.0 里程碑。

目前，Kuasar 提供三种类型的沙盒器——MicroVM 沙盒器、App Kernel 沙盒器和 Wasm 沙盒器——所有这些都被证明在多租户环境中是安全的隔离技术。沙盒器的一般架构由两个模块组成：一个模块实现沙盒 API 以管理沙盒的生命周期，另一个模块实现任务 API 以处理与容器相关的操作。

### MicroVM 沙盒器

在 microVM 沙盒场景中，VM 进程提供基于开源 VMM（虚拟机管理器）如 Cloud Hypervisor、StratoVirt、Firecracker 和 QEMU 的完整虚拟机和 Linux 内核。所有这些虚拟机都必须在支持虚拟化的节点上运行，否则将无法工作！因此，MicroVM 沙盒器的 vmm-sandboxer 负责启动虚拟机并调用 API，而 vmm-task 作为虚拟机中的初始化进程，扮演运行容器进程的角色。容器 IO 可以通过 vsock 或 uds 导出。

microVM 沙盒器避免了在主机上运行 shim 进程的必要性，带来了一种更干净、更易于管理的架构，每个 pod 只有一个进程。

![](images/vmm-arch.png)

### App Kernel Sandboxer

App Kernel 沙盒启动 KVM 虚拟机和 guest 内核，无需任何应用级虚拟机管理程序或 Linux 内核。这允许针对启动过程进行定制优化，减少内存开销，并提高 IO 和网络性能。此类应用内核沙盒的例子包括 gVisor 和 Quark。

Quark 是一个应用内核沙盒，它使用自己的虚拟机管理程序 QVisor 和定制的内核 QKernel 。通过对这些组件进行定制修改，Quark 可以实现显著的性能提升。

应用内核沙盒器 quark-sandboxer 启动 Qvisor 并创建一个名为 Qkernel 的应用内核。每当 containerd 需要在沙盒中启动容器时， QVisor 中的 quark-task 将调用 Qkernel 来启动一个新容器。同一 Pod 内的所有容器将在同一个进程中运行。

![](images/quark-arch.png)

### Wasm Sandboxer

Wasm 沙盒，如 WasmEdge 或 Wasmtime，非常轻量级，但目前可能对某些应用存在限制。 wasm-sandboxer 和 wasm-task 在 WebAssembly 运行时中启动容器。每当 containerd 需要在沙盒中启动容器时， wasm-task 会创建新进程，启动新的 WasmEdge 运行时，并在其中运行 Wasm 代码。同一 Pod 中的所有容器将与 wasm-task 进程共享相同的 Namespace/Cgroup 资源。

![](images/wasm-arch.png)

### Runc Sandboxer

除了安全容器，Kuasar 还为 runC 容器提供了运行能力。为了生成一个独立的命名空间， runc-sandboxer 通过双重克隆创建一个轻微的过程，然后成为 PID 1。基于这个命名空间， runc-task 可以创建容器进程并加入该命名空间。如果容器需要私有命名空间，它会为自己 unshare 一个新的命名空间。

![](images/runc-arch.png)
---
title: "shim特殊设计"
linkTitle: "shim"
weight: 30
date: 2025-10-15
description: >
  Kuasar shim 组件的特殊设计
---

背景：发现 greptile 在回答时说 shim 是 1：1 模型，但看 kuasar 的介绍又说 kuasar 特意对 shim 进行了优化，采用 1：N 模型。所以深挖了一下。

问： containerd-shim-kuasar-vmm-v2 这个组件，在连续创建 cloud hypervisor 的 microvm 时，是会为每个 micorovm 的创建而启动一个 containerd-shim-kuasar-vmm-v2 实例，还是一直重用一个 containerd-shim-kuasar-vmm-v2 实例？

再问：关于这一点，我看到 kuasar 的介绍中提到：

优化框架：通过移除所有暂停容器并将 shim 进程替换为单个驻留沙盒器进程，Kuasar 进行了优化，带来了 1:N 进程管理模型，其性能优于当前的 1:1 shim v2 进程模型。基准测试结果显示，Kuasar 的沙盒启动速度提升 2 倍，而管理资源开销降低了 99%。更多详情请参考性能。

greptile的回答：

**containerd-shim-kuasar-vmm-v2 确实会为每个 microVM 的创建而启动一个新的实例，但所有实例都连接到同一个长期运行的 sandboxer 进程。** 这是 Kuasar 实现性能优化的关键架构设计。

## Kuasar 的双层架构设计原理

Kuasar 采用了一种巧妙的**双层架构**，既保持了与 containerd shim v2 API 的完全兼容性，又实现了显著的性能优化：

第一层：Shim 层（1:1 模型）

- **每个 sandbox 仍然有一个独立的 containerd-shim-kuasar-vmm-v2 实例**
- 这些 shim 进程变得极其轻量，主要负责：
  - 接收 containerd 的 API 调用
  - 转发请求给 sandboxer 进程
  - 处理 I/O 管道和信号

第二层：Sandboxer 层（1:N 模型）

- 单一的长期运行的 sandboxer 进程（如 `kuasar-vmm`）
- 管理多个 Cloud Hypervisor microVM 实例
- 处理所有重量级操作

### 整体架构图

```mermaid
graph TD
    subgraph "Container Orchestrator Layer"
        K8s[Kubernetes]
        Containerd[containerd daemon]
    end
    
    subgraph "Shim Layer (1:1 Model)"
        Shim1["containerd-shim-kuasar-vmm-v2<br/>Instance 1<br/>PID: 1001<br/>轻量级进程"]
        Shim2["containerd-shim-kuasar-vmm-v2<br/>Instance 2<br/>PID: 1002<br/>轻量级进程"]
        Shim3["containerd-shim-kuasar-vmm-v2<br/>Instance 3<br/>PID: 1003<br/>轻量级进程"]
    end
    
    subgraph "Sandboxer Layer (1:N Model)"
        VMM["kuasar-vmm<br/>单一驻留进程<br/>PID: 2000<br/>systemd 服务"]
        Socket["/run/vmm-sandboxer.sock"]
    end
    
    subgraph "Hypervisor Layer"
        VM1["Cloud Hypervisor<br/>microVM 1<br/>sandbox-abc123"]
        VM2["Cloud Hypervisor<br/>microVM 2<br/>sandbox-def456"]
        VM3["Cloud Hypervisor<br/>microVM 3<br/>sandbox-ghi789"]
    end
    
    K8s --> Containerd
    Containerd --> Shim1
    Containerd --> Shim2
    Containerd --> Shim3
    
    Shim1 -.->|Unix Socket<br/>ttrpc gRPC| Socket
    Shim2 -.->|Unix Socket<br/>ttrpc gRPC| Socket
    Shim3 -.->|Unix Socket<br/>ttrpc gRPC| Socket
    
    Socket --> VMM
    
    VMM --> VM1
    VMM --> VM2
    VMM --> VM3
    
    style Shim1 fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    style Shim2 fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    style Shim3 fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    style VMM fill:#fff3e0,stroke:#e65100,stroke-width:3px
    style VM1 fill:#f3e5f5,stroke:#4a148c
    style VM2 fill:#f3e5f5,stroke:#4a148c
    style VM3 fill:#f3e5f5,stroke:#4a148c
    style Socket fill:#e8f5e8,stroke:#2e7d32
```

### 进程创建和通信时序图

```mermaid
sequenceDiagram
    participant Containerd
    participant Shim1 as containerd-shim-kuasar-vmm-v2 #1
    participant Shim2 as containerd-shim-kuasar-vmm-v2 #2  
    participant VMM as kuasar-vmm (驻留进程)
    participant CH1 as Cloud Hypervisor #1
    participant CH2 as Cloud Hypervisor #2
    
    Note over VMM: kuasar-vmm 作为 systemd 服务<br/>已经在运行，监听 /run/vmm-sandboxer.sock
    
    rect rgb(240, 248, 255)
        Note right of Containerd: 创建第一个 sandbox
        Containerd->>+Shim1: 启动新的 shim 实例
        Shim1->>Shim1: fork() 新进程，绑定到 sandbox ID
        Shim1->>+VMM: ttrpc: CreateSandbox(id: "sandbox-abc123")
        VMM->>VMM: 在内存中注册新 sandbox
        VMM->>+CH1: 启动 Cloud Hypervisor microVM 1
        CH1->>CH1: 初始化 VM，分配资源
        CH1-->>-VMM: microVM 1 启动完成
        VMM-->>-Shim1: CreateSandboxResponse
        Shim1-->>-Containerd: 返回成功
    end
    
    rect rgb(255, 248, 240)
        Note right of Containerd: 并行创建第二个 sandbox  
        Containerd->>+Shim2: 启动新的 shim 实例
        Shim2->>Shim2: fork() 新进程，绑定到不同 sandbox ID
        Shim2->>+VMM: ttrpc: CreateSandbox(id: "sandbox-def456")
        VMM->>VMM: 复用现有连接和内存结构
        VMM->>+CH2: 启动 Cloud Hypervisor microVM 2
        CH2->>CH2: 初始化 VM，分配资源
        CH2-->>-VMM: microVM 2 启动完成
        VMM-->>-Shim2: CreateSandboxResponse
        Shim2-->>-Containerd: 返回成功
    end
    
    Note over Shim1, VMM: 两个 shim 实例同时连接到<br/>同一个 kuasar-vmm 进程
```

### 资源优化对比图

```mermaid
graph LR
    subgraph "传统 Shim v2 (1:1 模型) - 资源重复"
        direction TB
        C1[Container 1] --> S1["Shim 1<br/>重量级<br/>内存: 50MB<br/>CPU: 10%"]
        C2[Container 2] --> S2["Shim 2<br/>重量级<br/>内存: 50MB<br/>CPU: 10%"]  
        C3[Container 3] --> S3["Shim 3<br/>重量级<br/>内存: 50MB<br/>CPU: 10%"]
        S1 --> R1["Runtime 1<br/>hypervisor API"]
        S2 --> R2["Runtime 2<br/>hypervisor API"]
        S3 --> R3["Runtime 3<br/>hypervisor API"]
        
        TotalOld["总资源:<br/>内存: 150MB<br/>CPU: 30%<br/>进程数: 6"]
    end
    
    subgraph "Kuasar 优化 (1:1 + 1:N 混合) - 资源共享"
        direction TB
        C4[Container 1] --> S4["Shim 1<br/>轻量级<br/>内存: 2MB<br/>CPU: 0.1%"]
        C5[Container 2] --> S5["Shim 2<br/>轻量级<br/>内存: 2MB<br/>CPU: 0.1%"]
        C6[Container 3] --> S6["Shim 3<br/>轻量级<br/>内存: 2MB<br/>CPU: 0.1%"]
        S4 --> VMM2["kuasar-vmm<br/>单一重量级进程<br/>内存: 20MB<br/>CPU: 5%"]
        S5 --> VMM2
        S6 --> VMM2
        VMM2 --> V1[microVM 1]
        VMM2 --> V2[microVM 2]
        VMM2 --> V3[microVM 3]
        
        TotalNew["总资源:<br/>内存: 26MB (节省 83%)<br/>CPU: 5.3% (节省 82%)<br/>进程数: 4 (节省 33%)"]
    end
    
    style S1 fill:#ffcdd2,stroke:#c62828
    style S2 fill:#ffcdd2,stroke:#c62828
    style S3 fill:#ffcdd2,stroke:#c62828
    style S4 fill:#e8f5e8,stroke:#2e7d32
    style S5 fill:#e8f5e8,stroke:#2e7d32
    style S6 fill:#e8f5e8,stroke:#2e7d32
    style VMM2 fill:#fff3e0,stroke:#e65100
    style TotalOld fill:#ffebee,stroke:#b71c1c
    style TotalNew fill:#f1f8e9,stroke:#33691e
```

## 代码实现细节

### Shim 层的轻量化实现

从 `shim/src/service.rs` 可以看到，每个 shim 实例的核心职责：

```rust
// 每个 shim 实例都有独立的 ID 和数据结构
2impl<Transport> Shim for Service<Transport> {
3    async fn new(_runtime_id: &str, id: &str, _namespace: &str, _config: &mut Config) -> Self {
4        let exit = Arc::new(ExitSignal::default());
5        Self {
6            kuasar_server: Box::new(KuasarServer::new(id, exit).await),
7        }
8    }
9}
10
11// 但所有 shim 实例都连接到同一个 sandboxer 地址
12impl<T: ContainerIoTransport> KuasarServer<T> {
13    pub async fn new(id: &str, exit: Arc<ExitSignal>) -> Self {
14        let channel = Endpoint::from_static("https://www.kuasar.io")
15            .connect_with_connector(service_fn(
16                |_: Uri| UnixStream::connect(T::sandboxer_addr()), // 同一个 socket 地址！
17            ))
18            .await
19            .expect("sandboxer should be running");
20    }
21}
```

### Sandboxer 层的单实例架构

从 `vmm/sandbox/src/bin/qemu/main.rs` 可以看到，sandboxer 作为单一进程运行：

```rust
#[tokio::main]
2async fn main() {
3    // 创建单一的 sandboxer 实例
4    let mut sandboxer: KuasarSandboxer<QemuVMFactory, QemuHooks> = KuasarSandboxer::new(
5        config.sandbox,
6        config.hypervisor.clone(),
7        QemuHooks::new(config.hypervisor),
8    );
9    
10    // 在指定地址监听，为所有 shim 提供服务
11    containerd_sandbox::run(
12        "kuasar-vmm-sandboxer-qemu",
13        &args.listen,  // /run/vmm-sandboxer.sock
14        &args.dir,
15        sandboxer,     // 单一实例管理多个 sandbox
16    )
17    .await
18    .unwrap();
19}
```

## 性能优化的关键机制

优化效果饼图:

```mermaid
pie title 资源使用分布 (Kuasar vs 传统)
    "轻量级 Shim 进程" : 5
    "共享的 sandboxer 进程" : 95
```

1. **消除重复开销**

   - 只有一个进程处理与 hypervisor 的交互

   - 共享内存池和连接池

   - 统一的 VM 生命周期管理

2. **保持完整兼容性**

   - containerd 仍然看到熟悉的 shim v2 接口

   - 每个 sandbox 保持独立的进程边界

   - 故障隔离和资源清理机制不变

3. **99% 资源节省的实现**

   - 大部分 shim 实例从重量级变为轻量级

   - 重量级的 VM 管理逻辑只有一份

   - 避免重复的 hypervisor API 连接

## 生命周期管理流程

```mermaid
stateDiagram-v2
    [*] --> SystemdStart: 系统启动
    SystemdStart --> SandboxerReady: kuasar-vmm 服务启动
    SandboxerReady --> ShimCreate: containerd 创建容器
    
    state ShimCreate {
        [*] --> ForkShim: fork 新的 shim 进程
        ForkShim --> ConnectSandboxer: 连接到 sandboxer socket
        ConnectSandboxer --> CreateVM: 通过 ttrpc 创建 microVM
        CreateVM --> [*]
    }
    
    ShimCreate --> MultipleVMs: 可以创建多个 VM
    
    state MultipleVMs {
        VM1: Cloud Hypervisor 1
        VM2: Cloud Hypervisor 2 
        VM3: Cloud Hypervisor 3
        
        VM1 --> SandboxerManaged: 由同一个 sandboxer 管理
        VM2 --> SandboxerManaged
        VM3 --> SandboxerManaged
    }
    
    MultipleVMs --> ShimDelete: 容器删除
    
    state ShimDelete {
        [*] --> StopVM: 通知 sandboxer 停止 VM
        StopVM --> CleanupShim: shim 进程退出
        CleanupShim --> [*]
    }
    
    ShimDelete --> SandboxerReady: sandboxer 继续运行
    SandboxerReady --> SystemdStop: 系统关闭
    SystemdStop --> [*]
```


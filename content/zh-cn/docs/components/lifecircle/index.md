---
title: "生命周期"
linkTitle: "生命周期"
weight: 20
date: 2025-10-15
description: >
  Kuasar 组件生命周期
---

仅考虑创建 cloud hypervisor microvm 的场景。

## 组件生命周期分析

长时间运行的服务组件（守护进程）

1. **containerd**

    - **类型**: 外部长期运行的系统服务
    - **生命周期**: 系统级守护进程，通过systemd管理
    - **功能**: 容器运行时管理，接收CRI调用

2. **VMM Sandboxer** (`cloud_hypervisor`)

    - **类型**: Kuasar提供的长期运行服务
    - **生命周期**: 通过systemd服务启动并持续运行
    - **服务文件**: `vmm/service/kuasar-vmm.service`
    - 功能:
       - 监听Unix Socket等待沙箱创建请求
       - 管理多个虚拟机实例的生命周期
       - 与containerd shim通信

3. **cloud-hypervisor进程**

    - **类型**: 外部hypervisor守护进程
    - **生命周期**: 每个VM对应一个进程，VM存在期间持续运行
    - **功能**: 实际的虚拟机监控和硬件模拟

4. **vmm-task** (VM内部)

    - **类型**: VM内部的任务服务器
    - **生命周期**: 随VM启动而启动，VM存在期间持续运行
    - **功能**: 在VM内部处理容器操作请求

短时间运行的临时组件

1. **containerd-shim-kuasar-vmm-v2**

    - **类型**:  临时进程
    - 生命周期 :
      - 由containerd为每个Pod启动
      - Pod删除时退出
    - **功能**: 作为containerd和VMM sandboxer之间的桥梁

```mermaid
timeline
    title 组件生命周期时序

    section 系统启动
        系统启动 : containerd启动
                : VMM Sandboxer启动 (通过systemd)

    section Pod创建请求
        请求到达 : containerd接收创建请求
        Shim启动 : containerd启动shim进程
        VM创建 : VMM Sandboxer启动cloud-hypervisor
               : VM启动，vmm-task启动

    section Pod运行期间
        稳定运行 : containerd (持续运行)
                : VMM Sandboxer (持续运行)
                : cloud-hypervisor (持续运行)
                : vmm-task (持续运行)
                : containerd-shim (持续运行)

    section Pod删除
        清理阶段 : containerd-shim退出
                : VM被销毁
                : cloud-hypervisor进程退出
                : vmm-task随VM退出
```

## Cloud Hypervisor 场景

以连续创建并删除3个Cloud Hypervisor microVM为例，各个组件的参与流程和生命周期：

```mermaid
gantt
    title Kuasar Cloud Hypervisor - 3个Pod的组件生命周期
    dateFormat YYYY-MM-DD
    axisFormat %m-%d
    
    section 持久服务 (长期运行)
    containerd daemon     :done, containerd, 2024-01-01, 2024-01-05
    vmm-sandboxer service :done, vmmsandboxer, 2024-01-01, 2024-01-05
    virtiofsd daemon      :done, virtiofsd, 2024-01-01, 2024-01-05
    
    section Pod-1 (临时组件)
    Cloud Hypervisor-1    :active, chv1, 2024-01-01, 2024-01-02
    vmm-task-1 (Guest)    :active, task1, 2024-01-01, 2024-01-02
    Container Process-1   :active, container1, 2024-01-01, 2024-01-02
    
    section Pod-2 (临时组件)
    Cloud Hypervisor-2    :active, chv2, 2024-01-02, 2024-01-03
    vmm-task-2 (Guest)    :active, task2, 2024-01-02, 2024-01-03
    Container Process-2   :active, container2, 2024-01-02, 2024-01-03
    
    section Pod-3 (临时组件)
    Cloud Hypervisor-3    :active, chv3, 2024-01-03, 2024-01-04
    vmm-task-3 (Guest)    :active, task3, 2024-01-03, 2024-01-04
    Container Process-3   :active, container3, 2024-01-03, 2024-01-04
```

长时间运行，服务于全部 3 次创建和删除的组件是：

- containerd daemon
- vmm-sandboxer service
- virtiofsd daemon

### 时间线视图

```mermaid
timeline
    title Kuasar Cloud Hypervisor - 连续3个Pod生命周期
    
    section 系统启动
        系统初始化 : containerd daemon启动
                  : vmm-sandboxer service启动
                  : virtiofsd daemon启动
                  
    section Pod-1 创建
        MicroVM-1启动  : Cloud Hypervisor-1进程启动
                      : vmm-task-1 (Guest PID 1)启动
                      : 容器进程-1运行
                     
    section Pod-2 创建
        MicroVM-2启动  : Cloud Hypervisor-2进程启动
                      : vmm-task-2 (Guest PID 1)启动
                      : 容器进程-2运行
                     
    section Pod-3 创建
        MicroVM-3启动  : Cloud Hypervisor-3进程启动
                      : vmm-task-3 (Guest PID 1)启动  
                      : 容器进程-3运行
                     
    section 清理阶段
        清理Pod-1     : 容器进程-1停止
                     : Cloud Hypervisor-1退出
                     : vmm-task-1终止
                     
        清理Pod-2     : 容器进程-2停止
                     : Cloud Hypervisor-2退出
                     : vmm-task-2终止
                     
        清理Pod-3     : 容器进程-3停止
                     : Cloud Hypervisor-3退出
                     : vmm-task-3终止
                     
    section 持续服务
        后台运行     : containerd daemon (持续运行)
                    : vmm-sandboxer service (持续运行)
                    : virtiofsd daemon (持续运行)
```

### 详细交互流程图

```mermaid
sequenceDiagram
    participant User as kubectl/crictl
    participant CD as containerd
    participant VMS as vmm-sandboxer
    participant CHV1 as Cloud Hypervisor-1
    participant Task1 as vmm-task-1 (Guest)
    participant Runc1 as runc-1 (Guest)
    participant CHV2 as Cloud Hypervisor-2
    participant Task2 as vmm-task-2 (Guest)
    participant CHV3 as Cloud Hypervisor-3
    participant Task3 as vmm-task-3 (Guest)
    
    Note over User,Task3: === Pod-1 创建流程 ===
    
    User->>CD: CreateSandbox(pod-1)
    CD->>VMS: CreateSandbox API
    VMS->>VMS: Factory.create_vm()
    VMS->>CHV1: 启动 cloud-hypervisor 进程
    activate CHV1
    CHV1->>Task1: 启动 vmm-task (PID 1)
    activate Task1
    Task1->>VMS: vsock连接注册
    VMS-->>CD: CreateSandboxResponse
    
    User->>CD: CreateContainer(container-1)
    CD->>VMS: CreateContainer API  
    VMS->>Task1: CreateContainer (via vsock)
    Task1->>Runc1: runc create + start
    activate Runc1
    Task1-->>VMS: Success
    VMS-->>CD: Success
    
    Note over User,Task3: === Pod-2 创建流程 ===
    
    User->>CD: CreateSandbox(pod-2)
    CD->>VMS: CreateSandbox API
    VMS->>CHV2: 启动 cloud-hypervisor 进程
    activate CHV2
    CHV2->>Task2: 启动 vmm-task (PID 1)
    activate Task2
    Task2->>VMS: vsock连接注册
    VMS-->>CD: CreateSandboxResponse
    
    User->>CD: CreateContainer(container-2)
    CD->>VMS: CreateContainer API
    VMS->>Task2: CreateContainer (via vsock)
    Task2->>Task2: runc create + start
    VMS-->>CD: Success
    
    Note over User,Task3: === Pod-3 创建流程 ===
    
    User->>CD: CreateSandbox(pod-3)
    CD->>VMS: CreateSandbox API
    VMS->>CHV3: 启动 cloud-hypervisor 进程
    activate CHV3
    CHV3->>Task3: 启动 vmm-task (PID 1)
    activate Task3
    Task3->>VMS: vsock连接注册
    VMS-->>CD: CreateSandboxResponse
    
    Note over User,Task3: === 删除流程 (按创建顺序) ===
    
    User->>CD: StopSandbox(pod-1)
    CD->>VMS: StopSandbox API
    VMS->>CHV1: SIGTERM/shutdown
    CHV1->>Task1: VM shutdown
    deactivate Runc1
    deactivate Task1
    deactivate CHV1
    
    User->>CD: StopSandbox(pod-2)
    CD->>VMS: StopSandbox API
    VMS->>CHV2: SIGTERM/shutdown
    deactivate Task2
    deactivate CHV2
    
    User->>CD: StopSandbox(pod-3)
    CD->>VMS: StopSandbox API
    VMS->>CHV3: SIGTERM/shutdown
    deactivate Task3
    deactivate CHV3
```

### 架构组件图:

```mermaid
graph TB
    subgraph "Host System (持久层)"
        CD[containerd daemon<br/>📍 长期运行]
        VMS[vmm-sandboxer service<br/>📍 长期运行<br/>Unix Socket:/run/vmm-sandboxer.sock]
        VFS[virtiofsd<br/>📍 长期运行]
        Shim[containerd-shim-kuasar-vmm-v2<br/>📍 可选组件]
    end
    
    subgraph "Pod-1 (临时)"
        CHV1[Cloud Hypervisor-1<br/>⏱️ 临时进程]
        subgraph "Guest VM-1"
            Task1[vmm-task-1<br/>⏱️ Guest PID 1]
            Runc1[runc-1<br/>⏱️ 容器管理器]
            Proc1[container-process-1<br/>⏱️ 应用进程]
        end
    end
    
    subgraph "Pod-2 (临时)"
        CHV2[Cloud Hypervisor-2<br/>⏱️ 临时进程]
        subgraph "Guest VM-2"
            Task2[vmm-task-2<br/>⏱️ Guest PID 1]
            Runc2[runc-2<br/>⏱️ 容器管理器]
            Proc2[container-process-2<br/>⏱️ 应用进程]
        end
    end
    
    subgraph "Pod-3 (临时)"
        CHV3[Cloud Hypervisor-3<br/>⏱️ 临时进程]
        subgraph "Guest VM-3"
            Task3[vmm-task-3<br/>⏱️ Guest PID 1]
            Runc3[runc-3<br/>⏱️ 容器管理器]
            Proc3[container-process-3<br/>⏱️ 应用进程]
        end
    end
    
    CD --> VMS
    VMS --> CHV1
    VMS --> CHV2  
    VMS --> CHV3
    
    CHV1 --> Task1
    Task1 --> Runc1
    Runc1 --> Proc1
    
    CHV2 --> Task2
    Task2 --> Runc2
    Runc2 --> Proc2
    
    CHV3 --> Task3
    Task3 --> Runc3
    Runc3 --> Proc3
    
    VMS -.vsock.-> Task1
    VMS -.vsock.-> Task2
    VMS -.vsock.-> Task3
    
    VFS -.virtio-fs.-> CHV1
    VFS -.virtio-fs.-> CHV2
    VFS -.virtio-fs.-> CHV3
    
    style CD fill:#e1f5fe
    style VMS fill:#e1f5fe
    style VFS fill:#e1f5fe
    style CHV1 fill:#fff3e0
    style CHV2 fill:#fff3e0
    style CHV3 fill:#fff3e0
    style Task1 fill:#f3e5f5
    style Task2 fill:#f3e5f5
    style Task3 fill:#f3e5f5
```

### 进程生命周期状态图

```mermaid
stateDiagram-v2
    [*] --> SystemInit: 系统启动
    
    SystemInit --> ContainerdRunning: 启动containerd
    ContainerdRunning --> VMSandboxerRunning: 启动vmm-sandboxer
    VMSandboxerRunning --> VirtioFSRunning: 启动virtiofsd
    VirtioFSRunning --> Ready: 持久服务就绪
    
    Ready --> ParallelCreation: 开始并行创建3个Pod
    
    state ParallelCreation {
        [*] --> Pod1Creating
        [*] --> Pod2Creating
        [*] --> Pod3Creating
        
        state Pod1Creating {
            [*] --> CHV1Starting: Cloud Hypervisor-1启动
            CHV1Starting --> Task1Ready: vmm-task-1初始化
            Task1Ready --> Container1Running: 容器-1运行
            Container1Running --> Pod1Ready: Pod-1就绪
        }
        
        state Pod2Creating {
            [*] --> CHV2Starting: Cloud Hypervisor-2启动
            CHV2Starting --> Task2Ready: vmm-task-2初始化
            Task2Ready --> Container2Running: 容器-2运行
            Container2Running --> Pod2Ready: Pod-2就绪
        }
        
        state Pod3Creating {
            [*] --> CHV3Starting: Cloud Hypervisor-3启动
            CHV3Starting --> Task3Ready: vmm-task-3初始化
            Task3Ready --> Container3Running: 容器-3运行
            Container3Running --> Pod3Ready: Pod-3就绪
        }
        
        Pod1Ready --> AllPodsReady
        Pod2Ready --> AllPodsReady
        Pod3Ready --> AllPodsReady
    }
    
    ParallelCreation --> AllPodsRunning: 所有Pod创建完成
    AllPodsRunning --> ParallelDeletion: 开始并行删除
    
    state ParallelDeletion {
        [*] --> Pod1Deleting
        [*] --> Pod2Deleting
        [*] --> Pod3Deleting
        
        state Pod1Deleting {
            [*] --> Container1Stopping: 停止容器-1
            Container1Stopping --> CHV1Shutdown: 关闭Cloud Hypervisor-1
            CHV1Shutdown --> Task1Terminated: vmm-task-1终止
            Task1Terminated --> Pod1Deleted: Pod-1已删除
        }
        
        state Pod2Deleting {
            [*] --> Container2Stopping: 停止容器-2
            Container2Stopping --> CHV2Shutdown: 关闭Cloud Hypervisor-2
            CHV2Shutdown --> Task2Terminated: vmm-task-2终止
            Task2Terminated --> Pod2Deleted: Pod-2已删除
        }
        
        state Pod3Deleting {
            [*] --> Container3Stopping: 停止容器-3
            Container3Stopping --> CHV3Shutdown: 关闭Cloud Hypervisor-3
            CHV3Shutdown --> Task3Terminated: vmm-task-3终止
            Task3Terminated --> Pod3Deleted: Pod-3已删除
        }
        
        Pod1Deleted --> AllPodsDeleted
        Pod2Deleted --> AllPodsDeleted
        Pod3Deleted --> AllPodsDeleted
    }
    
    ParallelDeletion --> ServicesContinue: 所有Pod清理完成
    ServicesContinue --> Ready: 持久服务继续运行
    
    note right of Ready
        持久服务层组件：
        - containerd daemon
        - vmm-sandboxer service
        - virtiofsd daemon
        这些组件持续运行
    end note
    
    note right of AllPodsRunning
        3个Pod并行运行中：
        - Pod-1: CHV1 + Task1 + Container1
        - Pod-2: CHV2 + Task2 + Container2  
        - Pod-3: CHV3 + Task3 + Container3
    end note
```

简化版本：

```mermaid
stateDiagram-v2
    [*] --> SystemReady: 持久服务启动
    
    state SystemReady {
        [*] --> ContainerdRunning
        [*] --> VMSandboxerRunning
        [*] --> VirtioFSRunning
    }
    
    SystemReady --> CreatePods: 开始创建Pod
    
    state CreatePods {
        [*] --> CreatingPod1
        [*] --> CreatingPod2
        [*] --> CreatingPod3
        
        CreatingPod1 --> Pod1Running: CHV1+Task1+Container1
        CreatingPod2 --> Pod2Running: CHV2+Task2+Container2
        CreatingPod3 --> Pod3Running: CHV3+Task3+Container3
    }
    
    CreatePods --> AllRunning: 3个Pod都在运行
    AllRunning --> DeletePods: 开始删除Pod
    
    state DeletePods {
        [*] --> DeletingPod1
        [*] --> DeletingPod2
        [*] --> DeletingPod3
        
        DeletingPod1 --> Pod1Deleted: 清理CHV1+Task1+Container1
        DeletingPod2 --> Pod2Deleted: 清理CHV2+Task2+Container2
        DeletingPod3 --> Pod3Deleted: 清理CHV3+Task3+Container3
    }
    
    DeletePods --> SystemReady: 持久服务继续运行
```

### 资源管理视图

```mermaid
graph TD
    subgraph "持久服务层 (长期运行)"
        A[containerd daemon]
        B[vmm-sandboxer service]
        C[virtiofsd daemon]
    end
    
    subgraph "Pod-1 (临时组件)"
        D1[Cloud Hypervisor-1]
        E1[vmm-task-1]
        F1[runc + container-1]
    end
    
    subgraph "Pod-2 (临时组件)"
        D2[Cloud Hypervisor-2]
        E2[vmm-task-2]
        F2[runc + container-2]
    end
    
    subgraph "Pod-3 (临时组件)"
        D3[Cloud Hypervisor-3]
        E3[vmm-task-3]
        F3[runc + container-3]
    end
    
    A --> B
    B -.管理.-> D1
    B -.管理.-> D2
    B -.管理.-> D3
    
    D1 --> E1
    E1 --> F1
    D2 --> E2
    E2 --> F2
    D3 --> E3
    E3 --> F3
    
    C -.virtio-fs.-> D1
    C -.virtio-fs.-> D2
    C -.virtio-fs.-> D3
    
    style A fill:#e8f5e8,stroke:#2e7d2e,stroke-width:2px
    style B fill:#e8f5e8,stroke:#2e7d2e,stroke-width:2px
    style C fill:#e8f5e8,stroke:#2e7d2e,stroke-width:2px
    style D1 fill:#ffe8e8,stroke:#d32f2f,stroke-width:2px
    style D2 fill:#ffe8e8,stroke:#d32f2f,stroke-width:2px
    style D3 fill:#ffe8e8,stroke:#d32f2f,stroke-width:2px
    style E1 fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    style E2 fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    style E3 fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
```


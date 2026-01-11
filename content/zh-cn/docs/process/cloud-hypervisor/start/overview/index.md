---
title: "启动概述"
linkTitle: "概述"
weight: 10
date: 2025-10-15
description: >
  启动 Cloud Hypervisor 的流程概述
---

Kuasar 中 Cloud Hypervisor microVM 启动全流程

## 1. 整体架构

Kuasar 通过以下组件实现 Cloud Hypervisor microVM 的管理：

- **containerd-shim-kuasar-vmm-v2**: containerd 的 shim 插件
- **kuasar-vmm-sandboxer-clh**: Cloud Hypervisor 专用的 sandbox 服务
- **CloudHypervisorVM**: VM 实例管理器
- **CloudHypervisorVMFactory**: VM 工厂类

## 2. 详细启动流程

#### 第一阶段：Sandbox 创建 (create)

1. **containerd 调用 shim**

   - containerd 通过 CRI API 调用 `containerd-shim-kuasar-vmm-v2`
   - shim 连接到 `kuasar-vmm-sandboxer-clh` 服务

2. **Sandboxer 初始化**

   ```
   rustCopy code1// vmm/sandbox/src/bin/cloud_hypervisor/main.rs
   2let mut sandboxer: KuasarSandboxer<CloudHypervisorVMFactory, CloudHypervisorHooks> =
   3    KuasarSandboxer::new(config.sandbox, config.hypervisor, CloudHypervisorHooks::default());
   ```

3. **VM 实例创建**

   ```
   rustCopy code1// CloudHypervisorVMFactory::create_vm
   2let mut vm = CloudHypervisorVM::new(id, &netns, &s.base_dir, &self.vm_config);
   ```

4. **设备配置**

   - 添加 rootfs (pmem 设备): `Pmem::new("rootfs", &image_path, true)`
   - 添加随机数生成器: `Rng::new("rng", &entropy_source)`
   - 添加 vsock 设备: `Vsock::new(3, &guest_socket_path, "vsock")`
   - 添加控制台设备: `Console::new(&console_path, "console")`
   - 添加文件系统设备: `Fs::new("fs", &virtiofsd_socket, "kuasar")`

5. **Cgroup 设置**

   ```
   rustCopy code1sandbox_cgroups = SandboxCgroup::create_sandbox_cgroups(&cgroup_parent_path, &s.sandbox.id)?;
   2sandbox_cgroups.update_res_for_sandbox_cgroups(&s.sandbox)?;
   ```

#### 第二阶段：VM 启动 (start)

1. **网络准备**

   ```
   rustCopy code1if !sandbox.data.netns.is_empty() {
   2    sandbox.prepare_network().await?;
   3}
   ```

2. **启动 virtiofsd**

   ```
   rustCopy code1async fn start_virtiofsd(&self) -> Result<u32> {
   2    create_dir_all(&self.virtiofsd_config.shared_dir).await?;
   3    let params = self.virtiofsd_config.to_cmdline_params("--");
   4    let mut cmd = tokio::process::Command::new(&self.virtiofsd_config.path);
   5    cmd.args(params.as_slice());
   6    // 在指定的网络命名空间中启动
   7    set_cmd_netns(&mut cmd, self.netns.to_string())?;
   8}
   ```

3. **启动 Cloud Hypervisor**

   ```
   rustCopy code1async fn start(&mut self) -> Result<u32> {
   2    let mut params = self.config.to_cmdline_params("--");
   3    for d in self.devices.iter() {
   4        params.extend(d.to_cmdline_params("--"));
   5    }
   6    
   7    let mut cmd = tokio::process::Command::new(&self.config.path);
   8    cmd.args(params.as_slice());
   9    set_cmd_netns(&mut cmd, self.netns.to_string())?;
   10    let child = cmd.spawn()?;
   11}
   ```

4. **API 客户端创建**

   ```
   rustCopy code1match self.create_client().await {
   2    Ok(client) => self.client = Some(client),
   3    Err(e) => return Err(e),
   4};
   ```

#### 第三阶段：Guest 系统配置

1. **内核启动参数**

   ```
   rustCopy code1const DEFAULT_KERNEL_PARAMS: &str = "console=hvc0 \
   2root=/dev/pmem0p1 \
   3rootflags=data=ordered,errors=remount-ro \
   4ro rootfstype=ext4 \
   5task.sharefs_type=virtiofs";
   ```

2. **Agent 通信建立**

   - VM 通过 vsock 设备与 host 通信
   - agent_socket: `"hvsock://{guest_socket_path}:1024"`

3. **文件系统挂载**

   - rootfs 通过 pmem 设备挂载
   - 共享目录通过 virtiofs 挂载

#### 第四阶段：监控和管理

1. **进程监控**

   ```
   rustCopy code1let sandbox_clone = sandbox_mutex.clone();
   2monitor(sandbox_clone);
   ```

2. **Cgroup 管理**

   ```
   rust
   Copy code
   1sandbox.add_to_cgroup().await?;
   ```

## 3. 关键配置参数

#### VM 配置示例

```
tomlCopy code1[hypervisor]
2path = "/usr/local/bin/cloud-hypervisor"
3vcpus = 1
4memory_in_mb = 1024
5kernel_path = "/var/lib/kuasar/vmlinux.bin"
6image_path = "/var/lib/kuasar/kuasar.img"
7hugepages = true
8entropy_source = "/dev/urandom"
9
10[hypervisor.virtiofsd]
11path = "/usr/local/bin/virtiofsd"
12log_level = "info"
13cache = "never"
14thread_pool_size = 4
```

#### 生成的 Cloud Hypervisor 命令行

```
bashCopy code1cloud-hypervisor \
2    --api-socket /path/to/api.sock \
3    --cpus boot=1 \
4    --memory size=1073741824,shared=on,hugepages=on \
5    --kernel /var/lib/kuasar/vmlinux.bin \
6    --cmdline "console=hvc0 root=/dev/pmem0p1 ..." \
7    --pmem file=/var/lib/kuasar/kuasar.img,id=rootfs,readonly=on \
8    --rng src=/dev/urandom,iommu=off \
9    --vsock cid=3,socket=/path/to/task.vsock \
10    --console file=/tmp/task.log \
11    --fs tag=kuasar,socket=/path/to/virtiofs.sock
```

## 4. 设备热插拔支持

Cloud Hypervisor 还支持设备的热插拔：

```
rustCopy code1async fn hot_attach(&mut self, device_info: DeviceInfo) -> Result<(BusType, String)> {
2    let client = self.get_client()?;
3    let addr = client.hot_attach(device_info)?;
4    Ok((BusType::PCI, addr))
5}
6
7async fn hot_detach(&mut self, id: &str) -> Result<()> {
8    let client = self.get_client()?;
9    client.hot_detach(id)?;
10    Ok(())
11}
```

## 5. 错误处理和恢复

系统具备完整的错误处理和恢复机制：

- 启动失败时自动清理网络和 cgroup
- 支持 sandbox 状态恢复
- 进程监控和异常退出处理

这个流程展示了 Kuasar 如何将 Cloud Hypervisor 集成到 containerd 生态系统中，提供了完整的 microVM 容器运行时解决方案。

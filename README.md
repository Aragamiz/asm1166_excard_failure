# ASM1166 + NVIDIA 加速卡/显卡在 AMD 平台 POST 循环重启（无法进入 BIOS）排查与解决方案

> 本文记录了一起在 AMD B550 平台上，**M.2 转 6 口 SATA 扩展卡（祥硕 ASMedia ASM1166 主控）** 与 **NVIDIA 独立加速卡/显卡（以 Tesla P4 为例）** 共存时，导致主机上电自检（POST）阶段无限复位重启（Boot Loop）、无法进入 BIOS 设置界面的完整排查过程、底层根因分析与免拆卡固化解决方案。

---

## 一、硬件环境与故障现象

### 1.1 硬件配置
* **主板**：ASUS TUF GAMING B550M-PLUS（AMD B550 芯片组，AMI Aptio V UEFI BIOS）
* **CPU**：AMD Ryzen 7 3700X（无集成显卡）
* **存储卡**：M.2 NVMe (Key-M) 转 6 口 SATA 扩展卡（ASMedia ASM1166 主控，PCIe Gen3 x2）
* **显卡/计算卡**：NVIDIA Tesla P4（GP104 核心，PCIe Gen3 x16，插于主板南桥 `PCIEX16_2` 物理槽）
* **其他 PCIe 设备**：NVMe SSD（M.2 / PCIe 转接卡）及 Mellanox ConnectX-4 Lx 多口网卡

### 1.2 故障现象
* 当主板 BIOS 开启 **`Above 4G Decoding = Enabled`（4G 以上解码）** 且使用纯 **UEFI 模式（CSM = Disabled）** 时：
* 只要将 **ASM1166 扩展卡** 与 **NVIDIA 卡** 同时插在主板上：
  * 主机通电开机约 3~5 秒后自动断电，随后再次尝试通电，**陷入无限 POST 重启死循环（Cold Reset Loop）**；
  * 显示器无信号输出，按 `Del` / `F2` 完全无法进入 BIOS 设置界面，主板卡在自检早期。
* **对照测试**：
  * **拔掉 ASM1166 卡**（机上保留 NVIDIA 卡 + 其它全部 NVMe / 网卡等 7 个设备）：**正常开机**；
  * **拔掉 NVIDIA 卡**（机上保留 ASM1166 卡 + 其它全部设备）：**正常开机**；
  * **拔掉 ASM1166 上连接的全部 SATA 硬盘数据线**（仅留 ASM1166 光板插在 M.2 槽位上）并插上 NVIDIA 卡：**依然 100% 触发无限重启**；
  * **互换槽位**（将 ASM1166 从 CPU 直连的 `M.2_1` 挪到南桥 `M.2_2`，或与其它 NVMe 互换）：**依然 100% 触发无限重启**。

---

## 二、排查过程与假说推翻（避坑实录）

在定位过程中，曾先后提出并验证排除了以下几个常见假说：

1. **假说 1：PCIe 32-bit MMIO 地址空间用尽 / 芯片组上行桥窗口上限？**
   * *推测*：此前怀疑南桥分配的 32-bit MMIO 被锁死在 19MB 附近导致分配失败。
   * *证伪*：在 Linux 下提取 `/proc/iomem` 发现，平台 32-bit MMIO 根窗口拥有 688MB 的巨大空闲区（`d0000000-fc3fffff`）；后续成功启动时，上行桥实际分配了 `288MB` 预取窗口 + `20MB` 非预取窗口，证实**并不存在硬件 19MB 上限**。
2. **假说 2：多块 3.5 寸机械硬盘起转浪涌电流拉垮电源？**
   * *推测*：4 块企业级机械盘上电起转电流导致 12V 掉压触发电源保护。
   * *证伪*：**拔掉全部 SATA 硬盘连接线（仅留 ASM1166 光板）**，开机依然死循环，排除供电及硬盘分区表问题。
3. **假说 3：服务器拆机卡 PCIe 金手指 `B5/B6`（SMBus）冲突？**
   * *推测*：Tesla P4 等计算卡在 PCIe 边沿引脚 `B5 (SMCLK)` 与 `B6 (SMDAT)` 上挂有 BMC I2C 总线，可能与主板或扩展卡引脚电平产生干扰。
   * *证伪*：使用绝缘高温胶带屏蔽 P4 显卡金手指 B 面左起第 5、6 根针脚（`B5/B6`），插回测试**依然死循环**，排除物理 SMBus 冲突。

---

## 三、底层根因分析（深度硬核剖析）

在通过规避配置成功进入 Linux 系统后，通过提取 `dmesg` 与 `lspci -vv` 底层寄存器，彻底查明了导致 POST 死锁的两大核心机制：

### 3.1 固件缺陷：ASM1166 虚标 30 个端口且附带 512KB Option ROM
* 在 Linux 内核日志中观察到该 ASM1166 扩展卡暴露的硬件特征：
  ```text
  pci 0000:01:00.0: [1b21:1166] type 00 class 0x010601 PCIe Endpoint
  pci 0000:01:00.0: ROM [mem 0xfcb00000-0xfcb7ffff pref] [size=512K]
  ahci 0000:01:00.0: 30/32 ports implemented (port mask 0xffffff3f)
  ahci 0000:01:00.0: scsi host0..host31
  ```
* **问题 1（ROM 异常偏大）**：市面上大部分廉价 ASM1166 扩展卡板载 SPI Flash（8 脚芯片）中刷入了完整的 UEFI/Legacy Option ROM 启动固件，其体积高达 **512KB**（普通扩展卡通常仅数 KB 到几十 KB）。
* **问题 2（端口虚标）**：物理硬件仅引出 6 个 SATA 口，但固件向主板报告的 AHCI `PI`（Ports Implemented）掩码寄存器竟为 `0xffffff3f`，即向 BIOS 谎称实现了 **30 个 SATA 端口**！

### 3.2 固件冲突链：UEFI `AhciBusDxe` 在 Above 4G 环境下崩溃
1. **显卡 BAR 空间特性**：
   * 很多人误以为 Tesla P4 / 计算卡必须依赖 `Above 4G Decoding`。但实测证明，Tesla P4（GP104 核心）其显存 `BAR1` 仅为 **`256MB`**，加上 `BAR0 (16MB)` 与 `BAR3 (32MB)`，总共仅需 **304MB**（与消费级的 GTX 750 Ti / GTX 1070 完全一致），完全可以直接映射在 4GB 以下的 32-bit MMIO 空间。
2. **死锁发生过程**：
   * 当 BIOS 开启 `Above 4G Decoding = Enabled` 且关闭 CSM 时：
   * AMI Aptio V 固件在执行 UEFI DXE 驱动调度阶段，一方面为显卡分配 64-bit 预取空间，另一方面强制执行 ASM1166 板载 512KB 的 Option ROM，并调用 UEFI `AhciBusDxe` 驱动去逐一扫描并初始化这 30 个虚拟 AHCI 端口。
   * 在处理跨 4G 边界的大量虚拟端口数据结构时，触发了 AMI BIOS 的内存/驱动调度异常，引发致命故障并导致主板硬件看门狗触发冷复位（Cold Reset），表现为 **POST 循环重启**。

---

## 四、解决方案与 BIOS 黄金配置

不需要动用烙铁拆除板载芯片，通过调整 BIOS 配置即可完美解决该问题，使全设备正常工作且**不影响任何硬件直通（IOMMU）能力**：

### 4.1 核心配置步骤（关键两步）

1. **保持 `Above 4G Decoding` 为关闭状态（破局核心 1）**：
   * **路径**：`Advanced（高级）` → `PCI Subsystem Settings（PCI 子系统设置）`
   * **设置**：
     * **`Above 4G Decoding`** → 设为 **`Disabled`**
     * **`Re-Size BAR Support`** → 设为 **`Disabled`**
     * **`SR-IOV Support`** → 设为 **`Enabled`**（若需使用网卡 VF 等虚拟化）
   * *原理*：显卡 256MB BAR 直接分配在 32-bit MMIO（如 `0xd0000000`），完全不需要 Above 4G。

2. **利用 CSM 忽略第三方存储 Option ROM 加载（破局核心 2）**：
   * **路径**：`Boot（启动）` → `CSM (Compatibility Support Module)`
   * **设置**：
     * **`Launch CSM`** → 设为 **`Enabled`**
     * **`Boot Device Control`** → 设为 **`UEFI Only`**（确保操作系统仍然使用纯 UEFI 方式引导）
     * **`Boot from Storage Devices`** → 设为 **`Ignore`（或 `Do not launch`）** 👈 **核心！**
     * **`Boot from Network Devices`** → 设为 **`Ignore`**
     * **`Boot from PCI-E/PCI Expansion Devices`** → 设为 **`Ignore`**
   * *原理*：禁止主板 BIOS 在开机 POST 阶段执行 ASM1166 上的 512KB Option ROM 代码，跳过其对 30 个虚拟端口的死循环扫描，彻底规避崩溃；进入操作系统后，由 Linux / Windows 原生 `ahci` 驱动直接接管，6 个 SATA 接口全部满速正常运行。

### 4.2 补充设置（避免无显卡/无外设报错）
* **路径**：`Boot（启动）` → `Boot Configuration`
  * **`Wait For 'F1' If Error`** → 设为 **`Disabled`**（针对无视频输出的计算卡环境，防止卡在按 F1 提示界面）。

---

## 五、伴生问题：双 M.2 插槽占用引发的 Linux 网卡接口名漂移

在成功解决 POST 循环重启后，用户在 Linux（如 Proxmox VE / Debian / Ubuntu）下可能会遇到**「开机后板载网卡物理灯熄灭、无法获取 IP 地址、网络失联」**的次生问题。

### 5.1 根因
* 插入 ASM1166 扩展卡后，主板多占用了一个 PCIe 总线号（Bus），导致排在其后的所有 PCIe 设备的物理 BDF 整体向后平移 `+1` 位：
  * 板载 2.5G 网卡从 `06:00.0` 漂移至 **`07:00.0`**，Linux 可预测命名规则将其从 `enp6s0` 改名为 **`enp7s0`**；
  * 其他 PCIe 网卡（如 25G 双口卡）也由 `07:00.0` 改名为 **`08:00.0`**。
* 内核加载 `r8169` 网卡驱动时，会先关闭物理网口 PHY（此时网口灯熄灭）。若系统的网络配置文件（如 `/etc/network/interfaces`）中仍然写死旧名称 `enp6s0`，系统将无法正确将新网口 `UP` 起来，导致网口灯长暗、远程网络断连。

### 5.2 解决方案（通过 `systemd.link` 按 MAC 地址永久绑定网卡名称）

在系统 `/etc/systemd/network/` 目录下添加 `.link` 规则文件，基于网卡出厂唯一固化的 `PermanentMACAddress` 强制锁定接口名称：

1. **创建 `/etc/systemd/network/10-onboard-lan.link`**：
   ```ini
   [Match]
   PermanentMACAddress=xx:xx:xx:xx:xx:xx  # 填入板载网卡实际的物理 MAC 地址
   Driver=r8169

   [Link]
   Name=enp6s0
   WakeOnLan=magic
   ```
2. **更新引导镜像**：
   ```bash
   update-initramfs -u -k all
   ```
配置后，无论主板上插入多少张 PCIe/M.2 扩展卡导致总线号如何漂移，板载网卡在进入系统时都将固定命名为 `enp6s0`，彻底免疫改名失联问题。

---

## 六、致硬件厂商/固件开发者的建议

对于生产 ASM1166 扩展卡的硬件厂商，建议在出厂固件上进行如下优化：

1. **提供“无 Option ROM / 纯微码固件（Non-Bootable Firmware）”**：
   * 大多数多盘位 NAS、虚拟化用户将系统装在板载 NVMe SSD 上，完全不需要从 ASM1166 扩展卡引导系统。
   * 类似银欣（SilverStone ECS06）官方提供的优化固件，剔除 512KB 的 Option ROM 代码，仅保留芯片自身初始化必需的 Controller Firmware，既能彻底规避多显卡/Above 4G 环境下的 UEFI 调度死锁，又能显著缩短自检时间。
2. **修正 `Ports Implemented` 掩码**：
   * 将固件报告的端口掩码从虚标的 `0xffffff3f`（30 口）修改为实际物理端口对应的 `0x0000003f`（6 口），防止引发主板 UEFI AHCI 驱动的内存与数组越界异常。

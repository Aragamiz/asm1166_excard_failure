# 关于 M.2 转 6口 SATA (ASM1166) 扩展卡在特定 UEFI/显卡环境下的兼容性反馈与固件优化建议

### 一、测试环境概要
* **主板平台**：AMD B550 芯片组主板（AMI Aptio V UEFI 固件）
* **CPU**：AMD Ryzen 3000 系列（无核显）
* **PCIe 拓扑**：扩展卡插于 M.2 Key-M 槽位（PCIe Gen3 x2 / x4 物理通道）
* **并存硬件**：NVIDIA 独立加速卡（无视频输出的计算卡 / 类似普通 256MB~4GB 显存显卡）

---

### 二、问题复现现象
1. **默认设置下的死循环（POST Boot Loop）**：
   * 当主板 BIOS 开启 **`Above 4G Decoding`（4G以上解码）** 且使用纯 **UEFI 模式（CSM 关闭）** 时：
   * 只要插上该 ASM1166 扩展卡，主机上电开机约 3~5 秒后自动断电、复位重启，**陷入死循环，完全无法进入 BIOS 设置界面，也无法显示自检画面**。
   * **即使把扩展卡上的所有 SATA 硬盘线全部拔空（仅留光板插在 M.2 槽位上）**，依然 100% 稳定复现循环重启。
   * 拔掉该 ASM1166 卡后，主机恢复正常开机。

2. **最终成功避开冲突的规避配置**：
   * 必须在主板 BIOS 中强制开启 **`CSM = Enabled`**，并将子项设为：
     * `Boot Device Control` = `UEFI Only`
     * **`Boot from Storage Devices` = `Ignore`（或 `Do not launch`）**
   * **作用原理**：主板 BIOS 在 POST 自检阶段**强制跳过对该 ASM1166 板载 Option ROM 的加载与执行**。跳过该固件后，系统顺利通过自检并进入 Linux 系统，系统内核原生 `ahci` 驱动接管后，6 个 SATA 接口全部能正常读写与满速识别。

---

### 三、底层抓取到的固件参数与根因分析

在 Linux 系统下提取该卡底层寄存器与 PCIe 配置空间日志（`dmesg` / `lspci -vv`），定位到以下核心问题：

1. **Option ROM 容量异常偏大**：
   * 该卡在 PCIe 配置空间中映射了一个高达 **512KB 的 Expansion ROM**：
     ```text
     pci 0000:01:00.0: [1b21:1166] type 00 class 0x010601 PCIe Endpoint
     pci 0000:01:00.0: ROM [mem 0xfcb00000-0xfcb7ffff pref] [size=512K]
     ```
   * 相比于普通存储卡仅数 KB~几十 KB 的极简引导微码，该 512KB 的 ROM 代码在 UEFI DXE 调度执行阶段对内存/BAR 映射的要求较为苛刻。

2. **AHCI `PI`（Ports Implemented）寄存器掩码虚标**：
   * 系统加载通用 AHCI 驱动时抓取到的硬件报告：
     ```text
     ahci 0000:01:00.0: 30/32 ports implemented (port mask 0xffffff3f)
     ahci 0000:01:00.0: scsi host0..host31
     ```
   * **分析**：该卡硬件实际仅引出 6 个物理 SATA 接口，但板载固件向主机报告的端口掩码为 `0xffffff3f`（即向系统声称实现了 **30 个端口**）。
   * 在部分 UEFI BIOS 中（特别是配合特定显卡、开启 Above 4G 时），BIOS 自带的存储枚举模块（`AhciBusDxe`）在执行该 Option ROM 初始化这 30 个虚拟端口时发生资源溢出或死锁异常，从而触发主板硬件看门狗复位（Cold Reset Loop）。

---

### 四、对厂家的建议与优化诉求

1. **能否提供一份“纯微码 / 去除 Option ROM（Non-Bootable）”的刷机固件？**
   * 对于大部分工控、NAS（如 TrueNAS、Unraid、Proxmox VE、Linux）或仅作为从盘扩展卡的用户，**根本不需要从该卡的 SATA 接口引导开机**（引导都在板载 NVMe 上）。
   * 类似银欣（SilverStone）等品牌对其 ASM1166 产品提供的优化固件，清空或禁用板载 Option ROM 模块，仅保留芯片自身运行必需的 Controller Firmware，既能极大提高主板兼容性（彻底避免多显卡/Above 4G 环境下的开机卡死），又能缩短主机开机自检时间。

2. **修正固件中的 Ports Implemented（`PI`）掩码**：
   * 将端口掩码由 `0xffffff3f` 修正为实际引出接口对应的 `0x0000003f`（精确匹配 6 端口），避免 UEFI BIOS 扫描过多不存在的虚拟端口导致超时或异常。

---
*(若贵司技术支持有现成的 Flash 刷写工具或无 ROM 纯净固件，请提供一份，我们可在实验室环境中协助验证测试。)*

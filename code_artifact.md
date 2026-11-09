# OxBeats Lite Linux (`oxbeats-lite-linux`)

> **A Minimalist, Modular, Bit-Perfect Linux Audio Appliance Built with AI Agents**
>
> 一个面向大众发烧友与个人开发者的极简高保真 Linux 音乐播放系统开源方案。无需掌握复杂的 Linux 内核裁剪与底层C语言交叉编译知识，通过自然语言交互与工程化八阶段门禁工作流，由 AI Agent 自动为您完成硬件适配、内核编译、模块化音频引擎构建、Bit-Perfect 校验与 BIOS/UEFI 双引导启动镜像生成。

---

## 📖 目录 (Table of Contents)

1. [项目愿景与核心特性](#-项目愿景与核心特性)
2. [系统架构概览](#-系统架构概览)
3. [功能与硬件支持状态分级](#-功能与硬件支持状态分级)
4. [如何使用 AI Agent 一键构建](#-如何使用-ai-agent-一键构建)
5. [镜像烧录与实机使用指南](#-镜像烧录与实机使用指南)
6. [三级测试体系与社区硬件报告](#-三级测试体系与社区硬件报告)
7. [二次开发与目录结构](#-二次开发与目录结构)
8. [版本演进记录 (V1 -> V2)](#-版本演进记录-v1---v2)
9. [开源协议 (License)](#-开源协议-license)

---

## 🎯 项目愿景与核心特性

通用 Linux 发行版（或基于重型发行版套壳的数播系统）往往后台运行着上百个系统守护进程、复杂的声音混音服务（PulseAudio / PipeWire）以及庞大的脚本运行时（Python / Node.js）。

`oxbeats-lite-linux` 采用**“极简 Linux 单用途设备（Single-Purpose Appliance）”**路线，完全基于 Linux 社区公开、标准的开源技术栈，通过极致精简但工程严谨的架构实现高水准的数字音频回放：

* **极致精简的纯 C 用户态**：不依赖 `systemd`，不依赖任何图形桌面或脚本解释器。整个用户态由职责清晰的模块化 C 语言编写，基于 `musl-libc` 与 `alsa-lib` 全静态编译。
* **流式预取与三线程实时隔离架构**：
  * **Audio RT Thread (`SCHED_FIFO 50`)**：锁定物理内存（`mlock`），线程内零动态内存分配、零磁盘 I/O、零网络 I/O、零互斥锁等待，仅从内存无锁环形缓冲区提取音频帧直推 ALSA `hw:x,x` 硬件接口。
  * **Decoder & Prefetch Thread**：独立的流式预取引擎，分块读取本地磁盘或 NAS 网络音频文件并解码至预分配的 **32MB 无锁环形缓冲区（Lock-Free SPSC Ring Buffer）**，既隔离了存储读取抖动，又避免了超大体积无损/DSD 整轨解压导致的内存溢出（OOM）。
  * **Control Thread**：负责网络 DHCP 自举、存储只读挂载与安全的内嵌 Web 控制界面响应。
* **明确区分的两种播放模式**：
  * **`Pure Direct`（纯直通模式 - 默认）**：原始 PCM / DoP / Native DSD 数据不经过任何软件增益或数字处理，通过自动化测试确保 100% 数字级 Bit-Perfect 一致性。
  * **`Protected Transition`（受保护切换模式 - 可选）**：针对部分无硬件静音继电器的解码器，在手动切歌或格式跳变时提供微秒级边缘淡入淡出与 DSD 静音过渡缓冲，抑制切换冲击声。
* **开机自动解静音与全格式支持**：
  * 启动时自动遍历 ALSA Hardware Mixer，解除 DAC 硬件静音开关（Unmute）并推至 0dB 满幅直通；
  * 原生支持 **WAV、FLAC、MP3、DSF、DFF** 以及基于精确采样帧（Sample Frame）定位的 **CUE 虚拟分轨**；
  * 独立支持 **Native DSD**、**DoP (DSD over PCM)** 与 **标准高码率 PCM** 探测与传输路径。
* **Legacy BIOS + UEFI 双模启动**：采用标准 GPT 混合分区表（BIOS Boot + FAT32 ESP + 可写 Config 分区），无论老旧工控机（J1900）还是现代迷你小主机（Intel N100）均可直接引导，并支持配置持久化保存。

---

## 🏗️ 系统架构概览

```
+-----------------------------------------------------------------------------------+
|                              User Browser (WebUI)                                 |
+-----------------------------------------------------------------------------------+
                                         | HTTP REST API (Port 80, Token Auth)
                                         v
+-----------------------------------------------------------------------------------+
|                        OxBeats User-Space Engine (Static C)                       |
|                                                                                   |
|  +-----------------------+   +------------------------+   +--------------------+  |
|  |    Control Thread     |   | Decoder/Prefetch Thread|   |  Audio RT Thread   |  |
|  |     (SCHED_OTHER)     |   |     (SCHED_OTHER)      |   |  (SCHED_FIFO 50)   |  |
|  | - HTTP Web Server     |   | - Chunked File/NFS I/O |   | - mlock() Memory   |  |
|  | - Storage Auto-Mount  |-->| - WAV/FLAC/MP3 Decode  |==>| - Zero-Alloc Loop  |  |
|  | - CUE Frame Seeking   |   | - DSF/DFF De-interleave|   | - XRUN Recovery    |  |
|  | - Network / DHCP Mgr  |   | - DoP Frame Packing    |   | - Direct PCM Write |  |
|  +-----------------------+   +------------------------+   +--------------------+  |
|                                          || (32MB Lock-Free SPSC Ring Buffer)     |
+-----------------------------------------------------------------------------------+
|                        PID 1 Supervisor (ox_init)                                 |
|  - Mount /proc, /sys, /dev, /run      - Signal Handling & Zombie Reaping          |
|  - Hotplug Device Listener            - Crash Recovery & Safe Poweroff/Reboot     |
+-----------------------------------------------------------------------------------+
|                   Linux 6.6 LTS Kernel (Optional PREEMPT_RT)                      |
|  - ALSA Core & snd-usb-audio          - NVMe / AHCI / USB-UAS / SDHCI-MMC         |
|  - FAT32 / exFAT / NTFS3 / ext4 / NFS - Intel (e1000e/igb/igc) & Realtek (r8169)  |
+-----------------------------------------------------------------------------------+
```

---

## 🔌 功能与硬件支持状态分级

秉持诚实严谨的开源工程原则，本项目对所有特性采用四级状态标识（🟢 `Stable` | 🟡 `Experimental` | ⚪ `Not Tested` | 🔴 `Unsupported`）：

| 模块类别 | 具体功能 / 硬件范围 | 状态分级 | 工程说明 |
| :--- | :--- | :--- | :--- |
| **引导与内核** | Legacy BIOS + UEFI x86_64 双引导 | 🟢 `Stable` | 已通过 QEMU SeaBIOS 与 OVMF 双模式自动化验证 |
| **引导与内核** | Linux 6.6 LTS + 可选 `PREEMPT_RT` | 🟢 `Stable` | 严格锁定内核与 RT 补丁小版本，自动审计 Kconfig 依赖 |
| **音频引擎** | WAV / FLAC / MP3 流式播放与 CUE 分轨 | 🟢 `Stable` | `Pure Direct` 模式通过 SHA256 数字级 Bit-Perfect 验证 |
| **音频引擎** | ALSA 硬件直通与 Mixer 自动 0dB 解静音 | 🟢 `Stable` | 自动识别 USB DAC 并解除 Master/PCM/DAC 静音开关 |
| **DSD 支持** | DSF / DFF 解析与 DoP 帧封装输出 | 🟡 `Experimental` | 协议封包已通过字节级单元测试，实机效果取决于 DAC DoP 支持 |
| **DSD 支持** | USB DAC Native DSD (`DSD_U32_LE/BE`) | 🟡 `Experimental` | 依赖具体 USB DAC 界面（XMOS/Amanero 等）与内核 `quirks.c` 支持 |
| **存储支持** | SATA / NVMe / USB UASP / eMMC / TF 卡 | 🟢 `Stable` | 默认采用只读优先（`ro,noatime`）安全挂载策略 |
| **文件系统** | FAT32 / exFAT / NTFS3 / ext4 (含中文) | 🟢 `Stable` | 内建 `NLS_UTF8` 与 `NLS_CODEPAGE_437`，原生支持中文曲目 |
| **网络与存储** | Intel (`e1000e`/`igb`/`igc`) & Realtek (`r8169`) | 🟡 `Experimental` | 部分新版 Realtek 网卡需按需打包 `rtl_nic` 固件 |
| **网络与存储** | NFSv3 / NFSv4 远程 NAS 音乐库挂载 | 🟡 `Experimental` | 经由流式预取缓冲隔离网络抖动 |

---

## 🚀 如何使用 AI Agent 一键构建

本项目采用 **Prompt-as-Code（提示词驱动工程构建）** 模式。您只需将本仓库的 **`oxbeats-lite-linux-master-prompt.md`** 交给具备终端执行能力的 AI 编程助手（如 Claude Code、Cursor、Windsurf、Cline 等）：

### 步骤 1：准备构建宿主机
* **推荐环境**：Ubuntu 22.04 / 24.04 LTS (x86_64) 原生系统，或 Windows 10/11 下的 **WSL2 (Ubuntu)**。确保至少有 **2GB 可用内存** 与 **15GB 可用磁盘空间**。
* **跨平台沙箱模式**：若在未配置开发环境的宿主机上运行，可让 Agent 先执行阶段一中的 `build_sandbox.py`，自动拉取官方 Ubuntu 云镜像并在 QEMU 后台沙箱中完成构建。

### 步骤 2：向 AI Agent 发送构建指令
将 **`oxbeats-lite-linux-master-prompt.md`** 放入工作目录，并向 AI Agent 输入类似如下指令：

> “请阅读当前目录下的 `oxbeats-lite-linux-master-prompt.md`。我的目标硬件是一台 Intel N100 小主机（配备 Intel I226 2.5G 网卡、NVMe SSD，外接 XMOS USB 解码器），请启用 PREEMPT_RT 实时内核，并严格按照提示词中的 Stage 1 到 Stage 8 渐进式门禁流程为我构建系统镜像，每个阶段完成后请向我展示自动化测试结果。”

### 步骤 3：八阶段自动化门禁验收
Agent 将严格按顺序执行以下 8 个阶段（任何阶段测试未通过绝不跳关）：
1. **Stage 1**：宿主机环境检测与 `musl` 静态 `alsa-lib` 工具链编译验证；
2. **Stage 2**：Linux 6.6 LTS 内核与 RT 补丁校验、Kconfig 依赖合成与 `.config` 静态审计；
3. **Stage 3**：PID 1 (`ox_init`) 编译与 QEMU 最小系统启动/优雅关机测试；
4. **Stage 4**：多总线存储只读挂载、UTF-8 中文文件名读取与网络 DHCP 测试；
5. **Stage 5**：三线程流式引擎、CUE 帧寻道与 **PCM SHA256 数字级 Bit-Perfect 一致性验证**；
6. **Stage 6**：DSF/DFF 解析、DoP 封包与 Native DSD 字节序单元测试及回退验证；
7. **Stage 7**：嵌入式 WebUI 整合与 API 路径穿越/命令注入安全渗透测试；
8. **Stage 8**：GPT 双引导镜像打包，并在 QEMU SeaBIOS 与 UEFI (OVMF) 下完成端到端启动验收。

构建全部通过后，最终镜像将输出在 `dist/oxbeats-lite-x86_64.img.xz`。

---

## 💾 镜像烧录与实机使用指南

1. **写入启动盘**：
   * **Windows / macOS**：下载 [balenaEtcher](https://www.balena.io/etcher/) 或 [Rufus](https://rufus.ie/)，直接选择 `dist/oxbeats-lite-x86_64.img.xz` 写入 U 盘、移动 SSD 或 SD/TF 卡。
   * **Linux**：
     ```bash
     xzcat dist/oxbeats-lite-x86_64.img.xz | sudo dd of=/dev/sdX bs=4M status=progress oflag=sync
     ```
2. **开机与连接**：
   * 将启动盘与 USB 解码器（DAC）插上目标主机，插好网线并开机。
   * 系统将在数秒内完成启动，并在外接显示器（或串口）中央清晰打印当前获取到的 IP 地址（例如 `http://192.168.1.105`）。
3. **Web 选曲播放**：
   * 使用同一局域网下的手机、平板或电脑浏览器访问该 IP，即可浏览本地硬盘（SATA/NVMe/USB/TF）或挂载远程 NFS NAS 曲库进行播放。

---

## 🧪 三级测试体系与社区硬件报告

我们坚持工程透明，严格区分三种测试层级：
* **Level A (编译测试)**：静态编译通过，无符号缺失。
* **Level B (QEMU 虚拟化测试)**：验证内核启动、分区挂载、网络通信、PCM 数据流 SHA256 一致性与虚拟声卡输出。
* **Level C (真实物理硬件测试)**：**QEMU 虚拟声卡无法替代真实 USB DAC 测试**。异步 USB 时钟同步、硬件 Mixer 行为及 Native DSD 硬解点灯必须在真实物理设备上验证。

欢迎社区玩家在实机测试后，通过 GitHub Issues 提交您的硬件兼容性报告：
* **测试主机与 CPU**：（如 Intel N100 / J1900）
* **USB DAC 型号与 USB ID (`lsusb`)**：（如 `152a:8750`）
* **实测通过项**：PCM 最高采样率 / DoP 状态 / Native DSD 指示灯状态 / 连续播放 1 小时 XRUN 计数。

---

## 🛠️ 二次开发与目录结构

本项目生成的标准工程结构非常便于 C 语言与嵌入式开发者扩展（例如添加新的音频格式、自定义 Web 前端或针对特定多核 CPU 添加 `isolcpus` 调优）：

```
oxbeats-lite-linux/
├── oxbeats-lite-linux-master-prompt.md  # V1+V2 终极整合版 AI Agent 主提示词
├── README.md                            # 项目总览与使用说明文档
├── configs/                             # 内核版本锁与模块化 Kconfig 配置片段
├── src/                                 # 模块化纯 C 用户态源码 (10 个独立模块)
├── web/                                 # 单页响应式 Web 控制端前端源码
├── scripts/                             # Stage 1 ~ Stage 8 自动化构建脚本
└── tests/                               # Bit-Perfect 校验、DSD 单元测试与 QEMU 验收套件
```

---

## 📝 版本演进记录 (V1 -> V2)

本整合版完整融合了 V1 的极简敏捷思路与 V2 的系统工程规范，修正了早期概念设计中的关键技术问题：
1. **术语与内核配置规范化**：将“微内核”表述修正为“极简 Linux 单用途系统”；引入 `merge_config.sh` 与 `.config` 静态审计，确保 PCI/ACPI/xHCI/SCSI/NLS 等底层依赖完整无缺。
2. **标准分区与双模引导**：将 V1 的无分区表裸 FAT 镜像升级为标准 GPT 混合分区表，同时支持 Legacy BIOS 与 UEFI x86_64 启动，并增加可写配置分区。
3. **流式预取替代整轨全内存解压**：引入预分配 `mlock` 的 32MB 无锁 SPSC 环形缓冲区与独立 Prefetch 线程，彻底解决大体积 DSD/WAV 整轨驻留导致的内存溢出（OOM）问题。
4. **模块化代码与实时安全**：将单文件代码拆分为 10 个职责清晰的 C 模块；严格确保 `Audio RT Thread` 零动态内存分配与零 I/O 阻塞；修正 CUE 解析为基于采样率的精确采样帧（Sample Frame）寻道。
5. **严谨的播放模式与八阶段门禁**：明确分离 `Pure Direct` 与 `Protected Transition` 模式；将构建流程重构为带自动化测试断言的八阶段门禁工作流。

---

## 📄 开源协议 (License)

本项目采用 **[GNU General Public License v3.0 (GPL-3.0)](https://www.gnu.org/licenses/gpl-3.0.html)** 开源协议。您可以自由使用、审计、修改与分发本项目代码及生成的系统镜像，任何衍生修改版本须保持相同的开源许可证。
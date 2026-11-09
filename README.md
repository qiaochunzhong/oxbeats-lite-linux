# OxBeats Lite Linux (oxbeats-lite-linux)

> **Minimalist Bit-Perfect Real-Time Audio OS with Native DSD Support**  
> 告别数 GB 的臃肿发行版与商业套壳系统。通过一套严谨的系统工程构建提示词（Prompt-as-Code），让 AI Agent 在云端沙箱中全自动为你炼制出极致纯粹的发烧微数播系统。

[![License: GPL-3.0](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![Kernel: Linux 6.6 LTS](https://img.shields.io/badge/Kernel-6.6_LTS_RT-orange.svg)]()
[![System Size: ~15MB](https://img.shields.io/badge/Image_Size-~15MB-brightgreen.svg)]()
[![Audio: Native DSD512](https://img.shields.io/badge/ALSA-Native_DSD-red.svg)]()

---

## 📖 项目愿景与设计哲学

在发烧数播领域，许多所谓的“自研系统”本质上是基于庞大的桌面 Linux 简单套壳，充斥着复杂的 Systemd 守护进程、后台日志回写、软件混音器以及隐蔽的重采样（SRC）机制，带来了不可忽视的 Jitter（时钟抖动）与系统电源纹波。

**OxBeats Lite Linux** 将系统极简推向了物理极限：
* **拒绝虚胖**：最终生成的全功能磁盘镜像（包含内核、引导、应用态与 WebUI）经极限压缩后**体积仅约 15MB**。
* **绝对点对点直通（Bit-Perfect）**：绕过一切中间件，由单二进制纯 C 引擎直连物理声卡硬件节点，拒绝二次污染。
* **极客开放性**：本项目以开源的 **Master Prompt（全流程自动化架构提示词）** 为核心分发媒介，任何玩家都可以通过本地或云端 AI Agent（如 Claude Code, Cursor, Windsurf, DeepSeek Harness 等）一键复现、审计、魔改并生成专属镜像。

---

## ⚡ 核心硬核特性

### 1. 纯物理级的极简微架构
* **Linux 6.6 LTS + PREEMPT_RT 实时微内核**：深度打入实时补丁，开启 1000Hz 高精度定时器与无抖动时钟（`NO_HZ_FULL`），单内核体积控制在 8MB 以内。
* **纯 C 单二进制 PID 1 (`/init`)**：基于 `musl-libc` 全静态链接，彻底剔除 Systemd、D-Bus、Glibc 及任何第三方动态库。整个操作系统用户态只有一个单核进程在调度。
* **全内存驻留运行（RAM-Resident）**：启动后解压进物理内存运行，曲目载入直接通过 `mlock()` 锁定于物理内存，播放期间**彻底切断磁盘 I/O**。

### 2. 针对高端解码器（DAC）的专业治理
* **开机自动解除静音（Hardware Unmute & 0dB Gain）**：许多 USB DAC（XMOS/ESS 方案）在 Linux 启动时默认处于静音或衰减状态。系统开机自动遍历 ALSA 硬件 Mixer，解除所有静音开关并将主音量推至 100%（0dB 直通）。
* **换轨与跨格式防爆音保护（Anti-Pop Protection）**：
  * **软淡入淡出（Soft Ramp）**：PCM 退出/切歌时，对缓冲末端执行微秒级淡出衰减并在零交叉点切断，开声执行平滑淡入。
  * **跨格式静音隔离窗**：PCM ↔ DSD 切换时提供 60ms 缓冲静音窗；DSD 模式下严格填充 **DSD 专用静音编码（`0x69`/`0x96`）**，绝不写入纯零，杜绝 DAC 继电器吸合与模拟爆音。
* **Native DSD 零拷贝直推**：手写 DSF/DFF 裸流解析，支持 ALSA Native DSD 直通（点亮 DAC 物理 DSD 灯），并提供 DoP（DSD over PCM）自动回退机制。

### 3. 全介质存储与开箱即用网络
* **全内建总线支持**：原生内建 NVMe、SATA 以及**闪存存储（板载 eMMC、TF/MicroSD 卡）**控制器。
* **四大多文件系统只读挂载**：支持 **FAT32、exFAT、NTFS 以及 Linux ext4**，自动扫描并以发烧级只读模式（`MS_RDONLY`）挂载，支持通过 Web 挂载 NAS (NFS)。
* **多网卡免配置自举**：自动检测插有网线的物理网口并完成 DHCP 握手，控制台屏幕醒目打印访问 IP。
* **内嵌轻量 Web 控制端**：集成 Mongoose HTTP 引擎，常驻 80 端口，无需下载 App，手机/电脑浏览器直接选歌切歌。

---

## 🚀 如何使用本项目？（两步极速体验）

### 途径 A：使用 Master Prompt 让 Agent 为你全自动编译（推荐）

本项目将整个构建管线（宿主机环境侦测 -> 云镜像下载 -> QEMU 沙箱启动 -> 内核编译 -> 静态 C 链接 -> 镜像打包与 QEMU 自动化断言）封装为了结构化提示词。

1. **下载或复制本仓库中的提示词文件**：
   打开 [`oxbeats_lite_linux_master_prompt.md`](./oxbeats_lite_linux_master_prompt.md)。
2. **交给你的 AI Agent**：
   将该提示词全文直接粘贴给具备终端执行能力的 AI 编程助手（如 Claude Code, Cursor, OpenDevin, Cursor Composer 等）。
3. **静待产出**：
   Agent 会自动调用宿主机的 Python 调度器并拉起 Ubuntu Cloud 极速沙箱，几分钟内自动完成编译、打包与自动化验证，在本地目录下吐出成品镜像：
   ```bash
   oxbeats.img.xz  (~15MB)
   ```

---

### 途径 B：直接下载已打包好的 Release 固件（懒人通道）

如果你不需要修改内核或源码，可以直接前往本仓库的 [Releases 页面](../../releases) 下载最新版本的预编译镜像 `oxbeats.img.xz`。

#### 写入 U 盘 / TF 卡步骤：
1. 准备一个普通 U 盘或 TF/SD 卡（哪怕是闲置的老旧 2GB/4GB 盘均可）。
2. 下载免安装写盘工具：[Rufus](https://rufus.ie/) 或 [balenaEtcher](https://www.balena.io/etcher/)。
3. 选中下载好的 `oxbeats.img.xz`（无需手动解压，写盘工具原生支持），目标设备选择你的 U 盘，点击**“开始写入”**。
4. 将写入完成的 U 盘插入目标 PC/工控机/小主机，开机选择 USB 启动即可。

---

## 🌐 快速上手体验

1. **开机连接**：将小主机接入有线网络并连接 USB DAC 或功放，开机后屏幕将打印：
   ```text
   ====================================================
     OxBeats Lite Linux (Kernel 6.6 RT - RAM Resident)
     Web Control URL: http://192.168.x.x
   ====================================================
   ```
2. **手机/电脑选歌**：在同一局域网的手机或电脑浏览器中输入屏幕上的 IP 地址，即可进入暗黑极客风 Web 控制台。
3. **拷入曲目**：支持随时插入装有曲库的 U 盘、移动硬盘或 TF 卡，系统将在半秒内完成只读挂载并自动刷新曲目列表。

---

## 🛠️ 项目声明与开源协议

* **开源协议**：本项目基于 [GNU General Public License v3.0 (GPL-3.0)](./LICENSE) 开源。欢迎广大发烧友、技术极客自由审计、魔改与分发，但任何衍生版本必须保持开源。
* **致谢**：感谢 Linux Kernel 社区、`PREEMPT_RT` 团队、`musl-libc`、`dr_libs` 以及 `Mongoose` 项目为极致嵌入式音频架构提供的基石。

---
*注：OxBeats Lite Linux 是作为 OxBeats（蛮牛）全内存实时架构的开源轻量参考实现而生。*

# Project Specification & Master Prompt: `oxbeats-lite-linux`

## 1. 项目背景与定位 (Role & Vision)
你是一名顶尖的高性能嵌入式 Linux 系统架构师、实时内核（PREEMPT_RT）专家与高保真音频（Hi-Fi Audio）底层工程师。
你的任务是全自动构建一套名为 **`oxbeats-lite-linux`** 的极简发烧级微内核音乐播放系统（作为旗舰 FreeBSD 蛮牛系统的开源轻量化参考实现）。

### 核心设计哲学
* **极致裸化（Bare-Metal Purity）**：彻底抛弃 `systemd`、`glibc`、`PulseAudio/PipeWire`、Python/Node.js 等臃肿包袱。整个系统仅由一个 **8MB 以内的 RT 实时微内核（`bzImage`）** 和一个由 `musl-gcc` 静态编译的 **纯 C 单二进制进程 `/init`（PID 1）** 组成。
* **纯内存驻留运行（RAM-Resident）**：系统通过 `initramfs` 完整加载于物理内存中运行；音乐播放前通过 `mlock` 将音频数据锁定在物理内存，播放期间实现零磁盘 I/O 读写抖动。
* **Bit-Perfect 原生直通**：绕过一切软件重采样（SRC）与混音插件，直接通过 ALSA `hw:x,x` 硬件节点向 DAC 倾泻 PCM 与 Native DSD 裸流。
* **开箱即用与高普适性**：采用通用实时抢占调度（`SCHED_FIFO`），不硬编码锁定特定 CPU 核心，完美兼容从单核老工控机（J1900/Atom）到现代多核迷你主机（N100/i226）。

---

## 2. 自动化执行架构与沙箱策略 (Execution Pipeline)

为确保在 Windows、macOS、Linux 宿主机上均能 100% 稳定复现并快速产出镜像，执行过程分为四个自动化阶段：

### 阶段一：跨平台宿主机调度与云镜像沙箱拉起 (`build_sandbox.py`)
1. **零依赖 Python 调度器**：
   * 仅使用 Python 3 标准库（`urllib.request`, `subprocess`, `platform`, `shutil`, `socket`），禁止依赖第三方 `pip` 包。
   * 自动检测宿主机的 `qemu-system-x86_64` 与 `qemu-img`；若缺失则输出对应系统的极简 CLI 安装指令（`winget` / `brew` / `apt`）。
2. **高速拉取官方 Cloud Image**：
   * 优先从国内高速镜像源（清华源 TUNA / 阿里云）拉取 `ubuntu-24.04-server-cloudimg-amd64.img`（约 500MB），若超时则自动降级至官方源。
   * 使用 `qemu-img create -f qcow2 -b ...` 秒级创建 20GB 的 Copy-On-Write 差量工作盘（`sandbox.qcow2`），保持原镜像纯净可复用。
3. **无头免交互注入（Cloud-Init）**：
   * 自动生成 `user-data` 与 `meta-data` 并打包为 `seed.img`。
   * 配置默认 root 免密/固定密码登录，自动替换沙箱内 apt 源为国内镜像，并预装编译工具链：`build-essential`, `musl-tools`, `musl-dev`, `libasound2-dev`, `bison`, `flex`, `libncurses-dev`, `libssl-dev`, `libelf-dev`, `bc`, `cpio`, `syslinux`, `syslinux-common`, `xz-utils`, `mtools`, `dosfstools`。
   * 启动后台无头 QEMU 实例（4 核 / 4GB 内存），映射宿主机 `2222` 端口至沙箱 SSH `22` 端口，轮询等待 SSH 通道就绪后自动投递编译任务。

---

## 3. 系统底层组件详细设计规范

### 阶段二：Loader 引导与 PREEMPT_RT 实时微内核编译

#### 2.1 引导层 (Bootloader)
* **方案**：采用轻量级预编译 **Syslinux**（BIOS 引导，直接从宿主机 `/usr/lib/syslinux/` 提取 `mbr.bin`, `ldlinux.c32`, `libcom32.c32`, `libutil.c32`）。
* **启动参数 (`syslinux.cfg`)**：
  * `LINUX /bzImage`
  * `INITRD /rootfs.cpio.xz`
  * `APPEND console=ttyS0 console=tty0 quiet init=/init`（保持普适性，不硬编码 `isolcpus`，但在文档与注释中预留极客自定义核隔离传参说明）。

#### 2.2 内核层 (Kernel 6.6 LTS + PREEMPT_RT)
* **源码与补丁**：下载 Linux `6.6.x` LTS 源码及完全匹配的 `patch-6.6.x-rt.patch.xz`（优先阿里云/清华源），自动合并补丁并校验无 `.rej` 冲突。
* **核心实时性与调度配置**：
  * `CONFIG_PREEMPT_RT=y`（完全实时抢占内核）
  * `CONFIG_HZ_1000=y`（1000Hz 高精度定时器）
  * `CONFIG_HIGH_RES_TIMERS=y`、`CONFIG_NO_HZ_FULL=y`
  * `CONFIG_BLK_DEV_INITRD=y`、`CONFIG_RD_XZ=y`、`CONFIG_DEVTMPFS=y`、`CONFIG_DEVTMPFS_MOUNT=y`
* **全内建硬件驱动矩阵（严禁编译为 `.ko` 模块，全部设为 `=y`）**：
  1. **有线网卡驱动（覆盖 98% 工控机与小主机）**：
     * Intel 千兆/工控主力：`CONFIG_E1000=y`, `CONFIG_E1000E=y`, `CONFIG_IGB=y` (I210/I211)
     * Intel 2.5G 新世代小主机：`CONFIG_IGC=y` (I225-V / I226-V)
     * Realtek 千兆/2.5G 系列：`CONFIG_R8169=y`
     * 虚拟化调试网卡：`CONFIG_VIRTIO_NET=y`
  2. **多总线存储控制器驱动（NVMe / SATA / 闪存介质 / USB）**：
     * NVMe 固态硬盘：`CONFIG_BLK_DEV_NVME=y`（识别 `/dev/nvme*`）
     * SATA / mSATA 控制器：`CONFIG_ATA=y`, `CONFIG_SATA_AHCI=y`（识别 `/dev/sd*`）
     * 板载闪存 eMMC / SD / TF 读卡器：`CONFIG_MMC=y`, `CONFIG_MMC_BLOCK=y`, `CONFIG_MMC_SDHCI=y`, `CONFIG_MMC_SDHCI_PCI=y`, `CONFIG_MMC_SDHCI_ACPI=y`（识别 `/dev/mmcblk*`）
     * USB 移动存储与高速 UASP：`CONFIG_USB_STORAGE=y`, `CONFIG_USB_UAS=y`
  3. **四大本地文件系统与网络存储（原生内核驱动）**：
     * **FAT32**：`CONFIG_VFAT_FS=y`, `CONFIG_FAT_DEFAULT_IOCHARSET="utf8"`
     * **exFAT**：`CONFIG_EXFAT_FS=y`
     * **NTFS**：`CONFIG_NTFS3_FS=y`（使用 Linux 5.15+ 原生高性能 NTFS3 驱动）
     * **Linux ext4**：`CONFIG_EXT4_FS=y`, `CONFIG_EXT4_USE_FOR_EXT2=y`（兼容 ext2/3/4）
     * **NAS 网络共享**：`CONFIG_NFS_FS=y`, `CONFIG_NFS_V4=y`（原生 NFS 客户端支持）
  4. **声卡与解码器（DAC）驱动**：
     * `CONFIG_SOUND=y`, `CONFIG_SND=y`
     * **发烧 USB Audio 异步直通（核心）**：`CONFIG_SND_USB_AUDIO=y`（内建 Linux 原生 XMOS / Amanero / Savitech / ComTrue 等 USB 界面支持与 Native DSD Quirks 白名单）
     * **板载声卡保底**：`CONFIG_SND_HDA_INTEL=y`
* **极致裁减项**：关闭 `CONFIG_DEBUG_INFO`、`CONFIG_DRM`（显卡渲染）、`CONFIG_WIRELESS`（无线网卡）、`CONFIG_BT`（蓝牙）、`CONFIG_SOUND_OSS_CORE` 及一切安全审计钩子。

---

### 阶段三：用户态纯 C 单二进制引擎设计 (`/init`)

使用 `musl-gcc -O3 -static` 将整個应用态编译为单一静态可执行文件 `/init`，涵盖以下六大功能模块：

#### 3.1 系统初始化与网络免配置自举 (DHCP & Static Fallback)
* 挂载 `/proc`, `/sys`, `/dev` (`devtmpfs`), `/mnt`。
* **多网卡热探测**：遍历 `/sys/class/net/`（排除 `lo`），将所有网卡 `IFF_UP` 唤醒，读取 `/sys/class/net/<iface>/carrier` 检测物理网线插入状态（`carrier == 1`）。
* **内置极简 DHCP 客户端**：对插有网线的网卡发起标准 DHCP Discover/Request 握手；若 5 秒内未获取到动态 IP，自动回退绑定至备用静态 IP（如 `192.168.1.188`）。
* **控制台醒目输出**：在屏幕中央大字打印当前工作的网卡名、IP 地址以及 Web 控制端访问 URL（`http://<IP>:80`）。

#### 3.2 全介质存储扫描与发烧级只读挂载 (Storage Auto-Mounter)
* 后台扫描线程自动匹配三类块设备分区节点：
  * SATA / USB：`/dev/sd[a-z][0-9]*`
  * NVMe SSD：`/dev/nvme[0-9]n[0-9]p[0-9]*`
  * 闪存 eMMC / SD / TF：`/dev/mmcblk[0-9]p[0-9]*`
* **只读零干扰挂载**：对探测到的分区，按 `"ext4"`, `"ntfs3"`, `"exfat"`, `"vfat"` 顺序依次调用 `mount()`，强制附加 `MS_RDONLY | MS_NODEV | MS_NOEXEC` 标志挂载至 `/mnt/local/<dev_name>`，杜绝任何元数据回写对系统供电造成的纹波干扰。
* 提供 Web API 接口支持用户输入 NAS 地址，直接通过 `mount -t nfs -o ro,nolock,tcp` 挂载远程 NFS 音乐库至 `/mnt/nfs`。

#### 3.3 解码器（DAC）自动寻优与硬件解静音 (Hardware Unmute & 0dB Gain)
* **痛点解决**：许多高端 USB 解码器（XMOS/ESS/AKM 方案）在 Linux 下开机默认处于 Hardware Mute 或低增益状态。
* **实现机制**：
  1. 扫描 `/proc/asound/cards`，优先选择带有 `USB-Audio` / `DAC` 特征的声卡作为主输出硬件节点（如 `hw:1,0` 或 `hw:0,0`）。
  2. 调用 ALSA Mixer API（`snd_mixer_open` -> `snd_mixer_first_elem`）遍历目标声卡的所有硬件控制单元。
  3. 凡遇到名称包含 `Master`, `PCM`, `Playback`, `DAC`, `Digital`, `Speaker` 的通道：
     * 若存在静音开关（Playback Switch），强制执行 `snd_mixer_selem_set_playback_switch_all(elem, 1)` **解除静音**；
     * 若存在硬件音量衰减（Playback Volume），获取其最大范围并强制设置为 **100% (0dB 满幅直通)**，确保数字信号不经任何精度截断直达 DAC 芯片。

#### 3.4 切歌防爆音与硬件保护机制 (Anti-Pop & Soft Ramp Protection)
* **PCM 微秒级软淡入淡出（Soft Ramp）**：
  * 主动切歌、暂停或停止前，对当前输出缓冲区末尾的 256 个采样帧执行余弦/线性快速衰减至 0（Fade-out），并在零交叉点（Zero-crossing）断开；新曲目开声前 256 帧执行平滑淡入（Fade-in）。
* **PCM ↔ DSD 跨格式切换静音窗（Mute Window）**：
  * 当检测到上下两首曲目发生采样率跳变或 PCM 与 DSD 模式切换时，先向声卡推入 **60ms 的标准静音缓冲帧**，让解码器内部模拟继电器或锁相环（PLL）平稳过渡：
    * **PCM 模式**：填充全 `0x00` 字节；
    * **DSD 模式（关键保护）**：严禁填充 `0x00`（会导致直流偏置爆音），必须严格填充 **DSD 标准静音码 `0x69` 与 `0x96` 交替序列**（或全部 `0x69`）。

#### 3.5 多格式无损解码与 Native DSD 引擎 (Audio Pipeline)
* **运行调度**：音频推流线程调用 `pthread_setschedparam` 设置为 `SCHED_FIFO`（优先级 95），并将待播音频数据通过 `mlock()` 锁入内存。
* **全格式支持矩阵**：
  1. **WAV**：手写轻量 RIFF/fmt/data 头部解析器，支持 16/24/32-bit 最高 768kHz 裸 PCM 直推。
  2. **FLAC**：集成单头文件库 `dr_flac.h`，全内存解压为 32-bit/16-bit 线性 PCM。
  3. **MP3**：集成单头文件库 `dr_mp3.h`。
  4. **CUE 虚拟分轨**：内置状态机解析 `.cue` 索引文件（识别 `FILE`, `TRACK`, `TITLE`, `INDEX 01`），将 `MM:SS:FF`（75帧/秒）转换为精确的采样字节偏移量（Sample Offset），无需物理切割整轨 WAV/FLAC/DSF 即可实现毫秒级选曲与无缝连播（Gapless Playback）。
  5. **DSF / DFF (DSDIFF) 与 Native DSD / DoP 直通**：
     * 手写原生 DSF（解析 `DSD `, `fmt `, `data` 块，处理声道交织 Block Interleave）与 DFF（解析 `FRM8`, `PROP`, `DSD ` 块）无损解析器，支持 DSD64 / DSD128 / DSD256 / DSD512。
     * **优先 Native DSD 握手**：向 ALSA `hw:x,x` 依次请求 `SND_PCM_FORMAT_DSD_U32_BE` 或 `SND_PCM_FORMAT_DSD_U32_LE`，直接将 1-bit 裸 DSD 比特流零拷贝推入 USB 异步端点，点亮解码器 Native DSD 指示灯。
     * **自动 DoP (DSD over PCM) 回退**：若目标 DAC 不支持 Linux Native DSD Quirks，自动将 16-bit DSD 数据加上 `0x05` / `0xFA` 交替标志头，封装为 24-bit/176.4kHz（或 352.8kHz）标准 DoP 帧输出。

#### 3.6 内嵌极简极客风 Web 控制界面 (Embedded Mongoose WebUI)
* 集成单文件 `mongoose.c` / `mongoose.h` 作为 HTTP Server（监听 80 端口）。
* 将响应式单页 HTML5/CSS/JS 界面作为 C 常量字符串直接编译进 `/init` 二进制内（不依赖外部静态网页文件）：
  * 视觉风格：纯黑底色（`#0e0e0e`）+ 荧光绿/青色状态指示（`#00ffcc`），展示当前 DAC 型号、工作模式（`Native DSD` / `DoP` / `Bit-Perfect PCM`）、实时采样率与位深。
  * 功能支持：目录树浏览（本地 NVMe/SATA/eMMC/USB 及 NFS 挂载点）、CUE 虚拟音轨自动展开、播放/暂停/上一下一曲/停止、NAS 挂载配置弹窗。

---

## 4. 镜像打包与 QEMU 端到端自动化验证

### 阶段四：打包 `.img.xz` 成品镜像与回归测试
1. **构建最小 Rootfs 内存盘**：
   * 创建纯净目录结构（`/bin`, `/dev`, `/proc`, `/sys`, `/mnt`, `/etc`），仅放入 `strip -s` 后的静态 `/init`。
   * 使用 `find . -print0 | cpio --null -ov --format=newc | xz -9 --check=crc32 > rootfs.cpio.xz` 极限压缩。
2. **制作可写盘物理镜像 (`oxbeats.img.xz`)**：
   * 使用 `dd` 生成 64MB 全零 raw 镜像 `oxbeats.img`。
   * 使用 `mkfs.vfat -F 32 -n "OXBEATS"` 格式化，并执行 `syslinux --install oxbeats.img` 写入引导区。
   * 通过 `mcopy` 将 `bzImage`、`rootfs.cpio.xz` 和 `syslinux.cfg` 写入镜像。
   * 执行 `xz -9 -T0 --keep oxbeats.img` 生成最终分发包 **`oxbeats.img.xz`**（体积控制在 15MB~20MB 左右，可直接用 Rufus / balenaEtcher 烧录至 U 盘或 eMMC/TF 卡）。
3. **QEMU 自动化闭环测试断言**：
   * 启动测试虚拟机（挂载虚拟 HDA 声卡、虚拟磁盘与端口转发 `8088->80`）。
   * 自动读取控制台输出日志并完成四项断言：
     1. `[PASS]` 内核成功显示 `PREEMPT_RT` 标识并拉起 `/init`；
     2. `[PASS]` 存储引擎与网络 DHCP 初始化完成；
     3. `[PASS]` ALSA 声卡探测成功并完成 Mixer Unmute 0dB 初始化；
     4. `[PASS]` 宿主机请求 `http://127.0.0.1:8088/` 返回 `HTTP 200 OK`。
   * 测试通过后将沙箱内的 `oxbeats.img.xz` 自动回传至宿主机当前工作目录，并清理后台 QEMU 进程。

---

## 5. 给 AI Agent 的执行指令 (Action Trigger)
请严格按照上述四个阶段的设计规范，依次生成并执行：
1. 宿主机跨平台沙箱拉起脚本 `build_sandbox.py`；
2. 沙箱内内核下载、RT 补丁合并与全内建驱动编译脚本 `01_build_rt_kernel.sh`；
3. 完整无省略的纯 C 音频引擎与 Web 服务源码 `ox_init.c`（必须包含完整的 DSF/DFF/CUE/WAV/FLAC/MP3 解析、ALSA Mixer 自动解静音、0x69/0x96 DSD 防爆音静音窗、多分区自动只读挂载与内嵌 WebUI）；
4. 镜像打包与 QEMU 自动化断言测试脚本 `02_pack_and_test.sh`。

确保所有网络请求均具备国内镜像源自动降级能力，全程免人工干预，直至在宿主机目录成功输出可烧录的 `oxbeats.img.xz` 成品镜像。
```

这份 Markdown 文档已经将我们讨论的所有细节（从跨平台 Python 沙箱拉起、Syslinux 引导、6.6 RT 实时内核、网卡/NVMe/SATA/闪存驱动、四大文件系统、声卡解静音、切歌防爆音到 Native DSD 直通）全部结构化整合完毕。您可以直接点击右上角下载或复制全文，交给您的 AI 编程助手直接开跑！

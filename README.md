# OKayPi (Rockchip RK3576) 嵌入式 Linux 6.1 开发指南与全流程构建手册

本项目为基于立创 / DLCV **OKayPi (Rockchip RK3576)** 的嵌入式 Linux 深度定制开发工程，集成 **Linux 6.1 内核（imx334 高性能图像传感器分支）**、**Debian 12 (Bookworm arm64 GNOME)** 原厂加速桌面系统，并已全量托管至个人 GitHub 账号体系。

---

## 目录 (Table of Contents)
1. [硬件平台规格与芯片特性](#1-硬件平台规格与芯片特性)
2. [远端仓库全景与新环境拉取](#2-远端仓库全景与新环境拉取)
3. [系统身份与环境定制配置](#3-系统身份与环境定制配置)
4. [Docker 隔离容器构建流水线](#4-docker-隔离容器构建流水线)
5. [固件一键烧录与敏捷开发指南](#5-固件一键烧录与敏捷开发指南)
6. [AI Agent 自动化技能 (Skill) 速查](#6-ai-agent-自动化技能-skill-速查)

---

## 1. 硬件平台规格与芯片特性

| 模块 | 核心规格与特性 |
| :--- | :--- |
| **主控芯片** | Rockchip RK3576（4× Cortex-A72 @ 2.2GHz + 4× Cortex-A53 @ 1.8GHz） |
| **NPU 算力** | 6 TOPS 强劲端侧 AI 算力，支持 INT4/INT8/INT16/FP16 混合精度 |
| **GPU 图形** | ARM Mali-G52 MC3 图形渲染核心，支持 OpenGL ES 1.1/2.0/3.2, Vulkan 1.1/1.2, OpenCL 2.0 |
| **VPU 编解码** | 4K @ 120fps H.264/H.265/AV1 视频硬解，4K @ 60fps H.264/H.265 视频硬编 |
| **ISP 图像引擎** | 16M 独立 ISP 核心，支持 3DNR, HDR, 畸变矫正与超宽动态范围 |
| **板载内存** | 4GB LPDDR4X / 8GB LPDDR4X 高速内存 |
| **板载存储** | 32GB / 64GB 高速 eMMC 5.1 芯片 |
| **板级系统** | Debian 12 Bookworm (arm64, Linux 6.1 内核, GNOME 桌面) |
| **传感器配置** | IMX334 4K 超高清 CMOS 图像传感器（`kernel:imx334` 分支） |

---

## 2. 远端仓库全景与新环境拉取

所有代码仓库均已全量镜像至个人 GitHub 账号体系（`OKayJH`），主清单默认绑定 `kernel:imx334` 分支与个人仓库：

| 模块目录 | 个人仓库地址 | 默认分支 | 说明 |
| :--- | :--- | :--- | :--- |
| **SDK 清单主库** | `https://github.com/OKayJH/DLCV_SDK.git` | `linux` | 包含 `rk3576_linux_release.xml` |
| **`device/`** | `https://github.com/OKayJH/device.git` | `main` | 板级 defconfig 与打包配置 |
| **`kernel-6.1/`** | `https://github.com/OKayJH/kernel.git` | `imx334` | Linux 6.1 内核与 IMX334 驱动 |
| **`u-boot/`** | `https://github.com/OKayJH/u-boot.git` | `rk3576` | 引导加载程序与 SPL |
| **`rkbin/`** | `https://github.com/OKayJH/rkbin.git` | `rk3576` | 原厂闭源固件与签名打包库 |
| **`debian12/`** | `https://github.com/OKayJH/debian12.git` | `rk3576` | Debian 12 根文件系统与定制包 |

### 新电脑 / 新目录一键拉取指令：
```bash
# 1. 初始化 repo 清单（直接指向个人仓库）
repo init -u https://github.com/OKayJH/DLCV_SDK.git -b linux -m rk3576_linux_release.xml

# 2. 全量并发同步所有子项目
repo sync -c -j$(nproc)
```

---

## 3. 系统身份与环境定制配置

当前生成的 Debian 12 系统固件已预装并固化以下深度定制：

1. **主机名 (Hostname)**：
   - 固定为 **`OKayPi`**（底层 `dlcvcam-init.sh` 脚本已锁死，防止每次开机根据 OTP ID 改写）。
2. **系统用户体系**：
   - 默认普通用户：**`okay`**；
   - 登录与 sudo 密码：**`168428`**；
   - 权限：赋予 `sudo`, `adm`, `video`, `dialout`, `plugdev`, `netdev` 等全量权限；
   - GDM3 桌面自动登录设置为 `okay`，强制开启 Xorg 会话以保证 x11vnc 兼容性。
3. **免密 SSH 直连**：
   - 固化宿主机开发公钥至 `/home/okay/.ssh/authorized_keys` 与 `/root/.ssh/authorized_keys`；
   - 开发机可直接免密连接：`ssh okay@<OKayPi_IP>` 或 `ssh root@<OKayPi_IP>`。
4. **桌面便捷交互**：
   - 预设全局快捷键 **`Ctrl+Alt+T`** 唤起 `gnome-terminal`；
   - 预配置国内清华源（`mirrors.tuna.tsinghua.edu.cn/debian`）与阿里云安全源，保证 `apt-get` 极速顺畅。

---

## 4. Docker 隔离容器构建流水线

### 4.1 架构设计
- **完全隔离**：构建环境运行在基于 Ubuntu 22.04 LTS 的 `okaypi3576-builder:22.04` 容器中，规避宿主机高版本 GCC/Python 兼容性冲突；
- **真实监控**：桌面中央自动弹出 **24pt 超大醒目字号原生 GTK 置顶悬浮小窗**，终端同步展示 30 格高对比度进度条，绝无虚假 Mock 进度；
- **成功自毁**：100% 满格绿字后持续停驻 **2 秒**，让开发者明确感知结果后再优雅退出。

### 4.2 一键构建指令速查

```bash
# 场景 A: 全量出厂固件构建 (从头到尾生成 update.img，包含内核、U-Boot、Debian12与DLCV分区)
./scripts/build_full/start_build.sh all

# 场景 B: 仅重新构建 Debian 12 根文件系统
./scripts/build_full/start_build.sh debian

# 场景 C: 日常内核与驱动敏捷增量编译 (仅更新 boot.img)
./scripts/build_full/start_build.sh kernel

# 场景 D: 仅编译 U-Boot
./scripts/build_full/start_build.sh uboot

# 场景 E: 仅重新打包已有固件为 update.img
./scripts/build_full/start_build.sh updateimg
```

---

## 5. 固件一键烧录与敏捷开发指南

### 5.1 硬件连接与进入烧录模式

```
[步骤 1: 物理接线]
  • 用 Type-C 数据线连接 PC 与 OKayPi 板载【Download / OTG】接口；
  • 接入 12V 直流电源适配器并供电。
        │
        ▼
[步骤 2: 触发握手时序]
  1. 按住板载【BOOT】(或 Maskrom/Recovery) 按键不松开；
  2. 短按一下【RESET】复位按键；
  3. 保持按住【BOOT】按键 2~3 秒后松开。
        │
        ▼
[步骤 3: 握手成功]
  • 宿主机识别到：Vid=0x2207, Mode=Loader 或 Maskrom。
```

### 5.2 烧录指令速查

统一使用 `scripts/flash_firmware/flash_firmware.sh` 脚本，基于 PTY 伪终端无缓冲输出，并自动弹出 24pt 置顶 GTK 烧录小窗：

```bash
# 1. 出厂完整固件一键全量烧录 (默认)
./scripts/flash_firmware/flash_firmware.sh

# 2. 【3 秒极速单分区更新】仅单独烧录内核与设备树 boot 分区 (免全量刷机，重启即生效)
./scripts/flash_firmware/flash_firmware.sh boot

# 3. 仅单独烧录 Debian 12 根文件系统 rootfs 分区
./scripts/flash_firmware/flash_firmware.sh rootfs

# 4. 仅单独烧录引导加载程序 uboot 分区
./scripts/flash_firmware/flash_firmware.sh uboot

# 5. 仅单独烧录 dlcv 专用应用分区
./scripts/flash_firmware/flash_firmware.sh dlcv
```

### 5.3 调试串口参数
- 波特率：**`1500000`**
- 数据位：8，停止位：1，奇偶校验：无，流控：关闭 (1500000 8N1)

---

## 6. AI Agent 自动化技能 (Skill) 速查

在与 AI Assistant 交互时，可直接使用以下口令唤醒对应专项技能：

| 触发意图 / 口令 | 唤醒技能名称 | 执行效果与能力 |
| :--- | :--- | :--- |
| “构建okaypi固件” / “全量构建” / “编译update.img” | [`okaypi3576-build`](file:///.agents/skills/okaypi3576-build/SKILL.md) | 自动检查 Docker 特权容器、拉起 QEMU 跨架构构建、桌面置顶 24pt GTK 真实进度弹窗，生成出厂镜像并停驻 2 秒通知 |
| “烧录okaypi” / “刷机” / “单独烧录boot” | [`okaypi3576-flash`](file:///.agents/skills/okaypi3576-flash/SKILL.md) | 自动检测 upgrade_tool 与开发板握手、PTY 流式读盘、原生 GTK 实时烧写弹窗、烧写完成自动重启 |

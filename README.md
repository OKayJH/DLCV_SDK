# DLCV_SDK — RK3576 Linux SDK Manifest

本仓库是 **DLCV RK3576 Linux SDK** 的 Repo 清单仓库。

---

## 拉取 SDK

```bash
export REPO_URL='https://mirrors.tuna.tsinghua.edu.cn/git/git-repo'
repo init -u https://github.com/OKayJH/DLCV_SDK.git -b linux -m rk3576_linux_release.xml
repo sync -c -j4
```

---

## 构建

### 加载配置

```bash
make dlcvcam_rk3576_debian12_gnome_defconfig
```

### 构建内核

```bash
./build.sh kernel
```

输出：
- `kernel-6.1/arch/arm64/boot/Image`
- `kernel-6.1/arch/arm64/boot/dts/rockchip/dlcvcam-rk3576.dtb`
- `output/firmware/boot.img`

### 构建根文件系统

```bash
rm -f debian12/linaro-rk3576-gnome-rootfs.img
./build.sh debian
```

输出：
- `debian12/linaro-rk3576-gnome-rootfs.img`
- `output/firmware/rootfs.img`

> 必须先删除旧镜像，否则 `build.sh` 会直接复用缓存。

### 构建完整固件

```bash
./build.sh all
```

输出：
- `output/update/Image/update.img` — 完整烧录固件
- `output/firmware/` — 各分区镜像

> **注意**：`./build.sh all` 会先检查各组件缓存。若 kernel / debian / u-boot 等已编译且未变更，会直接复用缓存。如需强制重新编译某组件，先执行对应的 `clean` 命令。

---

## 智能相机环境打包文档

### 环境准备

确认交叉编译器与依赖：

```bash
which aarch64-linux-gnu-gcc
# 确保预编译工具链存在：prebuilts/gcc/linux-x86/aarch64/
```

### 设备端环境打包（dlcv_env.tar.gz）

固件构建时，`mk-bookworm-rootfs.sh` 会从共享目录提取 `dlcv_env.tar.gz` 并解压到 rootfs 的 `/root/dlcv`，作为预置 Python 环境。因此**构建固件前必须先完成设备端打包**。

**打包步骤（在 170 设备上执行）**：

```bash
# 1. 确保 smartcam_bs 已编译安装（camos + camos-admin whl 已安装到 /root/dlcv）
# 2. 停止服务，避免运行时文件被占用
systemctl stop camos camos-admin

# 3. 打包 /root/dlcv 目录
cd /root
tar -czf /tmp/dlcv_env.tar.gz \
  --exclude='*.pyc' \
  --exclude='__pycache__' \
  --exclude='.cache' \
  --exclude='*.log' \
  dlcv

ls -lh /tmp/dlcv_env.tar.gz
```

**拷贝到共享目录**：

170 上 `/share` 通常未挂载，需通过 scp 传回 WSL：

```bash
# 在 WSL 上执行
scp root@192.168.1.170:/tmp/dlcv_env.tar.gz /share/4-智能相机/0-环境安装/dlcv_env.tar.gz
```

> 构建脚本中的固定路径：`DLCV_ENV_TAR="/share/4-智能相机/0-环境安装/dlcv_env.tar.gz"`

---

### 完整固件编译流程

**步骤 1：加载 SDK 配置**

```bash
cd /path/to/dlcv_rk3576_sdk
make dlcvcam_rk3576_debian12_gnome_defconfig
```

**步骤 2：编译完整固件**

```bash
./build.sh all
```

编译过程包含：
1. U-Boot 编译
2. Kernel 编译（含设备树）
3. Debian 12 根文件系统构建（如缓存不存在或已删除）
4. 智能相机 Python 环境打包成 `dlcv.img`
5. 各分区镜像打包（boot.img、rootfs.img、oem.img、userdata.img、dlcv.img 等）
6. 生成 update.img（完整烧录固件）

**输出路径与文件**：

| 文件 | 路径 | 说明 |
|------|------|------|
| 完整固件 | `output/update/Image/update.img` | RKDevTool / SD 卡升级用的完整镜像 |
| boot 分区 | `output/firmware/boot.img` | FIT 格式，含 kernel + dtb + resource |
| rootfs 分区 | `output/firmware/rootfs.img` | Debian 12 GNOME 根文件系统 |
| dlcv 分区 | `output/firmware/dlcv.img` | 智能相机 Python 环境 `/root/dlcv` 独立分区镜像 |
| uboot 分区 | `output/firmware/uboot.img` | 引导程序 |

**步骤 3：固件命名与拷贝**

编译完成后，按以下规范重命名并拷贝到共享目录：

```bash
# 示例：2026-06-09 编译的固件
SRC_IMG="output/update/Image/update.img"
DST_NAME="dlcvcam_rk3576_debian12_gnome_update_$(date +%Y%m%d).img"
DST_DIR="/share/4-智能相机/0-环境安装"

cp -v "$SRC_IMG" "$DST_DIR/$DST_NAME"
ls -lh "$DST_DIR/$DST_NAME"
```

命名规范：`dlcvcam_rk3576_debian12_gnome_update_YYYYMMDD.img`

### 仅更新环境（dlcv 分区）

若只更新了 `/root/dlcv` 环境包 `dlcv_env.tar.gz`，无需重建 rootfs：

```bash
cd debian12
./mk-dlcv-part.sh
cd ..
./build.sh firmware
```

此路径会重新解压环境到 `dlcv_part_src/`，重打 `dlcv.img`，并重新打包 `update.img`，rootfs 复用缓存。

### 根文件系统强制重建

若需重新构建 Debian 根文件系统（如 debian12 仓库有更新、系统依赖或 overlay 变更）：

```bash
rm -f debian12/linaro-rk3576-gnome-rootfs.img
./build.sh debian
# 然后再执行 ./build.sh all 重新打包
```

> 必须先删除旧镜像，否则 `build.sh` 会直接复用缓存。

### 仅更新内核的快捷流程

若只修改了 kernel（设备树 / 驱动 / 配置），无需重新构建 rootfs：

```bash
./build.sh kernel
./build.sh all
```

此时 `all` 会复用已有的 rootfs.img，仅重新编译 kernel 并打包 update.img。

---

## 烧写固件

### 方式一：烧写完整固件（推荐）

使用 **RKDevTool** 或 **SD 卡升级**，直接烧写 `output/update/Image/update.img`。

这是最简单可靠的方式，无需关心分区表和地址。

### 方式二：分区烧写

若使用 RKDevTool **分区烧写**，**必须同时勾选 `parameter.txt` 和对应分区的镜像**。`parameter.txt` 定义了 GPT 分区表，不更新会导致系统无法识别分区位置和大小，从而无法启动。

#### 分区地址表

| 分区 | 起始地址（扇区） | 大小（扇区） | 镜像文件 |
|------|-----------------|-------------|---------|
| uboot | 0x00004000 | 0x00002000 | `output/firmware/uboot.img` |
| misc | 0x00006000 | 0x00002000 | — |
| boot | 0x00008000 | 0x00020000 | `output/firmware/boot.img` |
| recovery | 0x00028000 | 0x00040000 | — |
| backup | 0x00068000 | 0x00010000 | — |
| oem | 0x00078000 | 0x00040000 | `output/firmware/oem.img` |
| userdata | 0x000B8000 | 0x00200000 | `output/firmware/userdata.img` |
| **dlcv** | **0x002B8000** | **0x00600000** | **`output/firmware/dlcv.img`** |
| **rootfs** | **0x008B8000** | **自动扩展** | **`output/firmware/rootfs.img`** |

> 扇区大小为 512 字节。`dlcv` 分区固定 3 GiB，存放 `/root/dlcv` Python 环境；`rootfs` 起始地址为 **0x008B8000**，保持 grow 末位。

### 清理

```bash
./build.sh clean-kernel    # 仅清理内核
./build.sh cleanall        # 清理全部
```

---

## DLCV 定制说明（相较于原版 LubanCat SDK）

### `device`（`OKayJH/device`）

| 修改项 | 说明 |
|--------|------|
| 新增 defconfig | `dlcvcam_rk3576_debian12_gnome_defconfig`：DLCVCAM 专用配置 |
| 新增 defconfig | `tspi_3m_rk3576_debian_bookworm_gnome_defconfig`：TaishanPi-3M 配置 |

### `kernel-6.1`（`OKayJH/kernel`）

| 修改项 | 说明 |
|--------|------|
| 设备树 | 新增 `dlcvcam-rk3576.dts` 及相关 `.dtsi` |
| 相机驱动 | IMX296 支持 on-board 晶振，禁用板级 MCLK |
| 触发功能 | 移植 RK3588 trigger-dev 外部触发到 RK3576 |
| 风扇控制 | 三档 PWM 风扇曲线，适配 DLCVCAM 散热方案 |
| 内核裁剪 | `dlcvcam_rk3576_kernel_cut.config` |

### `debian12`（`OKayJH/debian12`）

| 修改项 | 说明 |
|--------|------|
| 根文件系统定制 | 集成 DLCV 智能相机环境到构建流程 |
| 初始化脚本 | `dlcvcam-init.sh`：基于 OTP CPU ID 设置唯一主机名和 MAC 地址 |
| 启动修复 | 修复 7zz 解压退出码导致编译中断的问题；修复 chroot 执行 shell 问题 |
| 用户配置 | 默认用户 `dlcv`，密码 `dlcvai`；自动登录 GNOME；VNC 预配置 |
| 工具增强 | 预装 zsh、oh-my-zsh、btop、x11vnc、gnome-terminal |

### `u-boot`（`OKayJH/u-boot`）

| 修改项 | 说明 |
|--------|------|
| 设备树 | 默认设备树改为 `rk3576-tspi` |
| 新增 DTS | `arch/arm/dts/rk3576-tspi.dts` |
| 启动顺序 | 调整 bootcmd：优先尝试 `bootrkp`，最后 `distro_bootcmd` |
| eMMC 时序 | 保守时序配置，解决 u-boot 读写 eMMC 报错 |

---

## Manifest 说明

### 远程仓库

| 名称 | 地址 | 用途 |
|------|------|------|
| `lubancat` | `https://github.com/LubanCat/` | LubanCat 官方仓库（默认） |
| `dlcv` | `https://github.com/OKayJH/` | OKayJH 个人定制仓库 |

### 子仓库列表

| 本地路径 | 远程仓库 | 分支 | 说明 |
|----------|----------|------|------|
| `DLCV_SDK` | `OKayJH/DLCV_SDK` | `linux` | 清单仓库自身 |
| `device/rockchip` | `OKayJH/device` | `main` | 构建脚本与设备配置 |
| `kernel-5.10` | `LubanCat/kernel` | `lbc-develop-5.10` | Linux 5.10 内核 |
| `kernel-6.1` | `OKayJH/kernel` | `imx334` | Linux 6.1 内核（IMX334 定制） |
| `u-boot` | `OKayJH/u-boot` | `rk3576` | U-Boot 引导程序 |
| `rkbin` | `OKayJH/rkbin` | `rk3576` | Rockchip 固件二进制 |
| `debian12` | `OKayJH/debian12` | `rk3576` | Debian 12 根文件系统 |
| `tools` | `LubanCat/tools` | `main` | 烧录与打包工具 |
| `prebuilts/gcc/...` | `LubanCat/gcc-arm-10.3...` | — | ARM 交叉编译工具链 |

---

## 常用 Repo 命令

```bash
repo manifest -r              # 查看当前 manifest 信息
repo status                   # 查看所有子仓库状态
repo forall -c 'git log --oneline -5'
repo sync kernel-6.1          # 仅同步指定仓库
repo sync -c -j4              # 更新全部
repo start my-feature --all   # 创建 topic 分支
```

## 开发者工作流

### 单仓库修改

```bash
cd kernel-6.1
git checkout -b dev-fix
git add ... && git commit -m "..."
git push dlcv dev-fix:imx334
```

### 多仓库同时修改

```bash
repo start rk3576-feature-xxx --all
# 修改后分别 commit
repo forall -c 'git push $(git remote | head -1) $(git branch --show-current):rk3576'
```

### 本地修改与 `repo sync` 冲突处理

```bash
repo forall -c 'git stash' && repo sync -c -j4 && repo forall -c 'git stash pop'
# 或强制丢弃本地修改：
# repo forall -c 'git reset --hard HEAD && git clean -fd' && repo sync -c -j4
```

---

Maintained by [OKayJH](https://github.com/OKayJH)

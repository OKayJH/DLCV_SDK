# DLCV_SDK — RK3576 Linux SDK Manifest

本仓库是 **DLCV RK3576 Linux SDK** 的 Repo 清单仓库。

---

## 拉取 SDK

```bash
export REPO_URL='https://mirrors.tuna.tsinghua.edu.cn/git/git-repo'
repo init -u https://github.com/dl-cv/DLCV_SDK.git -b linux -m rk3576_linux_release.xml
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
./build.sh debian
```

输出：
- `debian12/linaro-rk3576-gnome-rootfs.img`
- `output/firmware/rootfs.img`

> 强制重新构建：先删除 `debian12/linaro-rk3576-gnome-rootfs.img`，再执行 `./build.sh debian`。

### 构建完整固件

```bash
./build.sh all
```

输出：
- `rockdev/update.img` — 完整烧录固件
- `output/firmware/` — 各分区镜像

### 清理

```bash
./build.sh clean-kernel    # 仅清理内核
./build.sh cleanall        # 清理全部
```

---

## DLCV 定制说明（相较于原版 LubanCat SDK）

### `device`（`dl-cv/device`）

| 修改项 | 说明 |
|--------|------|
| 新增 defconfig | `dlcvcam_rk3576_debian12_gnome_defconfig`：DLCVCAM 专用配置 |
| 新增 defconfig | `tspi_3m_rk3576_debian_bookworm_gnome_defconfig`：TaishanPi-3M 配置 |

### `kernel-6.1`（`dl-cv/kernel`）

| 修改项 | 说明 |
|--------|------|
| 设备树 | 新增 `dlcvcam-rk3576.dts` 及相关 `.dtsi` |
| 相机驱动 | IMX296 支持 on-board 晶振，禁用板级 MCLK |
| 触发功能 | 移植 RK3588 trigger-dev 外部触发到 RK3576 |
| 风扇控制 | 三档 PWM 风扇曲线，适配 DLCVCAM 散热方案 |
| 内核裁剪 | `dlcvcam_rk3576_kernel_cut.config` |

### `debian12`（`dl-cv/debian12`）

| 修改项 | 说明 |
|--------|------|
| 根文件系统定制 | 集成 DLCV 智能相机环境到构建流程 |
| 初始化脚本 | `dlcvcam-init.sh`：基于 OTP CPU ID 设置唯一主机名和 MAC 地址 |
| 启动修复 | 修复 7zz 解压退出码导致编译中断的问题；修复 chroot 执行 shell 问题 |
| 用户配置 | 默认用户 `dlcv`，密码 `dlcvai`；自动登录 GNOME；VNC 预配置 |
| 工具增强 | 预装 zsh、oh-my-zsh、btop、x11vnc、gnome-terminal |

### `u-boot`（`dl-cv/u-boot`）

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
| `dlcv` | `https://github.com/dl-cv/` | DLCV 定制仓库 |

### 子仓库列表

| 本地路径 | 远程仓库 | 分支 | 说明 |
|----------|----------|------|------|
| `DLCV_SDK` | `dl-cv/DLCV_SDK` | `linux` | 清单仓库自身 |
| `device/rockchip` | `dl-cv/device` | `main` | 构建脚本与设备配置 |
| `kernel-5.10` | `LubanCat/kernel` | `lbc-develop-5.10` | Linux 5.10 内核 |
| `kernel-6.1` | `dl-cv/kernel` | `rk3576` | Linux 6.1 内核（RK3576 定制） |
| `u-boot` | `dl-cv/u-boot` | `rk3576` | U-Boot 引导程序 |
| `rkbin` | `dl-cv/rkbin` | `rk3576` | Rockchip 固件二进制 |
| `debian12` | `dl-cv/debian12` | `rk3576` | Debian 12 根文件系统 |
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
git push dlcv dev-fix:rk3576
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

Maintained by [dl-cv](https://github.com/dl-cv)

# DLCV_SDK — RK3576 Linux SDK Manifest

本仓库是 **DLCV RK3576 Linux SDK** 的 Repo 清单仓库，用于通过 Google Repo 工具统一管理多个 Git 子仓库。

## 仓库信息

| 项目 | 内容 |
|------|------|
| 清单仓库 | `https://github.com/dl-cv/DLCV_SDK` |
| 默认分支 | `linux` |
| 默认清单 | `rk3576_linux_release.xml` |

## 快速开始

### 1. 安装 Repo 工具

```bash
# 方式一：通过包管理器安装
sudo apt update
sudo apt install repo

# 方式二：手动安装最新版
mkdir -p ~/.bin
curl -o ~/.bin/repo https://storage.googleapis.com/git-repo-downloads/repo
chmod a+x ~/.bin/repo
export PATH="${HOME}/.bin:${PATH}"
# 建议将 PATH 写入 ~/.bashrc
```

### 2. 初始化并同步 SDK

```bash
# 创建工作目录
mkdir -p ~/dlcv_rk3576_sdk && cd ~/dlcv_rk3576_sdk

# 初始化 Repo（指定清单仓库、分支、清单文件）
repo init \
  -u https://github.com/dl-cv/DLCV_SDK.git \
  -b linux \
  -m rk3576_linux_release.xml

# 同步所有子仓库（-c 只同步当前 manifest 指定的 commit，-j4 并发 4 个）
repo sync -c -j4
```

### 3. 验证目录结构

同步完成后，顶层目录应包含：

```
.
├── build.sh      -> device/rockchip/common/scripts/build.sh
├── Makefile      -> device/rockchip/common/Makefile
├── README.md     -> device/rockchip/common/README.md
├── rkflash.sh    -> device/rockchip/common/scripts/rkflash.sh
├── common        -> device/rockchip/common
├── kernel        -> kernel-6.1
├── kernel-5.10/
├── kernel-6.1/
├── u-boot/
├── rkbin/
├── tools/
├── debian11/
├── debian12/
├── ubuntu20.04/
├── ubuntu22.04/
├── device/rockchip/
├── prebuilts/
└── .repo/
```

## Manifest 说明

### 远程仓库 (remote)

| 名称 | 地址 | 用途 |
|------|------|------|
| `lubancat` | `https://github.com/LubanCat/` | LubanCat 官方仓库（默认） |
| `dlcv` | `https://github.com/dl-cv/` | DLCV 定制仓库 |

### 子仓库列表 (project)

| 本地路径 | 远程仓库 | 分支 | 说明 |
|----------|----------|------|------|
| `device/rockchip` | `LubanCat/device_rockchip` | `main` | 构建脚本与设备配置 |
| `kernel-5.10` | `LubanCat/kernel` | `lbc-develop-5.10` | Linux 5.10 内核 |
| `kernel-6.1` | `dl-cv/kernel` | `rk3576` | Linux 6.1 内核（RK3576 定制） |
| `u-boot` | `LubanCat/u-boot` | `rk3576` | U-Boot 引导程序 |
| `rkbin` | `LubanCat/rkbin` | `rk3576` | Rockchip 固件二进制 |
| `debian11` | `LubanCat/debian11` | `debian11` | Debian 11 根文件系统 |
| `debian12` | `LubanCat/debian12` | `debian12` | Debian 12 根文件系统 |
| `ubuntu20.04` | `LubanCat/ubuntu20.04` | `ubuntu20.04` | Ubuntu 20.04 根文件系统 |
| `ubuntu22.04` | `LubanCat/ubuntu22.04` | `ubuntu22.04` | Ubuntu 22.04 根文件系统 |
| `tools` | `LubanCat/tools` | `main` | 烧录与打包工具 |
| `prebuilts/gcc/...` | `LubanCat/gcc-arm-10.3...` | — | ARM 交叉编译工具链 |

> 所有项目均使用 `clone-depth="1"` 进行浅克隆，以节省时间和磁盘空间。

## 常用 Repo 命令

```bash
# 查看当前 manifest 信息
repo manifest -r

# 查看所有子仓库状态
repo status

# 对所有子仓库执行 git 命令
repo forall -c 'git log --oneline -5'

# 仅同步指定仓库
repo sync kernel-6.1

# 更新 manifest 本身
repo sync -c -j4

# 开始新特性开发（创建 topic 分支）
repo start my-feature --all
```

## 版本管理

若需发布新版本，请按以下流程操作：

1. 进入本地 `DLCV_SDK` 清单仓库
2. 更新 `rk3576_linux_release.xml` 中各项目的 `revision` 为最新 commit hash
3. 提交并推送至远程
4. 用户执行 `repo sync -c -j4` 即可同步到对应版本

---
Maintained by [dl-cv](https://github.com/dl-cv)

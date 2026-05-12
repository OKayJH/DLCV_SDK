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
| `debian12` | `dl-cv/debian12` | `RK3576` | Debian 12 根文件系统（DLCV 定制） |
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

## 开发者工作流：本地修改如何更新到仓库

### 单仓库修改（以 kernel-6.1 为例）

```bash
cd ~/dlcv_rk3576_sdk/kernel-6.1

# 1. 创建本地开发分支（repo sync 后默认处于 detached HEAD）
git checkout -b dev-rk3576-usb-fix

# 2. 修改代码并提交
git add drivers/usb/dwc3/
git commit -m "fix(dwc3): fix USB3.0 host mode on TaishanPi-3"

# 3. 推送到远程（dlcv/kernel 的 rk3576 分支）
git push dlcv dev-rk3576-usb-fix:rk3576
```

### 多仓库同时修改

```bash
cd ~/dlcv_rk3576_sdk

# 1. 为所有仓库创建统一的 topic 分支
repo start rk3576-feature-xxx --all

# 2. 修改代码后，批量查看状态
repo status

# 3. 在各仓库分别 commit，然后批量推送
repo forall -c 'git push $(git remote | head -1) $(git branch --show-current):rk3576'
```

### 本地修改与 `repo sync` 冲突处理

```bash
# 方式 A：暂存后同步，再恢复
repo forall -c 'git stash'
repo sync -c -j4
repo forall -c 'git stash pop'

# 方式 B：强制丢弃本地修改，完全以 manifest 为准
repo forall -c 'git reset --hard HEAD && git clean -fd'
repo sync -c -j4

# 方式 C：rebase 到最新 manifest 版本
repo sync -c -j4 --rebase
```

### 将修改同步更新到 Manifest（Release 流程）

当各子仓库修改已推送并合入后，更新清单仓库使所有开发者同步到最新代码：

```bash
# 1. 生成当前工作区的完整锁定清单（含实际 commit hash）
cd ~/dlcv_rk3576_sdk
repo manifest -r -o locked_manifest.xml

# 2. 进入清单仓库工作区
cd .repo/manifests

# 3. 更新 rk3576_linux_release.xml 中对应项目的 revision

# 4. 提交并推送清单仓库
git add rk3576_linux_release.xml
git commit -m "release: bump kernel-6.1/u-boot/rkbin for RK3576 v1.1"
git push origin linux

# 5. 用户端执行同步
repo sync -c -j4
```

---

## 版本管理

若需发布新版本，请按以下流程操作：

1. 进入本地 `DLCV_SDK` 清单仓库
2. 更新 `rk3576_linux_release.xml` 中各项目的 `revision` 为最新 commit hash
3. 提交并推送至远程
4. 用户执行 `repo sync -c -j4` 即可同步到对应版本

---
Maintained by [dl-cv](https://github.com/dl-cv)

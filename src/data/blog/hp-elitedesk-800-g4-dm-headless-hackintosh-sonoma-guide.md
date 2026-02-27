---
author: Cola
pubDatetime: 2026-02-27T12:00:00+08:00
title: HP EliteDesk 800 G4 DM 黑苹果无头服务器安装指南（Sonoma 14.x）
featured: false
draft: false
tags:
  - Hackintosh
  - macOS
  - HP
  - Homelab
description: 面向 HP EliteDesk 800 G4 DM（i5-8500T/8600T）的黑苹果无头服务器完整安装指南，覆盖制盘、EFI 定制、BIOS 配置、系统安装与远程运维上线全流程。
---

# HP EliteDesk 800 G4 DM 黑苹果无头服务器安装指南（终极保姆版）

目标系统：macOS Sonoma 14.x（最新版）  
硬件配置：i5-8500T / 8600T，无无线网卡/蓝牙，纯有线网络连接  
使用场景：7x24 小时无头服务器（Headless Server），无物理显示器，依靠远程桌面管理  
制盘环境：Mac mini M4

## 第一阶段：准备工作与软件下载

在 Mac mini M4 上先准备以下硬件和软件。

### 1. 硬件准备

- 一个 16GB 或以上的优质 U 盘（会被全盘格式化，先备份数据）。
- 一个 DP 接口显卡欺骗器（Dummy Plug，搜索关键词可用“DP 显卡欺骗器 1080P”或“4K 60Hz”）。
- 千兆网线、显示器、键盘鼠标（仅安装和 BIOS 配置阶段使用）。

### 2. 核心软件下载

- 完美 EFI 引导包：
  - 下载地址：[cmalf / HP EliteDesk 800 G4 Releases](https://github.com/cmalf/HP-EliteDesk-800-G4-G5-Hackintosh/releases)
  - 操作：下拉到带有 `Sonoma 14.7.1` 或类似标题的版本（避开 Tahoe），下载 `xxx-EFI.zip` 并解压。
- OpenCore Configurator（OCC）：
  - 下载地址：[OCC 官网下载页](https://mackie100projects.altervista.org/download-opencore-configurator/)
  - 操作：下载 Latest Version，拖入“应用程序”文件夹。Sonoma 版 EFI 通常基于 OpenCore 1.0.1 左右，用最新版 OCC 向下兼容即可。
- BetterDisplay（可选）：
  - 下载地址：[BetterDisplay Releases](https://github.com/waydabber/BetterDisplay/releases)
  - 操作：后期若 DP 欺骗器分辨率异常，用它强制设置 1080P 60Hz。

### 3. 下载 macOS Sonoma（14.x）完整安装程序

由于 App Store 可能隐藏旧版本，建议用终端下载官方完整安装器。

```sh
softwareupdate --list-full-installers
```

找到 14.x 中版本号最高的一个（如 14.8），执行：

```sh
softwareupdate --fetch-full-installer --full-installer-version 14.8
```

下载完成后，“应用程序”目录会出现“安装 macOS Sonoma”。

## 第二阶段：制作离线安装 U 盘

1. 格式化 U 盘（磁盘工具）：
- 显示所有设备，选择 U 盘顶层物理盘。
- 抹掉参数：
  - 名称：`USB`
  - 格式：`Mac OS 扩展（日志式）`
  - 方案：`GUID 分区图`

2. 写入 macOS 安装器：

```sh
sudo /Applications/Install\ macOS\ Sonoma.app/Contents/Resources/createinstallmedia --volume /Volumes/USB
```

输入开机密码并确认 `Y`，等待出现 `Install media now available`。

## 第三阶段：定制无头服务器专属 EFI

1. 打开 OCC，菜单选择 `Mount EFI`，挂载 U 盘 EFI 分区。  
2. 将解压后的 `EFI` 文件夹整体复制到 U 盘 EFI 分区根目录。  
3. 打开 U 盘 `EFI/OC/config.plist`，执行以下定制：

### A. 清理多余驱动（Kernel -> Add）

- 取消启用：
  - `AirportItlwm.kext`
  - `IntelBluetoothFirmware.kext`
  - `IntelBTPatcher.kext`
  - `BlueToolFixup.kext`
- 必须保留：
  - `IntelMausi.kext`（有线网卡）

提示：若看到 `SampleUsbKexts`（如 `USBPorts-noHS14`），本方案默认不使用，可忽略。

### B. 优化启动参数防黑屏（NVRAM -> boot-args）

- 在 `7C436110...` 对应项中编辑 `boot-args`。
- 添加参数：`igfxonln=1 keepsyms=1`
- 删除参数：`darkwake=x`、`igfxagdc=0`、`igfxfw=2` 等
- 参考示例：`keepsyms=1 debug=0x100 alcid=xx igfxonln=1`

### C. 隐藏启动菜单（Misc -> Boot）

- `ShowPicker`：`False`
- `Timeout`：`0`

保存并退出，拔出 U 盘。

## 第四阶段：配置 HP 主板 BIOS

首次安装阶段建议连接真实显示器和键鼠。

1. 开机按 `F10` 进 BIOS。若历史改动较多，先 `Apply Factory Defaults and Exit` 恢复默认，再重进设置。  
2. `Advanced -> Boot Options`：
- `Fast Boot` -> `Disable`
- `CD-ROM Boot` -> `Disable`
- `Network (PXE) Boot` -> `Disable`
- `After Power Loss` -> `Power On`
3. `Advanced -> Secure Boot Configuration`：
- 选择 `Legacy Support Disable and Secure Boot Disable`
- 若重启时提示输入随机数字确认，按提示完成
4. `Advanced -> System Options`：
- `Virtualization Technology (VTx)` -> `Enable`
- `Virtualization Technology for Directed I/O (VTd)` -> `Enable`
- `M.2 WLAN/BT` -> `Disable`
5. `Advanced -> Built-in Device Options`：
- `Wake On LAN` -> `Disable`
- `Video Memory Size` -> `64MB`
- `Audio Device` -> `Enable`
6. `Advanced -> Power Management Options`：
- `Runtime Power Management` -> `Enable`
- `Extended Idle Power States` -> `Enable`
7. `F10` 保存退出（`Save Changes and Exit`）。

## 第五阶段：正式安装 macOS

1. U 盘插到 HP 800 G4 背部 USB 口。  
2. 开机按 `F9` 选择 UEFI U 盘启动。  
3. 进入恢复模式后，打开“磁盘工具”：
- 显示所有设备
- 抹掉内部硬盘
- 名称：`Macintosh HD`
- 格式：`APFS`
- 方案：`GUID 分区图`
4. 返回恢复主界面，选择“安装 macOS Sonoma”，目标盘选刚抹掉的内部硬盘。  
5. 安装过程会自动重启多次，不要拔 U 盘，不要断电。

## 第六阶段：服务器收尾与无头上线

进入系统桌面后，完成以下关键动作。

### 1. 迁移 EFI 到内置硬盘

- 使用 OCC 同时挂载 U 盘与系统盘 EFI 分区。
- 将 U 盘中的 `EFI` 文件夹复制到系统盘 EFI。
- 拔掉 U 盘重启，验证可独立引导。

### 2. 生成专属三码（强烈建议）

默认 EFI 里的序列号是公开值，不建议直接登录 Apple ID。

- 用 OCC 打开系统盘 `EFI/OC/config.plist`
- 在 `PlatformInfo` 中确认 `SystemProductName = Macmini8,1`
- 为 `SystemSerialNumber`、主板序列号和 UUID 重新生成
- 保存并重启生效

### 3. 配置远程管理

- `系统设置 -> 通用 -> 共享`：
  - 打开“屏幕共享”
  - 打开“远程登录（SSH）”
- 建议为主机配置静态 IP（路由器 DHCP 绑定或手动网络设置）。

### 4. 配置防睡死

- `系统设置 -> 用户与群组`：开启自动登录
- `系统设置 -> 显示器 -> 高级`：开启“当显示器关闭时防止自动睡眠”
- `系统设置 -> 节能`：开启“唤醒以供网络访问”

### 5. 切换到无头模式

1. 关机并拔掉真实显示器。  
2. 插入 DP 显卡欺骗器。  
3. 开机。  
4. 在 Mac mini M4 上按 `Cmd + K`，输入 `vnc://你的HP主机IP` 连接。  
5. 在远程桌面中将分辨率调为 1080p（1920x1080）；若异常可用 BetterDisplay 调整。

至此，HP EliteDesk 800 G4 DM 黑苹果无头服务器即完成上线。

---
layout: default
title: openKylin 3.0 空间智能 Alpha · 奇遇 MIX 安装指南
permalink: /wiki/mix-spatial-alpha/
---

# openKylin 3.0 空间智能 Alpha · 奇遇 MIX 安装指南

本版面向奇遇 MIX（A081、SM8250）。默认启动到 UKUI Wayland 桌面。空间桌面、原生 XR 应用和调试工具随镜像安装。进入立体空间会话还需要下文列出的外部运行时。

## 安装条件

- 头显能正常进入已取得 Magisk root 的 Android B，已开启 USB 调试并授权电脑。核对型号为 A081。
- 头显已有名为 `osstore` 的 GPT 分区，文件系统为 ext4，至少有 24 GiB 加 64 MiB 可用。系统镜像保存在该分区的独立目录中。仅有原厂分区布局的机器不能直接使用本安装流程。遇到这一情况请停在检查步骤。
- 使用 Linux 电脑，准备 Python 3.8 或更高版本、Android Platform Tools（`adb`）、OpenSSL、zstd 和 GNU Coreutils。电脑至少预留 32 GiB，用于下载、解压和保存本机原 recovery。给头显充足电，保持 USB 连线稳定。
- 原 Android 和已有 osstore 文件应另有备份。安装器在电脑保存这台头显原有的完整 `recovery_b`。每台头显使用独立备份目录。

## 下载和解压

从同一个发布目录下载以下文件，放在同一空目录。另可下载发布说明、构建清单和软件包清单。

| 文件 | 用途 |
| --- | --- |
| `openKylin-3.0-Spatial-Alpha-MIX-arm64.img.zst` | 压缩的可写 ext4 根文件系统，解压得到 24 GiB 的 `.img` |
| `openKylin-3.0-Spatial-Alpha-MIX-recovery.img` | A081 专用启动镜像，100 MiB |
| `install-mix.py` | 检查设备、安装系统、恢复原 recovery |
| `SHA256SUMS` | 发布文件 SHA-256 |

本批次压缩系统镜像的 SHA-256 为 `6fced2a48fdf68caa8aaa69b4ef0b5b5cfe873f007c9cc5809e33df1da7294d1`，解压后的原始镜像为 `f37a7611a9344086805a9a52f2d29e6e740d972150f04f9ac114627661acf3ab`，recovery 为 `1cf9df52553acbe09a450c36c4b2eacc90e1b877cce3543643d6340040729778`。

先与官网下载页公布的 SHA-256 比对，再执行以下命令。网站若同时提供原始 `arm64.img`，可直接下载原始文件并跳过解压。

```sh
sha256sum -c --ignore-missing SHA256SUMS
zstd -d --sparse openKylin-3.0-Spatial-Alpha-MIX-arm64.img.zst
sha256sum -c --ignore-missing SHA256SUMS
adb devices -l
python3 install-mix.py check
```

`check` 只读挂载 osstore 并报告容量。设备型号、Android 插槽、root、分区或文件系统检查报错时，先解决该项再继续。多台设备同时连接时，对每个安装器命令都传 `--serial <ADB序列号>`。

## 安装和首次启动

```sh
python3 install-mix.py install --backup ./mix-device-backup
```

先核对型号、剩余空间和备份。依提示输入 `INSTALL`，为本机的 `mix` 用户设置密码。安装器先传输完整根镜像并在头显读回 SHA-256，然后才更新 `recovery_b` 并读回全分区。传输 24 GiB 和两次校验可能耗时较长。原 Android 的标定文件和传感器注册表存在时，安装器会在这台头显内复制，供该机首次启动使用。请保管 `mix-device-backup/`。

看到“系统和 recovery 均已读回校验”后执行：

```sh
adb reboot recovery
```

启动程序按 GPT 分区名寻找 `osstore`，从 `/systems/openkylin-alpha/rootfs.img` 启动。首次启动会生成本机身份，并应用安装时设置的密码。桌面账户是 `mix`。如果 USB ACM 串口可用，可从 Linux 电脑用串口终端连接对应的 `/dev/ttyACM*`，以同一用户和密码登录。在头显系统内核对：

```sh
cat /etc/a081-rootfs-id
findmnt /
findmnt /osstore
systemctl status display-manager.service --no-pager
dpkg-query -W xr-shell-session xr-cinema rvvr-saberv
```

应看到标记 `a081-openkylin-3.0-arm64`，根文件系统来自 loop 设备，外层 ext4 挂载在 `/osstore`。请分别检查实际画面、头姿、手柄按键、控制器射线和应用有效帧。软件包存在只说明已安装。

## 空间会话和内容

系统已安装 Monado 驱动、Turnip、传感器与头部/手柄追踪、手部后端、空间窗口和任务栏、青空环境、XR 影院、SaberV 以及调试工具。影院可使用自己的媒体文件。SaberV 需要用户导入有权使用的曲目。手部模型通过 `sudo xr-hand-import-assets --from-bundle <本地模型包目录>` 导入，还需按手部跟踪组件说明配置本机标定并启用手部会话。

立体会话需要自行准备有权使用且版本相配的以下文件。缺少其中任一项时，请先使用 UKUI 桌面。

| 本机路径 | 内容和检查 |
| --- | --- |
| `/opt/steamvr/` | 原生 Linux ARM64 SteamVR 运行时，包含 `bin/linuxarm64/vrserver`、`vrstartup` 和 `steamxr_linuxarm64.json`。工程基线使用 2.17.10 |
| `/usr/bin/gamescope`、`/usr/bin/gamescopereaper` | 配套 ARM64 构建。Gamescope 须支持 `--allow-deferred-backend --backend openvr --vr-session-manager` |
| `/usr/lib/aarch64-linux-gnu/xr/monado/pose-watch` | 与本版 xr-monado 源码锁相符、由用户在本地构建的姿态观察器。构建入口见对应组件的 `tools/pose-watch/README.md` |
| `/opt/mix-alpha-xr/tracking/input-config/calibration.xml` 与 `/usr/share/qcom/sm8250/iqiyi/a081/sensors/registry/` | 只能使用正在安装的这台头显的标定和注册表。安装器会从可读的原厂 persist 数据导入，并按需在同一台设备上复制可用的 SLPI 直接通道库 |

可以从已挂载的本地介质复制用户自行准备的运行时。下例假定本地目录中分别有完整的 `steamvr/`、`gamescope/bin/` 和用户构建的 `pose-watch`。先用 `df -h /` 确认剩余容量，并保留 SteamVR 文件原有的子目录和符号链接。

```sh
cd /你的本地介质目录
sudo cp -a steamvr /opt/steamvr
sudo chown -R root:root /opt/steamvr
sudo install -m 0755 gamescope/bin/gamescope gamescope/bin/gamescopereaper /usr/bin/
sudo install -m 0755 pose-watch /usr/lib/aarch64-linux-gnu/xr/monado/pose-watch
ldd /usr/bin/gamescope
gamescope --allow-deferred-backend --vr-session-manager --help
```

首次配置时使用以上路径。若已安装其他版本，请先备份原文件，按所用 Gamescope 构建说明安装配套库与资源。确认依赖库和本机标定后，从本机终端启动限时会话：

```sh
sudo env QIYU_VR_SESSION_PROFILE=/etc/xr/session.json xr-session start --seconds 60
sudo env QIYU_VR_SESSION_PROFILE=/etc/xr/session.json xr-session status
sudo env QIYU_VR_SESSION_PROFILE=/etc/xr/session.json xr-session shutdown
```

[姿态观察器构建说明](https://github.com/bitsk/xr-monado/tree/main/tools/pose-watch) · [手部跟踪与本机模型导入](https://github.com/bitsk/xr-hand-tracking) · [空间会话管理](https://github.com/bitsk/xr-session-manager)

## 返回 Android 或恢复

从 openKylin 执行 `sudo systemctl reboot` 正常重启，默认进 Android B。要再次进入 openKylin，从 Android 执行 `adb reboot recovery`。

要恢复安装前的 recovery，先进入同一台头显的 Android B，连接 USB，保留本版安装器、校验清单和原备份：

```sh
python3 install-mix.py restore --backup ./mix-device-backup
```

输入 `RESTORE` 后，工具核对设备身份、备份长度和 SHA-256，再写回并校验原 `recovery_b`。恢复不会删除 osstore 中的 Alpha 根镜像，里面仍有账户、文档及本机标定，请按个人数据管理。遇到断线或读回失败，应留在 Android，重新连接原设备并用同一份备份执行恢复。不要使用其他头显的 recovery 备份。根文件系统镜像只作为文件放入 osstore，切勿用 `dd` 写到 UFS 整盘、userdata、boot 或 recovery 分区。

## 验证范围

通用安装流程依赖目标机已有 ext4 osstore。本批次通过 ext4 全盘只读检查。通用 ARM64 QEMU 已使用公开版 ramdisk 和最终根镜像进入登录，确认根目录、osstore、UID 1000 以及 LightDM 服务。构建清单记录软件包、镜像摘要与离线验证。真机显示、空间会话、手柄/手势、音视频和 Android 往返仍以本批次完整镜像的实机验收为准。现场反馈请附型号、镜像摘要、失败步骤和脱敏日志；请保留本机标定、备份与密码在设备所有者手中。

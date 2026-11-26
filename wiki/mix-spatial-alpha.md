---
layout: default
title: openKylin 3.0 空间智能 Alpha · 奇遇 MIX 安装指南
permalink: /wiki/mix-spatial-alpha/
---

# openKylin 3.0 空间智能 Alpha · 奇遇 MIX 安装指南

本版供奇遇 MIX（A081、SM8250）使用。从 Android 进入 recovery 后，openKylin 会自动启动空间桌面。镜像已内置原生 ARM64 SteamVR 2.17.10、配套 Gamescope、姿态观察器、Monado 驱动和 XR Shell。安装和首次启动均可离线完成。

## 安装条件

- 头显能正常进入已取得 Magisk root 的 Android B，已开启 USB 调试并授权电脑。核对型号为 A081。
- 头显已有名为 `osstore` 的 GPT 分区，文件系统为 ext4，至少有 24 GiB 加 64 MiB 可用。系统镜像保存在该分区的独立目录中。仅有原厂分区布局的机器不能直接使用本安装流程。遇到这一情况请停在检查步骤。
- 使用 Linux 电脑，准备 Python 3.8 或更高版本、Android Platform Tools（`adb`）、OpenSSL、GNU tar、XZ Utils 和 GNU Coreutils。电脑至少预留 36 GiB，用于下载、解压和保存本机原 recovery。给头显充足电，保持 USB 连线稳定。
- 原 Android 和已有 osstore 文件应另有备份。安装器在电脑保存这台头显原有的完整 `recovery_b`。每台头显使用独立备份目录。若原来通过 recovery 启动 Debian 等系统，安装 Alpha 会更换该启动入口。恢复本机原 recovery 后可重新使用。

## 下载和解压

从发布页下载同一版本的镜像包 `openKylin-3.0-Spatial-Alpha-MIX.tar.xz`、整包校验文件 `openKylin-3.0-Spatial-Alpha-MIX.tar.xz.sha256`、镜像校验清单 `SHA256SUMS` 和安装器 `install-mix.py`。将这四个文件放进一个空目录，在该目录执行以下命令。压缩包根目录只有主镜像和 recovery 两个文件，解压后它们与安装器位于同一目录。

```sh
sha256sum -c openKylin-3.0-Spatial-Alpha-MIX.tar.xz.sha256
tar -xJf openKylin-3.0-Spatial-Alpha-MIX.tar.xz
sha256sum -c SHA256SUMS
adb devices -l
python3 install-mix.py check
```

| 包内镜像 | 用途 |
| --- | --- |
| `openKylin-3.0-Spatial-Alpha-MIX-arm64.img` | 24 GiB 可写 ext4 系统盘，安装器把它放在 osstore 的独立目录 |
| `openKylin-3.0-Spatial-Alpha-MIX-recovery.img` | 100 MiB 启动镜像，由安装器写入并读回校验 `recovery_b` |

`check` 只读挂载 osstore 并报告容量。设备型号、Android 插槽、root、分区、文件系统、原厂标定或传感器注册表检查报错时，先解决该项再继续。多台设备同时连接时，对每个安装器命令都传 `--serial <ADB序列号>`。

## 安装和首次启动

```sh
python3 install-mix.py install --backup ./mix-device-backup
```

先核对型号、剩余空间和备份。依提示输入 `INSTALL`，为本机的 `mix` 用户设置密码。安装器先传输完整根镜像并在头显读回 SHA-256，然后才更新 `recovery_b` 并读回全分区。传输 24 GiB 和两次校验可能耗时较长。原 Android 的标定文件和传感器注册表存在时，安装器会在这台头显内复制，供该机首次启动使用。请保管 `mix-device-backup/`。

看到“系统和 recovery 均已读回校验”后执行下面的命令。

```sh
adb reboot recovery
```

启动程序按 GPT 分区名找到 `osstore`，挂载 `/systems/openkylin-alpha/rootfs.img`。首次开机会应用安装时设置的密码和该头显的原厂标定。默认目标 `mix-alpha-xr.target` 等待头显面板和 DSP 就绪，然后通过 `xr-session` 启动追踪、合成器和空间桌面。会话持续运行，到用户退出或关机为止。

## 空间会话与诊断

正常启动后应在双眼中看到空间桌面。移动头部检查视角，再用物理手柄打开任务栏或 XR 影院。可使用自己的媒体文件和 SaberV 曲谱。可选手部追踪使用独立模型，其导入方式见[手部追踪说明](https://github.com/bitsk/xr-hand-tracking)。

若没有画面，可从 Linux 电脑连接头显的 USB ACM 串口，以 `mix` 和安装时设置的密码登录，运行以下命令检查启动目标、设备和会话日志。

```sh
systemctl get-default
findmnt /
findmnt /osstore
systemctl status mix-alpha-xr.service qiyu-openkylin-xr.service --no-pager
sudo journalctl -b -u mix-alpha-xr.service -u qiyu-openkylin-xr-tracking.service -u qiyu-openkylin-xr.service --no-pager
sudo env QIYU_VR_SESSION_PROFILE=/etc/xr/session.json xr-session status
```

`mix-alpha-xr.service` 负责发起会话，实际运行由 `qiyu-openkylin-xr.service` 托管。确认空间会话可用时应同时检查有效追踪和头显双眼画面。

如需临时使用 UKUI 普通桌面，可在会话正常运行时执行下列命令。下一次从 Android 进入 recovery，仍会自动进入 XR。

```sh
sudo env QIYU_VR_SESSION_PROFILE=/etc/xr/session.json xr-session stop
sudo systemctl start display-manager.service
```

## 返回 Android 或恢复

从 openKylin 执行 `sudo systemctl reboot` 正常重启，默认进 Android B。要再次进入 openKylin，从 Android 执行 `adb reboot recovery`。

要恢复安装前的 recovery，先进入同一台头显的 Android B，连接 USB，保留本版安装器、校验清单和原备份。

```sh
python3 install-mix.py restore --backup ./mix-device-backup
```

输入 `RESTORE` 后，工具核对设备身份、备份长度和 SHA-256，再写回并校验原 `recovery_b`。恢复不会删除 osstore 中的 Alpha 根镜像，里面仍有账户、文档及本机标定，请按个人数据管理。遇到断线或读回失败，应留在 Android，重新连接原设备并用同一份备份执行恢复。不要使用其他头显的 recovery 备份。根文件系统镜像只作为文件放入 osstore，切勿用 `dd` 写到 UFS 整盘、userdata、boot 或 recovery 分区。

## 验证范围

所有内置运行时均来自锁定输入。ARM64 QEMU 用发布版 ramdisk 启动了完整根镜像。虚拟机确认默认 XR 服务自动运行、普通桌面未启动，服务在发现缺少真实头显标定时输出诊断。账户登录、sudo 和关机卸载通过。本批次尚待 MIX 真机验证冷启动后的双眼有效帧、头部和手柄跟踪、退出和返回 Android。反馈请附型号、镜像摘要和脱敏日志，妥善保管原厂标定及本机备份。

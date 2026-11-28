---
layout: default
title: openKylin 3.0 空间智能 Alpha · 奇遇 MIX 安装指南
permalink: /wiki/mix-spatial-alpha/
---

# openKylin 3.0 空间智能 Alpha · 奇遇 MIX 安装指南

本版供奇遇 MIX（A081、SM8250）使用。从当前 Android 槽进入 recovery 后，openKylin 默认尝试启动空间桌面。镜像已内置原生 ARM64 SteamVR 2.17.10、配套 Gamescope、姿态观察器、Monado 驱动和 XR Shell。安装和首次启动均可离线完成。

## 现有 Alpha 镜像的适用范围

若主镜像 SHA-256 以 `b0efe26d` 开头，内置的 MIX 追踪程序仍要求参考设备的 XML 摘要。即使安装成功，其他头显也可能只看到静止画面或平面桌面。此修订版安装器可以保留当前槽位并导入本机资料。修复追踪启动需要更新主镜像。若 recovery 摘要以 `8605b996` 开头，Windows 也可能无法枚举 USB 串口。新 USB 描述符需要更新 recovery。希望装完直接使用 XR 的用户，请等这两份镜像更新并完成真机验证后再安装。

## 安装条件

- 头显型号为 A081，当前运行的 Android 已取得 Magisk root，USB 调试已授权。A 槽和 B 槽都可以使用。安装器从当前系统读取槽位，A 槽对应 `recovery_a`，B 槽对应 `recovery_b`。整个流程保留当前槽位。
- 头显已有名为 `osstore` 的 GPT 分区，文件系统为 ext4，至少有 24 GiB 加 64 MiB 可用。系统镜像保存在该分区的独立目录中。若 `check` 提示缺少 `osstore`，请停在这一步。此安装器只使用已有分区，切槽不能创建 `osstore`。
- 使用 Linux 电脑，准备 Python 3.8 或更高版本、Android Platform Tools（`adb`）、OpenSSL、GNU tar、XZ Utils 和 GNU Coreutils。电脑至少预留 36 GiB，用于下载、解压和保存本机原 recovery。给头显充足电，保持 USB 连线稳定。
- 原 Android 和已有 osstore 文件应另有备份。安装器在电脑保存当前槽位的完整 recovery、槽位和设备身份。每台头显、每个槽位使用独立备份目录。若原来通过 recovery 启动 Debian 等系统，安装 Alpha 会更换该启动入口。恢复本机原 recovery 后可重新使用。

## 下载和解压

可以先用体积较小的文档脚本包检查头显。把 `openKylin-3.0-Spatial-Alpha-MIX-docs-scripts.tar.xz` 和对应的 `.sha256` 放进同一空目录，在 Linux 终端运行以下命令。

```sh
sha256sum -c openKylin-3.0-Spatial-Alpha-MIX-docs-scripts.tar.xz.sha256
tar -xJf openKylin-3.0-Spatial-Alpha-MIX-docs-scripts.tar.xz
adb devices -l
python3 install-mix.py check
```

确认输出显示预期的 A/B 槽位、对应的 recovery 和可用空间后，把配套镜像包下载到这个目录，再执行以下命令。

```sh
sha256sum -c openKylin-3.0-Spatial-Alpha-MIX.tar.xz.sha256
tar -xJf openKylin-3.0-Spatial-Alpha-MIX.tar.xz
sha256sum -c SHA256SUMS
```

| 包内镜像 | 用途 |
| --- | --- |
| `openKylin-3.0-Spatial-Alpha-MIX-arm64.img` | 24 GiB 可写 ext4 系统盘，安装器把它放在 osstore 的独立目录 |
| `openKylin-3.0-Spatial-Alpha-MIX-recovery.img` | 100 MiB 启动镜像，由安装器写入并读回校验当前槽位的 recovery |

`check` 只读挂载 osstore 并报告容量。设备型号、Android 插槽、root、分区、文件系统、原厂标定或传感器注册表检查报错时，先解决该项再继续。若连接了多台设备，从 `adb devices -l` 选择目标，对每个安装器命令都传 `--serial <ADB序列号>`，后续重启也用 `adb -s <ADB序列号> reboot recovery`。已在 A 槽取得 root 的用户可直接留在 A 槽。整个安装流程均从正常启动的 Android 发起，无需进入 fastboot 或 EDL。

## 安装和首次启动

```sh
python3 install-mix.py install --backup ./mix-device-backup
```

先核对型号、剩余空间和备份。依提示输入 `INSTALL`，为本机的 `mix` 用户设置密码。安装器先传输完整根镜像并在头显读回 SHA-256，然后才更新刚才确认的 recovery 分区并读回全分区。传输 24 GiB 和两次校验可能耗时较长。安装器在头显内复制该机的原厂标定、传感器注册表，以及原厂提供的注册表版本、索引和传感器配置。请保管 `mix-device-backup/`。

看到“系统和 recovery 均已读回校验”后执行下面的命令。

```sh
adb reboot recovery
```

启动程序按 GPT 分区名找到 `osstore`，挂载 `/systems/openkylin-alpha/rootfs.img`。首次开机会应用安装时设置的密码和该头显的原厂标定。默认目标 `mix-alpha-xr.target` 等待头显面板和 DSP 就绪，然后通过 `xr-session` 启动追踪、合成器和空间桌面。会话持续运行，到用户退出或关机为止。

## 空间会话与诊断

正常启动后应在双眼中看到空间桌面。移动头部检查视角，再用物理手柄打开任务栏或 XR 影院。可使用自己的媒体文件和 SaberV 曲谱。可选手部追踪使用独立模型，其导入方式见[手部追踪说明](https://github.com/bitsk/xr-hand-tracking)。

若画面是黑屏或平面桌面，可从 Linux 电脑连接头显的 USB ACM 串口（通常为 `/dev/ttyACM0`）。

在 Windows 上查看串口时，打开设备管理器的“端口（COM 和 LPT）”，使用插入头显后出现的“USB Serial Device (COMx)”登录。Android 阶段使用 ADB，进入 openKylin 后调试入口改为串口。若只出现带黄色叹号的“USB Composite Device”（代码 10），先核对上文的 recovery 版本。`Qualcomm HS-USB QDLoader 9008` 表示头显处于 EDL 模式，该端口无法登录 openKylin。

在串口以 `mix` 和安装时设置的密码登录，执行以下命令检查启动目标、设备和会话日志。

```sh
systemctl get-default
findmnt /
findmnt /osstore
systemctl status mix-alpha-xr.service qiyu-openkylin-xr.service --no-pager
sudo journalctl -b -u mix-alpha-xr.service -u qiyu-openkylin-xr-tracking.service -u qiyu-openkylin-xr.service --no-pager
sudo env QIYU_VR_SESSION_PROFILE=/etc/xr/session.json xr-session status
```

`mix-alpha-xr.service` 负责发起会话，实际运行由 `qiyu-openkylin-xr.service` 托管。确认空间会话可用时应同时检查有效追踪和头显双眼画面。

若日志显示 `Missing or untrusted SEE runtime directory`，应核对本机传感器资料导入和文件属主。若显示 `Both distinct SEE accel and gyro identities are required`，应查看下列命令是否同时列出 `qcom-see-accel` 和 `qcom-see-gyro`。若旧镜像报告 `stage=xml_hash`，请保留本机标定并核对镜像版本。其他头显的标定文件不能用于本机。

```sh
cat /sys/bus/iio/devices/iio:device*/name
systemctl status mix-alpha-calibration.service a081-sensors-files.service --no-pager
```

若头显已有桌面画面但手柄不能操作，可用 `sudo journalctl -k -b` 查看故障时段是否有 `fastrpc` 和 `Oops`，同时保留上述会话日志。遇到这种情况请反馈手柄按键或位姿是否有响应。进程仍在运行、桌面帧数仍在增加，都不能单独证明手柄已恢复。

如需临时使用 UKUI 普通桌面，可在会话正常运行时执行下列命令。下一次从 Android 进入 recovery，仍会自动进入 XR。

```sh
sudo env QIYU_VR_SESSION_PROFILE=/etc/xr/session.json xr-session stop
sudo systemctl start display-manager.service
```

## 返回 Android 或恢复

从 openKylin 执行 `sudo systemctl reboot` 正常重启，通常返回原 Android 槽位。要再次进入 openKylin，从 Android 执行 `adb reboot recovery`。

要恢复安装前的 recovery，先回到创建备份时的同一台头显、同一 Android 槽位，连接 USB，保留本版安装器、校验清单和原备份。

```sh
python3 install-mix.py restore --backup ./mix-device-backup
```

输入 `RESTORE` 后，工具核对设备身份、槽位、备份长度和 SHA-256，再写回并校验该槽位的原 recovery。旧版安装器创建的备份只适用于 B 槽。恢复不会删除 osstore 中的 Alpha 根镜像，里面仍有账户、文档及本机标定，请按个人数据管理。遇到断线或读回失败，应留在 Android，重新连接原设备并用同一份备份执行恢复。不要使用其他头显的 recovery 备份。根文件系统镜像只作为文件放入 osstore，切勿用 `dd` 写到 UFS 整盘、userdata、boot 或 recovery 分区。

## 验证范围

所有内置运行时均来自锁定输入。ARM64 QEMU 用发布版 ramdisk 启动了完整根镜像。虚拟机确认默认 XR 服务自动运行、普通桌面未启动，服务在发现缺少真实头显标定时输出诊断。账户登录、sudo 和关机卸载通过。本批次尚待 MIX 真机验证冷启动后的双眼有效帧、头部和手柄跟踪、退出和返回 Android。反馈请附型号、镜像摘要和脱敏日志，妥善保管原厂标定及本机备份。

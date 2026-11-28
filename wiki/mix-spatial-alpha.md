---
layout: default
title: openKylin 3.0 空间智能 Alpha · 奇遇 MIX 安装指南
permalink: /wiki/mix-spatial-alpha/
---

# openKylin 3.0 空间智能 Alpha · 奇遇 MIX 安装指南

本版供奇遇 MIX（A081、SM8250）使用。从当前 Android 槽进入 recovery 后，openKylin 默认尝试启动空间桌面。镜像已内置原生 ARM64 SteamVR 2.17.10、配套 Gamescope、姿态观察器、Monado 驱动和 XR Shell。安装和首次启动均可离线完成。

## 先辨认手里的镜像

| 本地 IMG 的 SHA-256 前八位 | 适用步骤 |
| --- | --- |
| 主镜像 `b0efe26d`、recovery `8605b996` | 官网首版。已安装的用户走下方“原地修复官网首版”，无需重传 24 GiB 系统盘。只有首版下载包、尚未安装的用户，使用首版配套的更新文档脚本包完成安装，然后从 Android 执行修复，再首次启动。 |
| 主镜像 `ae296671`、recovery `a29af040` | R2 修订版。新安装直接使用 R2 文档脚本包和双镜像包，跳到“安装条件”。 |

以上前缀方便识别，下载时请验证配套清单里的完整摘要。首版会用开发头显的 XML 摘要拒绝另一台 MIX 的原厂标定。R2 读取当前头显的标定和传感器注册表；两台头显使用同一份公共 IMG。USB 序列号也在各自机器上生成。安装器用设备身份和当前槽位防止跨机、跨槽误写。

## 原地修复官网首版

1. 在原 Android 槽开启 USB 调试和 Magisk root。从 Linux 电脑解压本次更新的文档脚本包，保留里面的 `mix-alpha-repair.tar`、`mix-alpha-repair.service` 和 `mix-alpha-repair.json`。先对文档包运行配套的 `sha256sum -c`。只修复已安装的系统时不用下载主 IMG。建议备份已有个人文档，保管当初的 recovery 备份。
2. 在解压目录运行下面两条命令。看到正确的设备和槽位后按提示输入 `REPAIR`。工具从本机 Android 补齐标定、注册表和传感器配置，把更新放入现有可写根盘。已有账户、应用、文档和控制器配置保留。即使旧系统无法登录或 Windows 没有 COM 口，也能从 Android ADB 发起。

```sh
python3 install-mix.py check
python3 install-mix.py repair
```

3. 若还要修复首版的 Windows 串口枚举，将单独下载的 **R2 recovery IMG** 放到安装器旁，保持原文件名。确认其完整 SHA-256 与 `mix-alpha-repair.json` 中的 `recovery_sha256` 相同，使用这台头显当前槽位最初的备份目录执行下列命令，按提示输入 `UPDATE`。如缺少这份备份，可以完成上一步系统修复，保留当前 recovery。旧版没有记录槽位的备份只对应 B 槽。

```sh
python3 install-mix.py update-recovery --backup ./mix-device-backup
```

4. 执行 `adb reboot recovery`，等待第一次启动安装离线软件包。登录 openKylin 后运行以下命令。应看到 `Result=success`、`xr-head-tracking 0.1.18` 和 `xr-session-manager 0.3.25`。若更新中断，查看服务日志；下次启动会再次处理尚未完成的更新。

```sh
systemctl show -p Result mix-alpha-repair.service
dpkg-query -W xr-head-tracking xr-session-manager
sudo journalctl -b -u mix-alpha-repair.service --no-pager
```

此次修复同步更新会话实际调用的追踪程序和插件，修正传感器资料属主，等待加速度计与陀螺仪注册，并支持旧安装使用的带 `rounds=` 的 SHA-512 密码格式。继续使用自己的标定和密码；旧系统已保存的个人数据无需恢复成出厂镜像的摘要。

## 安装条件

- 头显型号为 A081，当前运行的 Android 已取得 Magisk root，USB 调试已授权。A 槽和 B 槽都可以使用。安装器从当前系统读取槽位，A 槽对应 `recovery_a`，B 槽对应 `recovery_b`。整个流程保留当前槽位。
- 头显已有名为 `osstore` 的 GPT 分区，文件系统为 ext4，至少有 24 GiB 加 64 MiB 可用。系统镜像保存在该分区的独立目录中。若 `check` 提示缺少 `osstore`，先按下节准备分区，再返回安装。
- 使用 Linux 电脑，准备 Python 3.8 或更高版本、Android Platform Tools（`adb`）、OpenSSL、GNU tar、XZ Utils 和 GNU Coreutils。电脑至少预留 36 GiB，用于下载、解压和保存本机原 recovery。给头显充足电，保持 USB 连线稳定。
- 原 Android 和已有 osstore 文件应另有备份。安装器在电脑保存当前槽位的完整 recovery、槽位和设备身份。每台头显、每个槽位使用独立备份目录。若原来通过 recovery 启动 Debian 等系统，安装 Alpha 会更换该启动入口。恢复本机原 recovery 后可重新使用。

## 创建 osstore 分区

已有可挂载的 ext4 `osstore` 时直接跳过本节。新建分区建议分配 **30 GiB 或更多**，安装器要求格式化后至少剩余 24 GiB 加 64 MiB。Android 设置里显示的可用容量属于 `userdata`，请以 GPT 显示的连续未分配范围为准。

**先识别设备并做备份。** 在 Android 能正常启动时记录当前槽位、设备及 `userdata` 的实际节点：

```sh
adb shell getprop ro.boot.slot_suffix
adb shell su -c 'readlink -f /dev/block/by-name/userdata'
adb shell su -c 'ls -l /dev/block/by-name/osstore'
```

下面的磁盘操作在**头显的独立 Linux 维护环境**以 root 执行。维护环境需提供 `lsblk`、`parted`、`sgdisk` 和 ext4 工具，且目标 LUN 的文件系统均已卸载。电脑/WSL 的系统盘不属于头显。发布包内的 Alpha recovery 自身依赖 osstore 启动；原厂满盘设备需要先准备能独立启动的维护环境。把本机 GPT 主备表备份和完整、离线的 userdata 块备份存到外接盘或电脑，校验备份；若使用分区加密，保管原厂恢复所需的元数据。不要使用另一台头显的 GPT 文件。

```sh
lsblk -o NAME,PATH,PKNAME,SIZE,LOG-SEC,PARTLABEL,FSTYPE,MOUNTPOINTS
# 在维护环境设置 DISK 为上一步辨认出的、承载 userdata 的头显整块 LUN。
: "${DISK:?请先设置为已确认的头显 LUN}"
sgdisk --verify "$DISK"
parted "$DISK" unit s print free
```

确认 `UD` 指向本机离线 userdata 分区，`BACKUP` 指向已挂载的外接备份目录且容量足够。以下是将 GPT 和整个 userdata 读到外接盘的示例。请使用新的空目录，完整核对源块设备与备份文件的字节数、摘要后再改盘。

```sh
: "${UD:?本机离线 userdata 分区}" "${BACKUP:?外接备份目录}"
sgdisk --backup="$BACKUP/gpt.bin" "$DISK" &&
dd if="$UD" of="$BACKUP/userdata.img" bs=4M conv=excl,fsync &&
test "$(blockdev --getsize64 "$UD")" -eq "$(stat -c %s "$BACKUP/userdata.img")" &&
sha256sum "$UD" "$BACKUP/userdata.img"
```

**有连续未分配区域。** 用上一步 `Free Space` 中的范围选择起点、终点。起点按 1 MiB 向上对齐，终点必须留在该段内。将两个值填入 `START`、`END`，单位为该盘逻辑扇区，记录本次预期容量。在写入前确认备份已移到头显外。命令只在这段空闲空间创建名为 `osstore` 的 GPT 条目。

```sh
: "${START:?填入已核对的起始扇区}" "${END:?填入同一空闲段内的末扇区}"
parted --align optimal "$DISK" unit s mkpart osstore ext4 "${START}s" "${END}s"
parted "$DISK" unit s print
sgdisk --verify "$DISK"
```

对比前后两份分区表，原有条目的起迄、GUID 和槽位属性应保持一致。如果内核未重新识别分区表，重新启动到同一维护环境再继续。用 `lsblk` 找到**刚创建**的分区，核对父 LUN、起点、终点和名称后设置 `NEW`；只对这个新节点创建文件系统。发现已有 osstore 或文件系统签名时先核对数据，已有分区不用格式化。示例采用两端内核均支持的 ext4 特性。

```sh
: "${NEW:?请设置为刚刚创建并核对过的分区节点}"
test -b "$NEW" && test "$(lsblk -dnro PARTLABEL "$NEW")" = osstore &&
test "$(lsblk -dnro PKNAME "$NEW")" = "$(basename "$(readlink -f "$DISK")")" &&
test "$(blockdev --getsize64 "$NEW")" -eq "$(( (END - START + 1) * $(blockdev --getss "$DISK") ))" &&
test -z "$(lsblk -dnro MOUNTPOINT "$NEW")" &&
test -z "$(wipefs -n -o TYPE --noheadings "$NEW")" &&
mke2fs -t ext4 -b 4096 -m 0 -L osstore -O none,has_journal,ext_attr,resize_inode,dir_index,filetype,extent,64bit,flex_bg,sparse_super,large_file,huge_file,dir_nlink,extra_isize,metadata_csum "$NEW" &&
e2fsck -fn "$NEW"
```

**若 userdata 占满整个 LUN。** 保留 Android 数据需要离线缩小 F2FS，再缩短 GPT 中 userdata 的末端。先在完整块备份的副本上按本机逻辑扇区大小和原始分区偏移预演，使用支持收缩的 `resize.f2fs`。本项目在 A081 上使用过 f2fs-tools 1.17.0。保留原厂尾部元数据空间，并把文件系统末端按 F2FS 段对齐。使用原厂兼容的 `fsck.f2fs` 核对缩容前后两份超级块、段边界、配额和文件树。数据占用过高、加密映射或尾部布局不明、检查报错时，应在原 GPT 仍完整时停止。

预演中，`FS_SECTORS` 是计划保留给 F2FS 的**扇区数**，需要扣除本机尾部保留区，`CLONE_UD` 是工作副本上对应的未挂载分区。请从整盘副本映射该分区，验证其逻辑扇区大小、分区起点和原设备相同。原备份另行保留。示例中的 `-t` 取分区内相对长度，不能填 GPT 的绝对结束扇区。

```sh
: "${CLONE_UD:?已核对的副本分区}" "${FS_SECTORS:?预演的文件系统目标扇区数}"
blockdev --getss "$CLONE_UD"
resize.f2fs -s -t "$FS_SECTORS" "$CLONE_UD"
```

只有在保留原偏移的副本上用相同工具和参数跑通、缩容后原厂 fsck 无错误、抽查数据可读，才将已验证的过程用于真实离线分区。若原厂在 userdata 末端存有加密页脚，须按本机布局迁移并逐字节验证新末端；布局不明时保留原分区。

缩容成功且新的文件系统与保留尾部均落在新末端以内后，才在维护环境用 `parted "$DISK" unit s resizepart "$UDNUM" "${UDEND}s"` 调整**原 userdata 条目**的末端。`UDNUM` 是本机条目号，`UDEND` 是已经预演通过的绝对结束扇区。保留原起点、类型、GUID、属性与 A/B 设置，复核主备 GPT。随后按“有连续未分配区域”创建 osstore。单独缩短 GPT 会截断文件系统。维护时若没有可还原的块备份或适合本机的缩容条件，请先取得这些条件再继续。[GNU Parted 的 resizepart 说明](https://www.gnu.org/software/parted/manual/html_node/resizepart.html)

完成后回到**原 Android 槽**，确认用户数据可读、USB 调试和 root 可用，再运行 `python3 install-mix.py check`。只有它识别出 ext4 osstore 和足够可用空间后，才继续安装。

## 下载和解压

可以先用体积较小的文档脚本包检查头显。以下命令以 R2 为例。使用官网首版双镜像包时，将命令中的 `-r2` 去掉，并下载首版配套的更新文档脚本包。请把文档脚本包和对应的 `.sha256` 放进同一空目录，在 Linux 终端运行以下命令。

```sh
sha256sum -c openKylin-3.0-Spatial-Alpha-MIX-r2-docs-scripts.tar.xz.sha256
tar -xJf openKylin-3.0-Spatial-Alpha-MIX-r2-docs-scripts.tar.xz
adb devices -l
python3 install-mix.py check
```

确认输出显示预期的 A/B 槽位、对应的 recovery 和可用空间后，把配套镜像包下载到这个目录，再执行以下命令。

```sh
sha256sum -c openKylin-3.0-Spatial-Alpha-MIX-r2.tar.xz.sha256
tar -xJf openKylin-3.0-Spatial-Alpha-MIX-r2.tar.xz
sha256sum -c SHA256SUMS
```

| 包内镜像 | 用途 |
| --- | --- |
| `openKylin-3.0-Spatial-Alpha-MIX-arm64.img` | 24 GiB 可写 ext4 系统盘，安装器把它放在 osstore 的独立目录 |
| `openKylin-3.0-Spatial-Alpha-MIX-recovery.img` | 100 MiB 启动镜像，由安装器写入并读回校验当前槽位的 recovery |

`check` 只读挂载 osstore 并报告容量。设备型号、Android 插槽、root、分区、文件系统、原厂标定或传感器注册表检查报错时，先解决该项再继续。若连接了多台设备，从 `adb devices -l` 选择目标，对每个安装器命令都传 `--serial <ADB序列号>`，后续重启也用 `adb -s <ADB序列号> reboot recovery`。已在 A 槽取得 root 的用户可直接留在 A 槽。已有 osstore 时，系统安装与原地修复均从正常启动的 Android 发起，无需进入 fastboot 或 EDL。

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

所有内置运行时均来自锁定输入。ARM64 QEMU 用发布版 ramdisk 启动了完整根镜像。虚拟机确认默认 XR 服务自动运行、普通桌面未启动，服务在发现缺少真实头显标定时输出诊断。账户登录、sudo 和关机卸载通过。R2 已用 ARM64 QEMU 从 osstore 内的可写根盘启动，首版根盘的写时复制副本也已通过首次开机修复，保留了原有用户文件，并验证了会话实际调用的追踪程序。本批次尚待 MIX 真机验证冷启动后的双眼有效帧、头部和手柄跟踪、退出和返回 Android。反馈请附型号、镜像摘要和脱敏日志，妥善保管原厂标定及本机备份。

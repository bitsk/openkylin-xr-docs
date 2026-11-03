# openKylin 独立打包

## 维护边界

功能组件使用 xr 前缀，发行版集成使用 openkylin-xr 前缀。内核、Mesa、Monado 等定制内容各自维护，采用独立源码包、版本化内核 ABI 和私有库目录。系统常规内核、图形库和默认 runtime 保持自己的选择。

内核与 Mesa 使用 `tools/prepare-source.py` 准备真实上游和补丁栈，再由 `packaging/build-debs.sh` 构建。其余有根目录 debian/ 的组件使用目标发行线的干净构建环境运行 dpkg-buildpackage。NDK、Cargo 缓存、厂商资产和开发包在输入准备阶段完成。

## 依赖

`xr-kernel-uapi-dev` 提供自有内核接口；`libxr-device-dev` 导出 XRDevice CMake 包；`libxr-hexagonrpc-dev` 提供共享静态客户端，`xr-hexagonrpc-source` 是 ARM32/Bionic 构建输入。头部与手柄协议分别从其开发包消费，运行时不维护第二份协议头文件。

## 型号与安装组合

集成仓库提供公共基础、串流、SteamVR 候选和调试元包。设备配置通过独立型号包选择，原生 OpenXR 组合在对应后端完成后增加。镜像工具继续按 systems、devices、profiles 组合，支持增加系统和型号。

固件安装到 `/usr/lib/firmware/xr/a081`，型号启动参数指定 firmware_class.path；initramfs hook 只收集该型号公共固件。私有 Bionic 环境、运行时插件和 Mesa ICD 均使用私有目录。包安装不刷写设备、不启动竞争采集，也不替换活动图形栈。

## 验证

已完成的源码和包构建范围见[索引](repositories.md)。后续在目标发行线验证依赖闭包、安装升级卸载、服务生命周期、硬件功能及回退，再评审正式组合。

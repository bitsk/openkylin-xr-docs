# 仓库与软件包索引

下列维护入口彼此独立；本地工作目录统一为 `openkylin-xr-repos/`。源码、打包和验收状态分别记录。新增组件继续按功能独立维护。

| 仓库 | 维护形式 | 源码包 | 二进制包 |
|---|---|---|---|
| [xr-kernel](https://github.com/bitsk/xr-kernel) | patch-stack | `linux-xr` | `linux-image-<kernelrelease>`, `linux-headers-<kernelrelease>`, `xr-kernel-uapi-dev`, `linux-image-xr-arm64` |
| [xr-mesa](https://github.com/bitsk/xr-mesa) | patch-stack | `mesa-xr` | `xr-mesa-vulkan-drivers` |
| [xr-monado](https://github.com/bitsk/xr-monado) | upstream-derived | `monado-xr` | `xr-monado-steamvr-driver` |
| [xr-hexagonrpc](https://github.com/bitsk/xr-hexagonrpc) | upstream-derived | `xr-hexagonrpc` | `libxr-hexagonrpc-dev`, `xr-hexagonrpc-source` |
| [xr-device-io](https://github.com/bitsk/xr-device-io) | component | `xr-device-io` | `xr-device-tools`, `libxr-device-dev` |
| [xr-head-tracking](https://github.com/bitsk/xr-head-tracking) | component | `xr-head-tracking` | `xr-head-tracking`, `xr-head-tracking-tools`, `libxr-head-dev` |
| [xr-controller-tracking](https://github.com/bitsk/xr-controller-tracking) | component | `xr-controller-tracking` | `xr-controller-worker-bionic`, `libxr-controller-dev`, `xr-controller-tools` |
| [xr-session-manager](https://github.com/bitsk/xr-session-manager) | component | `xr-session-manager` | `xr-session-manager` |
| [xr-debug-console](https://github.com/bitsk/xr-debug-console) | component | `xr-debug-console` | `xr-debug-server`, `xr-debug-web`, `xr-debug-video`, `xr-debug-tools` |
| [xr-vendor-assets](https://github.com/bitsk/xr-vendor-assets) | asset-catalog | `xr-firmware-a081`, `xr-vendor-a081` | `firmware-xr-a081`, `firmware-xr-a081-tracking`, `xr-vendor-a081-controller`, `xr-vendor-a081-dsp-host` |
| [openkylin-xr-integration](https://github.com/bitsk/openkylin-xr-integration) | component | `openkylin-xr-integration` | `openkylin-xr-defaults`, `xr-device-a081`, `xr-device-a081-vr`, `openkylin-xr-base`, `openkylin-xr-streaming`, `openkylin-xr-debug`, `openkylin-xr-steamvr` |
| [xr-image-tools](https://github.com/bitsk/xr-image-tools) | component | `xr-image-tools` | `xr-image-builder` |
| [openkylin-xr-docs](https://github.com/bitsk/openkylin-xr-docs) | component | `openkylin-xr-docs` | `openkylin-xr-doc` |

机器可读目录见 [xr-image-tools/repositories.json](https://github.com/bitsk/xr-image-tools/blob/main/repositories.json)。组件的精确提交与公开头文件摘要由 xr-image-tools 的 components.lock.json 固定，型号资产由集成仓库中的 vendor.lock.json 固定。

## 当前验证范围

空间桌面新增组件与既有底座分别管理。`xr-shell`、`xr-dock`、`xr-spatial-kit`、`xr-wayland` 和 `xr-system-ui` 当前为设计中的维护入口，尚未发布实现；`xr-environments` 已准备独立的本地背景数据包。包边界和机器可读状态见 [桌面栈设计](xr-desktop-stack.md) 与 [desktop-stack.json](../desktop-stack.json)，不将这些提案加入上表作为已发布仓库。

- 内核：14 个独立提交补丁重放通过，Image、125 个模块及型号 DTB 编译通过。启动镜像、目标发行版和实机验收分别进行。
- 设备、FastRPC、头部跟踪：ARM64 交叉编译及开发接口消费通过。
- 手柄：3 个 ARM32 worker 与 2 个兼容 shim 构建通过；执行能力和真实跟踪结果不由编译替代。
- Monado：独立源码树的 ARM64 SteamVR driver 构建通过；OpenXR 后端单独推进。
- 调试：ARM64 Rust 后端和 5 个原生 helper 构建通过，包含 CPU/GPU NV12 路径；实际视频链路与硬件编码分别验收。
- 包装验证：多个数据、开发、工具和元包在主机 Debian 环境构建通过；厂商资产的 4 个独立包已生成。
- 资产：已从 Gitee 下载并核验 247 个 OEM 文件；型号公共资产不含 per-device 校准。

这些结果用于源码和打包迁移验证，不能视为目标 openKylin 发行线或新设备组合已经验收。

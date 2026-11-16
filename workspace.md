# MIX openKylin XR workspace

产品正式名称为 **openKylin XR 空间智能版本**；HyperSpace 是 XR 空间桌面，Vultra 是空间合成器，晴空是空间主题。参见[产品命名与系统边界](wiki/product-architecture.md)。

The only maintained application/component sources are the repositories listed below. Source builds use `xr-image-tools/components.lock.json` and `xr-build-components`; historical qiyu component checkouts are audit/evidence archives.

The kernel implementation remains in [bitsk/qiyu-kernel](https://github.com/bitsk/qiyu-kernel), selected and packaged by `xr-kernel`. OEM binaries remain in the existing [bitsk/qiyu-vendor](https://github.com/bitsk/qiyu-vendor) release archive, consumed by digest through `xr-vendor-assets`. These are upstream/source dependencies, not second openKylin implementations.

| Component | Source owner | Binary package responsibility |
| --- | --- | --- |
| xr-kernel | [xr-kernel](https://github.com/bitsk/xr-kernel) | linux-image-<kernelrelease>, linux-headers-<kernelrelease>, xr-kernel-uapi-dev, linux-image-xr-arm64 |
| xr-mesa | [xr-mesa](https://github.com/bitsk/xr-mesa) | xr-mesa-vulkan-drivers |
| xr-monado | [xr-monado](https://github.com/bitsk/xr-monado) | xr-monado-steamvr-driver |
| xr-hexagonrpc | [xr-hexagonrpc](https://github.com/bitsk/xr-hexagonrpc) | libxr-hexagonrpc-dev, xr-hexagonrpc-source, xr-hexagonrpc-daemon |
| xr-device-io | [xr-device-io](https://github.com/bitsk/xr-device-io) | xr-device-tools, libxr-device-dev |
| xr-head-tracking | [xr-head-tracking](https://github.com/bitsk/xr-head-tracking) | xr-head-tracking, xr-head-tracking-tools, libxr-head-dev |
| xr-controller-tracking | [xr-controller-tracking](https://github.com/bitsk/xr-controller-tracking) | xr-controller-worker-bionic, libxr-controller-dev, xr-controller-tools |
| xr-session-manager | [xr-session-manager](https://github.com/bitsk/xr-session-manager) | xr-session-manager |
| xr-debug-console | [xr-debug-console](https://github.com/bitsk/xr-debug-console) | xr-debug-server, xr-debug-web, xr-debug-video, xr-debug-tools, xr-debug-gstreamer |
| xr-vendor-assets | [xr-vendor-assets](https://github.com/bitsk/xr-vendor-assets) | firmware-xr-a081, firmware-xr-a081-tracking, xr-vendor-a081-controller, xr-vendor-a081-dsp-host |
| openkylin-xr-integration | [openkylin-xr-integration](https://github.com/bitsk/openkylin-xr-integration) | openkylin-xr-defaults, xr-device-a081, xr-device-a081-vr, openkylin-xr-base, openkylin-xr-streaming, openkylin-xr-debug, openkylin-xr-steamvr, openkylin-xr-hands |
| xr-image-tools | [xr-image-tools](https://github.com/bitsk/xr-image-tools) | xr-image-builder |
| openkylin-xr-docs | [openkylin-xr-docs](https://github.com/bitsk/openkylin-xr-docs) | openkylin-xr-doc |
| xr-hand-tracking | [xr-hand-tracking](https://github.com/bitsk/xr-hand-tracking) | xr-hand-tracking, libxr-hand-dev, xr-hand-backend-quest-dsp |
| xr-dock | [xr-dock](https://github.com/bitsk/xr-dock) | xr-dock |
| xr-environments | [xr-environments](https://github.com/bitsk/xr-environments) | xr-environment-qingkong |
| xr-shell | [xr-shell](https://github.com/bitsk/xr-shell) | xr-shell, librust-xr-shell-protocols-dev |
| xr-spatial-kit | [xr-spatial-kit](https://github.com/bitsk/xr-spatial-kit) | librust-xr-spatial-kit-dev, xr-spatial-style-ukui |
| xr-system-ui | [xr-system-ui](https://github.com/bitsk/xr-system-ui) | xr-system-ui |
| xr-wayland | [xr-wayland](https://github.com/bitsk/xr-wayland) | xr-window-server |

## Build and migration

[Migration ledger](https://github.com/bitsk/openkylin-xr-docs/blob/main/wiki/component-migration.md) records the old capabilities, their single maintained owners and acceptance limits. The machine registry belongs to `xr-image-tools/repositories.json`; the workspace-level registry is a link to that maintained file.

Build outputs belong in a separate directory. The old `build/openkylin-xr-repos` entry is only a compatibility link to the same maintained workspace. It is not an alternative source selection.

Start a fresh build with `xr-build-components build --profile vr-hands --output NEW_DIRECTORY`. Explicit toolchain, SDK, OEM caches and private calibration/weight imports remain separate inputs. Build receipts identify exact source commits and package hashes. Package construction and image composition do not establish physical acceptance or fix the hand detector1042 fault.

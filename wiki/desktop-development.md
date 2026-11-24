---
layout: default
title: XR 桌面与应用协作开发
---

# XR 桌面与应用协作开发

本入口面向 Linux 主机上的 HyperSpace 桌面、Vultra 渲染、空间组件和应用开发。
项目源码、主题纹理、SDK、构建配方与接口说明均从 Git 仓库取得。
上层通过 OpenXR 接入运行时，不需要 MIX/VR700 的固件、校准、模型权重或本机文件。

## 分工与资源入口

| 工作方向 | 仓库与入口 | 负责的接口与边界 |
| --- | --- | --- |
| Shell / Vultra | [xr-shell](https://github.com/bitsk/xr-shell) | 唯一桌面 OpenXR 会话、Vulkan、空间布局、命中、焦点、输入路由；`protocols/` 是组件 SDK 的唯一来源 |
| 控件与设计系统 | [xr-spatial-kit](https://github.com/bitsk/xr-spatial-kit) | 控件状态、角尺寸、样式和 `art/` 纹理；不持有 GPU 或系统服务 |
| Dock 与应用入口 | [xr-dock](https://github.com/bitsk/xr-dock) | 应用目录、启动意图、任务与工作区 UI；通过 Shell 协议获取状态 |
| 系统组件 | [xr-system-ui](https://github.com/bitsk/xr-system-ui) | 控制中心、通知、剪贴板、键盘、OS 适配；不在渲染线程执行系统调用 |
| 桌面应用兼容 | [xr-wayland](https://github.com/bitsk/xr-wayland) | 私有 Wayland/Xwayland、窗口/弹出层、seat、SHM、选择与输入法；`compat/` 维护应用装饰适配 |
| 环境与美术 | [xr-environments](https://github.com/bitsk/xr-environments) | `environments/qingkong/` 含完整运行纹理、格式、方向与摘要；使用条款见仓库内美术说明 |
| 调试与性能 | [xr-debug-console](https://github.com/bitsk/xr-debug-console) | 前端、后端、API、帧时间线；设备专用采集不是桌面开发前置条件 |
| 构建与场景启动 | [xr-image-tools](https://github.com/bitsk/xr-image-tools) | 唯一仓库清单、提交锁、配方、`xr-shell-session` / `xr-shell-wivrn` |
| 通用诊断辅助 | [xr-session-manager](https://github.com/bitsk/xr-session-manager) | 本组合只安装 `xr-diagnostic-tools`；设备会话管理包不属于主机桌面安装集 |
| 跨仓接口与协作 | [openkylin-xr-docs](https://github.com/bitsk/openkylin-xr-docs) | 产品边界、协议设计、空间规范与共同开发入口 |

机器可读的仓库来源只维护在 `xr-image-tools/repositories.json`；
版本由 `components.lock.json` 固定，构建顺序由 `build-recipes.json` 管理。
`profiles/streaming-desktop.json` 同时定义源码组件与允许安装的包。

## 从空目录取得同一组源码

桌面协作基点固定构建工具提交为
`9c80c2e2f895f148d9e3235e5a4a42fd77edfd4c`。该提交在 `main`
分支可取得；每个组件的准确提交继续从它的组件锁读取。

```sh
git clone --branch main https://github.com/bitsk/xr-image-tools.git
git -C xr-image-tools checkout --detach 9c80c2e2f895f148d9e3235e5a4a42fd77edfd4c
builder="$PWD/xr-image-tools"
out="$PWD/xr-desktop-build"
python3 "$builder/tools/components.py" checkout \
  --profile streaming-desktop --output "$out"
```

`out` 首次必须不存在。此入口直接从公开 Git 远端恢复锁定的兄弟仓库，
产生 `sources/` 和 `build-manifest.json`；不需要 `--source-cache` 或旧工作区。
保持 `xr-shell`、`xr-spatial-kit` 与消费者的兄弟目录关系，不复制 SDK 源码。
调试仓库是可选模块，可另行 clone 后 checkout 到组件锁中 `xr-debug-console` 的提交。

## 原生 Linux 构建

以下在 Debian/openKylin 兼容的独立开发环境中执行。工具链使用 Rust/Cargo
1.95.0；包声明的下限为 1.88。Rust 工具链来源见
[rust-lang/rust](https://github.com/rust-lang/rust)。首次需要网络取得 Cargo.lock
固定的 crate；缓存是可重建的下载结果。无需复制已有机器的 SDK 或编译缓存。

```sh
sudo apt install build-essential debhelper dpkg-dev pkg-config python3 git \
  glslang-tools libcairo2-dev libpango1.0-dev librsvg2-dev libxkbcommon-dev
cargo --version
rustc --version
```

各组件的 `debian/control` 是发行版依赖的权威声明。先构建并安装两个源码 SDK，
使后续 `dpkg-buildpackage` 的依赖检查成立；这不会启动 XR 服务。

```sh
python3 "$builder/tools/components.py" build --profile streaming-desktop \
  --output "$out" --resume --only xr-spatial-kit \
  --environment "$builder/environments/desktop-native.json"
sudo apt install "$out"/packages/librust-xr-spatial-kit-dev_*.deb

python3 "$builder/tools/components.py" build --profile streaming-desktop \
  --output "$out" --resume --only xr-shell \
  --environment "$builder/environments/desktop-native.json"
sudo apt install "$out"/packages/librust-xr-shell-protocols-dev_*.deb

python3 "$builder/tools/components.py" build --profile streaming-desktop \
  --output "$out" --resume \
  --environment "$builder/environments/desktop-native.json"
```

构建包位于 `packages/`，各组件日志位于 `logs/`。安装桌面 profile 的明确包集：

```sh
sudo apt install \
  "$out"/packages/xr-shell_*.deb \
  "$out"/packages/xr-window-server_*.deb \
  "$out"/packages/xr-dock_*.deb \
  "$out"/packages/xr-system-ui_*.deb \
  "$out"/packages/xr-environment-qingkong_*.deb \
  "$out"/packages/xr-spatial-style-ukui_*.deb \
  "$out"/packages/xr-diagnostic-tools_*.deb \
  "$out"/packages/xr-shell-session_*.deb
```

不要用 `packages/*.deb` 全量安装：同次构建可产生设备会话管理包和镜像制作包，
它们不在主机桌面 profile 的安装集内。跨架构包需要目标工具链与独立验证，
不能把原生构建命令直接视作交叉构建方案。

## 运行与应用接入

串流入口使用 [WiVRn](https://github.com/WiVRn/WiVRn) 的主机服务和头显客户端，
以及主机 Vulkan 驱动、OpenXR loader。OpenXR loader 的源码入口是
[KhronosGroup/OpenXR-SDK](https://github.com/KhronosGroup/OpenXR-SDK)。
这些外部项目的构建与安装以各自仓库说明为准。

已连接头显后，以登录桌面的普通用户运行：

```sh
xr-shell-wivrn
```

也可在 WiVRn 的连接后启动应用中配置 `xr-shell-wivrn --wait-for-headset`。
选择其他兼容运行时，先显式设置该运行时安装的 `XR_RUNTIME_JSON`，再运行
`xr-shell-session`。场景入口创建私有 Wayland/Xwayland 与 D-Bus；安装包本身不启动场景。

没有头显时，可以先做编译、协议与组件工作。模拟运行的构建和启动入口见
[xr-image-tools/tools/qemu](https://github.com/bitsk/xr-image-tools/tools/qemu)。
其中 VM 装配流程要求显式准备辅助机，不提供预制开发盘；模拟结果不表示串流或真实佩戴通过。

- **普通 2D 应用**：维护应用自己的源码和 `.desktop` 文件，经 `xr-wayland` 启动，
  使用标准 Wayland/X11。应用选择策略见 `xr-wayland/src/application-backends.json`；
  原生 Wayland 与 Xwayland、弹出菜单、剪贴板、IME 分别验证。
- **空间桌面组件**：使用 [Shell SDK](https://github.com/bitsk/xr-shell/tree/main/protocols)
  和 Spatial Kit；Dock / System UI 是现有消费者。新增角色或能力先由协议所有者定义，
  不复制 wire 格式、不自行创建第二个桌面 OpenXR 会话。
- **独立 OpenXR 应用**：使用标准 OpenXR SDK 与选定运行时。应用自己的会话生命周期
  与桌面场景组件分开，接入启动/退出行为时与场景入口维护者协作。

## 协作约定

1. 每项工作选择上表的唯一代码所有者，在该仓库创建功能分支和 PR；任务说明列出
   输入、输出、依赖 PR 与验收方法。现有 checkout 是 detached HEAD，修改前先建分支。
2. 跨仓功能先确定接口：字段/能力/版本归接口仓库维护，生产者和消费者分别提交。
   OS 能力不可用时明确报告，窗口服务不代管空间布局，Shell 不接入设备采集。
3. 开发期间可在功能分支使用 `cargo check --locked` 和组件现有检查；集成构建前，
   先提交并推送各依赖，再通过单独的 `xr-image-tools` PR 更新组件锁，使用新的输出目录构建。
   不把未提交目录或旧部署 prefix 当作来源。
4. PR 说明区分编译/包构建、协议与应用交互、运行时场景、真实头显体验；为改动提供
   对应验证结果，不把模拟输入、帧计数或 GPU 时间直接当作物理显示/佩戴证明。
5. 共用规范：[产品边界](product-architecture.md)、[组件协议](xr-desktop-protocol.md)、
   [空间交互规范](xr-spatial-guidelines-and-issues.md)、[性能观察](frame-profiling.md)。
   历史审计按文内版本阅读，当前行为以固定提交的实现与对应验证结果为准。

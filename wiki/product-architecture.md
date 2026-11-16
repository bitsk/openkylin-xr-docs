---
layout: default
title: 产品命名与系统边界
---

# openKylin XR 空间智能版本

这是整体系统的正式名称。下表定义当前产品概念与代码的对应关系；仓库、包名与命令是工程标识，不另行作为产品名称。

| 名称 | 定义 | 当前实现 |
| --- | --- | --- |
| openKylin XR 空间智能版本 | 完整操作系统与发布组合 | openkylin-xr-integration、xr-image-tools 与锁定的组件 |
| HyperSpace · XR 空间桌面 | 用户使用的空间桌面和应用工作空间 | xr-shell、xr-dock、xr-system-ui、xr-wayland、xr-spatial-kit 与桌面会话入口 |
| Vultra · 高性能空间合成器 | Rust 模块化实现，基于 OpenXR 与 Vulkan 的双眼渲染和三维空间界面合成核心 | xr-shell；同仓当前还包含桌面布局、命中测试和组件协议 |
| 晴空（Qingkong）· 空间主题 | 可选择的环境主题，不是系统或桌面名称 | xr-environments 的 qingkong，包 xr-environment-qingkong |
| XROS 系统底层 | 设备、跟踪、XR 运行时与基础系统的工程分层名称 | xr-device-io、xr-head-tracking、xr-controller-tracking、xr-hand-tracking、xr-monado、xr-session-manager、内核与系统集成 |
| XRsig 控制台 | 既有 WiVRn 服务与连接管理入口 | openkylin-wivrn-dashboard；不承担空间合成 |

完整表述：**openKylin XR 空间智能版本提供 HyperSpace XR 空间桌面，HyperSpace 由 Vultra 驱动，并可选用晴空空间主题。**

任务栏、空间控件、窗口服务与系统控制采用解耦的独立组件架构。系统操作在各组件或服务中执行，不放进 Vultra 渲染线程。进程分离有助于故障隔离，但不等于这些组件已经完成安全沙箱验收；“高性能”是合成器的产品定位，具体帧率、延迟和功耗以对应平台实测为准。

## 两条项目线

**XROS 系统与运行时线**负责硬件接入、固件、采集、标定、跟踪、姿态预测、运行时合成、扫描输出、电源、系统服务和镜像。传感器保持唯一生产者，消费者使用已有共享接口。

**HyperSpace / Vultra 与应用线**负责空间桌面体验、渲染、窗口与输入路由、Dock、系统面板、输入法和应用兼容。通用跟踪、输入、环境混合和参考空间变更由 OpenXR 提供；系统操作使用显式服务提供者。

两条线独立提交与发布，通过 xr-image-tools 的组件锁和构建配方集成。WiVRn/Quest 可以验证通用桌面路径，MIX 原生必须验证设备与系统集成；任何一边都不能替代另一边的硬件或佩戴验收。

## 新接口边界

- Vultra 默认使用 OpenXR actions 与 interaction profiles；光学手部使用运行时提供的关节和手部动作，不读取相机、DSP 或设备手柄包，也不二次预测姿态。
- 运行时负责系统/Home 键与参考空间。桌面在 OpenXR 参考空间变更的生效时间取消捕获、重新定位锚点。
- 手柄模型是可选资源。XROS 通过 `XR_CONTROLLER_MODEL_DIR` 选择设备模型；未提供时使用内置通用模型与射线。通用包不依赖 MIX 模型包。
- 系统面板使用 `XR_SYSTEM_CONTROLS_COMMAND` 的 JSON argv 提供者，追加 `get` 或 `set FIELD VALUE`；提供者输出 xr-session-manager 所属的 controls schema。没有提供者时显示能力不可用，不自行寻找 `xr-session`。
- 观测文件通过 `XR_INPUT_STATUS_FILE`、`XR_SCENE_CONTROLS_STATUS_FILE` 显式传入，不从组件的私有 XDG 目录猜测路径。`XR_SESSION_CONTROLS_FILE` 是可选的系统管理策略；未配置时通用输入不依赖该服务。
- WiVRn 入口移除本机设备模型和系统控制提供者选择，以远端运行时为输入与呈现能力来源。系统面板不能把客户端未开放的权限开关伪装成可操作。

## 透视归属与迁移状态

Vultra 的 `XR_ENVIRONMENT_MODE=passthrough` 要求运行时枚举 `ALPHA_BLEND`，并提交透明背景与带 alpha 的空间界面；不支持时启动报错。默认 `opaque` 使用主题背景。

相机采集、RAW 解码、内外参、畸变校正、曝光时间对齐、透视重投影应归设备服务和 XR 运行时。WiVRn 可在支持的头显端合成透视，不代表主机获得原始相机访问权限。参见 [WiVRn](https://github.com/WiVRn/WiVRn) 和 [OpenXR 环境混合模式](https://registry.khronos.org/OpenXR/specs/1.1/html/xrspec.html#environment_blend_mode)。

**当前 MIX 运行时只报告 OPAQUE。** 新版移除了 Shell 内的 QSP1、RAW10、标定与手柄 Home 包解析，未把这些代码复制成另一份维护入口。旧实现保留在 Git 历史供运行时迁移参考。原生 MIX 彩透尚未完成运行时实现，系统控制器因此报告透视与双敲透视不可用；配置文件存在不算支持。

运行时后续接入门槛：复用既有相机生产者、真实时间与标定；完成运行时环境合成；仅在能力可用时发布 ALPHA_BLEND 或相应扩展；验证独立应用和 HyperSpace 的透视、退出、姿态与佩戴效果。源码拆分或离线构建不构成这一项验收。

此版本按新接口集成，不提供旧环境变量或相机直读接口的兼容路径。新系统会话、系统面板、Vultra 与组装锁必须作为同一组合消费；不在未验收时替换现用设备基线。

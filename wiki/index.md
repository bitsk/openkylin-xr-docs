---
layout: default
title: 工程 Wiki
---

# 工程 Wiki

从硬件到系统，再到 VR 应用：这里按主题整理实现方式和维护入口。先看[开发历程]({{ '/wiki/development-history/' | relative_url }})，也可以直接进入具体模块。

## 硬件与基础系统

- [openKylin 3.0 空间智能 Alpha · 奇遇 MIX 安装指南]({{ '/wiki/mix-spatial-alpha/' | relative_url }})

- [A081 硬件适配]({{ '/wiki/hardware/' | relative_url }})：原厂依据、主线接入、电源、无线与音频。
- [显示与 GPU]({{ '/wiki/display-and-gpu/' | relative_url }})：双 DSI、DSC、PHY、Mesa/Turnip 与呈现优化。
- [相机、IMU 与共享采集]({{ '/wiki/cameras-and-sensors/' | relative_url }})：六颗相机、SEE/IIO、QCFT 和共享输入。
- [从 Debian 到 SteamOS]({{ '/wiki/steamos/' | relative_url }})：持久 rootfs、桌面、原生 SteamVR 和多系统配方。

## VR 与调试

- [VR 运行时开发]({{ '/wiki/vr-runtime/' | relative_url }})：XR 桌面、QVR/CDSP、手柄 SDK、光学与会话。
- [调试系统与视频]({{ '/wiki/debug-and-video/' | relative_url }})：Rust/网页、相机预览、Venus H.264 与 WebRTC。
- [UKUI XR 桌面软件栈]({{ '/wiki/xr-desktop-stack/' | relative_url }})：Rust Shell、独立 Dock/组件、窗口服务、包与实施顺序。
- [XR Shell 最小 v1 draft 合同]({{ '/wiki/xr-desktop-protocol/' | relative_url }})：版本/能力、generation/revision、场景、capture、shm 与 focus_epoch。
- [UKUI XR 实施计划]({{ '/wiki/xr-desktop-implementation/' | relative_url }})：M0/M1 历史范围、空间视觉约束与原生 ARM64 上 M2–M4 待完成验收表。
- [XR 空间开发约束与平面化已知问题]({{ '/wiki/xr-spatial-guidelines-and-issues/' | relative_url }})：2D/平面化开发病灶清单（ISSUE-XR-01..14）与全仓库 XR 空间人体工学、6DoF 交互、光学材质与渲染架构强制约束。

## 开发与维护

- [桌面与应用协作开发]({{ '/wiki/desktop-development/' | relative_url }})：从仓库恢复环境、组件分工与跨仓接口协作。

- [仓库架构与职责]({{ '/wiki/architecture/' | relative_url }})：八个代码仓库与独立文档仓库。
- [原厂二进制归档]({{ '/wiki/vendor-assets/' | relative_url }})
- [构建与发布]({{ '/wiki/build-and-release/' | relative_url }})：组件构建、版本锁定与系统组合。
- [r1 源码发布记录]({{ '/wiki/source-publication/' | relative_url }})：首次拆库的具体提交和导入范围。
- [内核维护与 rebase](https://github.com/bitsk/xr-kernel/blob/main/devices/a081/MAINTAINING.md)。

[开发 Blog]({{ '/blog/' | relative_url }}) 按阶段讲述工程进展。需要追溯资料时，参阅[来源附录]({{ '/wiki/evidence-sources/' | relative_url }})。

- [头追发布、回抽与最终呈现]({{ '/wiki/tracking-and-presentation/' | relative_url }})：真实编译输入、姿态门禁、回抽归因与呈现证据。

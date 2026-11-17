# openKylin XR 开发与打包

XR SIG 的跨仓库工程文档、接口边界、软件包索引和设备参考。

- [独立仓库与软件包索引](wiki/repositories.md)
- [机器可读仓库目录](https://github.com/bitsk/xr-image-tools/blob/main/repositories.json)
- [openKylin 打包与安装边界](wiki/openkylin-packaging.md)
- [源码和资产交付入口](wiki/source-publication.md)
- [系统架构参考](wiki/architecture.md)
- [桌面与应用协作开发](wiki/desktop-development.md)：仓库分工、固定提交恢复、构建、运行与应用接入。
- [UKUI XR 桌面软件栈设计](wiki/xr-desktop-stack.md)
- [XR Shell 最小 v1 draft 合同](wiki/xr-desktop-protocol.md)
- [UKUI XR 项目实施计划（M0–M4）](wiki/xr-desktop-implementation.md)：历史 QEMU 范围、空间视觉约束与原生 ARM64 待验收边界。
- [XR 空间开发约束与平面化已知问题](wiki/xr-spatial-guidelines-and-issues.md)：平面化思维已知问题清单与全仓库 XR 空间开发强制规范。
- [桌面组件与依赖目录](desktop-stack.json)

内核、Mesa 使用源码补丁与打包仓；Monado、FastRPC 及功能组件各自维护。通用功能、设备后端、原厂资产、发行版组合和镜像工具分别管理。

当前构建状态以索引为准。既有硬件和运行时文章保留为历史参考，不代表迁移后 openKylin 组合已经实机验收。原有版本和设备基线独立保留。

维护入口与旧功能迁移范围见 [workspace.md](workspace.md) 和 [组件迁移清单](wiki/component-migration.md)。内核实现归 bitsk/qiyu-kernel，xr-kernel 只维护固定来源与打包；构建仓库清单归 xr-image-tools。

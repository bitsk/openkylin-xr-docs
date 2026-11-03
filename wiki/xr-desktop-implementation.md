---
layout: default
title: UKUI XR 项目实施计划
---

# UKUI XR 项目实施计划 · M0/M1

本计划把已确定的 UKUI 空间桌面目标落实到现有独立仓库、openKylin 开发环境和分阶段验收。当前执行边界为 M0 合同与 M1 Rust 全景 Shell；后续任务只有达到依赖门槛后才开始集成。本文、[协议合同]({{ '/wiki/xr-desktop-protocol/' | relative_url }})和[组件目录]({{ '/desktop-stack.json' | relative_url }})均不替代原始运行证据。

## 基线与交付边界

可复用底座是 openKylin 3.0 amd64 QEMU、UKUI、Venus/VirGL、Monado 模拟头显/手柄、视角调试输入以及独立晴空数据包。晴空采用最终六方向拼接的 4096×2048 全景，纹理摘要由环境包维护。开发观察端的 WSI CPU 拷贝和视频软件编码单独报告，不作为最终产品窗口传输路径。

M0 开始时，Shell 只有未验收 WIP 代码，尚无 Rust Shell 双眼运行证据；Dock、Kit、窗口服务与 system-ui 是设计入口。环境包构建通过不能晋升 Shell，Monado 示例运行通过也不能晋升本项目。后续实际状态由集成负责人在原始证据成立后更新。

本阶段仅操作开发工作区和现有 QEMU 内的隔离候选会话，不操作物理设备。保留既有 UKUI/Monado 调试入口，并通过独立 profile 明确选择 runtime、ICD、显示与环境；安装软件包不自动启用会话。

## 参考方案的工程适配

| 参考目标 | 本项目实施选择 | 保留或后置的理由 |
|---|---|---|
| 全项目 Monorepo 与六 crate | 各功能保留完整独立 Git；仓内可用 Cargo workspace，集成锁固定版本 | 延续现有维护边界，不压平内核/runtime 历史，不先生成空仓 |
| Rust + Bevy ECS | M1 使用 openxr + ash；ECS 是场景调度的后续候选 | 先验证现有 Vulkan/OpenXR 路径，ECS 不决定 wire 协议 |
| WGSL 程序化材质 | M1 保留 GLSL→SPIR-V；后续评估 WGSL 离线 SPIR-V 适配 | 记录路线选择，不同时引入第二套设备/渲染栈或两套手工材质 |
| core/render/input/widgets/bridge/ffi | 保留职责，分别映射到 Kit、Shell、窗口服务和 system-ui | Kit 不拥有 runtime 提交、Smithay 全部状态或 OS 守护进程 |
| dmm 与三模态交互 | token 可用角尺寸参考；wire 使用明确的米/弧度/逻辑像素；初版先验证可用输入 | dmm 是内部设计单位，手部/眼动须查询能力；capture 优先于 source 切换 |
| 轻薄玻璃、外置 Chrome、Dock | 保留视觉方向；分批实现基础控件、握把/窗口意图与 Dock | 透明、字形、遮挡和动画分别测量，不承诺单 draw call 或固定高帧率 |
| Phase 1 DMA-BUF 与办公浏览器 | 先 Rust 全景，再 IPC/Kit/Dock 与 shm 窗口，最后 DMA-BUF | 先形成正确、可见、可退出的闭环，再优化图像输入 |
| Quad/Cylinder 与 punch-hole | 先 projection 内正确合成，后按 runtime 能力评估混层 | 独立 layer 不自动共享任意深度遮挡；输入导入不等于直接替代 XR swapchain |
| Qt 去 CSD/强制 alpha | 后期显式 opt-in，按具体应用验证 | 主题插件不能保证控制所有 Qt/GTK/浏览器装饰，不全局改变存量应用内容 |
| zbus 系统联动与 C++ FFI | OS 接口由 system-ui 实测适配；稳定协议后做薄客户端 SDK | 先核对真实服务/配置端点，不把示意 API 或回调地址作为跨进程契约 |

这保留了参考方案的产品构想，并把示例代码、性能目标和未验证能力放回对应实施任务。六模块映射如下：core→Kit 纯布局/数据；render→Kit 绘制描述与 Shell Vulkan backend；input→Kit 控件状态、Shell 空间输入、Wayland seat；widgets→Kit 基础控件与独立 Dock 业务；bridge→窗口/图形/系统各自 owner；ffi→协议客户端 SDK。

## M0：合同与工作区

| 任务 | 负责人 | 交付与完成条件 |
|---|---|---|
| M0.1 范围与 source baseline | 主 Agent | 登记各仓 HEAD、分支、dirty 路径和所有者；保留 WIP checkpoint，不打功能验收标签 |
| M0.2 最小 v1 draft | 合同 Agent，Shell/窗口/UI 负责人审阅 | version/capabilities、generation/revision、snapshot/scene、capture/Cancel、shm Release、focus_epoch 定义一致 |
| M0.3 依赖与任务边界 | 合同 Agent | SDK 构建依赖与运行包依赖分开，SDK owner 为 Shell；每个接口和文件有唯一写入者 |
| M0.4 运行与证据规则 | 主 Agent + 验收 Agent | 指定 QEMU 隔离候选、正常退出方式、原始证据字段与晋升条件；不把计划状态改成已运行 |

M0 文档检查通过只代表设计可实施。主 Agent 完成跨仓审阅并记录所用文档提交后，才将 M0 标为集成就绪；本文不自授整项目验收。

## M1：最小 Rust 全景 Shell

M1 交付一个独立 Rust Shell 可执行程序和构建输入。此时可以使用仓内普通 Rust 模块，待接口稳定再提取 crate，不为目录外观先生成占位库。

| 任务 | 唯一实现边界 | 依赖与完成条件 |
|---|---|---|
| M1.1 环境输入 | Shell 环境模块；环境 schema/纹理由 xr-environments 拥有 | 校验 schema、投影/坐标、尺寸和 SHA-256；读取已选晴空，不复制美术到 Shell 源码 |
| M1.2 XR/Vulkan 生命周期 | Shell XR/GPU 模块 | 查询扩展与实际设备，建立 session/swapchain/每眼视图；使用 runtime 预测时刻提交 |
| M1.3 全景渲染 | Shell Vulkan renderer 与 shader | 每眼按世界视线采样 sRGB 全景，转头改变方向，忽略相机平移；分别记录两眼提交 |
| M1.4 视角与观察 | 隔离开发 profile 和现有模拟输入 | 实际改变 yaw/pitch/roll，留下相同输入前后的配对眼图与姿态；平移不产生假近景视差 |
| M1.5 退出与资源回收 | Shell + 自己拥有的候选会话 | 有界退出、正常用户退出与终止请求走清理路径；session/swapchain/GPU 及候选进程资源释放，不误停既有会话 |
| M1.6 包与记录 | 打包 Agent，Shell 负责人配合 | 固定 Cargo.lock、shader 工具与输入摘要；独立包内容/依赖检查；记录构建与目标运行分别成立 |

M1 明确不实现组件 IPC、Dock、Kit 业务控件、Wayland 服务、DMA-BUF、Quad 提升、FFI、Qt 主题修改或系统桌面接管。全景本身是远景，不提供 6DoF 场景几何、地面碰撞或定位质量证明。

M1 只有下列原始证据齐全才晋升：

- 源码提交、构建器/工具链、Cargo.lock 与二进制/环境摘要，启动时实际 runtime/ICD/device 选择。
- OpenXR 会话状态、两眼 view/swapchain/submission 记录；真实渲染出的配对眼图与可见镜像。
- yaw/pitch/roll 前后视图和姿态，以及平移时远景不产生位置视差的检查；两眼远景可能相同，不能仅凭图片不同判断双眼正确。
- Vulkan validation 的实际启用证据与错误计数；任何未启用的检查明确标出。
- 正常退出日志与候选拥有的进程/socket/资源清理记录；原会话存活独立核对。
- 包构建、解包内容与目标 QEMU 运行分别记录，观察端视频帧率与 XR 帧计时分别报告。

若某项失败，保留日志并将对应任务标为未通过，不因能够编译或单张镜像可见而晋升整阶段。

## 后续并行任务与门槛

| 阶段 | 并行范围 | 集成门槛 |
|---|---|---|
| M2 最小组件/窗口管线 | A：Shell 协议与输入；B：Kit/Dock；C：Wayland shm | 以本 v1 draft 为输入发布逐字段 SDK/schema，真实进程握手、快照与断连回收分别验证 |
| M3 真实应用闭环 | A：空间焦点/外框；B：Dock 启动任务；C：seat/resize/popup | 从 Dock 启动一个真实 Wayland 应用，输入、缩放、关闭与资源回收通过；mock 不算应用启动 |
| M4 系统能力 | UI/OS 适配与 IME/Xwayland 兼容分工 | 主题/音量/网络等真实接口、中文组合输入、通知所有权、剪贴板和工作区逐项验证 |
| M5 图形优化 | Shell 导入/合成与窗口服务同步配合 | 真实格式/modifier、acquire/release、帧预算与 layer 遮挡证据；CPU 拷贝与零拷贝分别报告 |
| M6 集成与 SDK | 打包、会话、C/C++ 客户端、独立审阅 | 干净目标系统依赖闭包与升级卸载；amd64、arm64、物理设备分别验收 |

M2 可以让 Kit/Dock 与 shm 窗口服务并行开发；真实 Linux 应用的 Dock 启动验收放在 M3。手部/眼动、overlay 共存、复杂混层和真实设备是额外能力门槛，不作为 M1 默认功能，也不因存在目录就标为支持。

## Agent 与 Git 规则

主 Agent 负责范围、合同合并、跨仓集成锁和验收状态；A 负责 Shell/OpenXR/Vulkan/input；B 负责 Kit/Dock；C 负责 Wayland surface/seat/buffer。会话/打包 D、独立审阅 E 在对应波次复用 worker 槽位，任一时刻不超过三名 worker 加主 Agent。M1 以 Shell、打包与文档任务分工，未到依赖门槛不创建 Dock/Wayland 空仓。

每仓保留真实完整 Git 历史、独立 branch/worktree、真实贡献者身份和实际提交时间；不得压平已有仓库、把他人未提交修改混入本任务，或以源码 ZIP 替代仓库。任务开始记录 base commit 和独占文件范围，结束提交并报告完整 hash；共享协议只能由唯一 owner 合并。集成锁固定源码、SDK、环境和包版本，源码提交不直接代表运行状态。

里程碑保留 git bundle --all、组件锁、包摘要和原始日志；dirty 工作与未跟踪输入先分类保存，凭据/校准/运行缓存不进入公开仓库。此规则是交付记录要求，M0 文档自身不声称 bundle、包或运行验收已经完成。公开文档只使用组件相对路径和版本标识，私有运行环境在本地原始记录中管理。

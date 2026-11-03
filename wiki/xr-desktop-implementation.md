---
layout: default
title: UKUI XR 项目实施计划
---

# UKUI XR 项目实施计划 · M0–M4

本计划把已确定的 UKUI 空间桌面目标落实到现有独立仓库、openKylin 开发环境和分阶段验收。M0 已就绪，M1 Rust 全景 Shell 的 amd64 QEMU 历史范围已接受；M2–M4 已有独立组件候选实现，后续验证目标改为 openKylin 3.0 原生 ARM64 开发机，最终逐项验收仍为 pending。本文、[协议合同]({{ '/wiki/xr-desktop-protocol/' | relative_url }})和[组件目录]({{ '/desktop-stack.json' | relative_url }})均不替代原始运行证据。

## 基线与交付边界

可复用底座是 openKylin 3.0 amd64 QEMU、UKUI、Venus/VirGL、Monado 模拟头显/手柄、视角调试输入以及独立晴空数据包。晴空采用最终六方向拼接的 4096×2048 全景，纹理摘要由环境包维护。开发观察端的 WSI CPU 拷贝和视频软件编码单独报告，不作为最终产品窗口传输路径。

M0 开始时，Shell 只有未验收 WIP 代码，尚无 Rust Shell 双眼运行证据；Dock、Kit、窗口服务与 system-ui 是设计入口。环境包构建通过不能晋升 Shell，Monado 示例运行通过也不能晋升本项目。后续实际状态由集成负责人在原始证据成立后更新。

历史 QEMU 候选与既有 UKUI/Monado 调试入口保留，不再用 QEMU 继续完成后续验收，也不把历史运行结果迁移为原生 ARM64 通过。新目标使用独立 profile 明确选择 runtime、ICD、显示与环境；安装软件包不自动启用会话。本项目范围仍不操作 MIX 头显。

原生 ARM64 目标先完成了只读工具链核对：Rust/cargo 1.93.1、Cairo/Pango/Rsvg、xkbcommon 和 Wayland 开发接口可查询。当时未发现 GLSL 编译器。前次 SSH 最小命令随后返回 `/bin/bash: Input/output error`，文件传输通道也曾不可用；之后目标地址/网络发生变化并重新启动，当前 SSH 已恢复。先前错误的具体原因尚未判定，旧 boot 的工具版本查询不代表新 boot 或图形环境已验收，最小系统依赖由集成负责人统一准备。

前次访问中断前，工程源码、完整 Git 与离线 registry 尚未传入新目标，也未在那里编译或运行组件。访问恢复后开始准备固定输入的迁移；本记录中的原生 Rust 组件编译、原生 Debian 打包和运行验证仍未开始，本地候选包不能称为 ARM64 产物。须先重新核对新 boot 的主机状态，再从固定提交和校验后的缓存执行 offline、locked、受控 jobs=3 构建（与并行构建统筹），并分别记录目标架构、源码、binary 和包摘要。原生构建通过与真实图形/输入/系统验收分别晋升，不改动全局驱动或既有桌面服务。

## 参考方案的工程适配

| 参考目标 | 本项目实施选择 | 保留或后置的理由 |
|---|---|---|
| 全项目 Monorepo 与六 crate | 各功能保留完整独立 Git；仓内可用 Cargo workspace，集成锁固定版本 | 延续现有维护边界，不压平内核/runtime 历史，不先生成空仓 |
| Rust + Bevy ECS | M1 使用 openxr + ash；ECS 是场景调度的后续候选 | 先验证现有 Vulkan/OpenXR 路径，ECS 不决定 wire 协议 |
| WGSL 程序化材质 | M1 保留 GLSL→SPIR-V；后续评估 WGSL 离线 SPIR-V 适配 | 记录路线选择，不同时引入第二套设备/渲染栈或两套手工材质 |
| core/render/input/widgets/bridge/ffi | 保留职责，分别映射到 Kit、Shell、窗口服务和 system-ui | Kit 不拥有 runtime 提交、Smithay 全部状态或 OS 守护进程 |
| dmm 与三模态交互 | token 可用角尺寸参考；wire 明确米、逻辑像素与各字段角度单位，`rotation_degrees` 使用度；初版先验证可用输入 | dmm 是内部设计单位，手部/眼动须查询能力；capture 优先于 source 切换 |
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

## 已接受的 M0/M1 结果

M0 的合同、独立工作区和依赖边界已完成集成审阅。M1 接受范围为 Rust Shell 在现有 openKylin 3.0 amd64 QEMU 中加载晴空、真实双眼渲染、视角变化与正常清理；不将此范围扩展为完整空间桌面或物理设备验收。

最终运行的是安装包中的 `/usr/bin/xr-shell`，二进制 SHA-256 为 `a11972d2ae1137fe24e16320293ad5cadefd308da706d7658296d0434d8860ee`。运行时选中 Monado 和 Venus GPU；晴空摘要与独立环境包一致。

| 原始验收案例 | 结果 |
|---|---|
| direct 参数，未创建或操作 GUI | 经固定 Monado remote 协议输入 front、yaw90、yaw-minus90、rear、pitch20、roll15、translated-x、reset 八组姿态；实际 OpenXR 视图与输入一致 |
| 真实双眼 | 每眼 960×1080、独立 swapchain；40 秒候选窗口内应用记录约 39.31 秒、1843 帧，每眼各提交 1843 次 |
| 全景方向与复位 | 旋转产生对应方向画面；x 平移 1 m 与 reset 均和 front 逐像素一致，符合无限远全景语义 |
| Khronos validation | 客户端 validation 实际启用，错误/警告计数为 0/0；这不是全部 runtime 日志零警告的声明 |
| duration 停止 | wrapper 到期发送 SIGTERM，客户端 request_exit→STOPPING/end；客户端/runtime 均退出 0，候选进程和 IPC 清理 |
| 客户端自退 | `--seconds 12` 记录约 12.02 秒、544 帧，每眼 544 次；客户端/runtime 退出 0，候选清理 |
| 既有入口回测 | 默认 hello_xr 有界 8 秒运行及退出清理通过 |
| 原会话保留 | 同一 boot 下原会话进程的 PID、启动身份、命令和可执行摘要保持；候选资源为空，Kasm 服务无重启 |

最终视角验收以 direct 参数及实际视图/像素为准。早期 GUI 快速点击的姿态结果没有计入通过；GUI 关闭记录仅作为辅助生命周期证据。

已知日志事实：客户端自退与 hello_xr 回测的 Monado 退出阶段出现 `ERROR [do_accept] Failed to wait for id 6`，随后记录 `Server exiting: '0'`。该行保留在原始日志中，不计作 Vulkan validation 错误，也不据退出码 0 删除这条日志。

Kasm 观察路径仍是 GPU 渲染、WSI CPU 拷贝和软件视频编码；这些与 OpenXR 帧计时分别报告。包构建、安装内容和 QEMU 运行已分别检查，干净 openKylin 构建环境中的完整依赖闭包仍待完成。组件 IPC/SDK、Kit/Dock、Wayland 服务、DMA-BUF、混层与 Qt/系统集成不属于 M1 已接受范围；后续候选实现另行验收。

无 GUI 参数验收入口示例，应在已经配置好并空闲的隔离开发会话内运行：

```sh
xr-shell-simulator --headless-input --shell-binary /usr/bin/xr-shell \
  --environment /usr/share/xr/environments/qingkong/environment.json \
  --capture-dir "${XDG_STATE_HOME:-$HOME/.local/state}/xr-shell/captures" \
  --validation --seconds 40
```

`--headless-input` 选择 remote 模拟设备并关闭滑块 GUI；它不会自行发送八组姿态，姿态输入须由对应固定版本的 Monado remote 客户端完成。默认不加此参数仍是交互滑块入口。wrapper 的 `--seconds` 控制候选时限；验证 Shell 自身的 12 秒退出时，改用 `--shell-arg --seconds --shell-arg 12`，不再同时设置 wrapper 时限。

## M2–M4 候选实现与接口边界

M2–M4 的代码和 Debian 配方分别保存在所属完整 Git 仓库。下表描述候选实现的职责，不授予稳定 ABI、目标发行版支持或运行验收。Shell 的协议 SDK 是 wire 定义的唯一代码入口；Kit、Dock、窗口服务和 system-ui 消费固定版本，不能各自复制一份协议维护。

| 仓库/进程 | 本轮候选实现范围 | 与相邻组件的边界 |
|---|---|---|
| `xr-shell` / `xr-shell` | OpenXR/Vulkan 双眼、组件 broker、场景栅格化、窗口空间外框、命中/capture、空间焦点和工作区 | 唯一 XR session owner；OS 调用和 Wayland serial 不进入 Shell；图片仍经 projection 合成 |
| `xr-shell/protocols` / 无独立进程 | 同用户 Unix seqpacket、版本/能力协商、身份外壳、FD 快照传输和 Rust 客户端 | 构建期 SDK；其包不是运行期守护进程，不复制 SDK 到其他仓维护 |
| `xr-spatial-kit` / 无常驻进程 | Widget/Scene、UKUI token、交互状态及捕获取消 | 纯 Rust 模型/数据；不打开显示、相机、runtime 或系统服务 |
| `xr-dock` / `xr-dock` | 实际 desktop entry 目录、固定应用、启动器、任务/工作区模型和窗口意图 | 根据权威快照更新任务；发送 launch/focus/close 意图不能代替应用实际创建或退出 |
| `xr-wayland` / `xr-window-server` | 私有 Smithay display、surface/commit/configure、seat、shm 快照、popup/subsurface、selection 和 IME 协议接入 | Wayland 对象/serial/grab/keymap 的唯一 owner；空间位置由 Shell 决定；Xwayland 是其受管子组件 |
| `xr-system-ui` / `xr-system-ui` | 控制中心、通知和键盘场景；UKUI/GSettings、Pulse、NetworkManager、Fcitx5 适配 | 系统操作在独立 worker，必须返回真实读数/错误；通知名仅显式私有 D-Bus，输入通过已确认窗口焦点 |
| `xr-session-manager` / 显式 desktop profile | 统一选择 runtime/ICD、私有用户目录、D-Bus、音频、Wayland、IME、应用与组件的启停 | 仅管理本轮拥有的进程组、socket 和状态；不复用原 UKUI 会话的 bus、剪贴板或 Pulse 端点 |

候选包按当前实际 `debian/control` 记录；规划中的独立 launcher/control-center/notifications/keyboard 子包和美术资源包不因同仓功能存在就视为已产出。SDK 源码开发包、样式数据、可执行程序与会话 profile 分别检查。安装包不自动启动 XR，也不接管现有桌面。

普通 Snapshot/StateChanged 只携带组件身份、scene revision、node count 和有界状态，避免完整节点树与剪贴板把元数据撑过 64 KiB。完整场景仅通过调试能力下的定向分页读取；剪贴板完整内容使用初连 GetClipboard 和后续 ClipboardChanged，preview 不可用作粘贴全文。组件、系统状态和 selection 必须按 owner/generation 在断连时撤销。

控件通过 Kit 匹配 press/interaction/release：Button 要求在原控件内释放，Cancel 清空交互；Slider 在同一次 capture 的最终 release 提交一次系统操作并读回。按键使用 Linux evdev，文本提交单独报告。Shell 的路由成功、窗口服务接收成功和应用读回文本是三层不同证据，不能用前两层替代最后一层。

## 空间视觉约束 · 新目标待验收

控制中心、键盘和应用窗口必须是分别放置的空间面板。控制中心不是常驻调试表单，键盘不再嵌在其下半部；通知与剪贴板按需独立弹出。默认控制中心只保留清晰分组的应用主题、音量、实际网络状态和基础开关。工作区、IME 诊断和最近操作移到按需详情页，不能为画面整洁而伪造 Wi-Fi、电池、音频端点或成功读数。

| 对象 | 逻辑尺寸 px | 物理尺寸 m | STAGE 中心 m | 旋转 `[Pitch,Yaw,Roll]` 度 |
|---|---|---|---|---|
| 主应用窗口基准 | 由应用发布真实逻辑尺寸 | 由 Shell 的窗口布局维护 | `[0,1.60,-1.80]` | 窗口布局单独记录 |
| 控制中心 | `[600,700]` | `[.60,.70]` | `[1.18,1.60,-1.74]` | `[0,-18,0]` |
| 独立键盘 | `[1000,280]` | `[1.35,.378]` | `[0,.60,-1.35]` | `[25,0,0]` |

正 Pitch 的 UI 语义是面板朝上，STAGE 仍为 +Y 向上、-Z 向前。键盘使用 `Group` 且 `spatial_panel:true`，其 world pose 独立于控制中心；子节点 rect 使用该面板局部逻辑像素。控制中心默认显示、键盘默认收起；实际按钮和 ComponentCommand 必须能展开/收起 control、keyboard、notifications，应用库按钮与 Dock 联动。Dock 保持独立紧凑胶囊，应用库使用按需空间面板，不能通过扩高 Dock 重新拼成长表单。

主窗与控制中心必须保留至少 32 逻辑像素 / .064 m 的视觉间隙，最终按实际投影 bounds 和物理几何分别记录。单独比较中心坐标或世界坐标轴间距不构成通过；主窗宽度、面板 Yaw 和观察位置改变后仍需满足门槛。按钮使用真实局部深度与 hover 抬升，Shell 的独立 GPU quad 和 3D 命中使用同一变换，不能只给平面截图增加阴影。

| 材质/文字 | 固定视觉值 |
|---|---|
| 空间玻璃 | RGBA `(22,26,35,.78)`，始终 dark |
| 主文字 / 次文字 | `#F2F6FC` / `#94A3B8` |
| 强调色 | `#3790FA` |
| 普通按钮 / hover | 白色 12% / 白色 24% |
| 键帽 | 白色 15% 底、清晰白字 |

应用主题按钮只改变真实 OS/应用主题，不把空间玻璃切成浅色。Kit 0.2.0 提供对应 token 与 spatial-group 模型，Shell 负责真实材质、字形、透明合成和 3D 命中，system-ui 负责面板业务与真实适配。本地源码和包已保留这些候选调整；原生 ARM64 上的视觉、深度、间隙和交互全部尚未验收，也不降低既有 M2–M4 功能门槛。

## M2–M4 最终验收表 · pending

早期 amd64 QEMU 候选已用于接通 Dock、真实应用、seat 输入、configure/ack/commit 和 xdg popup；这些是该历史目标上的局部证据，只用于收敛实现，不替代最终版本在原生 ARM64 完整隔离 profile 中的重新验收。最终验收须先固定各仓提交、目标架构、包/安装 binary 摘要、profile 和环境输入，并保存每个子进程的真实退出码。当前下列条目全部绑定新目标且保持 pending，未验证项不得靠 wrapper 忽略异常后晋升。

| 条目 | 最终验收所需原始证据 | 当前状态 |
|---|---|---|
| M2.1 协议与身份 | 真实进程 Hello/Welcome、能力交集、generation；旧身份/错误版本明确拒绝 | pending |
| M2.2 场景和状态 | 场景原子提交/拒绝、Group 几何字段校验、连续 revision、重新取快照；Dock 与 system-ui 同时在线且状态完整 | pending |
| M2.3 shm 生命周期 | 实际应用 commit、sealed memfd、图像可见；accepted/rejected BufferRelease、FD 关闭和断连清理 | pending |
| M2.4 capture 与焦点 | 控件按下/拖出/释放、Cancel、失焦与目标删除；FocusApplied 前不向旧窗口发送输入 | pending |
| M2.5 XR 与退出 | 最终安装 binary 的真实双眼提交、validation 计数；Shell 与组件按 SessionEnding/正常停止全部退出 0，候选资源清空 | pending |
| M3.1 真实启动与任务 | 从 Dock 的实际 scene 节点启动真实应用；surface 创建后任务出现，关闭并确认消失 | pending |
| M3.2 应用输入 | 已确认 focus_epoch、真实 seat 按键；应用 Ctrl+A/C 的 selection 精确读回预定文本，不只统计 InputAccepted | pending |
| M3.3 空间窗口 | 实际拖动、resize 意图和应用 configure/ack/commit 尺寸；逻辑坐标与 buffer 像素各自记录；独立 GPU 面板、32 px/.064 m 间隙与对应 3D 命中 | pending |
| M3.4 子窗口 | 真实 xdg popup/subsurface 的 parent、相对位置、seat/serial；图像与命中一致 | pending |
| M3.5 Dock 与输入策略 | 懒跟随头姿、左右手选择、capture 中不换来源；缺少该输入能力时显式声明不可用 | pending |
| M4.1 UKUI 与网络 | 私有配置下主题操作及读回；NetworkManager 实际链路/连接状态，无 Wi-Fi 设备时不显示虚假无线控制 | pending |
| M4.2 隔离音频 | 私有 PipeWire/Pulse 端点、音量/静音实际操作和读回；虚拟 sink 与物理发声分开 | pending |
| M4.3 通知 | 私有 D-Bus 的真实 Notify、关闭/动作/过期信号；退出释放名称，原 UKUI 通知 owner 保持 | pending |
| M4.4 剪贴板与工作区 | 真实 Wayland selection copy/paste、初连全文、owner 断线撤销；工作区权威快照与输入焦点同步 | pending |
| M4.5 中文与 IME 隔离 | `--no-ime` 下探测不激活 Fcitx；启用时真实客户端 preedit/候选/中文 commit；显式 Unicode 提交另列 | pending |
| M4.6 Xwayland 与会话 | 真实 X11 应用创建/图像/输入/selection；完整私有 profile 清理且原 UKUI/Monado/Kasm 会话身份保持 | pending |

历史 QEMU 中缺少声卡或 Wi-Fi 设备的结论不得套用到原生 ARM64 目标。新目标须重新发现实际端点；私有虚拟音频控制与物理发声分别记录。Fcitx 当前输入法切换及 D-Bus 名称存在不等于中文 preedit；未启动时状态探测不得触发服务自动激活。干净发行版构建闭包、DMA-BUF/显式同步、视频硬件编码、复杂材质、C/C++ FFI、原生 XR 应用切换和头显验收继续按各自后续门槛推进。

## 并行任务与后续门槛

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

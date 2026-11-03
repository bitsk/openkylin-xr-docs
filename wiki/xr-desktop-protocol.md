---
layout: default
title: XR Shell 最小组件合同
---

# XR Shell 组件合同 · v1 draft

合同标识：`xr-desktop/1.0-draft`。本页保留 M0 的最小合同约束。M1 已接受范围只实现 Rust 全景 Shell；M2–M4 已有 `xr-shell/protocols` 0.2.0 Rust SDK 与独立组件候选实现，**最终隔离 profile 互操作验收仍为 pending，尚无稳定 ABI**。逐字段 wire 定义以 Shell 拥有的 SDK 为唯一代码入口；本文与[机器目录]({{ '/wiki/data/xr-desktop-contract-v1.json' | relative_url }})跟随该 SDK，不成为第二份实现。候选范围和最终门槛见[实施计划]({{ '/wiki/xr-desktop-implementation/' | relative_url }})。

## 唯一所有者与通道

| 对象/接口 | 唯一所有者 | 消费者与约束 |
|---|---|---|
| 协议定义与客户端 SDK | `xr-shell` 的协议模块 | Dock、Kit 客户端适配器、窗口服务和 system-ui 消费固定版本；SDK 不是外部未知组件 |
| OpenXR 会话、Vulkan 资源、帧提交 | `xr-shell` | 组件发送场景意图，不发送 GPU 指针或创建第二个 XR session |
| 空间窗口变换、命中、capture、空间焦点 | `xr-shell` | 窗口服务接收版本化焦点和 surface-local 输入 |
| `wl_surface`、commit/configure、seat/grab/serial | `xr-wayland` / `xr-window-server` | Shell 不制造 Wayland serial；Dock 不修改窗口权威状态 |
| 控件模型、布局、样式 token | `xr-spatial-kit` | 提供库与数据，无常驻进程；材质 backend 在 Shell 内使用 |
| 任务与固定应用模型 | `xr-dock` | 缓存快照并发出意图；以 Shell/窗口服务返回结果更新任务状态 |
| 系统适配与组件业务 | `xr-system-ui` | D-Bus/GSettings/IME 适配不进入渲染线程 |
| 启停、XR 应用切换、服务资源归属 | 既有 `xr-session-manager` | 本合同不建立第二套生命周期 API |

场景和图像通道采用 Shell 创建的 Unix `SOCK_SEQPACKET`，位于 `$XDG_RUNTIME_DIR/xr-desktop/`，目录 0700；用 peer credentials 确认同一用户，并限制为会话配置显式启动的组件。它是会话内组件接口，不是任意第三方应用的权限沙箱。控制通道沿用 session-manager 的版本化接口；像素不经 D-Bus、网页或 JSON。

## 版本、能力与消息外壳

第一条数据报为 `Hello`，带 `protocol_major=1`、`protocol_minor=0`、`role`、`required_capabilities` 和 `optional_capabilities`。角色限定为 `dock`、`system_ui` 或 `window_server`。Shell 回送 `Welcome`，选择共同支持的 minor、返回能力交集、限制、`session_generation`、`component_id` 和 `component_generation`。不支持的 major 或缺少 required capability 明确拒绝，不降级成未声明协议。

`core.v1` 是共同基线；其余能力按需协商：

| capability | 消息与语义 | 典型消费者 |
|---|---|---|
| `core.v1` | Hello/Welcome、Error、generation、断连清理 | 所有组件 |
| `scene.v1` | SceneTransaction、SceneApplied、SceneRejected | Dock、system-ui |
| `snapshot.v1` | GetSnapshot、Snapshot、StateChanged、WindowState、WindowRemoved | Dock、system-ui、窗口服务 |
| `input.v1` | InputEvent、Cancel、capture 生命周期 | Dock、system-ui |
| `shm.v1` | ShmBufferCommit、BufferRelease；只读快照 FD | 窗口服务 |
| `focus.v1` | FocusChanged、FocusApplied、带 focus_epoch 的 SurfaceInput | 窗口服务 |
| `window.v1` | 应用窗口意图与窗口服务结果；不能把请求当成已创建/关闭 | Dock、窗口服务 |
| `workspace.v1` | WorkspaceIntent、WorkspaceApplied 与权威 active_workspace | Dock、system-ui |
| `system.v1` | SystemState、TextCommit、KeyIntent；系统调用属于独立适配器 | system-ui |
| `clipboard.v1` | GetClipboard、ClipboardSet、ClipboardChanged；只针对私有会话 selection | system-ui、窗口服务 |

新 capability 必须有实现与对应验收后才能作为已支持能力对外交付；隔离的 M2 验收候选可以启用正在验证的能力，但其机器状态仍为待验收，不把候选握手当成通过证据。M0/M1 均不实现或报告这些组件能力。首版不含 DMA-BUF、任意 shader、直接 OS 操作或远程 TCP 访问。

握手后每条消息包含 `type`、`request_id`、`session_generation`、`component_id`、`component_generation` 和 `payload`。所有 u64 对象 ID、计数器、generation、revision、frame ID 和 epoch 用十进制字符串编码，避免跨语言数值截断；major/minor、尺寸、索引等有界整数仍为 JSON number。浮点只接受有限数。Unix FD 由 SCM_RIGHTS 传递，payload 中的 `fd_index` 仅索引当前数据报附带的 FD 数组。

首版限制为每条元数据最多 65536 字节、每事务最多 128 个操作、每组件最多 512 个节点、节点局部 ID 匹配 `[A-Za-z0-9_-]{1,64}`、单个文本最多 2048 个 UTF-8 字节。`shm.v1` 每个提交恰好一个 FD；其他本页消息不得携带 FD。能力协商可以下调限制，不可超出所选协议定义。超过限制、缺少必需字段、未知消息/操作或消息角色不符时回送有类型的 Error 并释放收到的 FD，不能部分应用消息。Error 至少包含 code、关联 request_id 和可读说明；必要时附当前 revision/epoch。最小 code 为 unsupported_version、missing_capability、stale_generation、stale_revision、invalid_message、wrong_role、limit_exceeded、unknown_surface、stale_focus_epoch、invalid_buffer。

## Generation、revision 与快照

Shell 每次启动产生新的 `session_generation`。每次接受组件连接产生新的 `component_generation`；任何旧 generation 请求均拒绝。Shell 为每个连接建立不可删除的 `root` 节点，初始场景 revision 为 `"0"`；Welcome 返回该根标识。组件局部 node ID 在 `(session_generation, component_id, component_generation)` 内唯一；surface ID 在窗口服务 generation 内唯一。窗口服务重启后旧 surface 不得被新服务同号对象继承。

组件通过 `GetSnapshot` 请求 Shell 的当前模型，`Snapshot` 携带 `state_revision`、组件场景 revision、窗口列表、空间焦点与其 `focus_epoch`。其后的 `StateChanged` 逐条携带连续 `state_revision`；检测到缺号时停止应用增量并重新请求快照。Dock 不把本地“已发送启动/关闭请求”视为窗口已存在/已关闭。

普通快照中的组件只带身份、revision、node count 和 root 元数据，不内嵌全部节点树；完整场景使用显式调试能力下的有界定向分页查询。剪贴板只保留 preview/bytes，全文由初连 `GetClipboard` 和后续 `ClipboardChanged` 提供；null 表示当前无 selection。preview 不可用作粘贴全文。系统和剪贴板状态均需跟随其 owner/generation 在断连时撤销。

窗口服务是应用窗口协议状态权威，Shell 是合并空间状态的发布者。只有 window_server 角色可以发送带服务 generation、surface key 和完整窗口协议状态的 `WindowState` 或对应的 `WindowRemoved`；Shell 验证并更新其已发布模型。Snapshot 中每个窗口保留 `window_server_generation`、`surface_id`、`app_id`、标题、父子关系、逻辑尺寸与应用确认的尺寸状态；空间位置、物理尺寸和工作区另列。旧纹理可临时缩放，但 `resize` 意图不替代应用的 configure/ack/commit。

## SceneTransaction

`SceneTransaction` 带 `base_revision`、`revision` 和操作数组；成功要求 `base_revision` 等于已接受的组件场景 revision，`revision = base_revision + 1`。Shell 先校验整笔事务，再在帧边界原子应用并返回 `SceneApplied`。失败返回 `SceneRejected` 与当前 revision；没有部分场景或部分输入区域生效。

首版仅支持 `upsert_node`、`remove_node`、`set_root_policy`。node kind 为 Group、Card、Text、Icon、Button、Slider、SurfaceReference；父节点必须存在于提交后树中，禁止环和跨组件父引用。样式引用使用 Kit 已支持的 style ID，资源使用 Shell 已登记 ID；协议不传代码、回调地址、shader 或文件路径。root policy 限定 `world_locked` 和 `dock_lazy_follow`，其最终 pose 由 Shell 计算。

以下仅展示 payload，完整消息仍需上述外壳；数字字符串表示 wire u64：

```json
{
  "base_revision": "17",
  "revision": "18",
  "operations": [
    {"op": "upsert_node", "node_id": "launcher", "parent_id": "root",
     "kind": "Button", "style_id": "dock.application", "action_id": "open-launcher"}
  ]
}
```

Group、Card 材质、Text 文本、Icon 资源、Button action ID、Slider 范围和值、SurfaceReference 的 surface key 均由对应 primitive 版本定义；M2 实现时必须发布逐字段 schema 才能报告 `scene.v1` 可用。本文不把示意例子当作已可接收的 wire 实现。

0.2.0 候选 SDK 中，`Group` 可用 `spatial_panel:true` 定义独立空间面板，带 `logical_size`、`size_m`、`position_m` 和 `rotation_degrees`。它使用绝对 STAGE world pose，不继承组件根的世界平移/旋转；子节点仍使用该面板局部逻辑像素。`rotation_degrees` 顺序为 `[Pitch,Yaw,Roll]`，正 Pitch 表示面板朝上。普通 Group 的 rect 则作为树内局部偏移。`visible` 默认 true，隐藏节点的整个子树均不得绘制或命中。Group 的尺寸、有限数和父子关系也属于整笔事务校验，不能让无效变换部分生效。

## InputEvent、Cancel 与 capture

Shell 统一采集输入、命中场景并路由。`InputEvent` 携带 `event_id`、`pointer_id`、`interaction_id`、`source`、`phase`、目标 node/surface key、局部坐标、事件时间和当前 `focus_epoch`。source 限定为 controller_ray、head_ray、mouse；hand_ray/direct_poke 需要后续独立能力，不在首版伪造可用。phase 为 hover、press、drag、release、scroll。时间是会话声明的单调时钟纳秒，与 `XrTime` 的换算在 Shell 内维护，不能混用不同来源的原始时间值。

Shell 接受一次 press 时分配 `capture_id`，将后续 drag/release 送往同一目标，即使指针移出原热区。release 终止 capture。失去 XR 焦点、目标删除、source 切换或连接失效时，Shell 立即取消 capture，并向仍连接的相关方发送带 `capture_id`、`interaction_id` 和 reason 的 `Cancel`；断连一方的取消在 Shell 本地完成。不能把 source 固定优先级用于抢走正在进行的交互。

按下、释放、取消、焦点变化和状态快照不可丢弃或合并。相同 pointer/interaction 的 hover/drag 中间位置可以合并，但必须保持前后的离散事件顺序。客户端回调由客户端根据 action ID 触发，跨进程不传函数指针。

## focus_epoch 与 Wayland seat

Shell 的空间 focus 与窗口服务的 Wayland seat 状态通过 `focus_epoch` 对齐，不互相冒充权威：

1. Shell 决定激活窗口或清空焦点时递增 `focus_epoch`，发送 `FocusChanged`，带目标 surface key 或 null。
2. 窗口服务验证 surface 仍存活，执行合法 seat focus/grab 更新后返回 `FocusApplied`；失败时带原因和当前有效目标。
3. Shell 在对应 epoch 得到成功确认后才发送该窗口的键盘/文本 `SurfaceInput`。等待期间不把键盘送往旧窗口或重放给新窗口。
4. `SurfaceInput` 携带目标 surface generation、`focus_epoch` 和 pointer/interaction 信息。窗口服务拒绝过期 epoch；实际 Wayland serial、popup grab 与 keymap 由窗口服务产生和维护。
5. surface 删除、窗口服务重启或 XR 失焦时撤销 capture、清空对应焦点并建立新 epoch。hover/凝视本身不转移键盘焦点。

## shm BufferCommit/Release

首个窗口图像合同是 `ShmBufferCommit`，不是 DMA-BUF。窗口服务在应用 commit 后、应用仍遵守 wl_buffer 占用语义时，把内容复制到专用 memfd 快照，完成写入后加 `F_SEAL_WRITE | F_SEAL_GROW | F_SEAL_SHRINK | F_SEAL_SEAL`，再发送 FD；创建快照时启用允许 seals 的标志，发送/接收的自有 FD 均设置 close-on-exec。快照复制完成后，窗口服务可以 release 原应用 wl_buffer；接收端随后读取的是不可变快照，不再读取那个已释放的应用 buffer。

surface 必须先经 WindowState 登记；未登记或旧 generation 的提交返回 unknown_surface/stale_generation 并关闭收到的 FD。每个 ShmBufferCommit 包含：

```text
window_server_generation, surface_id, buffer_id, frame_id, commit_serial
width_px, height_px, stride_bytes, offset_bytes, length_bytes, fd_index=0
format: argb8888 | xrgb8888
alpha_mode: premultiplied | opaque
colour_space: srgb
damage_rects: 最多 256 个像素矩形
```

初始 width/height 上限为 8192，快照 length 上限为 134217728 字节；双方可在 Welcome 中下调。必须校验 stride 至少为 `width_px * 4`、offset/stride/height 组合不溢出且完全落在 FD 文件长度内，damage 矩形在图像范围内。argb8888/xrgb8888 使用 Wayland 对应格式的本机小端字节解释；首版 argb8888 对应 premultiplied，xrgb8888 对应 opaque，其他组合拒绝。格式转换和 alpha 解释不能只从文件名推断。

`buffer_id` 每次快照唯一，`frame_id` 在单 surface 中单调递增；首版不复用可写快照。Shell 验证 FD、边界、seals 后复制/上传，完成最后一次对快照的使用后发送恰好一次 `BufferRelease`，带 buffer/frame key 和 consumed、superseded、rejected 或 surface_destroyed 状态。较新帧可以替换尚未使用的旧帧，但旧帧仍需 release。关闭 FD 与释放原 Wayland buffer 是两种生命周期，不能混为同一个 ack。

断连时双方释放自己的 FD 引用并取消对应 surface/资源；不能等待断开的 peer 再回 ack。DMA-BUF 的多 plane、modifier、acquire/release fence 或 syncobj timeline 不属于 `shm.v1`，须在独立合同和真实图像导入验收后增加能力。OpenXR swapchain 由 runtime 创建，Shell 将应用内容采样/拷贝到自己的 swapchain；输入侧 DMA-BUF 导入不自动建立全链路零拷贝。

## 正常结束与系统输入结果

Shell 在自己拥有的正常关闭路径发送 `SessionEnding` 后关闭组件连接。收到该通知的客户端正常退出并回收自己的进程/资源；没有通知的 EOF/传输断线继续报告失败，supervisor 不把子进程异常统一改写为成功。

原生 socket 实测已覆盖握手/能力交集、错误版本、旧身份以及自有隐藏场景的原子拒绝/合法 Group。core-only 客户端收到未协商 StateChanged 的问题已记录；`ce9835d` 用统一 kind→cap 映射过滤广播与定向事件，本地 release 编译通过，但原生构建/复测仍待恢复连接。Error、SessionEnding 与撤销 capture 所需的 Cancel 保持基础控制语义，不因可选事件能力过滤而被屏蔽。

系统 UI 的按键使用 Linux evdev，按下与释放分开；显式字符串提交与真实 IME preedit 分开验收。系统 UI 点击自身控件不抢走应用键盘 focus。Shell 只能将输入送往已确认 epoch 的目标，窗口服务拒绝没有实际 text-input 对象的文本提交并回报错误。路由确认或服务接收不替代应用内容读回。

## M0/M1 状态与晋升条件

M0 可检查本文、机器目录、包/接口唯一所有者和任务依赖是否一致；这只说明合同有定义。M1 可接受的原始证据是 Rust Shell 在现有 QEMU 中加载固定摘要的晴空、真实双眼提交、视角变化与退出清理；M1 不因本文存在而提供 `scene.v1` 或 `shm.v1`。

M2 以后，每项能力分别经过真实进程握手、消息顺序、断线回收和对应可见功能验收，才把机器状态从候选/pending 晋升为 implemented/accepted。后续目标为 openKylin 3.0 原生 ARM64 开发机；历史 amd64 QEMU 结果只保留原目标的适用范围，不跨目标晋升。前次访问中断前未向新目标传输或构建工程；SSH 在网络变化与重启后恢复，固定输入迁移和五仓原生编译打包随后完成，最终逐项运行验收仍为 pending。UI、Wayland、DMA-BUF、原生 XR 应用切换与物理头显分别记录，不能用原生构建通过或一个画面代替整个合同验收。

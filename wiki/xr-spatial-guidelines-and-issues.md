---
layout: default
title: XR 空间开发约束与平面化已知问题
---

# XR 空间开发约束与平面化已知问题

本文档整理当前 `openKylin XR` 空间桌面组件（`xr-shell`、`xr-spatial-kit`、`xr-dock`、`xr-system-ui`、`xr-wayland`）在源码实现中暴露出的**“用 2D/平面 3D 思路开发 XR 应用”**的已知问题清单，并制定全仓库强制执行的 **XR 空间工程与交互约束规范**。


## 当前实现状态：空间交互候选

以下问题描述保留为整改基准。当前候选为 Shell 0.2.18、Spatial Kit 0.2.5、Dock 0.2.6、System UI 0.2.12；源码实现、离线构建、QEMU 场景验证和 MIX 佩戴验收分别记录。**这不是 14 项全部完成或实机基线晋升的声明。**

| 问题 | 本轮源码与验证 | 剩余门槛 |
| --- | --- | --- |
| XR-01 | 窗口由有效双目中心和水平朝向生成，创建/API 移动/抓取时解算无 Roll 的面向用户姿态；任务栏共享位置锚点，各岛分别转向用户 | 多窗口自动排布及所有面板角间隙尚未统一求解 |
| XR-02 | 通知中心从高空位置下移并补偿倾角；QEMU 验证中心仰角约 +2.9° | MIX 双眼图像已核对；颈部舒适度待佩戴确认 |
| XR-03 | 应用库、窗口选择器、控制中心、键盘、通知、剪贴板及 Wi-Fi 面板声明视野相对生成语义；控制中心不再走绝对世界根节点；普通/Wi-Fi 键盘上移以避让转向用户的任务栏 | 多个弹出面板同时显示时的空间避让；MIX 转头后唤出 |
| XR-04 | 对双眼姿态做最短弧四元数中点，双目位置取平均；头部布局与归中共用该结果；已有归中测试覆盖对称眼位倾角和 q/-q | 真实倾斜视场设备 |
| XR-05 | 抓取保存射线距离和局部锚点，在三维射线上移动、实时转向用户，支持手柄平移和摇杆深度，眼距限制 0.6–4 m；QEMU 与 MIX 开发参数射线通过 | 实体手柄抓握、推拉与佩戴手感 |
| XR-06 | 每个指针独立 Hover/Capture；双手抓同窗缩放、释放一手保留另一手已由 QEMU 和 MIX 开发参数射线验证；传统 Wayland seat 保留唯一应用鼠标按钮所有权 | 实体双手并发；组件同一控件仍保留单次动作所有权 |
| XR-07 | Shell 为小控件扩展至少 2.2° 的有效命中热区，按双目眼距及表面投影计算；Kit 提供 dmm 与角尺寸换算 | 密集小控件热区重叠仲裁与实机点选；现有文字布局尚未全部换算为 dmm |
| XR-08 | 按钮默认 +8 mm、Hover +18 mm、Press +2 mm，几何阴影随高度偏移；OpenXR Haptic action、绑定与分级脉冲已接入 | 实体振动器响应、按压行程佩戴确认；QEMU 不算触觉验收 |
| XR-09 | 删除跨窗口采样天空的伪透射；统一线性光 Premultiplied Alpha 合成，透明底色保留背后窗口 | 未实现多 Pass 场景模糊；MIX 光学与材质复核 |
| XR-10 | 卡片/按钮/滑块几何和颜色状态在 GPU 计算；高光依赖每眼视线与表面法线，Cairo 缓存文字与图标，空间路径不再使用九宫格材质高光 | 双目高光的佩戴观感 |
| XR-11 | 命中反馈携带平面/曲面局部法线，准星沿法线偏移并对齐；射线加入柔边、近端淡入、末端收窄 | 射线网格仍是交叉片，真正体积几何待替换；实体控制器与斜曲面画面 |
| XR-12 | 先绘制并写入不透明像素核心深度，再合成透明内容；交叉窗口不再只依赖中心排序 | 透明部分仍按组中心排序，尚未实现每眼透明排序或 OIT |
| XR-13 | 全景及 UI/窗口图集生成 Mipmap，使用线性级间过滤及设备支持的各向异性；图集补齐 16 px 边缘并将采样 LOD 限为 3，避免跨 tile 混色 | 焦点窗口原生 Quad/Cylinder 合成层；MIX 斜视文字与闪烁 |
| XR-14 | 背景、边框、滑块和状态颜色由 Shader 计算；现有回归测试确认 Hover/Press 复用相同位图；左右眼合并为一次录制/提交 | 每帧仍在 XR 图像释放前等待一次完成，图集内容更新仍同步；完整异步资源环尚未实现 |

验证范围：初版三个可执行组件完成锁定依赖的 ARM64 交叉构建；最终 Shell 0.2.18 / Dock 0.2.6 / System UI 0.2.12 和 Kit 0.2.5 在 MIX 原生 ARM64 上完成锁定依赖的离线构建，源码文件摘要与各组件提交逐文件一致。Shell 现有 5 项测试通过；QEMU Venus 独立会话已验证转头 45° 的视野相对唤出、双指针捕获、缩放/释放/深度/朝向/距离限制，以及控制中心、通知和键盘姿态。QEMU 使用模拟头姿及明确标记的开发参数射线，不作为 MIX 追踪、实体手柄、触觉、物理帧率或佩戴验收。原 UKUI 会话和 MIX 已验证基线保留。

MIX 原生范围：真实 QVR 头姿和双眼 1832×1920 图像、真实 Pluma 的输入/剪贴板回读，以及明确标记的开发参数射线共 23 项检查通过。修正了键盘与任务栏角间隙不足、Wi-Fi 密码表单被应用深度遮住两处实机问题。普通键盘与任务栏、Wi-Fi 密码表单与键盘的实测最小球面角间隙均超过 2°。完成旧版/候选/旧版/候选切换及正常场景退出；退出检查覆盖正常结束 OpenXR Session、客户端结束通知、私有总线和应用清理、DMA-BUF 与相关 FD。桌面恢复阶段的短暂多进程检查采用有界重试，停止阶段仍严格验证唯一所有权。快速退出额外试验中，图形资源和客户端握手已通过，但一次相机 producer 返回非零，快速生命周期健壮性仍待修复，不计通过。

原生没有安装 Khronos Vulkan 验证层，不把原生关闭验证时的零计数当作 Vulkan 验证通过。QEMU 验证层结果保留其独立范围。未进行实体双手、振动器、光学清晰度/闪烁、转头唤出或颈部舒适度的佩戴验收，不能沿用上一版 Wi-Fi 的佩戴结论。

---

## 第一部分：当前已知问题清单（2D/平面化思维病灶）

### 1. 空间布局与人体工学问题（把 XR 空间当成“固定 Z 平面上的 2D 坐标墙”）

- **ISSUE-XR-01：应用窗口死板平行于世界 XY 平面，无视用户朝向**
  - **位置**：`xr-shell/src/desktop.rs`（`WindowState` 新建窗口默认 `position`）、`xr-shell/src/spatial_ui.rs`（`window_frame` 固定传入 `[0.; 3]` 旋转角）。
  - **现象与根因**：新窗口生成在固定世界坐标 `[0.15*i, 1.6, -1.8 - 0.1*i]`，且姿态角永远硬编码为 `[0.0, 0.0, 0.0]`。无论用户朝向何方，也无论窗口被拖到左侧还是右侧，窗口始终死板地正对世界 $+Z$ 轴，缺少以用户双眼中点为圆心的球面/柱面切向朝向（Yaw/Pitch Billboarding）。
- **ISSUE-XR-02：通知面板置于反人体工学的高空位置（`Y = 2.38m`）且无俯仰倾角**
  - **位置**：`xr-system-ui/src/scene.rs`（`notifications` 函数：`position = [0., 2.38, -1.78]`，`rotation = [0., 0., 0.]`）。
  - **现象与根因**：基准人眼高度为 $1.60\text{m}$，通知面板中心被硬编码在 $2.38\text{m}$ 高处（高于视线 $0.78\text{m}$，仰角约 $+24^\circ$），且面板垂直于地面（`Pitch = 0°`）。用户在头显内必须大幅抬头（严重超出 XR 颈部舒适俯仰区 $[-25^\circ, +10^\circ]$），且只能以大斜角仰视面板底面。
- **ISSUE-XR-03：按需唤出的系统面板与应用库硬编码在绝对 `STAGE` 坐标，脱离用户当前视野**
  - **位置**：`xr-dock/src/model.rs`（`launcher-space` 固定在 `[0.0, 1.25, -1.68]`，`window-picker` 固定在 `[0.0, 1.2, -1.68]` 且无旋转角）、`xr-system-ui/src/scene.rs`（`control`、`keyboard`、`clipboard` 固定世界坐标）。
  - **现象与根因**：当用户转头（例如偏航 $45^\circ$）或移动位置后唤出应用库、键盘或控制中心时，面板仍然刷在世界原点的固定坐标处，不在用户当前视野正前方，用户必须转头寻找刚打开的界面。
- **ISSUE-XR-04：多处逻辑单取左眼 `views[0]` 代替双目中心（Center Eye / Head Pose）**
  - **位置**：`xr-shell/src/main.rs`（`d.update_head(head_position, [q.x, q.y, q.z, q.w], dt)` 取 `views[0].pose.orientation`）、`xr-shell/src/recenter.rs`（`let q = views[0].pose.orientation`）。
  - **现象与根因**：在非对称 FOV 或存在 canted views（倾斜视场）的头显上，直接用左眼姿态代表头部朝向会产生系统性偏航误差。

### 2. 6DoF 空间操纵与多模态输入问题（用“2D 鼠标拖拽像素”的方式做空间交互）

- **ISSUE-XR-05：窗口拖拽（Grab Move）退化为固定 2D 平面内的 XY 平移**
  - **位置**：`xr-shell/src/desktop.rs`（`Target::Move` 处理分支）。
  - **现象与根因**：抓取窗口移动时，代码将 3D 射线重新投射回原窗口所在的无限 2D 平面，仅修改 `w.position[0]`（X）和 `w.position[1]`（Y），`w.position[2]`（Z）与旋转角完全不变。
  - **后果**：
    1. 用户转动手腕或围绕身体搬运窗口时，窗口无法沿球面环绕用户，也无法自动转向面向用户；
    2. 无法通过手柄前后推拉或摇杆调节窗口远近深度（Z 轴）；
    3. 当手柄射线接近与原窗口平面平行时，平面求交距离 $t \to \infty$，导致窗口瞬间飞出视野。
- **ISSUE-XR-06：双手 6DoF 控制器被强制压缩为“单指针互斥模拟鼠标”**
  - **位置**：`xr-shell/src/input.rs`（`update_routes` 中 `let Some(hand) = hand` 单手选择逻辑）、`xr-shell/src/desktop.rs`（全局唯一 `capture: Option<Capture>` 与 `hover: Option<(u64, String)>`）。
  - **现象与根因**：每帧虽然读取了双手状态，但路由层按优先级只挑出一只手调用 `desktop.ray(..., SOURCES[hand])`。导致双手无法同时悬停两个控件、无法一手抓窗口一手点面板、无法双手捏合缩放窗口；甚至单手摇杆滚动（>`0.18`）都会抢走另一只手的指针路由。
- **ISSUE-XR-07：远场射线交互热区过小，违反 6DoF 射线命中角尺寸门限**
  - **位置**：`xr-dock/src/model.rs`（`tasks-prev` `[84.0, 24.0, 18.0, 24.0]`、`tasks-next` `[598.0, 24.0, 16.0, 24.0]`）。
  - **现象与根因**：在 $1.72\text{m}$ 距离、$0.0016\text{m/px}$ 比例下，$16\text{px}$ 宽的按钮物理宽度仅 $2.56\text{cm}$，视角宽度仅 $0.85^\circ$（远低于 6DoF 手柄射线稳定点选所需的 $\ge 2.2^\circ$ 生理阈值），极难瞄准。
- **ISSUE-XR-08：按钮按压缺乏 Z 轴下压行程，且全链路缺失 OpenXR 触觉反馈（Haptics）**
  - **位置**：`xr-shell/src/spatial_ui.rs`（`animate` 仅根据 `hovered` 抬升 `0.012m`，未区分 `pressed` 下压）、`xr-shell/src/input.rs`（未绑定 `/output/haptic` 及 `xrApplyHapticFeedback`）。
  - **现象与根因**：控件按下时只改变 2D 贴图颜色，没有物理按键向内回弹的 Z 轴行程变化，且跨越控件边界、按下、释放时没有任何手柄震动反馈，交互缺乏空间触感。

### 3. 空间光学、材质与深度感知问题（假折射穿模与 2D 烘焙贴图）

- **ISSUE-XR-09：空间玻璃材质（`material == 1.0`）直接采样无限远天空盒，导致后方窗口“隐身穿模”**
  - **位置**：`xr-shell/shaders/spatial.frag`（第 35–40 行 `texture(panorama, eq + ...)`）。
  - **现象与根因**：半透明玻璃面板（Dock、应用库、控制中心）在 Fragment Shader 中计算透射模糊时，直接采样了 `binding = 0` 的无限远 `panorama` 全景图，并将 `(transmitted - original)` 强行叠加到像素上。当玻璃面板位于某个应用窗口前方时，面板不仅没有模糊背后的应用窗口，反而把背后的应用窗口画面洗掉、透出了最后方的蓝天背景，造成严重的深度遮挡关系反转。
- **ISSUE-XR-10：使用静态 2D 切图（9-slice PNG）与固定 UV 渐变冒充空间高光，缺乏双目视差**
  - **位置**：`xr-shell/src/art.rs`、`xr-shell/src/raster.rs`（`canvas.skin`）、`xr-shell/shaders/spatial.frag`（`float spec = mix(.10, .02, clamp((local_uv.x + local_uv.y) * .5, 0., 1.))`）。
  - **现象与根因**：面板与按键的光泽来自静态 2D 位图贴皮和写死在 2D `local_uv` 对角线上的线性插值，左右眼看到的镜面高光完全停留在同一 UV 像素上，没有随每只眼的视线向量（`toward_eye`）产生双目高光视差（Binocular Specular Disparity），在立体视觉下呈现明显的“平面贴纸感”。
- **ISSUE-XR-11：控制器射线为硬边纯色十字片，且准星（Reticle）未对齐命中表面法线**
  - **位置**：`xr-shell/src/controller_model.rs`、`xr-shell/shaders/controller.vert`（第 17 行 `world += rotate_q(view.orientation, vec3(position.xy * view.controls.z, 0))`）。
  - **现象与根因**：射线没有手柄近端渐入与命中端汇聚的体积感；末端圆环准星始终朝向头显相机（`view.orientation`）而非贴合命中面板的表面法线（Surface Normal），无法在斜视或曲面窗口上向用户传达表面的三维倾角。

### 4. 深度遮挡、走样与渲染管线问题（CPU 逐控件光栅化与零 Mipmap）

- **ISSUE-XR-12：空间 UI 关闭深度写入，仅靠面板中心点距离排序（Painter's Algorithm），斜向/大尺寸面板易穿插**
  - **位置**：`xr-shell/src/renderer.rs`（`create_spatial_pipeline` 中 `depth_write_enable(false)`；`update_panel_poses` 按 `p.anchor` 到 `origin` 的欧氏距离排序）。
  - **现象与根因**：在三维空间中，宽 $1.59\text{m}$ 的窗口与侧面倾斜面板之间无法仅靠中心点距离决定逐像素前后关系；且 `origin` 仅传入了双眼中点，未针对左右眼分别排序或利用深度缓冲处理非透明主干区域。
- **ISSUE-XR-13：全景、Atlas 与窗口纹理全部 `mip_levels = 1` 且无各向异性过滤，斜视与头部微动时高频闪烁**
  - **位置**：`xr-shell/src/renderer.rs`（`upload` 与 `Atlas::new` 中 `mip_levels(1)`、`max_lod(0.0)`、`SampleCountFlags::TYPE_1`）。
  - **现象与根因**：XR 头显中人头时刻存在生理微颤，缺少 Mipmap 与各向异性过滤（Anisotropic Filtering）会导致斜向放置的文字、图标和细边框产生明显的摩尔纹与像素跳动；同时焦点应用窗口尚未提升至 OpenXR `CompositionLayerQuad` / `CompositionLayerCylinderKHR`，经历了额外的重采样损失。
- **ISSUE-XR-14：将 UI 几何背景/按钮状态与 CPU Cairo 位图耦合，且单眼渲染间同步阻塞 Fence**
  - **位置**：`xr-shell/src/spatial_ui.rs`（`panels()` 在 hover/press 变化时重新调用 Cairo 绘制整张控件位图并重传 Atlas）、`xr-shell/src/renderer.rs`（`submit()` 每次调用 `wait_for_fences(..., u64::MAX)`，每帧渲染左右眼加上传共阻塞等待 2~3 次）。
  - **现象与根因**：把圆角背景、按钮变色、滑块轨道当作 2D 截图在 CPU 上重画，违背 GPU 驱动的空间 UI 架构；单帧内多次 CPU-GPU 同步等待吞噬了宝贵的帧时间预算。

---

## 第二部分：全仓库 XR 空间开发与架构约束规范

自本规范生效起，`openkylin-xr-repos` 下所有涉及空间桌面、UI 组件、窗口服务与渲染的改动必须遵守以下五条硬性工程约束：

### 约束 1：空间坐标系与人体工学规则（Spatial Ergonomics & Placement）
1. **球面/柱面切向朝向法则（User-Facing Orientation Rule）**
   - **禁止**将任何应用窗口、独立空间面板或弹窗以固定 `rotation = [0, 0, 0]` 放置在偏离正前方的世界坐标上。
   - 所有世界锁定（`world_locked`）的窗口与面板在创建、重置或被拖拽移动时，必须根据其中心相对于当前用户**双眼中点（Center Eye Pose）**的向量自动解算朝向：
     - **水平朝向（Yaw）**：面板法线水平分量必须指向双眼中点；
     - **垂直倾角（Pitch）**：根据面板相对于眼高的俯仰角自动补偿倾斜（低于眼高时面板向上仰起，高于眼高时向下俯倾，保持面板法线与视线夹角 $\le 15^\circ$），保持 `Roll = 0°`（水平不歪斜）。
2. **颈部与视野舒适区（FOV & Neck Comfort Zone）硬门限**
   - 常驻与高频交互面板中心必须落在相对眼高俯仰角 **$[-25^\circ, +10^\circ]$** 区间内，**严禁**将通知或工具面板放置在仰角 $> +15^\circ$ 的高空位置。
   - 默认工作区水平展开角控制在 **$[-35^\circ, +35^\circ]$** 内；相邻空间面板之间必须保留至少 **$2.0^\circ\sim 3.0^\circ$ 的真实视角弧长间隙（Angular Gap）**。
3. **基于当前水平视线（Horizon-Relative Spawn）的动态生成**
   - **禁止**新启动的应用窗口或按需唤出的系统面板（应用库、键盘、控制中心、通知、窗口选择器）直接硬编码生成在绝对 `STAGE` 原点前方。
   - 必须以**唤出瞬间用户的水平视线方向（Horizon-projected Head Yaw）与当前眼高**为基准坐标系，在用户当前正前方或相对球面偏角（`yaw_offset_deg, pitch_offset_deg, distance_m`）处生成，生成后再锁定到世界空间。

### 约束 2：6DoF 空间操纵与多模态输入规则（6DoF Manipulation & Input）
1. **球面极坐标拖拽与深度推拉（Spherical Grab & Depth Control）**
   - **禁止**使用“射线与原 2D 平面求交计算 $\Delta x, \Delta y$”的方式实现窗口移动。
   - 抓取窗口移动时，必须绑定抓握点在控制器 `aim` 射线上的初始距离 $d_0$ 与局部锚点偏移：随手柄射线旋转，窗口沿以用户双眼中点为圆心的球面/柱面平滑移动并实时更新面向用户的 `rotation`；支持在抓握期间通过摇杆前后或手柄推拉调节窗口深度 $d \in [0.6\text{m}, 4.0\text{m}]$。
2. **双指针独立并发（Dual-Hand Concurrent Ray & Capture）**
   - **禁止**在输入层每帧将左右手压缩为单一活跃手。
   - 左右手控制器必须拥有各自独立的 `PointerId`、独立的 Hover 状态、独立的射线/准星渲染反馈与独立的 `Capture` 槽位，支持双手同时悬停、一手抓握窗口一手操作 UI、以及双手协同缩放窗口。
3. **空间热区角尺寸下限（Angular Hit-Target Minimums）**
   - 所有远场射线（Controller Ray / Head Ray）可点击控件，在所在面板基准距离下的**最小有效命中角尺寸不得小于 $2.2^\circ$**（等价于 $\ge 38\text{dmm}\sim 44\text{dmm}$）；视觉图形较小时必须提供扩展的空间碰撞热区（Hit Slop）。
4. **三维按压行程与触觉反馈（3D Z-Travel & Haptics）**
   - 可交互控件必须具备真实的三维深度状态：默认抬升 $z_0$、`Hover` 向前浮起（如 $+10\text{mm}$）、**`Press` 向面板内下压（如 $+2\text{mm}$）**，并在阴影偏移上同步反映高度变化。
   - 射线划入可交互控件、按下、释放与抓握吸附时，必须通过 OpenXR `xrApplyHapticFeedback` 提供分级触觉脉冲。

### 约束 3：空间光学、材质与真实深度线索（Spatial Optics & Depth Cues）
1. **严禁“假折射穿模”与 2D 静态贴皮高光**
   - **禁止**在空间面板 Shader 中跨过背后已渲染的应用窗口去采样 `panorama` 天空盒伪造透明折射；**禁止**依赖静态 2D PNG 切图（`art.rs`）冒充三维光影。
   - 在未引入多 Pass 场景级背景模糊前，空间玻璃必须使用正确合成的 Premultiplied Alpha 半透明底色 + 基于真实每眼视线向量 `toward_eye` 与表面法线的 **Fresnel 边缘高光和双目视差镜面高光（Binocular Specular Highlight）**；环境贴图仅可作为微弱表面反射（Reflection）叠加，不得破坏后方窗口的遮挡关系。
2. **三维体积射线与法线对齐准星（Normal-Aligned Reticle）**
   - 控制器射线必须具备近端渐隐、中段柔和、末端汇聚的深度渐变效果；射线命中准星（Reticle）必须贴合命中表面的三维法线方向（Surface Normal Aligned）并在命中点沿法线微移防闪烁，直观呈现平面与曲面的空间倾角。

### 约束 4：XR 排版度量、清晰度与反走样规则（Angular Typography & Clarity）
1. **统一 `dmm`（Distance-Independent Millimeter）度量**
   - 建立以 $1\text{m}$ 距离下 $1\text{mm}$ 物理尺寸（$1\text{dmm} \approx 0.0573^\circ$）为基准的排版规范：面板逻辑分辨率与物理尺寸保持统一密度（基准 $1000\text{px/m}$，结合超采样或图层直通）；正文字号 $\ge 20\text{dmm}$，次要文字 $\ge 16\text{dmm}$。
2. **强制 Mipmap、各向异性过滤与高频图层直通**
   - 所有 3D 空间中的 UI Atlas 与窗口纹理必须生成 Mipmap 链并启用线性 Mipmap 过滤（`SamplerMipmapMode::LINEAR`）与各向异性过滤，消除斜视与头部微动时的闪烁。
   - 焦点应用窗口在 Runtime 支持时优先提升为 OpenXR `CompositionLayerQuad` 或 `CompositionLayerCylinderKHR` 原生合成层，避免二次透视重采样损失文字锐度。

### 约束 5：渲染管线与组件协议边界（Pipeline & Protocol Architecture）
1. **几何材质全 GPU SDF 化，Cairo 仅负责字形与图标光栅**
   - `Card`、`Button`、`Slider` 的圆角矩形 SDF、玻璃/按键材质、侧壁厚度（Extrusion）、边框高光及 Hover/Press 颜色与位移过渡全部由 Vulkan Shader 实时计算；CPU Cairo/Pango 仅负责文字排版与图标光栅化缓存。Hover/Press 状态切换不得触发 CPU Cairo 重绘或 Atlas 纹理重传。
2. **组件协议原生支持球面锚定与朝向策略**
   - `xr-shell-protocols` 与 `xr-spatial-kit` 必须支持相对视野生成（`head_horizon_spawn`）、面向用户朝向（`user_facing_billboard`）及球面偏角定位（`yaw_deg, pitch_deg, distance_m`）；业务组件（`xr-dock`、`xr-system-ui`）声明空间语义与偏角，由 `xr-shell` 统一解算世界坐标与三维旋转矩阵。
3. **帧内异步录制，禁止单眼同步等待 Fence**
   - 渲染管线必须按帧管理 CommandBuffer 与同步原语，双目绘制在同一提交批次内完成，禁止在左右眼渲染之间调用 `wait_for_fences` 阻塞 CPU。

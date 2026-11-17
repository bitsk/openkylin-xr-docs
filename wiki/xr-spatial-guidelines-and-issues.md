---
layout: default
title: XR 空间开发约束、独立审计与画面防抖指南
---

# XR 空间开发约束、独立审计与画面防抖指南

本文档整合了 `openKylin XR` 空间桌面组件（`xr-shell`、`xr-spatial-kit`、`xr-dock`、`xr-system-ui`、`xr-wayland`）以及底层运行时（`xr-monado`、`xr-head-tracking`）的四个核心部分：
1. **Shell 0.2.18 / Kit 0.2.5 / Dock 0.2.6 / System UI 0.2.12 审计基点及后续修复记录**；
2. **转头画面抖动（Head-Turn Judder / Swim）的源码风险、运行时边界与实测门槛**；
3. **XR 手柄交互实施规范与验收清单（设计要求，以后文落地记录区分实现状态）**；
4. **全仓库 XR 空间开发与架构强制约束规范（含历史 14 项平面化问题对标）**。

---

## 第一部分：Shell 0.2.18 审计基点及后续修复记录

### 1.1 总体审计评级：**B+（空间化范式已完成转型，留有 1 处高优命中 Bug 与 5 处中低优体验尾项）**

对当前仓库最新源码的独立审计表明，此前暴露的“把 2D 桌面死贴在固定 Z 平面墙上”的 14 项平面化问题中，**10 项已在几何、协议和 Shader 层面实质性解决**：
- **双目对称中点与视线相对生成（`XR-01..04` 已落地）**：`xr-shell/src/recenter.rs` 实现了 `center_eye()`（双目位置均值 + 四元数半球符号对齐最短弧中点）；`spatial.rs` 实现了无 Roll 的 `user_facing()` 与 `horizon_spawn()`；通知面板从反人类高空 `Y = 2.38m` 下移至 `Y = 1.69m`（仰角约 $+2.9^\circ$），且协议层强制拦截 `yaw_deg.abs() > 35°` 与 `pitch_deg` 超出 `[-25°, +10°]`。
- **6DoF 射线距离抓握与双手并发（`XR-05, 06, 08` 已落地）**：`input.rs` 改为左右手双路独立路由，支持双手独立 Hover/Capture 与双手捏合缩放窗口（`Bimanual`）；窗口抓握改为沿 3D 射线保持抓握距离与局部锚点，支持摇杆深度推拉（$0.6\sim 4.0\text{m}$）；接入了按钮 Z 轴按压行程（默认 $+8\text{mm}$、Hover $+18\text{mm}$、Press $+2\text{mm}$）与 OpenXR 分级触觉脉冲。
- **光学材质、深度隔离与 Mipmap（`XR-09..14` 主体已落地）**：`spatial.frag` 删除了跨窗口采样天空盒的假透射，改为线性光 Premultiplied Alpha 合成与逐眼半角高光（具备真实双目高光视差）；拆分了 `opaque_pipeline`（写深度）与 `spatial_pipeline`（半透明）；全景与 UI Atlas 均启用了 Mipmap、各向异性过滤及 `16px` 重复边缘防渗色；Hover/Press 状态切换实现零 Cairo 重绘，双目合并为单次 `render_stereo` 提交。

### 1.2 独立审计新发现的代码缺陷清单（Audit Findings）

| 编号 | 级别 | 位置 | 缺陷现象与代码根因 | 修复要求 |
| :--- | :---: | :--- | :--- | :--- |
| **AUDIT-01** | **高优 Bug** | `xr-shell/src/spatial_ui.rs`<br>(`nearest_ray_pick` / `hit_local`) | **同平面扩展热区（Hit Slop）按遍历顺序吞噬相邻按钮**：同面板上的同级按钮处于相同三维平面与深度，射线交点距离 `t` 完全相等。先遍历的小按钮向外扩展 $2.2^\circ$ `slop` 后满足 `t < nearest` 并更新 `nearest = t`，导致后遍历的相邻按钮即便被射线**直接命中真实几何内部**，也因 `t < nearest` 为 `false` 而被吞掉。 | 当 `(t - nearest).abs() < 1e-4` 时，真实几何命中（`inside_rounded`）必须无条件优先于扩展 `slop` 命中；若同为 `slop` 命中，取距各自控件边界更近者。 |
| **AUDIT-02** | **中优缺陷** | `xr-shell/src/desktop.rs`<br>(`observe_horizon_head`: 2003–2008) | **多窗口生成大幅重叠且第 4 个窗口起 100% 共面重合**：主窗口宽 $1.59\text{m}$ 在 $1.8\text{m}$ 处视角宽约 $47.6^\circ$，而新窗口偏角按 `[0°, -24°, +24°][index % 3]` 循环。前 3 个窗口横向重叠约 $23.6^\circ$；第 4 个窗口（`index = 3`）与第 1 个窗口坐标、深度、旋转完全一致，引发共面遮挡/闪烁。 | 按已有窗口实际角宽度计算无重叠偏角，或对 `index >= 3` 引入深度阶梯（$\Delta d$）与俯仰偏移。 |
| **AUDIT-03** | **中优缺陷** | `xr-shell/src/spatial_ui.rs`<br>(`update_grab`: 1070–1100) | **双手捏合缩放（Bimanual）只放大物理尺寸，未重排应用分辨率**：双手缩放仅将 `w.size` 乘以 `ratio`，释放时未像单手 `Target::Resize` 那样发送 `WindowIntent::resize`，导致窗口拉大后像素密度减半、文字模糊。 | 双手缩放释放后按新的物理尺寸与基准 `dmm` 密度向 Wayland 窗口服务请求 `resize` 重绘（或明确限制最大无重排倍率）。 |
| **AUDIT-04** | **中优缺陷** | `xr-shell/src/desktop.rs`<br>(`Target::Resize`: 1844–1895) | **单手角落缩放未更新面向用户朝向，且伴随按钮无按压行程**：单手拖拽角落缩放改变了窗口中心 `w.position`，但未重新调用 `user_facing()` 更新 `w.rotation`；此外窗口底部伴随控件（Grab/Close/Minimize/Curve）未纳入 `animate()` 的 Z 轴按压位移。 | 缩放更新 `w.position` 后同步重算 `w.rotation`；将伴随控件纳入 Z 轴 Hover/Press 位移计算。 |
| **AUDIT-05** | **低优缺陷** | `xr-dock/src/model.rs` &<br>`xr-system-ui/src/scene.rs` | **跨组件弹出面板缺少空间避让**：`xr-dock` 的应用库（`yaw=0°, pitch=-10°, d=1.72m`）与 `xr-system-ui` 的空间键盘（`yaw=0°, pitch≈-11.5°, d≈1.65m`）同时打开时均位于正前方视野中央，发生前后遮挡。 | Shell 维护全局活跃弹出面板槽位，或按类别分配不冲突的俯仰/偏航频段。 |
| **AUDIT-06** | **低优缺陷** | `xr-shell/src/renderer.rs`<br>(`padded_pixels` / `update_panels`) | **动态窗口高频更新时仍存在 CPU 扩边拷贝与额外 Fence 阻塞**：当应用窗口每帧提交新 `ShmBufferCommit` 时，`padded_pixels` 在 CPU 上逐像素拷贝整窗并做 `16px` 扩边，且 `update_panels` 内单独执行一次 `submit() + wait_for_fences`，导致单帧两次阻塞等待。 | 用 GPU Blit / 纹理数组替代 CPU 端逐像素 `padded_pixels` 扩边，并将图集上传合入帧主 CommandBuffer。 |

### 1.3 审计修正候选（Shell 0.2.19 / Dock 0.2.7）

| 项目 | 源码修正 | 验证范围 |
| --- | --- | --- |
| AUDIT-01 | 近等深命中优先选真实圆角几何；同为扩展热区时比较到边界的米制距离 | 现有 Shell 回归检查增加相邻小按钮直接命中与近边界仲裁 |
| AUDIT-02 | 同深度层按实际角宽和 3° 间隙寻找槽位；不足时增加深度与俯仰层级 | QEMU 与 MIX 均验证四个真实应用窗口；分层可有前后遮挡，不宣称任意数量全局无重叠 |
| AUDIT-03 | 双手缩放保存基准尺寸/逻辑分辨率；首只手释放时请求应用 resize，保留另一只手抓握；取消恢复原尺寸 | QEMU 与 MIX 应用分辨率回读通过；缩放上限同步受逻辑分辨率限制 |
| AUDIT-04 | 角落缩放使用按下时朝向作基准，中心变化后重新面向双目；伴随按钮增加 8/18/2 mm 默认/悬停/按压行程 | QEMU 与 MIX 几何、应用重排与按压位移检查通过 |
| AUDIT-05 | 应用库/窗口选择器使用 +8° 上方频段，键盘保留约 -11.5° 下方频段 | QEMU 约 4.78°、MIX 约 4.61° 的面板边界间隙通过；尚非任意头姿变化下的全局排布求解器 |
| AUDIT-06 | CPU 仅顺序复制原始像素；GPU 扩展八块边缘，上传/Mipmap/双目共用帧提交；延后销毁 staging 至完成 fence | 本机 ARM64 构建与运行通过；QEMU 开启 Khronos 验证层，错误/警告 0/0；native 未启用该层 |

验证记录：Shell 现有 5 项测试通过（含补充的相邻按钮断言）；当前完整组件组合通过 QEMU 18 项交互检查、每眼 1,863 帧及正常退出；MIX 21 项功能检查、旧包回退与再次进入通过。开发参数射线不代表实体手柄验收。新候选保持运行，三次交接采样头部位置/朝向均为 tracked，但双手均未显示；没有由此推断追踪稳定性已修复。

佩戴、实体双手精度、触觉和转头稳定性继续保持待验；下文 0.2.18 表格保留为审计基线，不能当作新候选的验收结果。

---

## 第二部分：转头画面抖动（Head-Turn Judder）根因分析与 XR 防抖技术体系

在佩戴头显转头时，如果虚拟世界无法像真实物理世界一样“死死钉在空间中”，而是出现**画面滞后拖影、果冻摆动（Swim）、重影跳帧（Judder）或边缘拉伸变形**，前庭系统（内耳平衡觉）与视觉系统就会产生剧烈的感官冲突（Vestibulo-Ocular Conflict），在数秒至数分钟内诱发眩晕与恶心。

以下分开记录**源码中可确认的行为**与**仍需实机测量的假设**。当前原生会话使用 SteamVR/OpenXR；`xr-monado` 独立合成器与设备接入代码不能直接代表该会话的完整时序。源码中出现等待、缺少外参或没有预测，均不能单独证明佩戴时的具体抖动根因。

### 2.1 五个需要闭环的时序与标定问题

1. **独立 Monado 路径尚未按目标时刻预测。** `xr-monado/src/xrt/targets/common/a081_device.c::get_pose` 忽略 `at`，在新鲜度门限内返回最近的 QVR 关系或 IMU 融合朝向。该路径没有前向预测，但并不等于 ATW 完全失效：`comp_renderer.c::calc_pose_data` 会再次读取设备姿态，两次读取之间的新样本仍可形成旋转重投影差。两次读到同一个样本也不能单独证明 ATW 被禁用。需要记录 sample、locate、合成器重采样与显示时间，才能量化残余延迟；不能由源码直接断言滞后大于 20 ms。
2. **真实追踪原点到双目中心的外参仍待标定。** `xr-head-tracking/include/xr/head/a081_qvr_pose.h` 当前做坐标轴变换，不含经过实测确认的 tracker-to-eye 平移。缺少该标定是准确性门槛，但不能拿通用头部尺寸代替设备测量。真实 6DoF 不应叠加猜测的颈部模型；3DoF 不能通过伪造位置或位置有效标志来冒充 6DoF。
3. **上传与渲染等待需要区分。** Shell 0.2.18 的动态图集上传有独立提交与等待；0.2.19 将原始像素上传、GPU 扩边、Mipmap 和双目绘制放入同一提交。提交完成后的等待仍用于保护 OpenXR 图像释放、读回与资源生命周期。独立 Monado 的 `renderer_wait_queue_idle` 是否可去除，必须结合它的实际后端、fence 和 present 生命周期验证；它也不是当前 SteamVR 合成器的代码。不能仅凭这些调用推断必然从 72 跌至 36 fps。
4. **扫描方向、扫描耗时与发光时刻缺少本轮测量。** `calc_pose_data` 在设备报告 compositor info 时请求扫描起止姿态，但 A081 接入未提供这组实测信息。刷新周期不等于有效扫描或发光时间。不得把 12.5 ms、通用显示分辨率或其他配置的呈现偏移直接作为校准值写入驱动。
5. **光学与边缘稳定性仍需佩戴对照。** 原厂 FOV/畸变参数的接入不等于已确认个人眼位、镜片中心和全视场精度。需要同场景中心/边缘观察与真实运动参考，区分畸变误差、追踪误差、丢帧和纹理重采样。

当前 `qiyu-vr-runtime/patches/monado-a081.patch` 的 SteamVR 桥接路径使用 `a081_qvr_latest_pose` 和样本对应的 `poseTimeOffset`。 `qiyu-head-tracking/include/qiyu/head/production-v2/a081_qvr_pose.h` 还已有 OEM 运动状态到线速度、角速度及有效标志的转换，这与独立 `xr-head-tracking` 的较旧接入不同。下一步应核对已安装驱动所消费的具体头文件/版本和实测数据，而非再次从零增加一套预测。预测责任及速度坐标约定必须按这条实际链路检查，不能在独立 Monado、桥接层和 SteamVR 中重复预测。

### 2.2 防抖技术与本项目的验证边界

- **时钟与预测：** 保留传感器样本身份、时间基准、单位、轴向与新鲜度。仅在角速度/线速度确实有效且坐标系已确认时上报相应标志；按目标时刻预测前还须确认运行时是否已承担预测。QVR 字段名称不足以证明 `angular_s / angular_b / angular_bdt / angular_bdt2` 的物理含义。
- **晚期姿态更新与重投影：** 对比应用渲染姿态和合成器晚期姿态，分别测量应用帧率、合成帧率、呈现间隔和真实运动响应。队列优先级或 ATW 开关本身不是延迟验收结果。
- **位置重投影：** 深度层、运动矢量及运行时实现决定可补偿的范围，不能用无效位置或外推兜底掩盖追踪失效。
- **扫描补偿：** 只有测得扫描方向、起止时间且目标管线支持相应插值后才启用。零扫描时间表示缺少该信息，不能擅自填入刷新周期。
- **低余晖与呈现校准：** 必须以面板实际发光和显示模式为依据；本轮没有光电测量，因此没有 MTP 或低余晖通过结论。
- **原生 Quad/Cylinder 层：** 可以减少某些重采样步骤，但不保证消除所有闪烁或模糊。单独提升焦点窗口之前，需要验证运行时层数、alpha、空间、独立 swapchain，以及窗口与射线、键盘、伴随控件之间的遮挡顺序。
- **稳定参考与内容设计：** 保留清晰的地平线和适当的纹理频率；视觉内容改善不能替代追踪与时序验证。

### 2.3 后续运行时修正门槛

| 项目 | 必须先取得的证据 | 当前处理 |
| --- | --- | --- |
| 按 `at` 预测 | 样本时间、速度单位/参考系、有效性与实际预测责任 | 保持真实有效性门限；未验证前不添加外推或强行置位速度标志 |
| Tracker-to-eye | 本机真实刚体标定与坐标变换验证 | 不使用猜测的眼位/颈部偏移 |
| 扫描补偿 | 目标显示模式的扫描方向、耗时、发光时刻及后端支持 | 不写入猜测的 12.5 ms，不跨运行时套用 -11 ms |
| 去除合成器 QueueWaitIdle | 当前启用后端的同步、资源复用、呈现与退出证据 | 本轮先消除 Shell 的独立图集提交；保留必要完成等待 |
| 焦点 Quad 层 | 实际运行时层合成与跨层遮挡验证 | 保持当前投影路径；独立 swapchain/遮挡设计待闭环，不把普通 atlas image 直接作为 XR swapchain 提交 |

### 2.4 七项技术的实施顺序与交付物

先沿实际启用的 SteamVR 链路推进；独立 Monado 的改动作为单独候选验证，不能混用结论。每项保留开关、旧包和同场景 A/B/A，软件时序与物理 MTP 分开报告。

| 顺序 / 技术 | 实现归属与具体改动 | 通过条件 |
| --- | --- | --- |
| **P0 / 1. 运动状态与前向预测** | `xr-head-tracking` / `qiyu-head-tracking` 明确样本时间、陀螺仪单位、body/world 坐标和 tracker-to-eye 外参；`qiyu-vr-runtime` 核对 SteamVR 桥接的真实运动状态与 `poseTimeOffset`，确认唯一预测责任。独立 `xr-monado::get_pose` 在这些条件确认后再接 `m_predict_relation`，预测区间根据实际时序限定。先一阶，再评估二阶；不凭 OEM 字段名解释系数。 | 用真实转动/停转数据检查时钟、轴向、延迟和过冲；无效/陈旧样本不参与预测，不能重复预测或制造位置有效性。 |
| **P0 / 2. 晚期锁存与 ATW** | 从生产者到应用 locate、合成器采样、GPU 完成、present 记录同一时间线。SteamVR 路径先审计桥接输入和现有运行时统计；独立 Monado 再检查晚期采样位置、队列调度和每帧资源生命周期。Shell 已合并图集/双目提交，后续资源环按实测阻塞点推进。 | 分开统计应用帧率、合成呈现间隔、丢帧及 p50/p95/p99；同场景负载变化时检查晚期姿态更新。不得通过盲删 fence/WaitIdle 换取表面帧率。 |
| **P1 / 6. 原生 Quad/Cylinder 层** | `xr-shell` 查询目标运行时层数与格式；为焦点平面应用窗口建立独立 swapchain，直接上传窗口内容并提交 Quad。保留圆角 alpha、尺寸、空间和世界姿态；单独处理射线/键盘/伴随控件的前景层。跨层无法保持原有遮挡时回退投影；曲面随后单独实现 Cylinder。 | 原生双目文字捕获、远近遮挡、指针命中、切焦点/曲面切换和退出资源回收均通过，且 A/B/A 观察清晰度变化；不把一次采样等同于零抖动。 |
| **P1 / 7. 稳定参考与可选晕影** | `xr-shell` 保持世界锁定地平线、舒适位置和低频场景；检查头动时是否有不必要的重定位或 follow 抖动。动态晕影作为用户可关闭的辅助选项，默认不因真实转头收窄视野；若用于人工移动，单独定义触发与恢复曲线。 | 转头时世界参考稳定、文字和输入不被遮挡；分别佩戴对照，不用晕影掩盖追踪丢失。 |
| **P2 / 4. 滚动扫描补偿** | 在 `qiyu-kernel` / 面板资料与实测中确认扫描方向、双 DSI 时序、旋转映射、有效扫描时间；独立 Monado 才据此报告 compositor info 并贯通 begin/end 到实际选中的计算或图形畸变路径。SteamVR 内部实现单独核实，不视为可修改代码。 | 用真实旋转与高速/光电测量比较首尾扫描误差；方向与双眼时序均正确，不能以 72 Hz 倒数代替扫描耗时。 |
| **P2 / 5. 低余晖与发光锁相** | 先确认面板和驱动是否支持可控发光窗口。经原厂参数与测量确认后，在 `qiyu-kernel` 的面板驱动中做可回退配置，并通过 `qiyu-vr-session` 绑定验证过的显示模式与呈现参数；不盲写 DCS 或套用通用脉宽。 | 光电/高速测量发光窗口、亮度和 MTP，同时观察闪烁；若硬件不支持则明确记录限制，不能纯软件承诺实现低余晖。 |
| **P3 / 3. 位置重投影 / ASW** | 真实 6DoF 稳定后，`xr-shell` 在运行时支持时提供真实深度层，校对近远平面与深度约定；完整 ASW 还需对应运行时的运动矢量接口和合成实现。深度层扩展可用本身不代表 ASW 已启用。 | 真实平移、遮挡显露和低应用帧率场景中检查错误拉伸/重影；位置失效时按真实能力降级，不放宽有效性门限。 |

接口依据：[OpenVR Driver API](https://github.com/ValveSoftware/openvr/blob/master/docs/Driver_API_Documentation.md) 定义姿态时间偏移和运动数据坐标；[OpenXR 合成规范](https://registry.khronos.org/OpenXR/specs/1.1/html/xrspec.html#compositing) 定义层合成顺序；[SpaceWarp 层接口](https://registry.khronos.org/OpenXR/specs/1.1/man/html/XrCompositionLayerSpaceWarpInfoFB.html) 同时要求运动矢量与深度输入。目标运行时是否支持相应接口仍须现场枚举。

第一批具体交付应是**时间线与运动数据契约、单一预测链路、帧阻塞归因**。第二批交付**带可靠投影回退的焦点 Quad**。面板扫描和低余晖使用独立测量窗口；ASW 排在追踪与深度可信之后。


### 2.5 首批落实：运动审计、单一预测责任与实验 Quad

Shell 0.2.21 保留生产 SteamVR 路径，默认仍使用投影；`--focus-quad=auto` 或
`XR_FOCUS_QUAD=auto` 显式启用实验 Quad，`--focus-quad=off` 强制回退。

- **运动数据**：`qiyu-head-tracking/tools/audit-motion.py` 以会话用户订阅现有 relay，
  保存接收端 monotonic 时标及原样 240 字节 v2 包，不新增传感器所有者。30 秒
  14,812 包通过原始质量/时间门限，无序号缺口或时间倒退；观察端样本年龄
  p50/p95/p99 为 2.78/6.93/9.12 ms。这是基本静止的只读 tap 证据，既不是驱动
  实际 poseTimeOffset 采样，也不是转动、停转、过冲或物理 MTP 验收。
- **单一预测责任**：已核对活动驱动摘要、构建记录及生产 v2 头文件。生产者发布
  样本时刻状态，恢复滤波只推进到该样本时间，OpenVR 桥接原子读取姿态与时间，
  传递负样本年龄和有效速度，由 SteamVR 向显示时刻预测。Shell 使用运行时给定
  的显示时间 locate，不再次积分；本批没有叠加独立 Monado 预测器或放宽有效性。
  tracker-to-eye 平移外参、物理轴向/尺度仍未校准；不能据此认定全部运动问题已修复。
- **时间线**：Shell 保留最近 4096 个渲染帧的 CLOCK_MONOTONIC 阶段边界；捕获时
  保存 `timeline.json`，正常帧不写遥测磁盘。运行时支持 timespec 转换时才记录
  XrTime/monotonic 配对。应用 wait、locate、输入/图集、交换链等待、提交及完成等待、
  release/end 分开统计；完成等待不是 GPU 执行时间。另用既有 OpenVR 观察工具记录
  合成 GPU、应用间隔、累计提交/呈现/丢帧。SteamVR 内部逐帧晚期锁存和物理扫描
  时标仍不可见，不把该时间线称为完整 ATW/MTP 验证。
- **焦点 Quad**：独立交换链按窗口内容分辨率渲染，保留线性光预乘透明度、圆角、
  尺寸与世界姿态；按背景投影、焦点 Quad、前景投影的顺序提交。曲面、相交/共面
  几何、背向视图、资源不足及尚未分层的实体手柄几何均回退原投影。窗口命中几何
  不变；捕获明确区分背景双眼、前景双眼和 Quad，不能把背景图当完整头显画面。
- **性能门限未通过**：初始投影 / Quad / 回退投影的运行时帧序号节奏约
  66.45 / 33.56 / 34.78 Hz；合成 GPU 中位约 4.03 / 6.31 / 4.11 ms。
  回退未恢复初始节奏，说明热态/运行状态等因素尚未隔离，不能把帧率变化全归因于
  Quad；额外合成开销也不能忽略。因此保留实验开关，日常启动使用投影，暂不宣称
  Quad 改善延迟、吞吐或佩戴清晰度。最终版显式 Quad / 默认投影复测为约
  33.32 / 68.49 Hz，合成 GPU 中位 6.06 / 4.10 ms；应用提交及完成等待中位
  16.49 / 10.80 ms，交换链获取等待仅 0.025 / 0.013 ms。优先降低额外渲染/合成
  成本，仍需进一步区分 GPU 执行与队列等待，不能直接删除完成同步。

最终版本 QEMU 7 项检查、每眼 4516 帧（原生层提交 69 帧）、Khronos 0 错误/0 警告及正常退出通过；
实机最终版实验 Quad 的 10 项检查通过，退出前每眼 9774 帧，其中 1232 帧提交原生层；
最终版正常退出/采集退出与资源清理通过。较早一次回退停止中相机 capture exit 1，Shell 退出与
DMA/FD 清理仍通过；保留该原有生命周期故障观察，不将其标为整个运行栈通过。
扫描缓冲读回不是原子拍摄，局部黑条也不能直接归因于真实面板；物理显示仍待验。

原始证据保留在 `reports/device/openkylin-xr-prediction-quad/` 与
`reports/emulator/openkylin-xr-prediction-quad/`；构建/原生图层功能、软件呈现统计与
佩戴验收分别记录。待佩戴项目包括真实转动/停转、文字抖动与清晰度、远近遮挡、
实体手柄点选/拖动及触觉。扫描补偿、低余晖、ASW 未因本批审计而标记完成。


### 2.6 采集退出顺序与 Quad 开销优化

Shell 0.2.22 保留默认投影。实验 Quad 新增前景投影裁剪；
`XR_FOCUS_QUAD_CROP=off` 可保留全幅前景，用同一二进制比较开销。

- **采集退出**：旧异常的四路 QCFT 都有约 373.45 ms 的真实 SOF 间断，
  无效记录出现在同步门控关闭前约 65–78 ms，不能当作正常停止尾帧放行。
  已启动的命令场景改为先停唯一采集生产者：降门控、排空、释放 DSP 缓冲后，
  才请求 Shell 销毁图形资源。分别记录采集停止请求/完成与场景停止时刻；
  坏帧、曝光相位、时钟和清理检查均不放宽。最终 A/B/A 三次采集停止约
  0.92/1.03/0.97 秒，全部先于场景退出，capture 与清理通过，FastRPC 对象/字节
  均为 0、CDSP FD 为空。旧间断的直接触发原因仍未证明，历史失败继续保留；
  有限次正常退出不代表长时采集问题已全部根治。
- **透明层修复**：共享渲染 pass 的颜色附件采用 `DONT_CARE`，不能保证未写像素
  透明。读回检查实际发现缩放后的旧像素残留，现对前景和 Quad 在绘制前显式清零。
  发现此问题前的试跑和误用旧二进制的模拟器复测均排除出最终结果。
- **裁剪方式**：按真实左右眼视图分别包围全部前景平面及厚度，留两像素余量，
  同步裁剪 Vulkan 渲染区域、OpenXR sub-image 与 FOV。保持原像素密度、位姿、
  显示时间和三层遮挡顺序；跨近裁剪面或非有限值保留全幅。捕获时清零全图，
  另记实际提交矩形，便于检查透明边界。交换链分配与完成同步仍保留。

同版本、同编辑器内容与相同摆放规则的实机 A/B/A 结果如下。每段先预热，
随后采样约 30 秒；温度、CPU/GPU 频率与实际眼位姿随原始数据保存。

| 阶段 | 前景方式 | 合成 GPU 中位耗时 | 运行时帧序号节奏 |
| --- | --- | --- | --- |
| A1 | 全幅 | 6.19 ms | 42.24 Hz |
| B | 裁剪 | 5.86 ms | 33.73 Hz |
| A2 | 全幅 | 6.20 ms | 33.70 Hz |

本场景前景提交面积降至全幅的 5.05%，合成 GPU 中位耗时约降低 5.4%，
且回到全幅后该耗时恢复。但帧序号节奏未改善，回退也未恢复首段水平；
因此只记录局部开销收益，**整体吞吐门限仍未通过**，不能认定严格相同热态/运动
已被隔离，更不能把帧序号节奏当作物理刷新率或 MTP。Quad 继续保持实验开关。

最终 QEMU 每眼 4501 帧、62 帧原生 Quad，9 项检查、Khronos 0 错误/0 警告及
退出清理通过；6 张前景读回图通过透明边界检查。实机 A1/B/A2 分别通过
7/11/7 项检查，B 段 8 张前景读回图通过边界检查，曲面回退、缩放、输入回读及
参数射线释放正常。扫描缓冲复制期间发生帧缓冲切换，其图像仅作辅助记录，
不能作为原子合成或物理面板证明。原始证据：`reports/device/openkylin-xr-quad-cost/`。

佩戴文字清晰度、转头/停转稳定性、真实双手命中/拖动与触觉继续待验；
本轮不改内核、启动分区和预测责任，不晋升镜像基线。

---

### 2.7 手柄出现时抖动加重：隔离结果与 Shell 0.2.24

用户报告整幅画面与手柄模型/射线均抖动。Shell 0.2.23 增加开发态模型/射线
显示隔离及有界逐帧观察；隐藏绘制仍保持真实动作、grip/aim、命中与捕获。
实机双手有效时，全部显示／仅射线／全部隐藏／恢复显示的运行时帧序号节奏为
32.69／32.38／34.93／43.55 Hz。前三段均有两个应用窗口，最后一段变为一个；
用户运动轨迹也不相同，因此不能把数值差当成模型的独立成本。前三段没有显示
“去掉模型即恢复流畅”，暂不支持把网格绘制认定为唯一原因。

- 120 秒只读生产 tap 收到左/右 5,705/5,704 包，全部满足原有效性条件；样本间隔
  中位约 20.92 ms，接收时样本年龄中位约 0.56/0.64 ms，最大约 9.15/10.01 ms，
  没有时间倒退。生产发布约 47.5 Hz；对应驱动保留负 `poseTimeOffset`，当前包
  不携带速度，驱动也未填充速度。该链路需要后续结合预测与实际运动检查，
  不能仅凭更新率认定抖动根因，也不能私加一层预测或伪造有效姿态。
- 第一段有连续 7 帧头部 `POSITION_TRACKED` 清除（flags=7，首末样本相隔约
  392 ms）；朝向仍 tracked。其余测量段头部均 tracked。这个实际退化窗口保留，
  不能用双手有效包覆盖它，也未证明由手柄遮挡或模型开销触发。
- 同轮日志存在 11–14 MB 全图集上传及几十毫秒 CPU 准备峰值。源码在等待前
  和输入后各准备一次图集；后一次遇到尚未提交的批次会正确地退回全量快照，
  防止丢失已暂存的更新，但放大了成本。0.2.24 将准备合并到输入后每渲染帧一次，
  保留批次完整性、同步、原像素密度和 Mipmap；同时用一次父子索引遍历替代
  重复节点扫描，伴随控件动画只遍历可见窗口，不再构建整个场景几何。

同一 0.2.24 二进制提供仅开发态的 `spatial_fast_path` 开关。相同编辑器内容、
光标与悬停回放的原路径／优化／原路径／最终优化对照如下；统计排除采图帧。

| 路径 | 等待前与输入后 CPU 准备合计中位 | P99 | 运行时帧序号节奏 |
| --- | --- | --- | --- |
| 原路径 A1 | 3.47 ms | 17.11 ms | 67.82 Hz |
| 优化 B | 2.39 ms | 15.76 ms | 69.52 Hz |
| 原路径 A2 | 3.28 ms | 14.77 ms | 69.48 Hz |
| 最终优化 | 2.34 ms | 14.46 ms | 69.27 Hz |

准备中位耗时约下降 27–31%；P99 和吞吐未呈现同等幅度的稳定改善。此次重启后
双手均无有效姿态，而且四段都没有发生全量图集上传，因此该轮只证明当前场景的
CPU 收益，**没有验证有实体模型时的最终收益，也没有证明全量上传峰值已实机消除**。
不能与前面的双手活动场景直接比较 FPS。编辑器文字回读、正常关闭通过；
QEMU 12 项模拟双手/布局/伴随悬停检查、9,036 帧及 Khronos 错误/警告 0/0 通过。

首轮对照因非原子写入采图请求，Shell 读到短暂空标签并退出，随后记录
`pose_watch_log_drain` 清理失败；原始失败保留，未降低验收门槛。采样工具改用
同目录临时文件加原子替换后重跑完整对照，最终正常退出与清理通过，重入后
连续三次头部位置/朝向 tracked。当前为默认投影、全部手柄绘制开启、优化路径开启；
OEM 模型/握持/瞄准参数和追踪生产链没有修改。实体双手复测与佩戴抖动改善仍待验，
不晋升系统基线。

---

## 第三部分：XR 手柄完整交互逻辑实施规范（Agent 专项开发指导书）

### 3.1 现状诊断：当前手柄输入链路缺失了什么？

通过对 `xr-shell/src/input.rs`、`xr-shell/src/desktop.rs`、`xr-wayland/src/server.rs` 与 `xr-controller-tracking` 的逐行审计，当前手柄交互存在以下 **5 大功能断层**：

1. **扳机键（Trigger）与主选键（Select / A / X）混为一谈，且无模拟量迟滞（Hysteresis）**：
   - `input.rs:515` 直接写 `let pressed = state.select || state.trigger > 0.5;`。
   - **问题**：单阈值 `0.5` 在手指悬停半按扳机时极易发生临界抖动（Chatter），连续触发误点击；且将食指扳机（`trigger`）与拇指按键（`A/X`）完全混同，无法区分“主点击”与“辅助点击（右键）”。
2. **抓握键（Squeeze / Grip，中指侧键）只采集未消费（Dead Code）**：
   - `input.rs:342` 读取了 `squeeze`（对应 A081 硬件 `middle_finger_raw` 或 Oculus Touch `/input/squeeze/value`），并记录到了日志（第 396 行 `state.squeeze > 0.5`），但在 `update_routes` 中**没有任何消费逻辑**！
   - **问题**：在 XR 空间交互标准中，用户指望用**中指抓握键（Squeeze）直接随时随地抓取/移动眼前的整个窗口（即使射线指在窗口内容区，无需费力去瞄准底部细小的 Grab 胶囊）**，而用**食指扳机键（Trigger）点击窗口内的 2D 按钮/文本**。目前缺失抓握键逻辑，导致用户移动窗口极其费力。
3. **完全缺失鼠标右键（Right-Click / Context Menu，`BTN_RIGHT = 273`）**：
   - `desktop.rs:1196, 1515, 1921` 硬编码了 `"button": 272`（Linux `BTN_LEFT`）。
   - **问题**：在 Linux 桌面应用（如 Pluma 编辑器、Peony 文件管理器、Firefox 浏览器、终端）中，用户完全无法呼出右键上下文菜单！虽然 `xr-wayland/src/server.rs:671` 已经支持接收任意 `button` 编号并转发给 Smithay，但 `xr-shell` 从未发送过 `273`（`BTN_RIGHT`）或 `274`（`BTN_MIDDLE`）。
4. **完全缺失空间防抖双击（Double-Click Stabilization）與长按右键辅助**：
   - **问题**：在 6DoF 空中手持手柄连续按两次扳机键时，手腕不可避免会产生 $0.3^\circ \sim 1.0^\circ$ 的微小抖动，映射到 $1.8\text{m}$ 外的窗口上就是 $10\sim 30\text{px}$ 的坐标漂移。传统 X11/Wayland 应用（如 Peony 双击打开文件夹、Pluma 双击选中单词）要求两次点击之间指针位移 $\le 4\sim 5\text{px}$，导致在 XR 中用射线几乎无法成功双击！此外，窗口顶栏/伴随栏也缺乏双击最大化/恢复或双击归中的空间快捷语义。
5. **摇杆（Thumbstick）交互粗糙且存在多窗口抢占**：
   - `input.rs:599-606` 在任意一只手推动摇杆时，直接调用 `desktop.controller_scroll(scroll)` 发送给 `self.focus`（当前键盘焦点窗口），**完全不检查这只手的射线正指在哪个窗口或哪个组件面板上**！
   - 当一只手抓握窗口（`grabbing == true`）时，摇杆 Y 轴调节深度，但摇杆 X 轴被静默丢弃（无法在抓握时用摇杆 X 轴快速水平环绕或调节窗口曲率/大小）；且摇杆点击（Thumbstick Click `/input/thumbstick/click`）未绑定。

---

### 3.2 XR 手柄按键与动作映射总表（Target Interaction Matrix）

Agent 在修改 `xr-shell`、`xr-wayland` 及底层驱动时，**必须严格按以下交互矩阵实现**：

| 物理输入源 | OpenXR Action 路径 | 迟滞阈值 (Press / Release) | 在应用窗口表面 (`Target::Window` / `SurfaceReference`) 的行为 | 在空间外框 / 伴随栏 / 系统 UI (`Target::Node` / `Companion`) 的行为 | 在抓握态 (`Target::Move` / `Squeeze Grab`) 下的行为 |
| :--- | :--- | :---: | :--- | :--- | :--- |
| **食指扳机**<br>*(Trigger)* | `/input/trigger/value`<br>`/input/trigger/click` | **按下 $\ge 0.65$**<br>**释放 $\le 0.35$** | **鼠标左键 (`BTN_LEFT = 272`)**：<br>• 单击：激活焦点 + 左键按下/释放<br>• 拖拽：左键框选文本 / 拖动应用内滑块<br>• **空间防抖双击**：400ms 内二次按下且抖动 $< 1.2^\circ$（或 $< 24\text{px}$）时，强制锁死在第一次点击的 $(x_1, y_1)$ 坐标发送第二次点击 | • 激活系统 UI 按钮 / 拖动音量滑块<br>• 在窗口 Grab 胶囊上：启动 6DoF 球面抓握移动<br>• 在窗口 Grab 胶囊上**双击**：切换窗口曲面/平面或标准尺寸复位 | 若另一只手已用抓握键抓住同一窗口，第二只手扣扳机或抓握键抓住该窗口即进入**双手捏合缩放（Bimanual Scale）** |
| **中指抓握键**<br>*(Squeeze / Grip)* | `/input/squeeze/value`<br>`/input/squeeze/click` | **按下 $\ge 0.60$**<br>**释放 $\le 0.35$** | **任意位置直接 6DoF 空间抓握窗口（Direct Window Grab）**：<br>• 射线指在窗口**任意位置**按住抓握键，无需瞄准底部 Grab 胶囊，立即将该窗口切换为 `Target::Move` 6DoF 球面搬运模式（不向应用内发送左键点击，避免误触应用内容）<br>• 双手同时在窗口任意两处按住抓握键：立即进入**双手捏合缩放** | • 在可移动空间面板上：抓握移动或重锚定<br>• 在固定 UI 控件上：不触发误点击 | 保持抓握移动；释放抓握键（$\le 0.35$）结束窗口搬运或退出双手缩放 |
| **副按键 (A / X)**<br>或 **扳机静止长按** | `/input/a/click` (右)<br>`/input/x/click` (左)<br>`/input/trackpad/click` | 布尔边沿<br>或长按 $650\text{ms}$ | **鼠标右键 (`BTN_RIGHT = 273`)**：<br>• 射线指向窗口时按下 A/X 键，立即激活窗口焦点并在当前射线交点发送 `pointer_motion` + `BTN_RIGHT (273)` 按下与释放，唤出应用右键菜单<br>• **无 A/X 键的简单手柄回退**：扳机键在窗口内按下后保持 $650\text{ms}$ 且射线位移 $< 0.8^\circ$（`< 12px`），自动取消左键并触发一次右键点击 + 中等触觉脉冲 | 预留快捷上下文操作（如在 Dock 图标上按右键直接打开该应用的“窗口选择器 `window-picker`”） | 抓握窗口期间按 A/X 键：将窗口立即重置为正对用户的标准垂直/水平距离并切换曲面 (`Curve`) |
| **返回/菜单键**<br>*(B / Y / Menu)* | `/input/b/click` (右)<br>`/input/y/click` (左)<br>`/input/menu/click` | 布尔上升沿 | • 若存在活跃 Capture，先取消 Capture<br>• 若窗口有打开的 `xdg_popup`（如右键菜单）或对话框，向焦点窗口发送 `Escape (keycode = 1)` 关闭弹窗 | 收起当前打开的二级弹出面板（如 Wi-Fi 密码键盘、应用库、窗口选择器） | 立即取消当前窗口移动/缩放 Capture，将窗口还原至抓握前的位置与尺寸 |
| **摇杆二维推杆**<br>*(Thumbstick XY)* | `/input/thumbstick`<br>`/input/trackpad` | 径向死区 $r \le 0.18$<br>指数曲线响应 | **射线指向目标优先滚动（Hovered-Target Scroll）**：<br>• 优先向**当前该手射线所指向的窗口**发送二维平滑滚动（`scroll dx, dy`），若射线未指向任何窗口才回退到当前 `focus` 窗口<br>• 支持非线性加速（微推慢滚逐行阅读，满推快滚翻页） | 当射线指向支持翻页的组件面板（如 `xr-dock` 应用库、`window-picker`、`xr-system-ui` Wi-Fi 列表）时，摇杆左右/上下超过 $\pm 0.65$ 触发带冷却（$280\text{ms}$）的翻页 | • **摇杆 Y（前后）**：推拉调节抓握窗口深度 $d \in [0.6\text{m}, 4.0\text{m}]$（速度 $1.2\text{m/s}$）<br>• **摇杆 X（左右）**：微调窗口宽度缩放或沿球面偏航角平移 |
| **摇杆按下**<br>*(Thumbstick Click)* | `/input/thumbstick/click` | 布尔边沿 | **鼠标中键 (`BTN_MIDDLE = 274`)**：<br>• 在窗口内触发中键点击（用于 Linux Primary Selection 粘贴、浏览器中键打开/关闭标签页） | 在空闲空间双击或长按：快捷切换任务栏 `fixed / follow` 模式 | 抓握窗口时按下摇杆：将窗口距离一键复位至标准舒适阅读距离 $1.80\text{m}$ |
| **Home 键长按** | `A081_CONTROLLER_POSE_SOCKET` (`home.rs`) | 持续按住 $1000\text{ms}$ | 全局空间归中（`recenter`）：重置水平参考空间原点与偏航角，取消所有进行中的手部 Capture | 同左 | 取消抓握并归中 |

---

### 3.3 分模块详细代码实现指南（供执行 Agent 逐步落实）

#### 模块一：`xr-shell/src/input.rs` —— 动作集扩展、模拟量施密特触发器与双击/长按状态机

1. **分离 Action 定义与硬件 Profile 绑定**
   - **修正当前错误绑定**：当前 `input.rs:167-173` 将 Oculus Touch 的 `x/click` 和 `a/click` 绑到了 `select` 上，导致 `A/X` 键与扳机键完全混同。
   - **正确绑定方式**：
     - `trigger` (`Action<f32>`)：绑定 `/user/hand/{hand}/input/trigger/value`（Touch / Index / A081）及 `/input/trigger/click`（Vive）；
     - `select` (`Action<bool>`)：仅用于 `/interaction_profiles/khr/simple_controller` 的 `/input/select/click`；
     - **新增 `secondary` (`Action<bool>`)**：专门用于鼠标右键！绑定 Oculus Touch 的 `/user/hand/left/input/x/click` 与 `/user/hand/right/input/a/click`，以及 Vive 的 `/input/trackpad/click`；
     - `squeeze` (`Action<f32>`)：绑定 `/user/hand/{hand}/input/squeeze/value`（Touch）以及 `/user/hand/{hand}/input/squeeze/click`（Vive）；
     - **新增 `thumb_click` (`Action<bool>`)**：专门用于鼠标中键/复位！绑定 `/user/hand/{hand}/input/thumbstick/click`。

2. **为 Trigger 和 Squeeze 引入施密特触发器（Schmitt Trigger Hysteresis）**
   - 严禁使用单一阈值 `state.trigger > 0.5`。在 `Input` 中为每只手维护 `trigger_latched: [bool; 2]` 与 `squeeze_latched: [bool; 2]`：
   ```rust
   const TRIGGER_PRESS_THRESHOLD: f32 = 0.65;
   const TRIGGER_RELEASE_THRESHOLD: f32 = 0.35;
   const SQUEEZE_PRESS_THRESHOLD: f32 = 0.60;
   const SQUEEZE_RELEASE_THRESHOLD: f32 = 0.35;

   fn schmitt(latched: &mut bool, value: f32, press_th: f32, release_th: f32) -> bool {
       if *latched {
           if value <= release_th {
               *latched = false;
           }
       } else if value >= press_th {
           *latched = true;
       }
       *latched
   }
   ```

3. **实现抓握键（Squeeze）独立路由通道**
   - 当某只手 `squeeze_latched` 产生上升沿（`new_squeeze_press`）时：
     - 调用 `desktop.ray_squeeze(ray.origin, ray.direction, "press", SOURCES[hand])`；
     - 在 `desktop.rs` 中，若射线命中 `Target::Window(k)`、窗口的 `SurfaceReference` 或窗口伴随栏（`Grab / Orb`），**直接创建 `Target::Move(k)` 抓握 Capture**，而不向 Wayland 窗口发送鼠标点击！
     - 若左、右两只手同时对同一个窗口 `k` 持有 `Target::Move(k)`（无论是通过底部 Grab 胶囊还是通过 Squeeze 键抓握窗口任意位置），立即激活现有的 `Bimanual` 双手缩放逻辑（`spatial_ui.rs:994-1012`）！
     - 抓握成功瞬间触发一次沉稳的低频长触觉脉冲 `(amplitude: 0.55, duration_ms: 22)`，与普通按钮点击的清脆脉冲 `(0.35, 12ms)` 形成明显的触感区分。

4. **摇杆非线性响应曲线与射线悬停目标路由**
   - 将当前线性的死区映射升级为**二次方精细控制曲线（Quadratic Precision Curve）**，解决微推摇杆时滚动过快、满推时又不够快的问题：
   ```rust
   fn stick_curve(v: f32) -> f32 {
       const DEADZONE: f32 = 0.18;
       if !v.is_finite() || v.abs() <= DEADZONE {
           0.0
       } else {
           let norm = (v.abs() - DEADZONE) / (1.0 - DEADZONE);
           v.signum() * (0.35 * norm + 0.65 * norm * norm)
       }
   }
   ```
   - 滚动路由改为传入当前手的射线：`desktop.controller_scroll_at(SOURCES[hand], ray.origin, ray.direction, scroll)`，优先滚动当前手射线所指的窗口，无需先点击切换焦点即可边看参考文档边滚动。

---

#### 模块二：`xr-shell/src/desktop.rs` —— 鼠标左/右/中键、空间防抖双击与抓握仲裁

1. **扩展 `pointer_surface` 支持多鼠标按键（左键 `272` / 右键 `273` / 中键 `274`）**
   - Linux `input-event-codes.h` 定义：
     - `BTN_LEFT = 272` (`0x110`)
     - `BTN_RIGHT = 273` (`0x111`)
     - `BTN_MIDDLE = 274` (`0x112`)
   - 在 `Capture` 与 `pending_pointer` 中增加 `button: u32` 字段（默认 `272`）。
   - 当用户按下副按键 `secondary`（A/X 键）或中键 `thumb_click` 时，通过 `desktop.ray_button(origin, direction, phase, source, button_code)` 路由：
     - 若目标窗口尚未获得焦点（`!self.focus_applied`），`pending_pointer` 保存 `{"x": local[0], "y": local[1], "button": button_code}`，待 `FocusApplied` 到达后发送对应 `button` 的按下事件；
     - 若目标窗口已获得焦点，直接发送 `pointer_motion` + `pointer_button(button_code, pressed)`；
     - 当 `cancel_source` 取消 Capture 时，使用 `c.button` 发送对应的 `pressed: false` 释放事件（修复当前 `release_capture_surface` 硬编码释放 `272` 的问题）。

2. **实现“空间防抖双击稳定器（Spatial Double-Click Stabilizer）”**
   - **为什么必须由 Shell 实现**：Wayland 客户端（GTK/Qt/Xwayland）内部通过判断连续两次 `wl_pointer::button` 按下事件的时间差（通常 $\le 400\text{ms}$）和表面局部坐标差（通常 $|\Delta x|, |\Delta y| \le 4\text{px}$）来识别双击。6DoF 手柄射线在两次扣动扳机之间必然产生数像素到十余像素的抖动，如果不做空间防抖，应用会将双击拆成两次独立的单击。
   - **数据结构与算法**：
     在 `Desktop` 中维护每只手（`source`）最近一次成功释放点击的记录：
     ```rust
     struct ClickRecord {
         target_key: String,       // 窗口 key 或 companion id
         button: u32,              // 272 / 273 / 274
         local_xy: [f32; 2],       // 第一次按下时的锁定表面坐标
         ray_dir: [f32; 3],        // 第一次按下时的射线方向单位向量
         pressed_at: Instant,      // 第一次按下时刻
         count: u8,                // 连击计数 (1 -> 2)
     }
     ```
     - **双击判定条件**：当同一 `source`、同一 `button`（如左键 `272`）在同一窗口 `target_key` 上再次触发 `press`，且满足：
       1. 距离上次 `pressed_at` 时间差 $\Delta t \in [40\text{ms}, 420\text{ms}]$；
       2. 当前射线方向与上次 `ray_dir` 的夹角 $\arccos(\mathbf{d}_1 \cdot \mathbf{d}_2) \le 1.5^\circ$ **或者** 表面局部坐标距离 $\|(x, y) - (x_1, y_1)\|_2 \le 24.0\text{px}$；
     - **坐标钳位（Coordinate Clamping）**：一旦判定为双击（及紧随其后的双击释放），**强制将本次发送给 `pointer_surface` 的局部坐标替换为第一次点击的 `record.local_xy`**！这样 Wayland 应用收到的两次点击坐标误差为严格的 $(0.0, 0.0)$，100% 稳定触发双击打开文件、双击选中文本！
     - **空间外框双击能力**：当在窗口底部的 `Grab` 胶囊上触发双击时，自动切换该窗口的 `curved`（平面/曲面）或将其恢复至视野正前方标准尺寸。

3. **实现“长按静止触发右键（Hold-to-Right-Click）”可选辅助**
   - 对于没有独立 A/X 键的控制器（或单手仅用扳机操作时），在窗口表面 `Capture` 处于 `drag` 阶段时：
     - 若按住时长达到 $650\text{ms}$、尚未触发过长按右键、且从按下至今的最大表面位移 $\max \|(x, y) - \text{start}\|_2 \le 10.0\text{px}$：
     - 自动发送一次左键释放（`button: 272, pressed: false`），紧接着在原位发送一次右键点击对（`button: 273, pressed: true` -> `pressed: false`），并触发一次双脉冲触觉反馈，提示用户右键菜单已弹出。

---

#### 模块三：`xr-controller-tracking` & `xr-monado` —— 打通 A081 真实硬件按键与摇杆到 OpenXR

当前 `xr-controller-tracking/include/xr/controller/a081_controller_controls.h` 已经完整解码了 A081 手柄的底层无线报文，而 `xr-shell/src/home.rs` 证明了 `a081_controller_pose_wire.h` v2 协议在偏移 `60` 处已包含 `controls: u32` 字段（`let controls = u32_at(60);`）。Agent 在衔接底层硬件与 OpenXR Action 时须对齐以下硬件映射：

1. **A081 原厂手柄报文到语义控件的映射关系（参考 `controls-contract.json`）**：
   - `index_finger_raw`（`p[28] & 0x0f`，范围 `0..15`）$\longrightarrow$ **扳机键模拟量 `trigger`**（归一化为 `index_finger_raw as f32 / 15.0`）；
   - `middle_finger_raw`（`p[28] >> 4`，范围 `0..15`）$\longrightarrow$ **抓握键模拟量 `squeeze`**（归一化为 `middle_finger_raw as f32 / 15.0`）；
   - `joystick_raw[2]`（Mode 2 `signed8(p[28]), signed8(p[27])`，按左右手 JNI 规则交换/取反并限制在 `[-80, 80]`）$\longrightarrow$ **摇杆二维向量 `stick`**（归一化为 `[-1.0, 1.0]`）；
   - `buttons_raw`（Mode 1 `p[27] & 0xf0`）：
     - Home / System 键位（`0x40`）由运行时与 `home.rs`（长按 1s 归中）消费；
     - A/B（右手）与 X/Y（左手）、Menu、摇杆下按位分别映射到 OpenXR `/interaction_profiles/oculus/touch_controller` 的 `a/click` (`secondary`)、`b/click` (`back`)、`x/click` (`secondary`)、`y/click` (`back`)、`menu/click` (`menu`) 与 `thumbstick/click` (`thumb_click`)。
2. **安全门限（Safety Gates）保持不变**：
   - 任何按钮、扳机、抓握或摇杆值必须在 `a081_controller_controls_valid_mask(now, 100_000_000)` 校验新鲜度（$< 100\text{ms}$）且 `stock_enable == 1` 时才作为有效输入上报；
   - 超时、断连或会话失焦时，必须立即清零模拟量并释放所有按键，严禁卡键（Stuck Button）或持续自动滚屏。

---

### 3.4 手柄交互验收测试用例（Agent 自测与回归清单）

Agent 完成代码修改后，必须通过以下 6 项端到端验证（含 `xr-shell` 单元测试与 `DebugInput` / 实机验证）：

| 测试编号 | 验证项目 | 输入操作 | 预期输出与验收标准 |
| :--- | :--- | :--- | :--- |
| **TEST-CTRL-01** | **扳机键施密特防抖** | 扳机值按 `0.0 -> 0.52 -> 0.68 -> 0.48 -> 0.30` 序列输入 | `0.52` 不触发按下；`0.68`（$\ge 0.65$）触发一次 `press`；回落到 `0.48`（$> 0.35$）保持 `drag` 不误释放；降至 `0.30`（$\le 0.35$）触发 `release`。 |
| **TEST-CTRL-02** | **抓握键直接搬运窗口** | 射线指向 Pluma 窗口正中央文本区，按住中指抓握键（`squeeze = 0.8`）并移动射线 | 窗口进入 `Target::Move` 球面跟随移动并自动 `user_facing` 转向用户；**Pluma 内部不产生鼠标左键点击或文本选区变化**；双手同时 `squeeze` 同一窗口可缩放大小。 |
| **TEST-CTRL-03** | **鼠标右键呼出菜单** | 射线指向 Pluma 文本区或桌面应用，按下手柄 `A` / `X` 键（或 `DebugInput` `button: 273`） | `xr-wayland` 收到 `SurfaceInput` `pointer_button { button: 273, pressed: true/false }`，应用在射线落点弹出真实 `xdg_popup` 右键上下文菜单。 |
| **TEST-CTRL-04** | **空间防抖双击** | 在 $220\text{ms}$ 内连续两次扣动扳机，第二次按下时射线方向故意偏移 $0.8^\circ$（表面偏移约 $14\text{px}$） | Shell 识别为空间双击，第二次点击的 `pointer_motion` 与 `pointer_button` 坐标被严格钳位至第一次点击坐标 $(x_1, y_1)$，应用成功触发双击选词/打开目录。 |
| **TEST-CTRL-05** | **摇杆悬停滚动与抓握推拉** | 1. 焦点在窗口 A，射线悬停在窗口 B 上推摇杆；<br>2. 按住抓握键抓起窗口 B 后推拉摇杆 Y | 1. 滚动事件送达悬停窗口（或焦点同步后滚动），支持快慢非线性速度；<br>2. 抓握期间推拉摇杆 Y 只改变窗口距离 $d \in [0.6\text{m}, 4.0\text{m}]$，不向窗口内发送滚轮事件。 |
| **TEST-CTRL-06** | **失焦/断连安全释放** | 按住扳机或右键拖动中途触发 `focus_lost`、按 `B/Y` 返回键或控制器位姿失效 | 立即向组件发送 `Cancel`，向 Wayland 发送对应 `button`（`272` 或 `273`）的 `pressed: false`，且 `require_release` 置位防止恢复追踪后自动重按。 |

---

## 第四部分：历史平面化问题对标表（`ISSUE-XR-01` ~ `ISSUE-XR-14`）

| 编号 | 原平面化问题描述 | 0.2.18 审计状态 | 现状说明与剩余门槛 |
| :--- | :--- | :---: | :--- |
| **ISSUE-XR-01** | 应用窗口固定 `rotation = [0,0,0]` 平行于世界 XY 墙 | **已修（有尾项）** | 已实现双目 `user_facing` 与任务栏三岛独立朝向；需修补 `AUDIT-02`（第 4+ 窗口重合）。 |
| **ISSUE-XR-02** | 通知栏置于 `Y = 2.38m` 高空且无俯仰倾角 | **已彻底修复** | 降至 `Y = 1.69m`（仰角约 $+2.9^\circ$）并自动俯倾补偿，协议层拦截仰角 $> +10^\circ$。 |
| **ISSUE-XR-03** | 面板硬编码在绝对 `STAGE` 原点，转头唤出脱离视野 | **已修（有尾项）** | 全部改为 `head_horizon_spawn` 随水平视线生成；需补 `AUDIT-05` 跨组件弹出面板避让。 |
| **ISSUE-XR-04** | 单取左眼 `views[0]` 代替双目中心（Center Eye） | **已彻底修复** | `recenter::center_eye` 实现双目位置均值与四元数最短弧中点，含单元测试覆盖。 |
| **ISSUE-XR-05** | 窗口拖拽退化为固定 2D 平面内的 XY 平移 | **已彻底修复** | 改为 3D 射线距离锚定 + 面向用户旋转迭代 + 摇杆深度推拉（$0.6\sim 4.0\text{m}$）。 |
| **ISSUE-XR-06** | 双手 6DoF 控制器被压缩为单指针互斥模拟鼠标 | **已彻底修复** | 左右手独立 Hover/Capture/Ray，支持双手并发及双手抓握窗口缩放。 |
| **ISSUE-XR-07** | 远场射线热区过小（$< 1^\circ$），违反 $2.2^\circ$ 角尺寸门限 | **部分修复（含 Bug）** | 实现了 $2.2^\circ$ 动态 Hit Slop，但引入了 **`AUDIT-01` 同平面热区吞噬 Bug**，需优先修补。 |
| **ISSUE-XR-08** | 按钮缺乏 Z 轴按压行程，且缺失 OpenXR 触觉反馈 | **已修（有尾项）** | 组件按钮已实现 $+8/+18/+2\text{mm}$ 行程与分级震动；窗口伴随按钮待补行程（`AUDIT-04`）。 |
| **ISSUE-XR-09** | 空间玻璃直接采样天空盒，导致后方窗口隐身穿模 | **已彻底修复** | 移除天空盒假透射，改为线性空间 Premultiplied Alpha 合成，正确保留后方窗口。 |
| **ISSUE-XR-10** | 使用静态 2D 切图与固定 UV 渐变冒充空间高光 | **已彻底修复** | 改为 GPU 逐眼视线半角高光与 Fresnel，具备真实双目立体高光视差。 |
| **ISSUE-XR-11** | 射线为硬边纯色片，且准星未对齐命中表面法线 | **已彻底修复** | 射线加入锥度与首尾渐隐；准星按 `Frame::normal_at` 贴合平面/曲面三维法线。 |
| **ISSUE-XR-12** | 关闭深度写入仅靠中心距离排序，交叉面板易穿插 | **已大幅改善** | 引入 `opaque_pipeline` 写深度解决非透明穿插；半透明部分仍按组中心排序。 |
| **ISSUE-XR-13** | 零 Mipmap 与无各向异性过滤，头部微动高频闪烁 | **已修（有尾项）** | 全景与 Atlas 已开启 Mipmap + Anisotropy + 16px 防渗色；待做焦点窗口 Quad 层提升。 |
| **ISSUE-XR-14** | 逐控件 CPU Cairo 重绘与单眼同步阻塞 Fence | **已修（有尾项）** | Hover/Press 零 Cairo 重绘已验证，双目合并单次提交；动态窗口上传待优化（`AUDIT-06`）。 |

---

## 第五部分：全仓库 XR 空间开发与架构强制约束规范

`openkylin-xr-repos` 下所有涉及空间桌面、UI 组件、窗口服务、跟踪与运行时合成的改动，必须持续遵守以下六条硬性工程约束：

### 约束 1：空间坐标系与人体工学规则（Spatial Ergonomics & Placement）
1. **球面/柱面切向朝向法则（User-Facing Orientation Rule）**
   - **禁止**将任何应用窗口、独立空间面板或弹窗以固定 `rotation = [0, 0, 0]` 放置在偏离正前方的世界坐标上。
   - 所有世界锁定（`world_locked`）的窗口与面板在创建、重置、拖拽移动或缩放中心偏移时，必须根据其中心相对于当前用户**双眼中点（Center Eye Pose）**的向量自动解算朝向（Yaw 正对双眼中点、Pitch 按仰俯角自动补偿、保持 `Roll = 0°`）。
2. **颈部与视野舒适区（FOV & Neck Comfort Zone）硬门限**
   - 常驻与高频交互面板中心必须落在相对眼高俯仰角 **$[-25^\circ, +10^\circ]$** 区间内，**严禁**将通知或工具面板放置在仰角 $> +15^\circ$ 的高空位置。
   - 默认工作区水平展开角控制在 **$[-35^\circ, +35^\circ]$** 内；相邻或同时弹出的空间面板之间必须保留至少 **$2.0^\circ\sim 3.0^\circ$ 的真实视角弧长间隙（Angular Gap）**，禁止多个窗口或弹出面板共面完全重合。
3. **基于当前水平视线（Horizon-Relative Spawn）的动态生成**
   - **禁止**新启动的应用窗口或按需唤出的系统面板直接硬编码生成在绝对 `STAGE` 原点前方。必须以唤出瞬间用户的水平视线方向（Horizon-projected Head Yaw）与当前眼高为基准生成，再锁定到世界空间。

### 约束 2：6DoF 空间操纵与手柄多模态交互规则（6DoF Manipulation & Controller Input）
1. **扳机点选与抓握搬运职责分离（Trigger-to-Click vs. Squeeze-to-Grab）**
   - **食指扳机（Trigger）** 负责精细点选、文本拖拽与 UI 控件激活（映射为鼠标左键 `BTN_LEFT = 272`），必须具备双阈值施密特迟滞（按下 $\ge 0.65$，释放 $\le 0.35$）；
   - **中指抓握键（Squeeze / Grip）** 负责直接抓取与空间搬运（映射为对指向窗口的直接 `Target::Move` 6DoF 球面拖拽与双手捏合缩放），按住抓握键移动窗口时严禁向应用内部误发鼠标点击事件。
2. **完整桌面鼠标语义（右键、中键、空间防抖双击、摇杆滚动）**
   - 必须支持**鼠标右键（`BTN_RIGHT = 273`）**（通过手柄 `A/X` 副按键或扳机静止长按触发）与**鼠标中键（`BTN_MIDDLE = 274`）**（摇杆下按）；
   - 必须在 Shell 层实现**空间防抖双击稳定器**（在 $\le 420\text{ms}$ 且射线抖动 $\le 1.5^\circ$ 或 $\le 24\text{px}$ 时将第二次点击坐标钳位至首次点击坐标）；
   - **摇杆（Thumbstick）** 在非抓握态下采用二次方精细曲线驱动悬停目标的 2D 滚动或组件翻页，在抓握态下驱动窗口深度推拉（$0.6\sim 4.0\text{m}$）。
3. **双指针独立并发与同平面热区优先级（Dual-Hand Concurrent Ray & Hit Priority）**
   - 左右手控制器必须拥有各自独立的 `PointerId`、Hover 状态、射线渲染反馈与 `Capture` 槽位；
   - 所有远场射线可点击控件在基准距离下的**最小有效命中角尺寸不得小于 $2.2^\circ$**；在同一平面（`|\Delta t| < 1e-4`）下，**真实几何内部命中（Direct Hit）必须绝对优先于扩展热区命中（Slop Hit）**。
4. **三维按压行程与分级触觉反馈（3D Z-Travel & Haptics）**
   - 所有可交互控件（含窗口伴随控制栏按钮）必须具备默认抬升、`Hover` 浮起、`Press` 下压的三维深度行程，并在悬停边界、左键点击、右键呼出与窗口抓握时触发不同强度的 OpenXR `xrApplyHapticFeedback` 触觉脉冲。

### 约束 3：空间光学、材质与真实深度线索（Spatial Optics & Depth Cues）
1. **严禁“假折射穿模”与 2D 静态贴皮高光**
   - **禁止**在空间面板 Shader 中跨过背后已渲染的应用窗口去采样天空盒伪造透明折射；空间玻璃必须使用线性空间 Premultiplied Alpha 半透明合成 + 基于每眼独立视线向量 `toward_eye` 的 **Fresnel 与双目视差镜面高光**。
2. **三维体积射线与法线对齐准星（Normal-Aligned Reticle）**
   - 射线必须具备首尾渐隐与末端收窄过渡；射线命中准星（Reticle）必须严格对齐命中表面的局部三维法线（Surface Normal Aligned）并沿法线微移防 Z-fighting。

### 约束 4：XR 排版度量、清晰度与反走样规则（Angular Typography & Clarity）
1. **统一 `dmm`（Distance-Independent Millimeter）度量**
   - 以 $1\text{m}$ 距离下 $1\text{mm}$ 物理尺寸（$1\text{dmm} \approx 0.0573^\circ$）为基准：正文字号 $\ge 20\text{dmm}$，次要文字 $\ge 16\text{dmm}$。
2. **强制 Mipmap、各向异性过滤与高频图层直通**
   - 所有 3D 空间中的 UI Atlas 与窗口纹理必须生成 Mipmap 链并启用线性级间过滤与各向异性过滤；焦点平面窗口在 Runtime 支持时优先提升为 OpenXR `CompositionLayerQuad` 原生合成层。

### 约束 5：渲染管线与组件协议边界（Pipeline & Protocol Architecture）
1. **几何材质全 GPU SDF 化，Cairo 仅负责字形与图标光栅**
   - 圆角矩形 SDF、玻璃/按键材质、侧壁厚度、边框高光及 Hover/Press 颜色过渡全部由 Vulkan Shader 实时计算；Hover/Press 状态切换不得触发 CPU Cairo 重绘或 Atlas 纹理重传。
2. **帧内异步流水线**
   - 双目绘制合并提交，逐步消除动态纹理上传与合成器中的同步 `wait_for_fences / vkQueueWaitIdle` 串行阻塞。

### 约束 6：姿态预测、时间扭曲（ATW）与运动到光子防抖约束（Pose Prediction & Anti-Judder）
1. **明确预测责任，真实报告运动有效性**
   - 驱动应按已验证的样本时间和运动状态处理目标时刻 `at`；不能在多个层级重复预测。仅在传感器值、单位、轴向和参考系已确认时置位相应速度有效标志。缺少证据时保留未预测状态并明确标注限制，严禁伪造位置、速度、时间戳或放宽追踪有效性门限。
2. **验证合成器重投影与扫描补偿链路**
   - 分别记录应用采样、合成器晚期采样和实际呈现证据。按目标运行时检查扫描补偿支持与标定，不能把缺少预测直接等同于 ATW 禁用，也不能以进程存活或请求 72 Hz 作为稳定满帧证明。


## 手柄交互落地记录（Shell 0.2.26）

本批实现扳机迟滞、抓握键独立搬运、右键/中键、空间双击稳定器与悬停滚动。源码提交 `9f1b7a77b0c83d74464b406a6b37bc8acd9abe33`，原生软件包 SHA-256 为 `883df7cbcd9513969ab94852c40e6ecc7713580c595607b764430aa4e05e2527`。原有追踪、OEM grip/aim/model 坐标与单一预测链路保持不变。

| 项目 | 已实现行为 | 自动验证边界 |
| --- | --- | --- |
| 扳机与抓握迟滞 | 扳机 0.65/0.35；抓握 0.60/0.35；失效、失焦后须先释放再激活 | 状态机回归及模拟 OpenXR Action；实体模拟量待验 |
| 独立抓握 | 窗口表面直接搬运，不发左键；双手缩放；取消恢复原位置、旋转与尺寸 | 原生 Pluma 选区保持、双手缩放及恢复通过 |
| 右键与中键 | A/X→273、摇杆按下→274；焦点等待与取消保留按钮身份 | 真实右键菜单、菜单关闭后键盘回读；GTK 客户端收到 274 按下及取消释放 |
| 双击稳定器 | 同手、同窗口、同按钮，40–420 ms 内且 ≤1.5° 或 ≤24 px，锁定第二次按下/释放坐标；拖动、取消或目标变化不串联 | 实机清空旧剪贴板影响后，14 px 偏移、约169 ms 间隔选词读回通过；客户端自身双击设置仍生效 |
| 悬停滚动 | 0.18 径向死区与非线性曲线；优先命中窗口，遮挡 UI 不穿透；已有翻页控件 0.65 阈值/280 ms 冷却；抓握 Y 调深度 | 原生跨窗口滚动并读回移动后的文档行；组件翻页边界通过回归测试 |
| 弹出菜单焦点 | 菜单销毁后按原焦点代次重申父窗口，避免丢失 Escape 或其他按键释放 | 实机发现的问题已修复，真实键盘选择/复制通过 |

验证结果：11 项本地回归、15 项 QEMU 检查（5269 帧，Vulkan 验证错误/警告均 0）、22 项原生检查及 8 项退出/重进检查通过；正常清理后 FastRPC 对象/字节均 0，CDSP FD 为空。原生交互使用受保护的开发射线调用生产路由，未伪造实体追踪姿态。三次交接采样头部位置与朝向均 tracked；实体双手、触觉、佩戴抖动与光学验收继续待验，不晋升镜像基线。

当前 GTK 配置关闭主选区粘贴；本批未改用户设置，因此中键通过客户端按键协议验证，不声称 Pluma 中键粘贴已启用。A081 已部署驱动的实测归一化满量程为扳机 9、抓握 10，本批不以通用 0–15 字段范围覆盖已有归一化。长按右键辅助、伴随栏双击快捷操作、抓握态快捷复位/附加轴等完整矩阵余项仍需后续实现；窗口选择器尚无分页控件，不标记其摇杆翻页已支持。

构建采用锁定依赖的原生 release 构建，软件包复用已安装包元数据；未把完整 debhelper 构建依赖闭包标为通过。保留 0.2.24 回退包及中间失败记录，最终设备停留默认投影、模型/射线全开、已有渲染优化开启的正常 XR 会话。


## 三岛弧面与冰晶玻璃候选

采用中心620px、侧岛各340px、弧长间隙24px（0.0384m）及水平轨道半径1.72m。侧岛中心由半宽与间隙计算为 `x=±0.77718075m, z=-1.53440219m`，三岛高度均为0.86m，Shell统一读取俯仰并给胶囊及其子几何保留1.72m曲率。渲染和射线相交使用同一Frame；固定/跟随模式与捕获冻结继续共用一个锚点。1.72m是水平半径，低于双目0.74m时实际视距约1.87m。

Dock专用玻璃使用 `(0.88,0.93,0.99,0.55)`，加入垂直微光泽、内层柔光、顶/左0.65与下/右0.18的双层边缘。图标托盘静态Alpha0.14、悬停0.32及10mm抬升；运行条核心14×4px，附小范围冰蓝光晕。左岛日期/未知天气改为紧凑双行，右岛移除三条分隔线并保留真实电池读数的微胶囊。玻璃阴影扩大至40px，向后及向下各偏移12mm，以深蓝软暗晕托底。透明合成继续使用线性预乘Alpha与每眼高光。

源码版本为Shell0.2.27、Kit0.2.6、Dock0.2.8和SystemUI0.2.13。14项回归及8项QEMU场景检查通过，覆盖圆弧半径/间隙、共同高度/俯仰、曲面像素射线往返、跟随与冻结、图标悬停、曲面点击及状态布局；渲染验证层错误/警告均0。保留同视点前后截图、源码锁和二进制摘要。

这轮验证范围为离线构建与隔离QEMU。实机窗口仍由手柄测试占用，本候选尚未部署到头显；ARM64构建、深色实机场景及佩戴效果继续待验。实际天气/电量/应用状态来自原组件，不使用概念稿示例读数。


### 三岛材质修正：去除灰蓝底膜

前述Alpha0.55、双层高光及40px蓝色暗晕方案经用户视觉反馈后撤销。原因包括：阴影SDF在面板内部仍有0.22的Alpha，透过玻璃形成灰蓝底膜；图标托盘的独立大阴影进一步相互叠加。

当前候选采用中性 `(0.95,0.965,0.985,0.68)` Dock玻璃；移除垂直色调渐变、内发光和第二层描边，保留0.28/0.08的单层细边及每眼微弱高光。阴影为中性、12px范围、最高Alpha0.06，面板内部衰减为零；图标和电池托盘不再独立投影，运行条保留14×4px核心并取消外围光晕。三岛圆弧、尺寸、间距及交互逻辑沿用几何候选。

14项回归与10项隔离QEMU场景检查通过，另保留修正前后双眼截图和渲染验证统计。此为未发布候选的材质修订；实机部署与佩戴观感继续待验。数值通过不替代用户的视觉判断。


### 清理版原生部署记录

清理版已完成ARM64原生锁定依赖构建并部署到已安装的openKylin：Shell0.2.27、Dock0.2.8、SystemUI0.2.13和Kit0.2.6。21项原生检查通过，覆盖组件与实际运行二进制摘要、透视保留、三岛曲率/尺寸/高度、单层细边、12px轻阴影、无托盘投影、悬停及曲面点击。原有透视、追踪驱动与会话配置摘要保持一致。

实际透视启动目录原先保留独立旧Shell，首次仅安装系统包造成新旧组件不一致且ready失败，已自动回退并保留失败记录。随后将实际启动的Shell接到已管理的软件包二进制，核对进程exe摘要与新包一致；旧透视Shell与四个旧组件包均保存，可配套回退。

正常退出及清理通过，FastRPC对象/字节均0、CDSP FD为空，已重进并保留新版XR。重进初始三次头姿有效但未tracked，该次门槛失败原样保留；后续短采样观察到恢复，最后连续三次位置与朝向均tracked。此结果不代表长期跟踪稳定或佩戴观感通过，实体交互与视觉确认继续待验。内核和启动镜像未修改，不据此晋升新的内核镜像基线。

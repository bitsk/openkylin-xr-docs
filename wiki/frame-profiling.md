---
layout: default
title: MIX 逐帧性能采样与稳定性分析
---

# MIX 逐帧性能采样与稳定性分析

Shell 0.2.31 与 Debug Console 0.1.5 提供逐帧观测和离线分析入口。它们是诊断候选；包构建、模拟验证、MIX 查询可用性与佩戴改善分别验收。采样不改变头姿、手柄有效性、预测、队列策略或 GPU 同步语义。

## 当前采样覆盖

| 链路 | 数据 | 边界 |
| --- | --- | --- |
| 帧循环 | 帧号、wait 序号、预测显示时刻/周期、8 个主阶段、未渲染原因 | CPU 墙钟包含睡眠、阻塞和抢占；帧间隔不等于物理扫描 |
| 帧外工作 | 事件处理、快照请求/导出、记录/截图文件/日志 | 显式保留观测开销；截图帧默认从统计排除 |
| CPU | 线程 CPU 时间首末值、运行核首末值 | 核首末相同不能证明期间没有迁移；等待归因需调度 trace |
| 6DoF | 请求的 XrTime、头姿、valid/tracked flags、无有效朝向时未渲染记录 | 应用消费的预测姿态，不是原始传感器时间 |
| 手柄 | xrSyncActions、分别读取左右动作/位姿的 CPU 时间、grip/aim 与原始 flags | valid 与 tracked 分开；没有位姿的帧不证明模型运动性能 |
| 手部 | 关节定位、关节数组数量、动态网格准备、模型顶点数 | 有关节数组不等于全部关节有效，模型可拒绝无效网格 |
| UI | 动画/栅格化、几何上传、射线反馈、图集区域/源字节/全量替换、面板实例数 | 源字节不包含 GPU mipmap/blit 的全部内存流量 |
| 录制 | 彩透准备/姿态、图集、双眼、额外图层、命令结束、提交、fence、查询/读回/回收 | 双眼内部另有背景、控制器模型、手模型、面板、射线 CPU 录制段 |
| GPU | 可选 timestamp 查询：上传、左眼、右眼、额外图层 | 设备时钟的阶段间隔，可能含流水重叠；不是独占着色器成本，不强行映射 CPU 时轴 |
| 合成器 | OpenVR 历史帧号、应用/合成 GPU 与 CPU、Submit/Present/等待、重投影/误呈现/重复帧、阶段偏移 | 原始运行时时钟不假定与 MONOTONIC 一致；驱动零值不自动解释为零成本 |
| 系统 | 每线程 stat/schedstat/wchan/status、CPU/GPU 频率、温控状态、压力、缺页、内存、IO、中断累计数 | 默认 100ms 系统快照；保留读取跨度，不称为每帧原子采样 |
| 调度 | 私有 tracefs instance 中 sched/IRQ、频率、DRM、dma_fence、可用 MSM 事件 | 不改全局 trace；保留未提供事件及 ring 覆盖统计 |
| 调用栈 | 可选 perf cpu-clock 99Hz DWARF 栈 | 抽样热点，可用 perf report 展开调用图；不代表每个调用都被计时 |

尚未补成生产者逐帧事件的部分：曝光到 DSP/SLAM 内部阶段、每个相机帧与控制器融合结果的统一身份、应用帧到合成器帧的明确关联、GPU 校准时钟、真实扫描/光子与佩戴抖动。不能用主线程等待倒推出这些阶段的耗时。

## Shell 导出

CPU 观测保留最近 4096 个提交帧，未渲染记录另有 4096 条上限；`observed` 与 retained 数量同时保存。`summary.json` 中包含 `timeline.frames`；`timeline.json` 是快照格式。`wait_number` 跳跃与 `skipped` 可以区分未渲染帧，旧格式缺字段时显示未知。

在已有 Shell summary 目录中创建空文件 `timeline-request`，最多等待 30 次 wait 循环触发 JSON 快照。不会创建截图请求、读回交换链或重启服务。导出在帧前执行，其成本记录在下一帧的帧外工作中；渲染停止期间不会及时响应。最新截图帧可能尚未写完帧外计时，此时 `post_frame_complete=false`。

GPU 查询需在获准启动的诊断 Shell 环境中显式设置 `XR_SHELL_GPU_TIMING=1`。默认关闭。支持检查、查询失败/未就绪与正常结果分别记录；复用原有完成 fence，不加等待、不拆分双眼提交。GPU profiling 开/关应各采一次，检查 timestamp 本身的观察效应。仅通过编译不能证明目标 Turnip 支持或时间值可信。

## 系统采集

先核实当前硬件窗口、真实进程、会话目录和目标内核；硬件被占用时只做离线分析。采集器不操作相机，不启动或停止 XR，不修改调度/频率/温控策略。

```sh
xr-perf-capture --output /tmp/xr-perf-sample \
  --pid "$shell_pid" --pid "$compositor_pid" --pid "$tracking_pid" \
  --seconds 15 --interval-ms 100 --shell-dir "$scene_dir"
```

可用 `--json-source NAME=PATH` 订阅至多 8 个已有生产者 JSON 文件，保留来源原始时间戳和本次读取夹区间，不把读取时间当作曝光/推理完成时间；单份上限 1MiB。`--drm-fdinfo` 可记录所选进程暴露的 DRM GPU/内存累计计数，驱动不提供时为空。

按需添加 `--trace`（通常需 root）与 `--perf`。时长上限 120 秒，采样间隔 20–2000ms；trace 每 CPU 4MiB，因此高事件率下仍会覆盖，必须检查 `capture.json` 的 buffer stats。tracefs instances 不可用时只记录缺失，不回退改全局 tracing。SIGINT/SIGTERM 停止采集并清理自己建立的 instance/perf 子进程。

产物：`capture.json`（boot、PID/路径、时钟夹区间、能力/错误/覆盖计数）、`system.jsonl`、可选 `system.ftrace`、`perf.data`、结束系统采样后请求的 `timeline.json`。本地证据含进程与设备信息，公开前按工程隐私规则处理。`perf report -i perf.data --stdio -g` 可展开 CPU 调用栈；具体函数符号需要对应构建的调试符号。

已有只读 `vr-timing SECONDS COMPOSITOR_PID SESSION_FILE` 行为保持不变。加第四参数 `--history` 可每 100ms 读取最多 128 条历史，按帧号/时间去重，保留缺失索引、运行时回退 epoch，并暂缓最新一条以减少读取未完成数据。输出前缀 `MIX_FRAME_TIMING`；保留查询 MONOTONIC 夹区间和独立 runtime frame seconds，不宣称完成跨时钟校准。它仍需原有已运行 SteamVR 的用户与环境，不能作为启动器。

## 可视化

调试台“性能与时序”进入 `performance.html`。导入 `timeline.json` 或 Shell `summary.json`，点击总览/拖动选帧、定位最慢帧，查看嵌套 CPU 时序和独立 GPU 耗时。支持旧数据，但缺失项留空。截图默认不参与统计；72Hz 只是可调整的参考线，超预算计数不叫真实掉帧。

同次采样可追加 `system.jsonl`、`capture.json` 和 `system.ftrace`。同一 boot 与采样窗口校验后关联系统记录；调度区间仅由匹配的 switch 配对构成，包含 IRQ 时间，不能作为函数级 CPU 时间。未知尾部不补齐，覆盖/配对异常必须保留。

导出 Chrome JSON trace 可由 [Perfetto 支持的外部 trace 格式](https://perfetto.dev/docs/getting-started/other-formats)直接导入。CPU 段保持 MONOTONIC，GPU 耗时作为 fence 后观测的 counter，避免把未校准 GPU 时钟伪装为同步 GPU 轨道。页面条带是嵌套时序，不是调用栈火焰图。

## 分析与优化顺序

1. 用同一场景的静止、转头/平移、左右手柄单独移动、双手抓握/缩放区间，先确认 valid/tracked 与输入来源；保持分辨率、视频预览、温度及功能组合一致。
2. 找异常帧的关键路径：CPU wall 与线程 CPU 同时升高时查看栅格化/几何/模型；wall 上升而线程 CPU 不升高时查看调度等待、IPC 和 fence；GPU 查询与 fence 同时升高时再查图层/纹理/分辨率成本。各段分位数不相加。
3. 先分离控制器位姿间断、丢 tracked、预测/时间戳跳变与渲染长帧。位姿无效时不放宽门槛、不固定假坐标来“消抖”。
4. 现有每帧提交后等 fence 是明确的串行边界，但去掉它需要重新设计交换链释放、资源复用与多帧生命周期；采样补齐前不盲改。CPU 模型录制很短也不证明 GPU 模型无成本。
5. 对持续重栅格、全图集上传、逐帧重复几何/网格建立进行实测归因，再考虑缓存或脏区域。已有 CPU 准备优化曾改善中位数却未消除长尾，平均 FPS 不足以验收。
6. 系统策略用同场景 A/B/A；保留原 CPU/GPU 与温控基线。已有 cpuset/uclamp/EAS 组合没有证明提速，不按“更高级策略”直接替换默认值。
7. 分别报告提交节奏、P95/P99、跟踪有效性、合成器呈现、设备清理与佩戴稳定性。只在实际成立的范围内晋升。

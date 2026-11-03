---
layout: default
title: XR Shell 组件接口设计
---

# XR Shell 组件接口 · r1 草案

本文定义待实现的边界，不是已有 wire 实现或稳定 SDK。正式类型、协议版本和客户端代码将由 `xr-shell/crates/protocols` 唯一维护；其他组件消费其发布版本，不复制第二份定义。

## 三条通道

| 通道 | 用途 | 传输与所有权 |
|---|---|---|
| 会话控制 | 启停、运行状态、原生 XR 应用切换 | 既有 session-manager 的版本化控制接口；不在此另建第二套生命周期服务 |
| 组件场景 | Dock/UI 节点、布局意图、交互事件、窗口状态订阅 | Shell 的 Unix `SOCK_SEQPACKET`；以 component/session generation 限定对象生命周期 |
| 窗口图像 | wl_shm 或 DMA-BUF 资源及提交/释放 | Unix 消息附带 SCM_RIGHTS；像素不经 JSON、D-Bus 或网页 |

Socket 位于当前用户的 `$XDG_RUNTIME_DIR/xr-desktop/`，目录 0700；基于 peer credentials 确认连接用户。首版仅接收由会话配置显式启动的同用户组件；这不是任意第三方应用的系统级权限边界。组件通道不能启动任意命令或写任意文件。

首版元数据可以使用有界 JSON，每个数据报一个消息；资源 FD 单独通过 ancillary data 传递。实现前固定最大消息尺寸、节点数、FD 数和断连清理行为，握手包含协议 major/minor、组件角色与 capabilities。不兼容 major 明确拒绝；可选能力通过 capability 协商。

## 场景模型

客户端提交 retained scene，而不是 Vulkan 命令、GPU 指针或每帧 draw call。首版节点限于 Group、Card、Text、Icon、Button、Slider 和 SurfaceReference。节点引用由 Shell 分配的资源 ID；材质从 Kit 的已知样式选择，不能直接注入 shader。

事务例子（示意，字段仍待代码实现）：

```json
{
  "type": "SceneTransaction",
  "generation": 3,
  "revision": 18,
  "component": "dock",
  "commands": [
    {"op": "set_anchor", "node": "root", "policy": "dock_lazy_follow"},
    {"op": "upsert", "id": "launcher", "parent": "root", "kind": "button",
     "style": "dock.application", "action": "open-launcher"}
  ]
}
```

Root pose、世界变换、raycast、pointer capture 与焦点由 Shell 仲裁。Dock 发送布局参数，Shell 在 predictedDisplayTime 下更新跟随姿态。重连时分配新的 generation；旧 ID、旧事件和旧事务不能影响新场景。事务按 revision 原子应用，避免半更新布局接收输入。

Shell 回送 `HoverChanged`、`Press`、`Drag`、`Release`、`Cancel`、`FocusChanged`。事件携带 source、pointer/interaction ID、时间与局部坐标；失去会话焦点或客户端断连必须 Cancel 并释放捕获。滑块等连续输入可合并中间值，按下/释放/取消不可丢弃。

窗口权威在 Shell 与窗口服务，不在 Dock。Dock 只缓存 `WindowSnapshot` 和随后事件。`ActivateWindow`、`CloseWindow`、`LaunchApp` 请求经 Shell/session 规则处理后返回结果；应用拒绝关闭、缺包或启动失败都保留可见状态。

## 窗口与缓冲

窗口标识包含服务 generation 和 surface ID；应用 app_id、标题、父子关系、像素尺寸、buffer scale、输入区域、alpha 和 configure serial 分别维护。三维尺寸和位置由 Shell 持有，不将像素尺寸与米混用。

缓冲事件至少包含：

```text
resource_id, surface_id, frame_id, commit_serial
width, height, DRM fourcc, modifier, plane_count
每 plane: fd_index, offset_bytes, stride_bytes
alpha_mode, colour_space, damage_rects
sync_mode, acquire_sync_reference
```

`plane_count` 不等于 FD 数；多个 plane 可以引用同一个被传递的 FD。FD 传输后接收方拥有自己的引用，断连、导入失败和 retire 均需释放；发送方不得把跨进程的整数 fd 值当作句柄。

同步与释放顺序：

1. 应用 commit，窗口服务保存 buffer 引用。
2. 窗口服务向 Shell 发送 resource/commit 和约定的 acquire 同步。
3. Shell 在 acquire 完成后采样；无法导入时回报错误并使用已协商的拷贝路径。
4. Shell 的 GPU 最后一次读取完成后返回 Release；导入的持久资源可保留到独立 Retire。
5. 窗口服务据该缓冲协议发送 release/同步信号，应用才可以安全复用对应内容。

基线 `wl_shm` 用明确定义的复制完成语义。DMA-BUF 先实现并验收一种显式同步方式，再协商其他方式；`sync_file` FD 与 DRM syncobj timeline 是不同表示，不互相伪装。以真实格式/modifier 和 GPU 同步能力为准，不能只按“有 DMA-BUF FD”判断可用。

Shell 将应用图像采样/拷贝到它自己的 OpenXR swapchain。直接复用应用纹理为 OpenXR layer 并不天然可行，需遵守 runtime swapchain 的创建和使用约束。当前开发 VM 的 OPAQUE_FD runtime/client 图像共享是另一种协议与资源所有权关系。

## Kit、样式与布局单位

`xr-spatial-kit` 提供纯 Rust 控件模型、布局、交互状态和渲染抽象；`xr-spatial-style-ukui` 提供可替换的 token 数据。Dock 与 system-ui 通过客户端库构建节点，Shell 渲染同版本支持的 primitive。

Rust 边界草图：

```rust
pub struct ComponentId(pub u64);
pub struct SurfaceId { pub generation: u64, pub id: u64 }
pub struct LogicalSize { pub width: f32, pub height: f32 }
pub struct WorldSize { pub width_m: f32, pub height_m: f32 }

pub trait SpatialComponent {
    fn on_snapshot(&mut self, state: &DesktopSnapshot);
    fn on_event(&mut self, event: &InteractionEvent) -> Vec<Intent>;
    fn scene(&self) -> SceneTransaction;
}
```

此草图表达所有权与调用方向，不是可链接的现有 crate。协议序列化显式定义整数字段、版本和单位，不依赖 Rust struct 的内存布局。

初始视觉 token 提案：

| 项 | 初始值/行为 | 验收注意 |
|---|---|---|
| 品牌 | `#3790FA` 麒麟蓝，青蓝/淡紫辅助色 | 与目标系统的实际主题资产匹配，保持可替换 |
| 卡片 | 轻透灰蓝、连续圆角、薄外框 | 根据真实背景亮度检查文字对比；不强迫存量应用整体透明 |
| 正文 | 参考角高约 0.8° | 按头显清晰度与字体实际测量，不作为普适生理标准 |
| 点击热区 | 参考角宽高约 2.5° | 1 m 处约 44 mm；热区可以大于可见按钮 |
| Dock | 前下方约 18°，初始距约 0.9 m | 参数可调；延迟跟随有 dead zone、限速和关闭选项 |
| Hover / Press | 颜色、微位移与可选触觉 | reduced-motion 下禁用多余位移，依然保持状态可辨 |
| 场景 | 世界朝向稳定，背景忽略平移 | 不以无限背景提供近场位置/地面线索 |

环境贴图是完整远景，UI 卡片不把背景高光作为文字亮度。窗口外框与内容层独立：使用底部握把、控制胶囊和角部缩放锚点；平面内容与弯曲内容按应用能力选择。未来的碰撞、空间锚点和多用户能力不能从当前全景资产推导出来。

## 兼容与观测

日志分别报告 XR runtime、Vulkan device、窗口输入格式、buffer transport、WSI transport、composition layer 和环境资产版本。帧号、提交时间、GPU 完成和观察端呈现分别计量；CPU copy、DMA-BUF import 与最终零拷贝是不同状态。

第一轮实现只需要证明一个完整链路：Rust Shell 背景 → Dock 真正点击 → 启动一个 Wayland 应用 → 输入/resize → 关闭并释放。协议测试围绕实际出现的错序、缓冲复用和焦点故障编写，不用纯模型测试代替真实窗口与会话证据。

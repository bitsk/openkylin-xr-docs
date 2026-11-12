# 仓库与软件包索引

当前 20 个组件仓库统一发布在 bitsk 的 GitHub。完整职责和软件包表见 [统一维护入口](../workspace.md)，机器可读目录只有 [xr-image-tools/repositories.json](https://github.com/bitsk/xr-image-tools/blob/main/repositories.json) 一份。

本地唯一应用源码工作区是 `openkylin-xr-repos/`。旧 build 路径是指向同目录的兼容链接；历史 qiyu 用户态仓库只用于归档与比对。已有候选工作树改动独立保留，固定提交构建不会混入它们。

内核实现继续在 [bitsk/qiyu-kernel](https://github.com/bitsk/qiyu-kernel) 维护；xr-kernel 负责锁定和打包该来源。Mesa 使用固定上游与适配补丁，Monado 与 FastRPC 保留各自上游历史。资产消费固定 [bitsk/qiyu-vendor](https://github.com/bitsk/qiyu-vendor) 的 Release 与逐文件摘要。

`xr-image-tools/components.lock.json` 固定源码提交及公开接口摘要；`build-recipes.json` 规定 SDK 和消费组件的构建顺序。固件、相机标定、原权重模型及外部运行时的来源分别固定，不把私有资产混入公开源码。

旧功能迁移及已知限制见 [组件迁移清单](component-migration.md)。当前包构建结果以实际 build-manifest.json 和原始构建日志为准；源码发布、安装包生成、目标系统安装和硬件功能验收分别记录。手检测器1042仍是独立的待解决运行故障。

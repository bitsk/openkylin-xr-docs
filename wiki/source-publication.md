# 源码与交付入口

[统一工作区](../workspace.md) 列出已发布的 20 个 bitsk GitHub 组件仓库。代码、接口和打包规则由各自唯一仓库维护；完整选择由 xr-image-tools 的组件锁与构建配方决定。

内核实现属于 [bitsk/qiyu-kernel](https://github.com/bitsk/qiyu-kernel)，xr-kernel 只保存固定来源和打包入口。Mesa 通过固定上游历史及适配补丁准备；Monado 与 FastRPC 保留其真实上游历史。旧 Gitee 仓库和历史发布保留，后续镜像同步不成为第二套维护来源。

型号公共 OEM 数据继续消费 [bitsk/qiyu-vendor](https://github.com/bitsk/qiyu-vendor) 的 a081-vendor-r1 Release。集成仓库 vendor.lock.json 固定资产 ID、压缩包和全部文件 SHA-256；xr-vendor-assets 从校验缓存构建型号软件包。公共包不包含 per-device 标定和手模型权重。

当前 GitHub 源码不等于已经发布新的二进制 Release。构建产物由新的外部输出目录保存，其 manifest 记录精确提交和包摘要。构建、安装、服务运行和佩戴验收各自独立；历史测量不替代迁移后组合的验收。

---
layout: default
title: 构建与发布
---

# 构建与发布

代码仓库提供各组件的实际构建入口，openkylin-xr-docs 说明如何组合它们。源码版本首先固定，再构建目标产物，最后由 rootfs manifest 记录所选系统、型号和组件摘要。

## 内核

qiyu-kernel 保留真实 Linux 主线历史，驱动、binding/DTS 和配置按子系统提交；xr-kernel 负责固定来源与包构建。A081 的构建与 rebase 入口在 [devices/a081/MAINTAINING.md](https://github.com/bitsk/qiyu-kernel/blob/main/devices/a081/MAINTAINING.md)；内核专用文档与源码一起维护。

## 用户态组件

openKylin 的组件源码只从独立仓库的准确 Git 提交导出。
`xr-image-tools/components.lock.json` 固定提交与公共头摘要，`build-recipes.json`
决定构建顺序；正式产物通过 `xr-build-components` 构建，放在源码工作区之外。
工作树编辑、之前的构建目录和已安装 prefix 都不是项目代码输入。

```sh
python3 xr-image-tools/tools/components.py build \
  --profile vr-hands --source-cache /workspace/components \
  --environment /private/toolchain.json --output /build/new-components
```

工具链配置可声明外部 SDK、发行版依赖和交叉编译器，不能替换本次构建的组件接口。
组件专用编译选项只能扩展当前 include 路径。公共协议由唯一所属组件提供，
接口变更同步更新生产者、消费者、组件锁和部署。

## 实际编译输入

源码提交、包版本和包内头文件正确，并不证明编译器使用了这些头文件。
构建器检查实际编译依赖文件中的 `include/xr/` 路径和摘要：项目接口必须来自
本轮隔离源码或 sysroot。运行时消费的头姿接口必须出现在依赖记录中，并匹配
组件锁。部署导出再次检查这些记录，缺少证据的产物不能进入组合包。

构建后的 `build-manifest.json` 保存 `compiler_project_headers`。
不能用复制公共头文件、匹配包版本或核对安装后摘要替代这项检查。
真实进程的路径和摘要仍须与部署包核对；离线构建、实机遥测与佩戴确认分别记录。

Monado、Mesa 和内核的上游来源继续固定到真实历史；适配补丁只在各自仓库维护。
原厂库和模型通过独立资产锁导入，私有标定不进入公开源码。历史修改由 Git 保存，
项目不维护并行协议、兼容实现或回退包。

## 系统、型号与功能组合

```sh
python3 xr-image-tools/tools/rootfs.py plan \
  --system debian --device a081 --profile debug
python3 xr-image-tools/tools/rootfs.py plan \
  --system steamos --device a081 --profile vr
```

prepare 调用对应系统适配器准备基础归档；assemble 接收基础归档和显式产物清单，输出基础归档副本、设备 overlay 与 manifest。Debian foreign-stage 还需要原生第二阶段。新增系统、型号或 profile 使用相同装配入口。

## 发布记录

组合 manifest 固定组件提交、接口摘要和产物 SHA-256。工程里程碑与实现过程写入 Wiki/Blog；具体设备记录的测量范围留在来源附录，已发布标签、Release 与回退材料保持可追溯。

原厂依赖现在由既有 qiyu-vendor Release 提供，xr-image-tools 的 devices/<model>/vendor.lock.json 锁定下载内容；tools/vendor.py 先验证整包，再逐文件校验并生成安装映射。缓存准备好后 verify / assemble 均可离线执行。每台设备的 persist 校准单独注入；完整系统和设备验收仍是独立门槛。

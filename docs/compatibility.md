# 版本兼容矩阵

每个 Release 必须更新本文件，并在 Release 说明中引用。表中的版本是**最低要求**；
高于表中版本能否工作，以实机验证为准。

## 1. 当前发布矩阵

> 当前发布：**SDK 1.0.1（2026-11-23）**。本版发布 host / orin-nx / agx-thor；
> RV1126B 目标源码已支持，P3 板端验收前不发布公开包。

| SDK | UVDP | 目标 | 系统 | 内核 | VPI | TensorRT | CUDA | RKNN / MPP | 模型 | ZSViewer |
|---|---|---|---|---|---|---|---|---|---|---|
| 1.0.x | 2 | host | Ubuntu 22.04+ | — | — | — | — | — | 桩/回放 | 2.1.4 |
| 1.0.x | 2 | orin-nx | JetPack 6.2 / L4T 36.x | 5.15 | 3.x | 10.x | 12.x | — | 不随公开包 | 2.1.4 |
| 1.0.x | 2 | agx-thor | JetPack 7.x / L4T 39.x | 6.8 | 4.x | 10.x | 13.x | — | 不随公开包 | 2.1.4 |
| 1.0.x | 2 | rv1126b | Buildroot Linux | 5.10 | — | — | — | RKNN 2.x / MPP 1.x | 待交付（内部验证） | 2.1.4 |

说明：

- `host` 包只用于协议、客户端、回放和单元测试，不包含相机/深度硬件后端。
- Orin NX 与 AGX Thor 的 VPI 主版本、TensorRT engine、CUDA 架构不同，资产不可互换。
- 本版 Orin NX runtime 使用 `orin-nx-hw` 生产构建（VPI3 + NVBUF + NVJPEG），
  `build.vpi=true`；CPU-only 的 `orin-nx` preset 只用于回归，不用于生产包。
- AGX Thor VPI 路径使用 VPI 自有图像，禁止导入 NVBUFFER（否则约 6 分钟后 VPI
  会累积失败）。
- 公开包不随附模型/engine（再分发许可确认前）；engine 必须在目标板生成，详见
  `docs/packages.md`。
- RV1126B 的 RKNN 模型与 RKNPU 驱动版本绑定；升级系统后需重新确认模型。

## 2. 协议兼容

| SDK | UVDP 协议 | 客户端要求 |
|---|---|---|
| 1.0.x | 2 | 必须支持 `HELLO proto_min/proto_max`、64 字节头、CRC32C |
| 更早的 UVDP v1 | 1 | 不兼容；服务端返回明确版本错误 |

规则：

- 同一 major 内不改变头部布局、消息语义和帧格式。
- 新增消息类型/控制方法必须向后兼容：旧客户端忽略未知消息，服务端对未知方法返回
  `Unsupported`，不崩溃。
- 删除/重命名字段、改变像素格式编号、改变 CRC 算法 → 升 major。
- 协议版本与 SDK 版本独立；SDK minor 可以仍为 proto 2。

## 3. 模型与 engine

| 目标 | 模型 | 输入 | 精度 | 生成环境 | 分发 |
|---|---|---|---|---|---|
| orin-nx | LAS2-M | 640×544（以配置为准） | FP16 | JetPack 6.2 / TRT 10.x | 公开包不随附；目标板生成 |
| agx-thor | LAS2-M | 640×544（以配置为准） | FP16 | JetPack 7.x / TRT 10.x | 公开包不随附；Thor 上生成 |
| rv1126b | RKNN 双目模型 | 以配置为准 | — | RKNN Toolkit 2.x | 待交付人员提供 |

规则：

- engine 绑定 GPU 架构、TensorRT 版本和构建时的插件/工作区设置；换板或升级 JetPack
  必须重新生成，不能复制。
- 模型升级单独发 `models` 包或在 release notes 中标注；不得静默替换 runtime 包内模型。
- 单独发布 models 包时，`MODEL_INFO` 至少包含：模型名、版本、输入尺寸、输出单位
  （视差/深度）、精度、生成命令、生成环境、sha256。
- 视差与深度单位必须明确：`F32_DISPARITY` 是像素视差，`F32_METERS`/`U16_MILLIMETERS`
  是物理距离；旧 SDK 中“米”字段实际存视差的问题不得在新包中重现。

## 4. ABI 与构建

- 公开头文件不因平台宏改变类布局；平台状态全部 pimpl。
- dev 包提供静态库 `libzscamera.a`；客户程序静态链接，升级 SDK 后需重新编译。
- dev 包内头文件、库、CMake config 必须来自同一次构建，禁止混用不同版本。
- 客户交叉编译时使用与板端 runtime 相同目标（`orin-nx` / `agx-thor` / `rv1126b`）的 dev 包。
- 系统库（VPI/TensorRT/CUDA/RKNN/MPP/RGA）由客户环境提供，版本见上表。
- 交付包为 Release 构建，并对源码根做 `-ffile-prefix-map`，不携带构建机路径。

## 5. 工具兼容

- ZSViewer 通过 UVDP v2 连接；连接后以 `HELLO_ACK`/`META` 的能力声明为准，不按板型写死。
- 本版最低 ZSViewer 版本：**2.1.4**。
- 控制方法新增时，旧工具应忽略未知能力；删除/改变语义时升 SDK major。

## 6. 兼容性验证记录

| 日期 | SDK | 目标 | 系统 | 工具 | 结果 | 备注 |
|---|---|---|---|---|---|---|
| 2026-11-23 | 1.0.1 | host | Ubuntu 22.04 x86_64 | — | ctest 32/32 | 无硬件依赖 |
| 2026-11-23 | 1.0.1 | orin-nx | JetPack 6.2 / L4T 36.x | — | 构建 + ctest 32/32；`build.vpi=true` | 本版未做相机 smoke |
| 2026-11-23 | 1.0.1 | agx-thor | JetPack 7.x / L4T 39.x | ZSViewer 2.1.4 | 双目 30 fps、深度 10 fps、`u16_mm_zlib`，probe 15 帧无丢帧 | SOF 30.000 fps；新模组双路固定差 3.39 ms（AE 曝光差，待模组侧同步） |
| — | 1.0.1 | rv1126b | Buildroot | — | 未发布 | P3 板端验收后再发布 |

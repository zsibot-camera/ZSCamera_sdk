# Changelog

本文件遵循 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/) 格式，
版本号遵循 [Semantic Versioning](https://semver.org/lang/zh-CN/)。

## [1.0.1] - 2026-11-23

首个统一 ZSCamera SDK 公开发布。SDK 覆盖 RV1126B / Orin NX / AGX Thor，
本版发布 `host`、`orin-nx`、`agx-thor` 的 dev/runtime/tools 包。

### Added

- 统一 C++17 SDK：公开头文件、静态库 `libzscamera.a`、`find_package(zscamera)`
  CMake 配置，以及 `zscamera_stream`、`zscamera_probe`、`zscamera_calib`、
  `zscamera_ota`、`zscamera_demo` 工具。
- UVDP v2：64 字节小端头、CRC32C、TCP 9600，支持 left/right/depth/cloud/meta
  与 CONTROL/PING；保留 v1 兼容输出。
- Orin NX 生产包使用 `orin-nx-hw` 构建（VPI3 + NVBUF + NVJPEG；
  VPI 自有图像，不导入 NVBUFFER）；AGX Thor 包使用 VPI4 + TensorRT。
- 标定加载、A/B Flash 记录、`calib.get/set`、OTA 分片上传与状态查询。
- 三平台出包脚本统一：包名按 `<target>` 命名，`orin-nx-hw` 产物发布为
  `orin-nx`；runtime 只携带目标对应的配置。

### Changed

- 交付包改为 Release 构建，并对源码根做 `-ffile-prefix-map`；包内不再包含
  构建机路径、内部 README、发布检查单或 deb 打包文档。
- 公开包不随附模型/engine（再分发许可确认前）；`models/README.md` 说明
  获取方式和 engine 与目标板的绑定关系。
- 交付文档按实际包内容修正：dev 包为静态库，runtime 无自研 `.so`，
  不承诺内置 `examples/` 或启动脚本。

### Known Issues

- RV1126B 目标源码已支持，但 P3 板端验收完成前不发布公开 dev/runtime 包。
- TensorRT engine 必须在目标板生成；Orin 与 Thor 的 engine 不可互换。
- 双目相位取决于模组 FSYNC 与两路 AE：若两路曝光不同，SOF 会出现与曝光差
  对应的固定相位差（2026-11-23 新模组实测 3.39 ms，帧率仍锁定 30.000 fps）。
  需要紧同步时，先在模组侧做 AE 同步或固定相同曝光/增益后复测。

### Compatibility

- SDK 1.0.1，UVDP proto 2；最低工具版本 ZSViewer 2.1.4。
- Orin NX：JetPack 6.2 / L4T 36.x，VPI 3.x，TensorRT 10.x。
- AGX Thor：JetPack 7.x / L4T 39.x，VPI 4.x，TensorRT 10.x。
- 详细矩阵见 `docs/compatibility.md`。

# examples

客户集成示例随每个 `zscamera-sdk-<ver>-...-dev.tar.gz` 发布，内容包括：

- `CMakeLists.txt`：`find_package(zscamera CONFIG REQUIRED)` 的最小工程。
- `client_demo.cpp`：连接 UVDP v2、订阅 left/right/depth、打印帧信息。
- `runtime_demo.cpp`：板端创建采集/深度流水线的最小示例（按目标提供）。

本目录只保留示例索引和版本说明，不存放 SDK 实现源码。示例源码在内部源码仓库
`apps/`/`examples/` 维护，出包时复制到 dev 包。

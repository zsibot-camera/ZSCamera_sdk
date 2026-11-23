# examples

本目录只保留示例索引和版本说明，不存放 SDK 实现源码；客户示例不随公开包发布。

集成方式以包内 `docs/` 与公开头文件为准，最小工程如下：

```cmake
find_package(zscamera CONFIG REQUIRED)
target_link_libraries(your_app PRIVATE zscamera::zscamera)
```

```cpp
#include <zscamera/zscamera.hpp>

zscamera::Runtime runtime;
// 按包内 docs/ 初始化设备、订阅流、处理 FrameSet
```

dev 包提供公开头文件、静态库 `libzscamera.a` 和 CMake config，客户程序静态链接，
无需 `LD_LIBRARY_PATH`。协议与 API 说明见 `docs/protocol-v2.md`、`docs/architecture.md`。

示例实现由内部源码仓库维护，不进入交付仓库，也不复制进 dev 包。

# 发布包说明

本文件定义交付仓库的包命名、内容、目录结构和资产清单（manifest）格式。
维护者出包和客户选择包都以本文件为准。

## 1. 命名规则

```text
zscamera-<kind>-<version>-<os>-<arch>-<target>[-<variant>].tar.gz
```

| 字段 | 取值 | 说明 |
|---|---|---|
| `kind` | `sdk` / `runtime` / `tools` / `models` | 包类型 |
| `version` | `0.1.0`、`0.1.1-rc.1` | 不带 `v` 前缀；tag 才带 `v` |
| `os` | `linux` | 当前只发布 Linux |
| `arch` | `x86_64` / `aarch64` | host 为 x86_64，板端为 aarch64 |
| `target` | `host` / `orin-nx` / `agx-thor` / `rv1126b` | 一个包只服务一个目标 |
| `variant` | `dev` | 仅 sdk 包使用；runtime/tools 不需要 |

示例：

```text
zscamera-sdk-0.1.0-linux-aarch64-agx-thor-dev.tar.gz
zscamera-runtime-0.1.0-linux-aarch64-agx-thor.tar.gz
zscamera-tools-0.1.0-linux-aarch64-agx-thor.tar.gz
zscamera-models-0.1.0-agx-thor.tar.gz
```

禁止的命名：

- `zscamera-all.tar.gz`、`zscamera-latest.tar.gz`（无法判断目标和版本）
- `zscamera-orin-thor.tar.gz`（Orin/Thor 不可共用）
- 版本号和 tag 不一致的包

## 2. 包内容

### 2.1 dev 包

```text
include/zscamera/           公开 API 头文件
lib/libzscamera.so.<ver>    预编译共享库
lib/libzscamera.so          soname 软链
lib/cmake/zscamera/         zscameraConfig.cmake 等
examples/                   CMake + 最小客户端示例
docs/                       API、协议、标定、安装说明
LICENSE  NOTICE  VERSION  SHA256SUMS
```

约束：

- 头文件只能来自 `include/zscamera/`；不得带内部头文件。
- `libzscamera.so` 的依赖只有系统库和目标平台 SDK 库；不打包 VPI/TensorRT/CUDA/RKNN/MPP/RGA。
- 不包含 `src/`、测试、构建脚本、`.debug` 文件、CMake 构建缓存。
- `examples/` 只能是客户示例，不得复制内部测试/工具源码。

### 2.2 runtime 包

```text
bin/zscamera_stream
lib/                        运行所需 .so（自研）
config/<target>.conf        默认 profile
models/                     该目标模型/engine（许可允许时）
scripts/start.sh stop.sh selfcheck.sh
docs/quickstart.md
LICENSE  NOTICE  VERSION  SHA256SUMS
```

约束：

- `zscamera_stream` 只链接包内库和系统库，不引用内部构建路径。
- 配置中的相对路径按配置文件目录解析；默认安装目录 `/opt/zscamera`。
- 模型与目标板严格绑定；`models/` 中附 `MODEL_INFO`（模型名、版本、输入尺寸、精度、生成环境）。
- 不能分发的模型不放入包，`models/README` 说明获取方式。

### 2.3 tools 包

```text
bin/zscamera_probe
bin/zscamera_calib
bin/zscamera_demo
lib/                        工具依赖的自研库
docs/tools.md
LICENSE  NOTICE  VERSION  SHA256SUMS
```

### 2.4 models 包（可选）

只在模型单独升级、且客户不需要整包升级时发布。必须包含：

```text
models/<model>.rknn | <model>.engine | <model>.onnx
MODEL_INFO                  版本、输入尺寸、精度、目标、生成命令、校验和
LICENSE  NOTICE
```

禁止把 Orin 的 engine 放进 Thor 包，或把 FP32/FP16 版本混在同一目录而不标注。

## 3. 资产清单 manifest

每个 Release 必须有 `manifests/v<version>.json`，并更新 `manifests/latest.json`。
`latest.json` 结构：

```json
{
  "schema": 1,
  "latest": "0.1.0",
  "releases": [
    {
      "version": "0.1.0",
      "tag": "v0.1.0",
      "date": "2026-11-10",
      "protocol": 2,
      "status": "stable",
      "targets": ["host", "orin-nx", "agx-thor", "rv1126b"],
      "assets": [
        {
          "name": "zscamera-sdk-0.1.0-linux-aarch64-agx-thor-dev.tar.gz",
          "kind": "sdk",
          "target": "agx-thor",
          "os": "linux",
          "arch": "aarch64",
          "size": 12345678,
          "sha256": "…64 hex…",
          "min_system": "L4T 39.x / JetPack 7.x",
          "model_version": "las2-m-1.0"
        }
      ]
    }
  ]
}
```

字段规则：

- `schema`：manifest 结构版本，当前为 1。
- `latest`：最新稳定版本；预发布不要写进这里。
- `status`：`stable` / `rc` / `deprecated`。
- `assets[].sha256` 必须与 Release 资产一致；发布后不可修改。
- 目标版本、模型版本、最低系统版本以资产为单位记录，不使用全局默认值。

`manifests/v<version>.json` 只需要 `version` 及其 `assets`，结构与上面对应。

## 4. 校验和

每个 Release 附带一个 `SHA256SUMS`，覆盖该版本所有资产：

```text
<sha256>  zscamera-sdk-0.1.0-linux-aarch64-agx-thor-dev.tar.gz
<sha256>  zscamera-runtime-0.1.0-linux-aarch64-agx-thor.tar.gz
...
```

生成方式：

```bash
cd dist/0.1.0
sha256sum zscamera-* > SHA256SUMS
sha256sum -c SHA256SUMS
```

客户校验：

```bash
sha256sum -c SHA256SUMS
```

## 5. 本地出包目录

`dist/` 只用于维护者本地暂存，已加入 `.gitignore`，**不提交 git**：

```text
dist/
└── 0.1.0/
    ├── zscamera-sdk-...
    ├── zscamera-runtime-...
    ├── zscamera-tools-...
    ├── SHA256SUMS
    └── release-notes.md
```

上传 Release 时整目录作为资产上传；`dist/` 内文件不进入仓库历史。

## 6. 包体检查

出包后至少检查：

```bash
tar -tzf zscamera-sdk-<ver>-...-dev.tar.gz | grep -E '(^|/)src/|\.debug$|CMakeCache'
```

命中任一模式说明包不干净，禁止发布。再检查：

- `ldd lib/libzscamera.so` 不出现内部构建路径。
- `nm -D --defined-only lib/libzscamera.so` 不含内部符号前缀（如 `rill::`）。
- `readelf -p .comment`、`strings` 不泄露用户名/主机名/内部路径。
- dev 包头文件能独立编译 `examples/`。

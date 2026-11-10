# ZSCamera SDK（交付仓库）

智身科技（GENISOM AI）视觉传感器团队的统一相机与双目深度 SDK **二进制交付仓库**，
支持 RV1126B、Jetson Orin NX、Jetson AGX Thor。

> **本仓库不包含 SDK 实现源码。**
> 这里只发布：公开 C++ 头文件、预编译库、板端程序、模型、配置、协议/使用文档和校验和。
> 源码、测试、构建脚本和内部文档在内部研发仓库维护，不对外释放。

当前状态：**v0.1.0 开发中，尚未发布二进制包**。下表是发布矩阵，实际资产以下载页
（GitHub Releases）为准。

---

## 1. 支持平台

| 目标代号 | 硬件 | 系统 / SDK | 采集 | 深度 | 状态 |
|---|---|---|---|---|---|
| `host` | x86_64 Linux | Ubuntu 22.04+ | 回放/合成 | 桩 | 开发中 |
| `orin-nx` | Jetson Orin NX | JetPack 6.2 / L4T 36.x | V4L2 + VPI3 | TensorRT LAS2-M | 开发中 |
| `agx-thor` | Jetson AGX Thor | JetPack 7.x / L4T 39.x | V4L2 + VPI4 | TensorRT LAS2-M | 开发中 |
| `rv1126b` | RV1126B | Buildroot Linux | V4L2 NV12 + RGA | RKNN | 开发中 |

> Orin NX 与 AGX Thor 的 VPI ABI、TensorRT engine 均不兼容，**不能共用一个包**。
> 每个目标必须使用对应资产；版本不匹配时 SDK 会明确报错，不要强行混用。

---

## 2. 下载哪个包

从 **GitHub Releases** 下载，不要从 git 历史或 `dist/` 目录找二进制。

| 目标 | SDK 开发包（集成） | 运行时包（板端） | 工具包（诊断） |
|---|---|---|---|
| host | `zscamera-sdk-<ver>-linux-x86_64-host-dev.tar.gz` | — | `zscamera-tools-<ver>-linux-x86_64-host.tar.gz` |
| orin-nx | `zscamera-sdk-<ver>-linux-aarch64-orin-nx-dev.tar.gz` | `zscamera-runtime-<ver>-linux-aarch64-orin-nx.tar.gz` | `zscamera-tools-<ver>-linux-aarch64-orin-nx.tar.gz` |
| agx-thor | `zscamera-sdk-<ver>-linux-aarch64-agx-thor-dev.tar.gz` | `zscamera-runtime-<ver>-linux-aarch64-agx-thor.tar.gz` | `zscamera-tools-<ver>-linux-aarch64-agx-thor.tar.gz` |
| rv1126b | `zscamera-sdk-<ver>-linux-aarch64-rv1126b-dev.tar.gz` | `zscamera-runtime-<ver>-linux-aarch64-rv1126b.tar.gz` | `zscamera-tools-<ver>-linux-aarch64-rv1126b.tar.gz` |

选择规则：

1. **只在板端跑现成程序** → 只下 runtime 包。
2. **客户自己的程序要链接 SDK** → 下 dev 包；板端同时需要 runtime 包提供的库和配置。
3. **现场排查/写标定** → 下 tools 包。
4. **只升级模型** → 下对应目标的 models 包（如单独发布），不要跨板使用 engine。

包的详细内容、命名规则和目录结构见 [`docs/packages.md`](docs/packages.md)。

---

## 3. 包里有什么

### 3.1 SDK 开发包（dev）

```text
zscamera-sdk-<ver>-.../
├── include/zscamera/*.hpp     # 公开 API 头文件（不是实现源码）
├── lib/libzscamera.so*        # 预编译库
├── lib/cmake/zscamera/        # find_package(zscamera) 配置
├── examples/                  # 最小客户程序示例
├── docs/                      # API/协议/标定/移植说明
├── LICENSE  NOTICE  VERSION  SHA256SUMS
```

不含 `src/`、内部头文件、测试、构建脚本、调试符号表。

### 3.2 运行时包（runtime）

```text
zscamera-runtime-<ver>-.../
├── bin/zscamera_stream        # 板端常驻程序（UVDP v2 服务端）
├── lib/                       # 运行所需自研库
├── config/<target>.conf       # 该目标的默认配置
├── models/                    # 该目标的 RKNN / TensorRT engine（如许可允许）
├── scripts/                   # 启动/停止/自检
└── docs/quickstart.md
```

VPI、TensorRT、CUDA、RKNN、MPP、RGA 等系统库**不随包提供**，按兼容矩阵安装。

### 3.3 工具包（tools）

`zscamera_probe`（连通性/版本/流信息）、`zscamera_calib`（Flash/JSON 标定读写）、
`zscamera_demo`（离线回放）。工具与 runtime 使用同一 UVDP v2 协议。

---

## 4. 快速开始

### 4.1 校验下载

每个 Release 都带 `SHA256SUMS` 和 `manifests/v<ver>.json`：

```bash
sha256sum -c SHA256SUMS
```

校验失败不要解压使用；重新下载或联系交付人员。

### 4.2 板端部署（runtime）

```bash
sudo mkdir -p /opt/zscamera
sudo tar -xzf zscamera-runtime-<ver>-linux-aarch64-<target>.tar.gz -C /opt/zscamera --strip-components=1
sudo /opt/zscamera/bin/zscamera_stream --config /opt/zscamera/config/<target>.conf
```

默认 UVDP v2 端口为 TCP `9600`。启动后用工具确认：

```bash
zscamera_probe <board-ip>:9600
```

### 4.3 客户程序集成（dev）

```cmake
find_package(zscamera CONFIG REQUIRED)
target_link_libraries(your_app PRIVATE zscamera::zscamera)
```

```cpp
#include <zscamera/zscamera.hpp>

zscamera::Runtime runtime;
// 按 docs/api 与 examples/ 初始化设备、订阅流、处理 FrameSet
```

编译时使用 dev 包内头文件和库；运行时确保 `libzscamera.so` 在 `LD_LIBRARY_PATH`
或系统库路径中。完整 API 和协议见包内 `docs/`。

### 4.4 常见问题

| 现象 | 先检查 |
|---|---|
| 连不上 9600 | 板端进程是否启动、防火墙、IP/网段、`zscamera_probe` 输出 |
| 连上但无图 | `HELLO_ACK` 的 `capabilities`、`META` 的流状态、相机设备节点 |
| 深度全 0/全无效 | 标定是否加载、左右目是否配对、模型/engine 是否匹配目标板 |
| 加载 engine 失败 | TensorRT/GPU 架构是否与包匹配；换板必须重新生成 engine |
| 时间戳异常 | 板端授时/NTP、V4L2 时间戳域；客户端不要自行混用 monotonic 与 UTC |

---

## 5. 版本与兼容

- SDK 版本遵循 SemVer：`MAJOR.MINOR.PATCH`。
- 协议版本独立：当前 `UVDP proto = 2`；协议破坏性变更会升 SDK major。
- 每个目标单独构建；模型、engine、VPI 版本与目标板绑定。
- 完整矩阵见 [`docs/compatibility.md`](docs/compatibility.md)，每个 Release 也会在说明中
  给出“最低系统版本 / 最低工具版本 / 模型版本”。

| SDK | UVDP | 目标 | 系统 | VPI | TensorRT | RKNN | 模型 | ZSViewer |
|---|---|---|---|---|---|---|---|---|
| 0.1.x | 2 | orin-nx | L4T 36.x | 3.x | 10.x | — | LAS2-M | 待发布时确认 |
| 0.1.x | 2 | agx-thor | L4T 39.x | 4.x | 10.x | — | LAS2-M | 待发布时确认 |
| 0.1.x | 2 | rv1126b | Buildroot | — | — | 2.x | 双目模型 | 待发布时确认 |

> 表中版本是目标值，最终以对应 Release 的 `docs/compatibility.md` 为准。

---

## 6. 发布哪些包（维护者）

每次 Release 按“目标 × 包类型”出资产，**一个包只服务一个目标**：

| 包类型 | 是否必需 | 内容 | 说明 |
|---|---|---|---|
| `sdk-...-dev` | 是 | 头文件、`.so`、CMake config、examples | 客户集成用；不含实现源码 |
| `runtime-...` | 是（板端目标） | `zscamera_stream`、库、配置、模型 | 板端部署 |
| `tools-...` | 是 | probe/calib/demo | 现场诊断 |
| `models-...` | 可选 | RKNN / ONNX / engine | 模型单独升级时发布 |
| `debug-symbols` | 否 | `.debug` 符号 | **仅内部**，不进交付仓库 |

不发布的东西：源码包、测试程序、构建脚本、内部文档、客户标定数据、设备密码、
未确认许可的模型。

发布节奏：

- 功能版本按里程碑发布（minor）；缺陷修复按需发 patch。
- 安全或严重现场问题：评估后尽快发 patch，不等待下一个 minor。
- 预发布用 `vX.Y.Z-rc.N`，只给试点客户，不和正式版共用“latest”。
- 每季度至少复核一次兼容矩阵、旧资产可下载性和第三方许可。

---

## 7. 发布流程（维护者）

### 7.1 发布前

1. 内部源码仓库打 tag `vX.Y.Z`，确认主机单测、三平台交叉编译通过。
2. 在对应构建机/板端生成 `dist/<version>/` 资产。
3. 板上跑发布检查单（见 [`docs/maintenance.md`](docs/maintenance.md)）：
   双目出流、深度有效、META/标定、控制命令、断线重连、30 分钟 soak。
4. 生成 `SHA256SUMS`，写 `manifests/v<version>.json`。

### 7.2 发布

```bash
# 1. 更新交付仓库文档与 manifest
#    README.md 版本表、CHANGELOG.md、manifests/latest.json、manifests/v<ver>.json
git add README.md CHANGELOG.md manifests docs
git commit -m "release: v<ver>"
git push origin main

# 2. 打 tag 并创建 Release（资产只增不改）
git tag v<ver>
git push origin v<ver>
gh release create v<ver> dist/<ver>/* \
  --title "ZSCamera SDK v<ver>" \
  --notes-file docs/release-notes-v<ver>.md
```

### 7.3 发布后

1. 从 Release 重新下载一个 dev 包和一个 runtime 包，校验 `sha256sum`。
2. 解压 dev 包，用 `examples/` 做一次 `find_package` + 编译 + 运行。
3. 板上用 runtime 包启动，`zscamera_probe` 确认流和标定状态。
4. 更新 `manifests/latest.json` 指向新版本；确认 README 下载表无误。
5. 通知 ZSViewer/交付/客户支持对应版本与最低工具版本。

**已发布 tag 的资产不要替换。** 需要修复时升 patch 版本重新发布，旧版本保留可下载。

---

## 8. 仓库怎么管理

| 项目 | 规则 |
|---|---|
| 分支 | 只保留 `main`；临时修复分支在内部源码仓库，不推到交付仓库 |
| tag | `vX.Y.Z` / `vX.Y.Z-rc.N`；tag 指向发布提交，创建后不移动 |
| 二进制 | 只放 GitHub Releases；`dist/` 已 gitignore，不提交 git 历史 |
| 文档 | 协议/API/兼容矩阵随版本更新；历史版本文档可从对应 tag 查看 |
| PR | 不接受客户源码 PR；只接受文档纠错（不涉及内部实现） |
| Issue | 用于问题反馈和下载/安装问题；安全漏洞走私密渠道，不在公开 issue 贴细节 |
| 权限 | org owner 管理仓库；Release 权限仅维护者；客户只给读取/下载权限 |
| 备份 | 每个 Release 资产至少保留两份（GitHub + 内部构建机 `dist/`） |
| 保留 | 最近 3 个 minor 保留；更早版本保留资产但不再修复 |
| 支持 | 当前 major 全量支持；上一 major 只修安全/严重问题；更早只提供下载 |

**源码边界**：交付仓库里出现 `src/`、内部头文件、构建脚本、调试符号表、客户数据，
一律视为发布事故，立即回滚该提交并重新出包。

仓库目录：

```text
README.md  CHANGELOG.md  VERSION  LICENSE  NOTICE
docs/
  packages.md          包内容、命名、manifest 说明
  maintenance.md       维护手册与发布检查单
  compatibility.md     版本兼容矩阵
  protocol-v2.md       UVDP v2 协议（发布时同步）
manifests/
  latest.json          当前最新版本与资产索引
  vX.Y.Z.json          每个版本的资产、sha256、目标、依赖版本
examples/              客户集成示例（不含实现源码）
dist/                  本地出包暂存，gitignore
```

---

## 9. 许可与支持

本 SDK 为智身科技专有软件，仅授权客户在约定范围内使用；不得复制、再分发、
反向工程或向第三方披露。第三方组件和模型许可见 [`NOTICE`](NOTICE) 及包内
`docs/licensing.md`。

- 交付/商务：联系智身科技视觉传感器团队
- 问题反馈：GitHub Issues（不要贴密码、token、客户数据）
- 安全漏洞：走私密渠道，附复现步骤和影响版本

---

## 10. 相关仓库

- 内部研发仓库：`/home/zsxc/zzq_work/4.code/ZSCamera_sdk`（不对外）
- 上位机工具：`zsibot-camera/ZSViewer`（按 UVDP v2 对应升级）
- 平台驱动：`zsibot-camera/nvidia-jetson-camera-drivers`、
  `zsibot-camera/d-robotics-camera-drivers`

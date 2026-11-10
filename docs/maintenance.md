# 维护手册与发布检查单

面向交付仓库维护者。客户只需要看根目录 `README.md` 和 `docs/packages.md`。

## 1. 仓库职责与边界

| 仓库 | 职责 | 可见性 |
|---|---|---|
| `/home/zsxc/zzq_work/4.code/ZSCamera_sdk` | 源码、测试、构建、内部文档、出包脚本 | 内部 |
| `/home/zsxc/zzq_work/6.github/ZSCamera_sdk` | 文档、manifest、校验和、Release 二进制 | 按需（当前私有） |

硬规则：

- 交付仓库不出现实现源码、内部头文件、测试、构建脚本、调试符号表、客户数据。
- 二进制只进 GitHub Releases，不进 git 历史。
- 交付仓库只保留 `main` 和发布 tag；临时修复分支留在内部源码仓库。
- 发布权限只给维护者；客户只给读取/下载权限。

## 2. 版本、节奏与支持

### 2.1 版本号

- `MAJOR.MINOR.PATCH`，tag 为 `vX.Y.Z`。
- `MAJOR`：UVDP 协议破坏、公开 API 不兼容、ABI 破坏。
- `MINOR`：新增流/控制方法/平台/模型，向后兼容。
- `PATCH`：缺陷修复、文档、依赖安全更新，不改变协议和 ABI。
- 预发布：`vX.Y.Z-rc.N`，只给试点客户；不更新 `latest`。

### 2.2 节奏

| 类型 | 时机 |
|---|---|
| 功能版本（minor） | 按里程碑，P2/P3 板端验收通过后 |
| 缺陷修复（patch） | 影响客户使用时，评估后尽快 |
| 安全/严重现场问题 | 最高优先级，单独 patch，不等功能版本 |
| 兼容性复核 | 每季度一次 |

### 2.3 支持窗口

- 当前 major：全量支持。
- 上一个 major：只修安全/严重问题。
- 更早版本：保留 Release 资产供下载，不再修复。
- 弃用：提前一个 minor 在 `CHANGELOG.md` 和 Release 说明中公告，给出替代版本。

## 3. 发布前检查单

### 3.1 源码与构建

- [ ] 内部源码仓库 `git status` 干净，tag 已打 `vX.Y.Z`。
- [ ] 主机 `ctest` 全绿；协议一致性、标定 golden、Flash 回读测试通过。
- [ ] `orin-nx`、`agx-thor`、`rv1126b` 交叉编译通过，无新增 warning 阻断。
- [ ] 版本号、`VERSION`、`CHANGELOG.md`、协议文档一致。
- [ ] 公开头文件只依赖标准库/系统库，不含平台宏改变类布局。

### 3.2 出包

- [ ] 用固定构建机/工具链出包，记录工具链、JetPack/L4T、VPI、TensorRT、RKNN、MPP 版本。
- [ ] 包名符合 [`packages.md`](packages.md) 的命名规则。
- [ ] `tar -tzf` 检查无 `src/`、`.debug`、`CMakeCache`、内部路径。
- [ ] `ldd`/`nm`/`strings` 检查无内部构建路径和内部符号。
- [ ] dev 包 `examples/` 可独立编译。
- [ ] engine/模型与目标板匹配，`MODEL_INFO` 已写。
- [ ] 生成 `SHA256SUMS` 并自检通过。

### 3.3 板上验收

每个目标分别执行，记录结果和版本：

- [ ] `zscamera_stream` 启动，`zscamera_probe <ip>:9600` 能拿到 `HELLO_ACK` 和 `META`。
- [ ] 左/右目出流稳定，帧率符合配置，时间戳单调且为 UTC ns。
- [ ] 深度有效：近/中/远距离有合理值，无整帧 0/全无效。
- [ ] 标定：Flash 有参数时能加载；无参数时不崩溃、不阻塞出流，`META` 标记未标定。
- [ ] 控制：`device.status`、`stream.start/stop`、`calib.get` 正常。
- [ ] 断线重连：客户端断开/重连后能恢复，服务端不泄漏 fd/线程。
- [ ] 30 分钟 soak：RSS、fd、VPI/GPU 错误、温度无异常。
- [ ] 目标板特有项：
  - Orin NX：VPI3 路径、无 FSYNC 时左右目配对。
  - AGX Thor：VPI4 自有图像路径，确认无 NVBUFFER 导入；GMSL 无停帧。
  - RV1126B：RGA 旋转/校正、MPP 编码、RKNN 模型、Flash 标定。

### 3.4 文档与 manifest

- [ ] `README.md` 下载表、版本矩阵、最低系统/工具版本更新。
- [ ] `CHANGELOG.md` 按 Keep a Changelog 写清新增/变更/修复/已知问题。
- [ ] `docs/compatibility.md` 更新 SDK/协议/系统/VPI/TRT/RKNN/模型/工具版本。
- [ ] `manifests/v<ver>.json` 资产名、大小、sha256、目标、依赖版本完整。
- [ ] `manifests/latest.json` 在发布验证后指向新稳定版。
- [ ] 包内 `docs/protocol-v2.md`、`docs/api` 与源码版本一致。

## 4. 发布操作

```bash
cd /home/zsxc/zzq_work/6.github/ZSCamera_sdk

# 文档与 manifest
git add README.md CHANGELOG.md docs manifests
git commit -m "release: vX.Y.Z"
git push origin main

# tag + Release（资产只增不改）
git tag vX.Y.Z
git push origin vX.Y.Z
gh release create vX.Y.Z /path/to/dist/X.Y.Z/* \
  --title "ZSCamera SDK vX.Y.Z" \
  --notes-file docs/release-notes-vX.Y.Z.md
```

发布后回读：

```bash
tmp=$(mktemp -d)
gh release download vX.Y.Z --dir "$tmp"
(cd "$tmp" && sha256sum -c SHA256SUMS)
tar -xzf "$tmp"/zscamera-sdk-...-dev.tar.gz -C "$tmp"
# 用 examples 编译一次；runtime 包上板启动一次
rm -rf "$tmp"
```

## 5. 热修流程

1. 在内部源码仓库从问题 tag 拉 `fix/vX.Y.Z`。
2. 只改必要代码；补回归测试；确认影响范围。
3. 升 `PATCH`，重跑发布检查单（至少受影响目标和主机测试）。
4. 只重发受影响目标的资产，但 Release 版本整体升 patch。
5. `CHANGELOG` 写清问题、影响版本、规避方式、修复版本。
6. 不删除旧 Release，不覆盖旧 tag 资产。

## 6. 客户问题处理

| 类型 | 处理 |
|---|---|
| 下载/校验失败 | 确认资产名和 sha256；必要时重新上传缺失资产（不替换已有文件） |
| 安装/链接失败 | 收集 `ldd`、CMake 日志、系统版本；对照兼容矩阵 |
| 连不上/无图 | 要 `zscamera_probe` 输出、`META`、板端日志；先确认设备端出流，再查工具 |
| 深度异常 | 标定状态、左右目配对、模型/engine 与目标板是否匹配 |
| 崩溃/卡死 | 要 core/日志、复现步骤、运行时长；按安全流程处理 |
| 安全漏洞 | 走私密渠道；不在公开 issue 贴利用细节；评估后发 patch 和公告 |

客户 issue 里禁止出现密码、token、完整标定文件、客户数据；发现后提醒并删除。

## 7. 安全与合规

- SSH/设备密码只走 `sshpass -f` 的 600 权限临时文件，用完立即删除；禁止写入 argv、
  脚本、日志、issue。
- 不在仓库提交设备密码、客户 IP、客户标定、内部主机名。
- 交付包内 `NOTICE` 列明第三方组件和模型来源；不能确认再分发许可的模型不发布。
- 每季度检查依赖 CVE 和许可变化；LAS2/ONNX/RKNN 模型、OpenCV、zlib、prometheus-cpp
  等随包组件要保留许可文本。
- 公开仓库前必须完成 LICENSE/NOTICE 法务确认；未确认时保持私有或仅邀请客户。

## 8. 保留、备份与审计

- GitHub Release 保留最近 3 个 minor；更早资产保留但标记不再支持。
- 每个版本在内部构建机 `dist/<version>/` 保留一份完整资产和 `SHA256SUMS`。
- 每季度审计：
  - [ ] 旧 Release 资产可下载。
  - [ ] `SHA256SUMS` 与 `manifests` 一致。
  - [ ] README 链接无 404。
  - [ ] 兼容矩阵与在售板卡/系统版本一致。
  - [ ] Issue 中无敏感信息。
  - [ ] 许可证/NOTICE 仍准确。

## 9. 常见发布事故

| 事故 | 处理 |
|---|---|
| 把 Orin 包发给 Thor 客户 | 立即停止使用；补发正确包；在 README 强调目标不可互换 |
| 包内混入 `src/` 或 `.debug` | 撤回资产，重新出包，升 patch；检查打包脚本过滤规则 |
| tag 资产被覆盖 | 恢复原资产；无法恢复时升 patch 重发，旧 tag 标记异常 |
| engine 与板端 TensorRT 不匹配 | 重新生成对应 engine；在 Release 说明写清最低 TRT/GPU |
| 客户按旧协议连接新固件 | 明确提示协议 v2 不兼容，要求升级工具/固件；不要偷偷兼容 v1 |
| Flash 标定写坏 | 使用另一 slot 回滚；检查 sequence/CRC 和回读校验 |

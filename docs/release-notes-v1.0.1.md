# ZSCamera SDK v1.0.1

发布日期：2026-11-23。首个统一 ZSCamera SDK 公开发布，覆盖 host / Orin NX / AGX Thor。
RV1126B 源码已支持，P3 板端验收完成前暂不发布公开包。

## 资产

### host（x86_64）

- `zscamera-sdk-1.0.1-linux-x86_64-host-dev.tar.gz`
- `zscamera-tools-1.0.1-linux-x86_64-host.tar.gz`

### Orin NX（aarch64，VPI3 + NVBUF + NVJPEG 生产包）

- `zscamera-sdk-1.0.1-linux-aarch64-orin-nx-dev.tar.gz`
- `zscamera-runtime-1.0.1-linux-aarch64-orin-nx.tar.gz`
- `zscamera-tools-1.0.1-linux-aarch64-orin-nx.tar.gz`

### AGX Thor（aarch64，VPI4 + TensorRT）

- `zscamera-sdk-1.0.1-linux-aarch64-agx-thor-dev.tar.gz`
- `zscamera-runtime-1.0.1-linux-aarch64-agx-thor.tar.gz`
- `zscamera-tools-1.0.1-linux-aarch64-agx-thor.tar.gz`

校验和：`SHA256SUMS`；资产清单：`v1.0.1.json`。

## 亮点

- 统一 C++17 SDK：公开头文件、静态库 `libzscamera.a`、`find_package(zscamera)`
  CMake 配置；`zscamera_stream`、`zscamera_probe`、`zscamera_calib`、
  `zscamera_ota`、`zscamera_demo`。
- UVDP v2：64 字节小端头、CRC32C、TCP 9600；支持 left/right/depth/cloud/meta
  与 CONTROL/PING，并保留 v1 兼容输出。
- Orin NX 生产包使用 `orin-nx-hw` 构建（VPI3 + NVBUF + NVJPEG，VPI 自有图像，
  不导入 NVBUFFER）；AGX Thor 使用 VPI4 + TensorRT。
- 包内无内部 README、构建脚本、发布检查单或构建机路径；交付包为 Release 构建。
- runtime 只携带目标配置；公开包在模型许可确认前不随附模型/engine，
  `models/README.md` 说明获取与生成方式。

## 安装

```bash
# 校验
sha256sum -c SHA256SUMS

# 板端 runtime
sudo mkdir -p /opt/zscamera
sudo tar -xzf zscamera-runtime-1.0.1-linux-aarch64-<target>.tar.gz \
    -C /opt/zscamera --strip-components=1
sudo /opt/zscamera/bin/zscamera_stream \
    --config /opt/zscamera/config/<target>.conf
```

- Orin NX 工具模式：`orin_nx_vpi.conf`；ROS 模式：`orin_nx_ros.conf`。
- AGX Thor：`agx_thor.conf`。
- 默认 UVDP v2 端口 TCP `9600`；用 `zscamera_probe <board-ip>:9600` 验证。
- 所有模型/engine 必须在目标板生成；Orin 与 Thor 的 engine 不可互换。

## 兼容性

- SDK 1.0.1，UVDP proto 2；最低工具版本 **ZSViewer 2.1.4**。
- Orin NX：JetPack 6.2 / L4T 36.x，VPI 3.x，TensorRT 10.x。
- AGX Thor：JetPack 7.x / L4T 39.x，VPI 4.x，TensorRT 10.x。
- 完整矩阵见 `docs/compatibility.md`。

## 验证

- host：ctest 32/32。
- Orin NX：板端构建 + ctest 32/32；`--print-profile` 显示 `build.vpi=true`。
- AGX Thor：板端构建 + ctest 32/32；双目 30 fps、深度 10 fps、
  `u16_mm_zlib`，probe 15 帧无丢帧。

## 已知问题

- RV1126B 目标源码已支持，P3 板端验收前不发布公开 dev/runtime 包。
- 公开包不随附模型/engine；TensorRT engine 需在目标板生成。
- 双目相位取决于模组 FSYNC 与两路 AE。2026-11-23 新模组实测：SOF 周期
  33.3338 ms（30.000 fps，锁定），但双路固定相差 3.39 ms（std 9 µs、无漂移），
  与两路曝光差一致；需要紧同步时先在模组侧做 AE 同步或固定相同曝光/增益。

# manifests

每个 Release 对应一个 `v<version>.json`，`latest.json` 汇总当前稳定版本。
字段定义见 [`../docs/packages.md`](../docs/packages.md) 第 3 节。

维护规则：

- `latest.json` 只指向稳定版，不指向 `rc`。
- 资产发布后 sha256 不可修改；如需修复，升 patch 版本。
- `assets[].name` 必须与 Release 资产文件名完全一致。
- 预发布版本也保留 `v<version>.json`，但 `status` 为 `rc`，且不进入 `latest`。

发布后自检：

```bash
jq -e '.schema == 1' manifests/latest.json
jq -r '.releases[].assets[].sha256' manifests/latest.json | grep -Ev '^[0-9a-f]{64}$' && exit 1
```

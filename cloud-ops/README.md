# Cloud Ops

个人云服务器通用运维归档目录。

## 目录

- `AGENTS.md`：Agent/Codex 云端操作规范。
- `docs/ServerLayout.md`：Linux 目录、systemd、日志和端口约定。
- `docs/DirectoryGovernance.md`：云端目录职责、临时产物、备份、日志和缓存治理规范。
- `docs/SecretPolicy.md`：密钥、API Key、`.env` 管理约定。
- `docs/ServiceTemplate.md`：新增通用服务的文档模板。
- `docs/AiImageService.md`：通用 AI 图片服务方案。
- `history/`：重要操作、方案调整和验收记录。

## 当前重点

先落地 `ai-image-service`：

- 独立于 N2。
- N2 和其他项目都通过 HTTP 调用。
- 第三方中转 API Key 仅保存在云端 `.env`。
- 第一版使用 FastAPI + SQLite + systemd + 本地图片目录。

## N2 云端标准

- `docs/N2ServerCloudStandard.md`：N2 云端成熟部署标准，适用于后续长期测试服 / 最小正式服。
- `docs/N2ServerOperations.md`：N2 云端服务、目录、日志、配置和治理规范。
- `/home/ubuntu/n2/README.md`：N2 云端统一入口，聚合服务目录、控制台、文档、归档和脚本。

# 2026-06-14 ai-image-service v0.2

## 背景

在 v0.1 骨架基础上继续推进服务化能力。当前仍未配置真实第三方中转 Key，因此不执行真实生图。

## 本次变更

发布新 release：

```text
/opt/ai-image-service/releases/20260614-172026
/opt/ai-image-service/current -> /opt/ai-image-service/releases/20260614-172026
```

主要增强：

- 批量任务改为 SQLite 持久队列。
- 服务启动时自动恢复 `running` 状态任务为 `queued`，避免重启后卡死。
- 后台 worker 常驻处理队列，数量由 `WORKER_CONCURRENCY` 控制。
- 新增 `GET /v1/upstream/probe`，用于填入中转配置后探测 `/models` 可达性。
- `/healthz` 返回服务版本 `0.2.0`。
- `/v1/capabilities` 返回 `worker_concurrency` 和 `probe` 能力。

## 当前上游配置

```text
UPSTREAM_BASE_URL=EMPTY
UPSTREAM_API_KEY=EMPTY
UPSTREAM_MODEL=gpt-image-2
```

真实 Key 未写入文档。

## 验收结果

健康检查：

```json
{"status":"ok","service":"ai-image-service","version":"0.2.0","upstream_configured":false}
```

能力查询包含：

```text
generate
edit
multi_image_edit
mask_edit
batch_generate
probe
```

未配置上游时：

- `GET /v1/upstream/probe` 返回 503。
- 批量任务可创建、入库、由 worker 处理，并标记为 `completed_with_errors`。

自测批次项目名为 `selftest`，验收后已从 SQLite 清理。

## 后续步骤

1. 填写 `/etc/ai-image-service.env`：
   - `UPSTREAM_BASE_URL`
   - `UPSTREAM_API_KEY`
2. 重启 `ai-image-service`。
3. 运行 `/v1/upstream/probe`。
4. 验证文生图。
5. 验证图生图、多参考图和 mask。

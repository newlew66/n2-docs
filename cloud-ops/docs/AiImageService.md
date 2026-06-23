# Ai Image Service Plan

## 目标

部署一个通用 AI 图片生成服务 `ai-image-service`，用于封装第三方中转 API 和 `gpt-image-2` 能力。该服务不绑定 N2，N2 只是调用方之一，后续其他个人项目也可以复用。

## 第一版能力

- 文生图：prompt 生成图片。
- 图生图：输入图片和 prompt 生成新图片。
- 多参考图：多张输入图片作为参考。
- mask 局部编辑：输入图片、mask 和 prompt 做局部重绘。
- 批量异步生图：一次提交多个任务，后台生成，轮询结果。
- 能力查询：返回当前模型、支持尺寸、质量选项。
- 健康检查：用于部署验收和监控。

## 技术方案

```text
FastAPI + SQLite + systemd + 本地图片存储
```

第一版不上 Redis、Celery、对象存储和后台管理，降低实现成本。批量任务使用 SQLite 保存任务状态，由服务内后台 worker 顺序或小并发处理。

## 部署目录

```text
/opt/ai-image-service
  服务代码、Python 虚拟环境、启动入口。

/etc/ai-image-service.env
  第三方中转 API Key、服务访问 Key、模型、端口等配置。

/var/lib/ai-image-service
  sqlite 数据库、任务记录。

/var/lib/ai-image-service/images
  生成图片文件。
```

## systemd

服务名：

```text
ai-image-service.service
```

常用命令：

```bash
sudo systemctl status ai-image-service --no-pager
sudo systemctl restart ai-image-service
sudo journalctl -u ai-image-service -n 100 --no-pager
```

## 配置项

真实值只写入 `/etc/ai-image-service.env`。

```env
HOST=0.0.0.0
PORT=18082
SERVICE_API_KEY=<local-service-access-key>
UPSTREAM_BASE_URL=<relay-openai-compatible-base-url>
UPSTREAM_API_KEY=<relay-api-key>
UPSTREAM_MODEL=gpt-image-2
PUBLIC_BASE_URL=http://115.159.127.52:18082
STORAGE_ROOT=/var/lib/ai-image-service/images
DB_PATH=/var/lib/ai-image-service/service.db
DEFAULT_SIZE=1024x1024
DEFAULT_QUALITY=low
MAX_PROMPT_CHARS=4000
MAX_BATCH_ITEMS=20
WORKER_CONCURRENCY=2
```

## API 草案

```http
GET /healthz
GET /v1/capabilities
POST /v1/images/generate
POST /v1/images/edit
POST /v1/images/batch
GET /v1/images/batch/{batch_id}
GET /images/{filename}
```

### 文生图

```bash
curl -X POST http://127.0.0.1:18082/v1/images/generate \
  -H "Authorization: Bearer <SERVICE_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"project":"n2","prompt":"a fantasy house icon, game UI style","size":"1024x1024","quality":"low"}'
```

### 图生图

使用 `multipart/form-data`：

```bash
curl -X POST http://127.0.0.1:18082/v1/images/edit \
  -H "Authorization: Bearer <SERVICE_API_KEY>" \
  -F "project=n2" \
  -F "prompt=make this house look snowy" \
  -F "size=1024x1024" \
  -F "quality=low" \
  -F "image=@input.png"
```

### 多参考图

```bash
curl -X POST http://127.0.0.1:18082/v1/images/edit \
  -H "Authorization: Bearer <SERVICE_API_KEY>" \
  -F "project=n2" \
  -F "prompt=generate a game card using these references" \
  -F "image=@character.png" \
  -F "image=@style_ref.png" \
  -F "image=@background_ref.png"
```

### 批量任务

```bash
curl -X POST http://127.0.0.1:18082/v1/images/batch \
  -H "Authorization: Bearer <SERVICE_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"project":"n2","items":[{"type":"generate","prompt":"icon of a wooden sword"},{"type":"generate","prompt":"icon of a healing potion"}]}'
```

查询：

```bash
curl -H "Authorization: Bearer <SERVICE_API_KEY>" \
  http://127.0.0.1:18082/v1/images/batch/<batch_id>
```

## 响应约定

生成成功返回服务自己的图片 URL，不直接依赖第三方临时 URL。

```json
{
  "id": "img_xxx",
  "project": "n2",
  "model": "gpt-image-2",
  "url": "http://115.159.127.52:18082/images/img_xxx.png",
  "created": 1781430000
}
```

## 边界与约束

- 第一版使用一个 `SERVICE_API_KEY`，不做多项目 Key 管理。
- 请求保留 `project` 字段，用于日志和未来扩展。
- 不允许客户端传 `UPSTREAM_BASE_URL` 或第三方 API Key。
- 模型默认固定为 `gpt-image-2`，后续如需多模型再加白名单。
- 图片先落本地磁盘，后续可迁移到腾讯云 COS。
- 缺关键配置时服务启动失败，不做隐式 fallback。

## 验收方式

- `GET /healthz` 返回健康状态。
- 文生图生成成功并返回可访问图片 URL。
- 图生图生成成功并返回可访问图片 URL。
- 批量任务可提交、可查询、结果落盘。
- `journalctl` 中没有完整 API Key 或 Authorization。

## 后续扩展点

- 多项目独立 Key 和额度。
- 腾讯云 COS 存储。
- Nginx + HTTPS + 域名。
- 管理后台。
- 用量统计和成本估算。
- 更细粒度的并发、队列和失败重试策略。

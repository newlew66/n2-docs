# 2026-06-14 ai-image-service 骨架部署

## 背景

开始推进通用 AI 图片生成服务。第一步先部署可运行的服务骨架、systemd、目录结构和基础 API，不配置真实第三方中转 Key，不执行真实生图调用。

## 本次变更

新增服务：

```text
ai-image-service.service
```

部署目录：

```text
/opt/ai-image-service/releases/20260614-171142
/opt/ai-image-service/current -> /opt/ai-image-service/releases/20260614-171142
/etc/ai-image-service.env
/var/lib/ai-image-service
/var/lib/ai-image-service/images
```

安装系统依赖：

```text
python3.12-venv
python3-pip-whl
python3-setuptools-whl
```

Python 依赖：

```text
fastapi==0.115.6
uvicorn[standard]==0.34.0
httpx==0.28.1
python-multipart==0.0.20
```

## 当前能力

已部署 API：

```http
GET /healthz
GET /v1/capabilities
POST /v1/images/generate
POST /v1/images/edit
POST /v1/images/batch
GET /v1/images/batch/{batch_id}
GET /images/{filename}
```

当前 `UPSTREAM_BASE_URL` 和 `UPSTREAM_API_KEY` 为空，因此真实生成接口会返回 503：

```json
{"detail":"Upstream image provider is not configured"}
```

这是预期行为，用于避免未配置上游时静默 fallback 或返回假结果。

## 配置

真实配置文件：

```text
/etc/ai-image-service.env
```

已生成本服务访问 Key：

```text
SERVICE_API_KEY=<已生成，真实值仅保存在 /etc/ai-image-service.env>
```

仍需手动填写：

```env
UPSTREAM_BASE_URL=<第三方中转 /v1 base url>
UPSTREAM_API_KEY=<第三方中转 API key>
```

## 验收结果

服务状态：

```text
systemctl is-active ai-image-service -> active
```

健康检查：

```json
{"status":"ok","service":"ai-image-service","upstream_configured":false}
```

能力查询：

```json
{
  "model":"gpt-image-2",
  "upstream_configured":false,
  "supports":["generate","edit","multi_image_edit","mask_edit","batch_generate"]
}
```

未带 Authorization 查询能力接口返回 401。

未配置上游时调用生成接口返回 503。

## 常用命令

```bash
sudo systemctl status ai-image-service --no-pager
sudo systemctl restart ai-image-service
sudo journalctl -u ai-image-service -n 100 --no-pager
curl http://127.0.0.1:18082/healthz
```

## 后续步骤

1. 填写 `/etc/ai-image-service.env` 中的 `UPSTREAM_BASE_URL` 和 `UPSTREAM_API_KEY`。
2. 重启服务。
3. 探测第三方中转是否支持：文生图、图生图、多参考图、mask。
4. 通过真实生图验收后，再决定是否开放公网端口或接 Nginx/HTTPS。

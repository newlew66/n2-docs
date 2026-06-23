# 2026-06-14 ai-image-service 方案归档

## 背景

需要在个人腾讯云服务器上部署一个通用图片生成服务，封装第三方中转 API 和 `gpt-image-2`，供 N2 以及后续其他个人项目使用。

## 方案选择

放弃将能力直接写入 N2 Gateway 的方案，改为独立通用服务。

原因：

- 图片生成能力不是 N2 专属。
- 后续其他项目也需要复用。
- 独立 systemd 服务更便于重启、排障和扩展。
- 第三方中转 API Key 不应进入 N2 客户端或 N2 项目配置链路。

## 核心设计

- 服务名：`ai-image-service`。
- 技术：FastAPI + SQLite + systemd + 本地图片存储。
- 能力：文生图、图生图、多参考图、mask 编辑、批量异步生图。
- 目录：`/opt/ai-image-service`、`/etc/ai-image-service.env`、`/var/lib/ai-image-service`。
- 鉴权：第一版使用单个 `SERVICE_API_KEY`。

## 边界

- 本次只归档规范和方案，尚未部署服务代码。
- 不记录任何真实第三方 API Key、密码或私钥。
- N2 仅作为后续调用方，不改 N2 当前服务。

## 验收目标

后续实现完成后，应至少验证：

- `/healthz` 可用。
- 文生图可返回图片 URL。
- 图生图可返回图片 URL。
- 批量任务可提交和查询。
- systemd 可管理服务。
- 日志不泄露密钥。

## 后续扩展

- 腾讯云 COS。
- 多项目 Key 和额度。
- 域名和 HTTPS。
- 管理后台与用量统计。

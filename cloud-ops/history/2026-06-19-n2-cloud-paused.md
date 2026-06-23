# 2026-06-19 N2 云端游戏服停服保留

## 背景

当前开发阶段主要使用本地服，不需要云端 N2 游戏服持续运行。

本次操作目标是 **停服保留**：停止运行资源和日志输出，但不删除代码、发布产物、配置和数据。

## 停服前状态

- `n2-gateway.service`：enabled / active
- `n2-meta-silo.service`：enabled / active
- Docker Compose 基础设施运行中：PostgreSQL、Redis、Jaeger、OTel Collector
- 监听端口包括：5092、30000、11111、5432、6379、16686、4317、4318

## 执行操作

```bash
sudo systemctl stop n2-gateway
sudo systemctl stop n2-meta-silo
sudo systemctl disable n2-gateway
sudo systemctl disable n2-meta-silo
cd /home/ubuntu/n2-server/Server
sudo docker compose -f deploy/docker-compose.yml --env-file .env down
```

## 停服后状态

- `n2-gateway.service`：disabled / inactive
- `n2-meta-silo.service`：disabled / inactive
- Docker Compose `ps` 无运行容器
- 5092、30000、11111、5432、6379、16686、4317、4318 相关监听已释放
- Gateway `/healthz` 无法连接，符合停服预期
- nginx `/n2/` 控制台仍响应 Basic Auth 401，说明面板入口未受影响
- AI Image `/healthz` 仍响应，未受 N2 停服影响

## 控制台同步

已更新 `/home/ubuntu/.openclaw/canvas/n2/index.html`：

- N2 Gateway：`paused`
- N2 Meta Silo：`paused`
- PostgreSQL / Redis：`paused`
- Jaeger / OTel Collector：`paused`
- 快捷入口中的 Jaeger 标记为 N2 paused

控制台只展示状态和恢复命令，不直接执行操作。

## 恢复命令

```bash
cd /home/ubuntu/n2-server/Server
sudo docker compose -f deploy/docker-compose.yml --env-file .env up -d
sudo systemctl enable n2-meta-silo n2-gateway
sudo systemctl start n2-meta-silo
sudo systemctl start n2-gateway
curl http://127.0.0.1:5092/healthz
```

## 未执行

- 未删除 `/home/ubuntu/n2-server`
- 未删除 `/home/ubuntu/n2-server/publish`
- 未删除 `/etc/n2-server/n2-server.env`
- 未删除 `/home/ubuntu/n2-server/Server/.env`
- 未删除 Docker 数据卷
- 未清理历史日志和备份文件

## 后续建议

- N2 旧 cloud/verify 日志后续可归档。
- N2 控制台 `index.html.bak.*` 稳定后保留最近 5 份，其余归档。
- 如后续重新启用云端服，再处理 OpenTelemetry metrics 刷 journal 的问题。

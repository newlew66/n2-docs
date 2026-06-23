# N2 Server Operations

> 最后更新：2026-06-19
>
> 适用范围：当前个人云服务器上的 N2 服务端、基础设施、N2 控制台与相关日志/部署目录。
>
> 目标：记录 N2 云端现状，明确目录职责、服务关系、配置边界、日志治理与后续清理策略。

## 0. 当前运行状态

> 2026-06-19 23:22 更新：开发阶段当前使用本地服，云端 N2 游戏服已进入 **paused / retained** 状态。

停服保留含义：

- `n2-gateway.service` 与 `n2-meta-silo.service` 已停止并 disabled。
- N2 Docker Compose 基础设施已 `down`，PostgreSQL / Redis / Jaeger / OTel Collector 容器停止。
- N2 代码、发布产物、env 配置和 Docker 数据未删除。
- `/n2/` 控制台仍可访问，服务总览显示 N2 相关服务为 `paused`。
- 后续需要云端联调时，按本文恢复命令重新启动。

恢复顺序：

```bash
cd /home/ubuntu/n2-server/Server
sudo docker compose -f deploy/docker-compose.yml --env-file .env up -d
sudo systemctl enable n2-meta-silo n2-gateway
sudo systemctl start n2-meta-silo
sudo systemctl start n2-gateway
curl http://127.0.0.1:5092/healthz
```

## 1. 当前结论

2026-06-19 只读盘点结果：

- N2 Gateway 与 Meta Silo 正常运行。
- Gateway `/healthz` 返回 `HTTP/1.1 200 OK` 与 `Healthy`。
- PostgreSQL、Redis、Jaeger、otel-collector 通过 Docker Compose 运行。
- 当前最大治理风险不是服务不可用，而是 N2 进程持续向 journal/syslog 输出大量 OpenTelemetry 指标日志。
- `/home/ubuntu/n2-server/Server` 中存在早期 cloud/verify 输出日志和 pid 文件，可后续归档或清理。
- N2 控制台 `/home/ubuntu/.openclaw/canvas/n2` 中有大量 `index.html.bak.*`，稳定后应保留最近 5 份。

## 2. 目录现状

```text
/home/ubuntu/n2-server/
  Server/      # N2 服务端源/部署工作目录，约 105M
  publish/     # systemd 当前运行的发布产物，约 34M

/home/ubuntu/n2-server/publish/
  gateway/
  meta-silo/

/home/ubuntu/n2-server/Server/deploy/
  docker-compose.yml
  otel-collector-config.yaml
  README.md

/home/ubuntu/.openclaw/canvas/n2/
  index.html
  index.html.bak.*
```

大小快照：

```text
/home/ubuntu/n2-server        约 138M
/home/ubuntu/n2-server/Server 约 105M
/home/ubuntu/n2-server/publish 约 34M
/home/ubuntu/.openclaw/canvas/n2 约 596K
```

## 3. 服务关系

systemd：

```text
n2-meta-silo.service
n2-gateway.service
nginx.service
```

基础设施：

```text
n2-server-postgres
n2-server-redis
n2-server-jaeger
n2-server-otel-collector
```

依赖关系：

- `n2-meta-silo.service` 先启动，提供 Orleans Silo / Meta 状态能力。
- `n2-gateway.service` 依赖 `n2-meta-silo.service`。
- PostgreSQL / Redis / Jaeger / otel-collector 由 `Server/deploy/docker-compose.yml` 管理。
- nginx 负责公开面板和工具入口，N2 Gateway 当前直接监听 `0.0.0.0:5092`。

## 4. systemd 当前配置

`n2-meta-silo.service`：

```text
WorkingDirectory=/home/ubuntu/n2-server/publish/meta-silo
EnvironmentFile=/etc/n2-server/n2-server.env
ExecStart=/usr/bin/dotnet /home/ubuntu/n2-server/publish/meta-silo/Game.N2.MetaSiloHost.dll
```

`n2-gateway.service`：

```text
WorkingDirectory=/home/ubuntu/n2-server/publish/gateway
EnvironmentFile=/etc/n2-server/n2-server.env
Environment=ASPNETCORE_URLS=http://0.0.0.0:5092
ExecStart=/usr/bin/dotnet /home/ubuntu/n2-server/publish/gateway/Game.N2.GatewayHost.dll
Requires=n2-meta-silo.service
```

状态快照：

```text
n2-meta-silo.service active running, since 2026-06-13 14:18:54 CST, NRestarts=0
n2-gateway.service   active running, since 2026-06-13 14:19:02 CST, NRestarts=0
```

## 5. 端口

当前监听：

```text
5092        N2 Gateway, 0.0.0.0
30000       N2 Meta Silo, 127.0.0.1
11111       N2 Meta Silo, 127.0.0.1
5432        PostgreSQL, docker
6379        Redis, docker
16686       Jaeger UI, docker
18082       nginx / OpenClaw tool gateway
```

健康检查：

```bash
curl -sS -i --max-time 5 http://127.0.0.1:5092/healthz
```

期望：

```text
HTTP/1.1 200 OK
Healthy
```

## 6. 配置边界

当前配置文件：

```text
/etc/n2-server/n2-server.env
/home/ubuntu/n2-server/Server/.env
```

规则：

- systemd 正式运行读取 `/etc/n2-server/n2-server.env`。
- `/home/ubuntu/n2-server/Server/.env` 主要服务 Docker Compose / 本地部署脚本。
- 文档中只允许记录字段名，不记录真实连接串、密码、Token。
- 修改配置前必须先备份对应 env 文件。

字段类别包括：

```text
PostgreSQL 数据库名、用户、密码、端口
Redis 端口与连接串
OpenTelemetry / OTLP endpoint
ServerDb connection string
Persistence store 配置
```

## 7. 日志现状与风险

只读盘点发现，最近 1 小时 N2 journal 行数很高：

```text
n2-gateway   约 151284 行 / hour
n2-meta-silo 约 69840 行 / hour
```

最近日志主要是 OpenTelemetry / .NET runtime / HTTP metrics：

```text
http.client.request.duration
http.client.open_connections
http.client.request.time_in_queue
process.runtime.dotnet.assemblies.count
dns.lookup.duration
Histogram bucket 输出
```

这很可能是 `/var/log/syslog` 和 journal 膨胀的重要来源。

当前系统日志盘点：

```text
/var/log/syslog   约 3.3G
/var/log/journal  约 4.0G
```

优先级：高。

## 8. 日志治理建议

先不要直接清空日志。建议按顺序处理：

1. 确认 N2 是否启用了 OpenTelemetry Console Exporter 或指标直接写控制台。
2. 将 metrics 输出改为 OTLP exporter，不写 stdout。
3. 或在云端环境关闭高频 runtime/http metrics 控制台输出。
4. 重启 N2 服务后观察 10 分钟 journal 行数。
5. 确认刷日志来源解决后，再按 `DirectoryGovernance.md` 限制 journal 大小。

排查命令：

```bash
journalctl -u n2-gateway --since '10 minutes ago' --no-pager | wc -l
journalctl -u n2-meta-silo --since '10 minutes ago' --no-pager | wc -l
journalctl -u n2-gateway -n 120 --no-pager
journalctl -u n2-meta-silo -n 120 --no-pager
```

代码/配置排查方向：

```bash
grep -R "ConsoleExporter\|AddConsoleExporter\|Console" -n /home/ubuntu/n2-server/Server/src /home/ubuntu/n2-server/Server/*.props 2>/dev/null || true
grep -R "OpenTelemetry" -n /home/ubuntu/n2-server/Server/src /home/ubuntu/n2-server/Server/*.props 2>/dev/null || true
```

## 9. N2 目录内可整理项

`/home/ubuntu/n2-server/Server` 中存在：

```text
gateway-host.cloud.out.log  约 4.5M / 36923 行
gateway-host.cloud.err.log
meta-host.cloud.out.log     约 3.1M / 64462 行
meta-host.cloud.err.log
gateway-host.verify.out.log
meta-host.verify.out.log
gateway-host.pid
meta-host.pid
```

判断：这些更像 systemd 正式托管前的手动 cloud/verify 启动产物。

建议：

- 当前服务已由 systemd 托管后，可以移动到归档目录。
- 归档前确认最近 7 天没有脚本依赖这些 pid/log 文件。
- 不建议直接删除，先归档 7-14 天。

建议归档路径：

```text
/home/ubuntu/archive/n2-server-legacy-logs/YYYYMMDD/
```

## 10. N2 控制台备份治理

当前目录：

```text
/home/ubuntu/.openclaw/canvas/n2/
```

有大量：

```text
index.html.bak.20260619-*
```

规则：

- 迭代当天可以保留全部备份。
- 面板稳定后保留最近 5 份。
- 其余移动到：

```text
/home/ubuntu/archive/n2-panel-backups/YYYYMMDD/
```

清理前验收：

```bash
curl -I http://127.0.0.1:18082/n2/
```

## 11. Do Not Touch

没有明确任务时不要改动：

```text
/etc/n2-server/n2-server.env
/home/ubuntu/n2-server/publish/gateway
/home/ubuntu/n2-server/publish/meta-silo
/home/ubuntu/n2-server/Server/.env
/home/ubuntu/n2-server/Server/deploy/docker-compose.yml
/var/lib/docker
PostgreSQL / Redis 数据卷
```

不要执行：

```bash
rm -rf /home/ubuntu/n2-server
rm -rf /var/lib/docker
rm -rf /home/ubuntu/n2-server/publish
```

## 12. 常用只读诊断命令

```bash
systemctl status n2-meta-silo --no-pager
systemctl status n2-gateway --no-pager
journalctl -u n2-meta-silo -n 100 --no-pager
journalctl -u n2-gateway -n 100 --no-pager
curl -sS -i --max-time 5 http://127.0.0.1:5092/healthz
ss -ltnp | grep -E ':(5092|30000|11111|5432|6379|16686)' || true
```

Docker 基础设施：

```bash
docker compose -f /home/ubuntu/n2-server/Server/deploy/docker-compose.yml --env-file /home/ubuntu/n2-server/Server/.env ps
```

## 13. 发布与回滚建议

当前 `/home/ubuntu/n2-server/publish/gateway` 和 `publish/meta-silo` 是直接运行目录。

后续成熟化建议改成 releases/current：

```text
/home/ubuntu/n2-server/releases/<timestamp>/gateway
/home/ubuntu/n2-server/releases/<timestamp>/meta-silo
/home/ubuntu/n2-server/current -> /home/ubuntu/n2-server/releases/<timestamp>
```

systemd 后续可改为：

```text
WorkingDirectory=/home/ubuntu/n2-server/current/gateway
ExecStart=/usr/bin/dotnet /home/ubuntu/n2-server/current/gateway/Game.N2.GatewayHost.dll
```

本次不执行此调整，只作为成熟化方向。

## 14. 当前建议下一步

1. 优先处理 OpenTelemetry metrics 刷 journal/syslog 的问题。
2. 观察修复后 10 分钟内 `journalctl` 行数是否显著下降。
3. 对 N2 旧 cloud/verify 日志和 pid 文件做归档，不直接删除。
4. N2 控制台稳定后归档旧 `index.html.bak.*`，保留最近 5 份。
5. 后续如要长期测试服化，再评估 releases/current 发布结构。
## 15. 2026-06-19 停服后清理结果

已在 N2 停服保留状态下执行保守清理：

- 归档 `/home/ubuntu/n2-server/Server/*cloud*.log`、`*verify*.log`、`*host.pid` 到 `/home/ubuntu/archive/n2-server-legacy-logs/20260619/`。
- N2 控制台备份从 22 份收敛为最近 5 份，其余移动到 `/home/ubuntu/archive/n2-panel-backups/20260619/`。
- 写入 journald 大小限制 `/etc/systemd/journald.conf.d/size-limit.conf`，当前 journal 已进一步收敛到约 133M。
- 用户确认运行时临时日志无需保留后，已删除 syslog 压缩归档与旧 syslog 轮转，仅保留当前活跃 `/var/log/syslog`。

清理后验证：

- `n2-gateway.service` / `n2-meta-silo.service` 仍为 `disabled / inactive`。
- N2 相关端口 `5092/30000/11111/5432/6379/16686/4317/4318` 未监听。
- `/n2/` 控制台仍响应 Basic Auth。
- AI Image 服务入口仍响应。

未删除：N2 代码、发布产物、env 配置、Docker 数据卷。
## 16. 2026-06-19 N2 统一入口目录

已建立 `/home/ubuntu/n2/` 作为 N2 云端统一入口。该目录当前是索引层，不改变 systemd 和 nginx 的真实运行路径。

结构：

```text
/home/ubuntu/n2/
  README.md
  server/
    source  -> /home/ubuntu/n2-server/Server
    publish -> /home/ubuntu/n2-server/publish
  console/
    current -> /home/ubuntu/.openclaw/canvas/n2
  docs/
    operations.md     -> /home/ubuntu/cloud-ops/docs/N2ServerOperations.md
    cloud-standard.md -> /home/ubuntu/cloud-ops/docs/N2ServerCloudStandard.md
  archive/
    old-panel-backups/20260619/
    sync-packages/n2-server-sync-20260619.zip
  scripts/
    status.sh
    start-cloud.sh
    stop-cloud.sh
```

已移动纯归档/同步包：

- `/home/ubuntu/archive/n2-panel-backups/20260619` -> `/home/ubuntu/n2/archive/old-panel-backups/20260619`
- `/home/ubuntu/n2-server-sync.zip` -> `/home/ubuntu/n2/archive/sync-packages/n2-server-sync-20260619.zip`

未移动真实运行目录：

- `/home/ubuntu/n2-server/Server`
- `/home/ubuntu/n2-server/publish`
- `/home/ubuntu/.openclaw/canvas/n2`
- `/etc/n2-server/n2-server.env`

常用入口：

```bash
/home/ubuntu/n2/scripts/status.sh
/home/ubuntu/n2/scripts/start-cloud.sh
/home/ubuntu/n2/scripts/stop-cloud.sh
```


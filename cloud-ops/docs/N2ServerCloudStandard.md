# N2 Server Cloud Standard

## 定位

N2 云端服务是游戏服务器，不是个人通用工具服务。后续云端部署应按长期测试服 / 最小正式服的成熟标准设计，而不是继续沿用用户家目录下的临时联调工作区。

N2 与通用 `ai-image-service` 的关系：

- N2 是业务游戏服务器。
- `ai-image-service` 是跨项目通用能力。
- 两者独立部署、独立 systemd、独立配置和数据目录。
- N2 可以作为调用方访问 `ai-image-service`，但不承载其代码和密钥。

## 成熟目录标准

N2 云端部署使用以下目录：

```text
/opt/n2-server/
  releases/<timestamp>/
    gateway/
    meta-silo/
    deploy/
  current -> /opt/n2-server/releases/<timestamp>

/etc/n2-server/
  gateway.env
  meta-silo.env
  infra.env

/var/lib/n2-server/
  generated/
  backups/
  runtime/

/var/log/n2-server/
  可选文件日志目录。默认优先使用 systemd journal。

/home/ubuntu/cloud-ops/
  运维规范、部署标准、变更历史。
```

说明：

- `/opt/n2-server` 放服务发布产物和部署脚本，不放真实密钥。
- `/etc/n2-server` 放环境配置和敏感连接串，权限收紧。
- `/var/lib/n2-server` 放运行数据、备份、生成配置快照等持久数据。
- `/home/ubuntu/n2-server` 不作为成熟云端标准目录；如存在，只视为历史联调工作区或临时同步目录。

## 服务拓扑

第一阶段成熟单节点：

```text
Unity Client
  -> Nginx / HTTPS
  -> n2-gateway.service
  -> n2-meta-silo.service
  -> PostgreSQL / Redis

OTel Collector / Jaeger
  <- Gateway / Meta telemetry
```

systemd 服务：

```text
n2-gateway.service
n2-meta-silo.service
```

基础设施：

```text
PostgreSQL
Redis
OTel Collector
Jaeger
```

短期可以继续由 Docker Compose 承载基础设施，但 Compose 文件和 `.env` 应位于成熟部署目录或明确同步来源。

## 端口约定

```text
5092
  Gateway 内部 HTTP 端口。正式对外应由 Nginx/HTTPS 代理。

4317 / 4318
  OTel Collector，仅内网或本机访问。

16686
  Jaeger UI，默认不公网暴露。

5432 / 6379
  PostgreSQL / Redis，不公网暴露。
```

公网入口目标：

```text
https://<domain>/
```

过渡阶段可以保留：

```text
http://<server-ip>:5092
```

但只作为测试入口，并在文档中标注清理条件。

## 配置与密钥

N2 配置文件：

```text
/etc/n2-server/gateway.env
/etc/n2-server/meta-silo.env
/etc/n2-server/infra.env
```

权限建议：

```bash
sudo chown -R root:ubuntu /etc/n2-server
sudo chmod 750 /etc/n2-server
sudo chmod 640 /etc/n2-server/*.env
```

约束：

- 数据库密码、Redis 密码、JWT 密钥、第三方服务 Key 不写入 `/opt`、`cloud-ops` 文档或 Git。
- systemd service 文件只引用 `EnvironmentFile=`，不内嵌密钥。
- 缺关键配置时服务启动失败，不使用默认连接串或 fallback 配置掩盖问题。

## 发布结构

推荐发布结构：

```text
/opt/n2-server/releases/20260614-180000/
  gateway/
    Game.N2.GatewayHost
  meta-silo/
    Game.N2.MetaSiloHost
  deploy/
    docker-compose.yml
    otel-collector-config.yaml

/opt/n2-server/current -> /opt/n2-server/releases/20260614-180000
```

发布流程：

1. 上传或构建新 release 到 `/opt/n2-server/releases/<timestamp>`。
2. 校验 release 文件完整性。
3. 备份 `/etc/n2-server/*.env`。
4. 切换 `current` 软链接。
5. 重启 systemd 服务。
6. 执行健康检查和关键 API 验收。
7. 写入 `cloud-ops/history`。

切换命令：

```bash
sudo ln -sfn /opt/n2-server/releases/<timestamp> /opt/n2-server/current
sudo systemctl restart n2-meta-silo
sudo systemctl restart n2-gateway
```

回滚命令：

```bash
sudo ln -sfn /opt/n2-server/releases/<old-timestamp> /opt/n2-server/current
sudo systemctl restart n2-meta-silo
sudo systemctl restart n2-gateway
```

## systemd 标准

服务文件：

```text
/etc/systemd/system/n2-meta-silo.service
/etc/systemd/system/n2-gateway.service
```

核心要求：

- `WorkingDirectory` 指向 `/opt/n2-server/current/<service>`。
- `EnvironmentFile` 指向 `/etc/n2-server/<service>.env`。
- `Restart=always`。
- 服务启动用户可先使用 `ubuntu`，后续可收紧为专用用户 `n2`。
- Gateway 依赖 Meta 和基础设施健康，但不要通过脚本静默 fallback。

常用命令：

```bash
sudo systemctl status n2-meta-silo --no-pager
sudo systemctl status n2-gateway --no-pager
sudo journalctl -u n2-meta-silo -n 100 --no-pager
sudo journalctl -u n2-gateway -n 100 --no-pager
```

## 健康检查与验收

基础验收：

```bash
curl http://127.0.0.1:5092/healthz
curl http://127.0.0.1:5092/
```

公网验收，过渡期：

```bash
curl http://<server-ip>:5092/healthz
```

正式公网验收：

```bash
curl https://<domain>/healthz
```

业务验收至少包含：

- `auth.login`
- `auth.heartbeat`
- `player.getData`
- 当前登录引导依赖的 System / Task / Guide / Shop / Home 路由

远端模式不允许 fallback 到 `LocalMock`；缺路由直接失败并记录缺口。

## 数据与备份

成熟方案必须明确：

- PostgreSQL 数据卷位置。
- Redis 数据持久化策略。
- 数据库备份目录：`/var/lib/n2-server/backups/postgres`。
- 配置快照目录：`/var/lib/n2-server/backups/config`。
- 备份保留周期。

第一版最低要求：

```bash
sudo mkdir -p /var/lib/n2-server/backups/postgres
sudo mkdir -p /var/lib/n2-server/backups/config
```

并在发布前备份 `/etc/n2-server`。

## 与本地服和客户端的关系

- 本地开发服、云端测试服、正式服共用同一份 `Server/` 代码。
- 不维护 `ServerLocal` / `ServerCloud` 分叉。
- 环境差异通过 env、部署脚本、Gateway 地址表达。
- Unity 客户端只连接 Gateway。
- `RemoteStaging` / `RemoteProd` 都走 `RemoteHttpTransport`。
- 远端模式禁止回退 `LocalMock`。

## 阶段规划

### 阶段 1：成熟单节点测试服

- `/opt`、`/etc`、`/var/lib` 目录标准落地。
- Gateway + MetaSilo systemd 托管。
- PostgreSQL / Redis / OTel / Jaeger 单机部署。
- Gateway 提供公网测试入口。
- 补齐客户端登录引导所需服务端路由。

### 阶段 2：生产化单节点

- Nginx + HTTPS。
- DB / Redis / Jaeger 端口收口。
- PostgreSQL 自动备份。
- 日志、监控、告警基础化。
- Admin/GM 能力独立入口和审计。

### 阶段 3：服务拆分

- Gateway 多实例。
- MetaSilo 多实例。
- BattleHost 独立。
- JobWorker 独立。
- AdminHost 独立。
- 数据库主从或托管化。

## 当前决策

既然云端 N2 正式部署尚未开始，后续不再以 `/home/ubuntu/n2-server` 作为成熟目标目录。新的云端 N2 部署应直接按本标准进入 `/opt/n2-server`、`/etc/n2-server`、`/var/lib/n2-server`。

如已有历史联调目录 `/home/ubuntu/n2-server`，迁移前应先确认是否仍有运行中服务依赖；迁移完成并通过验收后，再清理旧目录和旧 systemd 引用。

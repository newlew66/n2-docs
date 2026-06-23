# 2026-06-19 N2 停服后保守清理

## 背景

N2 云端游戏服已进入停服保留状态。用户确认当前服务器还没有真正意义上的开启云端游戏服，因此执行保守清理。

本次清理不删除 N2 代码、发布产物、配置文件或 Docker 数据卷。

## 清理前状态

- `n2-gateway.service`：disabled / inactive
- `n2-meta-silo.service`：disabled / inactive
- N2 旧 cloud/verify 日志存在于 `/home/ubuntu/n2-server/Server`
- N2 控制台备份：22 份
- `/var/log/syslog`：约 3.3G
- `/var/log/journal`：约 4.0G

## 执行内容

### 1. 归档 N2 旧运行日志

移动以下文件到：

```text
/home/ubuntu/archive/n2-server-legacy-logs/20260619/
```

匹配范围：

```text
/home/ubuntu/n2-server/Server/*cloud*.log
/home/ubuntu/n2-server/Server/*verify*.log
/home/ubuntu/n2-server/Server/*host.pid
```

### 2. 收敛 N2 控制台备份

- 原有 `index.html.bak.*`：22 份
- 保留最近 5 份
- 其余归档到：

```text
/home/ubuntu/archive/n2-panel-backups/20260619/
```

### 3. 限制 journald 大小

新增：

```text
/etc/systemd/journald.conf.d/size-limit.conf
```

内容：

```ini
[Journal]
SystemMaxUse=1G
RuntimeMaxUse=256M
MaxRetentionSec=14day
```

执行：

```bash
sudo systemctl restart systemd-journald
sudo journalctl --vacuum-size=1G
```

### 4. 归档并截断过大的 syslog

由于 `/var/log/syslog` 仍超过 1G，执行：

- 复制并压缩到 `/home/ubuntu/archive/system-logs/20260619/syslog.before-truncate.gz`
- 截断当前 `/var/log/syslog`

说明：logrotate 因 `/var/log` 权限策略跳过 rsyslog 轮转，因此采用保留压缩副本后截断当前日志的方式。

## 清理后结果

- N2 旧日志归档大小：约 4.8M
- N2 控制台旧备份归档大小：约 428K
- syslog 归档压缩文件：约 355M
- 当前 `/var/log/syslog`：约 16K
- 当前 `/var/log/journal`：约 993M
- N2 控制台备份保留：5 份

## 验收

- `n2-gateway.service` / `n2-meta-silo.service` 仍为 `disabled / inactive`。
- N2 相关端口 `5092/30000/11111/5432/6379/16686/4317/4318` 未监听。
- `/n2/` 控制台仍响应 Basic Auth。
- AI Image 服务入口仍响应。

## 未执行

- 未删除 `/home/ubuntu/n2-server`
- 未删除 `/home/ubuntu/n2-server/publish`
- 未删除 `/etc/n2-server/n2-server.env`
- 未删除 `/home/ubuntu/n2-server/Server/.env`
- 未删除 Docker 数据卷
- 未删除 AI Image / OpenClaw / 通用工具服务
## 追加：运行时临时日志删除

用户确认当前云端游戏服还没有真正意义上开启，也没有客户端连接过，因此运行时临时日志不需要保留。

追加清理：

- 删除 `/home/ubuntu/archive/n2-server-legacy-logs/20260619/`。
- 删除 `/home/ubuntu/archive/system-logs/20260619/`。
- 删除旧 `/var/log/syslog.1` 和 `syslog.*.gz` 轮转文件。
- 将 journald 限制从 1G 调整为 200M：

```ini
[Journal]
SystemMaxUse=200M
RuntimeMaxUse=64M
MaxRetentionSec=7day
```

清理后：

- `/var/log/syslog`：约 24K
- `/var/log/journal`：约 133M
- N2 服务仍为 `disabled / inactive`
- N2 相关端口仍未监听
- `/n2/` 控制台和 AI Image 入口仍响应

仍未删除：N2 代码、发布产物、env 配置、Docker 数据卷、OpenClaw/AI Image 服务。


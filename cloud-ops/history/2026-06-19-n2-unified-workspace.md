# 2026-06-19 N2 云端统一入口目录

## 背景

N2 云端游戏服当前处于停服保留状态，用户希望整理 N2 相关内容，减少零散文件和入口。

本次只做第一阶段目录收拢：建立统一入口与软链接聚合，不移动真实运行目录，不修改 systemd，不删除配置和数据。

## 新增目录

```text
/home/ubuntu/n2/
  README.md
  server/
  console/
  docs/
  archive/
  scripts/
```

## 软链接

```text
/home/ubuntu/n2/server/source -> /home/ubuntu/n2-server/Server
/home/ubuntu/n2/server/publish -> /home/ubuntu/n2-server/publish
/home/ubuntu/n2/console/current -> /home/ubuntu/.openclaw/canvas/n2
/home/ubuntu/n2/docs/operations.md -> /home/ubuntu/cloud-ops/docs/N2ServerOperations.md
/home/ubuntu/n2/docs/cloud-standard.md -> /home/ubuntu/cloud-ops/docs/N2ServerCloudStandard.md
```

## 移动的纯归档/同步包

```text
/home/ubuntu/archive/n2-panel-backups/20260619
  -> /home/ubuntu/n2/archive/old-panel-backups/20260619

/home/ubuntu/n2-server-sync.zip
  -> /home/ubuntu/n2/archive/sync-packages/n2-server-sync-20260619.zip
```

## 新增脚本

```text
/home/ubuntu/n2/scripts/status.sh
/home/ubuntu/n2/scripts/start-cloud.sh
/home/ubuntu/n2/scripts/stop-cloud.sh
```

脚本不包含密钥。`start-cloud.sh` 只封装恢复云端 N2 的 docker compose 和 systemd 命令。

## 未执行

- 未移动 `/home/ubuntu/n2-server/Server`
- 未移动 `/home/ubuntu/n2-server/publish`
- 未移动 `/home/ubuntu/.openclaw/canvas/n2`
- 未修改 systemd unit
- 未修改 nginx
- 未删除 env、Docker 数据卷或发布产物

## 验收

- `/home/ubuntu/n2/scripts/status.sh` 正常执行。
- N2 仍为 `disabled / inactive`。
- N2 Docker Compose 无运行容器。
- N2 相关端口未监听。
- `/n2/` 控制台仍响应 Basic Auth。
- 归档和同步包已移动到 `/home/ubuntu/n2/archive/`。

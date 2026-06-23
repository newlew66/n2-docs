# 2026-06-14 N2 云端成熟部署标准归档

## 背景

确认 N2 云端正式部署尚未开始，因此不沿用 `/home/ubuntu/n2-server` 作为长期目标目录，改为从一开始按成熟游戏服务器部署标准设计。

## 决策

N2 云端使用：

```text
/opt/n2-server
/etc/n2-server
/var/lib/n2-server
```

而不是继续把正式运行目录放在 `/home/ubuntu/n2-server`。

## 原因

- N2 是游戏服务器，不是个人通用工具，也不是临时脚本。
- 服务代码、敏感配置、运行数据需要分离。
- releases/current 结构便于发布和回滚。
- systemd、Nginx、备份、日志和后续拆分都更容易标准化。
- 避免将用户家目录工作区和正式服务运行区混在一起。

## 核心标准

- 发布产物：`/opt/n2-server/releases/<timestamp>`。
- 当前版本：`/opt/n2-server/current`。
- 配置：`/etc/n2-server/*.env`。
- 数据和备份：`/var/lib/n2-server`。
- 服务：`n2-gateway.service`、`n2-meta-silo.service`。
- 公网入口最终走 Nginx + HTTPS。
- 远端客户端模式禁止 fallback 到 `LocalMock`。

## 影响范围

本次只更新云端运维归档，不修改运行中服务、不创建目录、不迁移 N2。

## 后续执行条件

开始 N2 云端部署时，先按 `docs/N2ServerCloudStandard.md` 创建目录、systemd、env 和验收脚本。

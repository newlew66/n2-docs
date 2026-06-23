# 2026-06-14 服务器运维规范归档

## 背景

在开始实现通用 `ai-image-service` 前，先为个人云服务器确定基础运维规范，避免通用服务与 N2 项目耦合，并减少后续密钥、目录、日志和回滚混乱。

## 核心规范

- 运维文档放在 `/home/ubuntu/cloud-ops`。
- 通用服务代码放在 `/opt/<service-name>`。
- 服务配置放在 `/etc/<service-name>.env`。
- 服务数据放在 `/var/lib/<service-name>`。
- systemd 作为服务管理入口。
- HTTP 服务必须提供 `/healthz`。
- 重要变更写入 `/home/ubuntu/cloud-ops/history`。

## /home/ubuntu 的定位

`/home/ubuntu` 是 `ubuntu` 用户的家目录，适合放个人配置、临时工作区、项目工作区和运维文档。不作为通用服务的正式部署目录。

## 端口约定

- `5090-5099`：N2 相关服务。
- `18080-18099`：个人通用 HTTP 服务。
- `19000-19099`：实验性服务。

## 密钥约束

- 真实密钥只写入 `/etc/<service-name>.env` 或明确的服务私有配置。
- 文档、history、Git 仓库、systemd service 文件不记录真实密钥。
- 日志不打印完整 Authorization、API Key、密码或私钥。

## 影响范围

本次只更新运维规范文档，不修改运行中的 N2 服务，不新增 systemd 服务，不改防火墙。

## 验收

- 已更新 `AGENTS.md`。
- 已更新 `docs/ServerLayout.md`。
- 已更新 `docs/SecretPolicy.md`。
- 已更新 `docs/ServiceTemplate.md`。
- 已新增本 history 记录。

## 后续

实现 `ai-image-service` 时，应先读取本规范，再按服务模板补充部署记录、验收命令和回滚方式。

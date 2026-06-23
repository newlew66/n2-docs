# Server Layout

## 标准目录

通用服务默认使用以下布局：

```text
/opt/<service-name>/
  服务代码、虚拟环境、启动入口。建议使用 releases/current 结构支持回滚。

/etc/<service-name>.env
  服务环境变量和敏感配置。不得写入 Git 或公开文档。

/var/lib/<service-name>/
  服务运行数据、sqlite 数据库、生成文件、持久化状态。

/var/log/<service-name>/
  可选日志目录。优先使用 systemd journal，必要时再落文件。

/home/ubuntu/cloud-ops/
  个人云服务器运维规范、服务方案和历史记录。
```

## /home/ubuntu 的职能

`/home/ubuntu` 是 Linux 用户 `ubuntu` 的家目录。SSH 登录 `ubuntu@<server>` 后默认进入这里。

适合放：

- 当前用户的个人配置，如 `.ssh`、`.bashrc`。
- 临时工作区和手动同步文件。
- 个人运维文档，例如 `/home/ubuntu/cloud-ops`。
- 项目级工作目录，例如已有的 `/home/ubuntu/n2-server`。

不建议放：

- 通用服务的正式运行代码。
- systemd 服务的敏感环境变量。
- 长期业务数据或生成资源。

原因：`/home/ubuntu` 更偏“用户工作区”，而不是标准服务部署目录。通用服务应使用 `/opt`、`/etc`、`/var/lib` 分离代码、配置和数据。

## systemd

通用服务应使用 systemd 托管：

```bash
sudo systemctl status <service-name> --no-pager
sudo systemctl restart <service-name>
sudo journalctl -u <service-name> -n 100 --no-pager
sudo journalctl -u <service-name> -f
```

服务应提供健康检查接口：

```bash
curl http://127.0.0.1:<port>/healthz
```

## 端口

建议端口分段：

```text
5090-5099
  N2 相关服务。

18080-18099
  个人通用 HTTP 服务。

19000-19099
  实验性服务。
```

`ai-image-service` 规划使用：

```text
18082
```

新服务部署前应检查端口占用：

```bash
ss -ltnp | grep ':<port>' || true
```

## 发布与回滚

建议通用服务使用 releases/current 结构：

```text
/opt/<service-name>/releases/<timestamp>
/opt/<service-name>/current -> /opt/<service-name>/releases/<timestamp>
```

发布：

```bash
sudo ln -sfn /opt/<service-name>/releases/<new-release> /opt/<service-name>/current
sudo systemctl restart <service-name>
```

回滚：

```bash
sudo ln -sfn /opt/<service-name>/releases/<old-release> /opt/<service-name>/current
sudo systemctl restart <service-name>
```

## 与项目目录的关系

- N2 项目目录：`/home/ubuntu/n2-server/`。
- 通用服务不放入 N2 项目目录，避免生命周期耦合。
- 通用服务文档归档在 `/home/ubuntu/cloud-ops/`。
## 目录治理

具体的现状盘点、home 根目录临时产物、备份文件、日志和缓存清理规范见 [`DirectoryGovernance.md`](./DirectoryGovernance.md)。


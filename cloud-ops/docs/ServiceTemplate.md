# Service Template

## 服务名

`<service-name>`

## 目标

说明该服务解决什么问题、服务哪些项目、是否为通用服务。

## 部署目录

```text
/opt/<service-name>/releases/<timestamp>
/opt/<service-name>/current
/etc/<service-name>.env
/var/lib/<service-name>
```

## systemd

```text
<service-name>.service
```

常用命令：

```bash
sudo systemctl status <service-name> --no-pager
sudo systemctl restart <service-name>
sudo journalctl -u <service-name> -n 100 --no-pager
sudo journalctl -u <service-name> -f
```

## 端口与健康检查

```bash
ss -ltnp | grep ':<port>' || true
curl http://127.0.0.1:<port>/healthz
```

## 配置项

```env
HOST=
PORT=
SERVICE_API_KEY=
```

只写字段名和用途，不写真实密钥。

## 变更流程

1. 查看当前状态。
2. 备份配置。
3. 部署新 release 或修改配置。
4. 重启服务。
5. 执行健康检查和关键接口验收。
6. 查看日志。
7. 写入 history。

## 验收

列出 curl 命令、预期响应和失败排查方式。

## 回滚

```bash
sudo ln -sfn /opt/<service-name>/releases/<old-release> /opt/<service-name>/current
sudo systemctl restart <service-name>
```

## 后续扩展

记录计划中的能力和清理条件。

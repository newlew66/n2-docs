# Secret Policy

## 原则

- 私钥、密码、API Key、Token 不写入文档、脚本、Git 仓库或聊天记录。
- 公钥可以复制到服务器授权位置；私钥只能留在可信本机。
- 云端服务的真实配置放在 `/etc/<service>.env`。
- 文档只记录配置字段名、用途和示例占位值。

## SSH

- 本机私钥用于登录服务器，不上传到服务器。
- 云服务器只保存对应公钥。
- 需要新增登录方式时，优先使用独立 key，便于吊销。
- `/home/ubuntu/.ssh/authorized_keys` 用于记录允许登录 `ubuntu` 用户的公钥。

## API Key

第三方中转 API Key 只允许存在于服务环境文件，例如：

```text
/etc/ai-image-service.env
```

不得写入：

- `/home/ubuntu/cloud-ops/docs/*.md`
- `/home/ubuntu/cloud-ops/history/*.md`
- 项目仓库
- 客户端代码
- systemd service 文件本体
- shell history 中的明文命令

## .env 权限

建议：

```bash
sudo chown root:ubuntu /etc/<service-name>.env
sudo chmod 640 /etc/<service-name>.env
```

变更前备份：

```bash
sudo cp /etc/<service-name>.env /etc/<service-name>.env.bak.$(date +%Y%m%d-%H%M%S)
```

## 日志

日志允许记录：

- time
- request_id
- project
- endpoint
- model
- size
- status
- duration_ms
- error_code

日志不得记录：

- 完整 Authorization
- 完整第三方 API Key
- 私钥内容
- 密码
- 第三方响应里的敏感字段

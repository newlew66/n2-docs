# Directory Governance

> 最后更新：2026-06-19
>
> 适用范围：`/home/ubuntu`、`/opt`、`/etc`、`/var/lib`、`/var/log`、OpenClaw canvas、N2 与通用 AI 工具服务。
>
> 目标：让云端目录可解释、可迁移、可清理，避免服务代码、运行数据、临时产物和个人工作区混在一起。

## 1. 当前目录现状快照

2026-06-19 盘点结果：

```text
/home/ubuntu/cloud-ops            运维规范、服务文档、历史记录，约 80K
/home/ubuntu/n2-server            N2 服务端目录，约 138M
/home/ubuntu/.openclaw/canvas      静态工具面板目录，含 n2/kling/video-pipeline
/opt/ai-image-service              AI 图片服务程序目录，约 130M
/var/lib/ai-image-service          AI 图片服务运行数据目录，约 68M
```

`/home/ubuntu` 根目录当前还存在若干临时或工具产物：

```text
pipeline_output/
pipeline_uploads/
kling_output/
kling_uploads/
tree_output/
n2-server-sync.zip
test_video.mp4
ai-img.py
```

这些可以短期保留，但不应作为长期标准位置。

## 2. 标准职责划分

| 目录 | 职责 | 允许内容 | 不建议内容 |
|------|------|----------|------------|
| `/home/ubuntu/cloud-ops` | 云端运维规范与归档 | AGENTS、docs、history | 密钥、真实 env、运行数据 |
| `/home/ubuntu/n2-server` | N2 项目服务端工作目录 | N2 Server、publish、部署脚本 | 通用工具服务、AI 媒体长期产物 |
| `/home/ubuntu/.openclaw/canvas` | OpenClaw/nginx 静态工具页面 | n2 控制台、kling 页面、pipeline 页面 | 大文件输出、长期上传数据 |
| `/opt/<service>` | 通用服务代码 | releases/current、venv、入口脚本 | 用户上传文件、生成结果、密钥明文 |
| `/etc/<service>.env` | 服务环境变量 | API Key、Token、DB 连接串 | 文档、代码、输出文件 |
| `/var/lib/<service>` | 服务运行数据 | sqlite、任务状态、上传、输出、缓存 | 源码、临时测试文件 |
| `/var/log` 或 journal | 系统和服务日志 | systemd journal、nginx 日志 | 无限制增长的调试输出 |

## 3. 推荐目标结构

- `/home/ubuntu/n2/`：N2 云端统一入口，聚合 server、console、docs、archive、scripts；当前以软链接为主，不替代真实运行路径。

### 3.1 通用 AI 媒体工具

后续建议逐步收敛为：

```text
/opt/ai-image-service/
  current 或服务代码

/var/lib/ai-image-service/
  db/
  uploads/
  outputs/
  tmp/

/opt/ai-media-tools/
  video-pipeline/
  kling/

/var/lib/ai-media-tools/
  pipeline/uploads/
  pipeline/outputs/
  kling/uploads/
  kling/outputs/
  tmp/
```

当前已有的 `/home/ubuntu/pipeline_output`、`/home/ubuntu/pipeline_uploads`、`/home/ubuntu/kling_output`、`/home/ubuntu/kling_uploads` 后续可迁移到 `/var/lib/ai-media-tools/`。

### 3.2 N2

```text
/home/ubuntu/n2-server/
  Server/
  publish/
```

N2 是项目服务端目录，可以保留在 `/home/ubuntu/n2-server`。如果未来变成长期正式服，再评估迁到 `/opt/n2-server` + `/var/lib/n2-server`。

### 3.3 OpenClaw 面板

```text
/home/ubuntu/.openclaw/canvas/n2/
  index.html
  index.html.bak.<timestamp>
```

`canvas` 目录只承载静态页面。不要把大文件、生成资源、视频、图片批量输出放在这里。

## 4. /home/ubuntu 根目录治理

`/home/ubuntu` 是用户工作区，不是长期服务数据目录。

允许短期存在：

- 临时上传测试文件。
- 手动同步包，如 `n2-server-sync.zip`。
- 一次性排查脚本。

要求：

- 临时文件超过 7 天应归档或删除。
- 输出目录超过 1G 前必须评估迁移到 `/var/lib/<service>`。
- 一次性脚本若仍有价值，移动到 `/home/ubuntu/cloud-ops/scripts` 或对应服务文档；否则删除。
- zip、mp4、图片输出等大文件不应长期堆在 home 根目录。

建议归档目录：

```text
/home/ubuntu/archive/
  tmp-YYYYMMDD/
  old-sync-packages/
```

## 5. 备份文件治理

当前 `/home/ubuntu/.openclaw/canvas/n2/` 中存在多份 `index.html.bak.*`，这是近期面板迭代产生的安全备份。

规则：

- 频繁编辑期：允许保留当天全部备份。
- 稳定后：保留最近 5 份。
- 重要里程碑：可移动到 `/home/ubuntu/archive/n2-panel-backups/YYYYMMDD/`。
- 删除备份前先确认当前 `index.html` 可访问且历史记录已归档。

推荐检查：

```bash
ls -lh /home/ubuntu/.openclaw/canvas/n2/index.html*
curl -I http://127.0.0.1:18082/n2/
```

## 6. 日志治理

2026-06-19 盘点发现：

```text
/var/log/syslog      约 3.3G
/var/log/syslog.1    约 226M
/var/log/journal     约 4.0G
```

这是当前最需要关注的磁盘风险。

处理顺序：

1. 先查是谁在刷日志，不要直接清空。
2. 确认是否有服务异常、循环报错、debug 输出过多。
3. 修复来源后，再限制日志大小。

只读排查命令：

```bash
sudo tail -n 200 /var/log/syslog
sudo journalctl -p warning -n 200 --no-pager
sudo journalctl --disk-usage
```

限制 journal 大小的建议配置：

```ini
# /etc/systemd/journald.conf.d/size-limit.conf
[Journal]
SystemMaxUse=1G
RuntimeMaxUse=256M
MaxRetentionSec=14day
```

应用：

```bash
sudo systemctl restart systemd-journald
sudo journalctl --vacuum-size=1G
```

注意：执行日志清理前必须先记录原因和影响范围到 `history/`。

## 7. 缓存治理

盘点中较大的用户缓存：

```text
/home/ubuntu/.cache        约 1.2G
/home/ubuntu/.npm          约 727M
/home/ubuntu/.openclaw/npm 约 715M
/home/ubuntu/.local        约 1.5G
/home/ubuntu/venv          约 1.3G
```

原则：

- `.cache`、`.npm` 可以清，但要避开服务运行中依赖。
- `venv`、`.local` 可能是工具或服务依赖，删除前必须确认使用方。
- `.openclaw/npm` 属于 OpenClaw 工具链依赖，非必要不删。

建议先只做统计：

```bash
du -sh /home/ubuntu/.cache /home/ubuntu/.npm /home/ubuntu/.openclaw/npm /home/ubuntu/.local /home/ubuntu/venv
```

## 8. 清理分级

### Safe：通常可以处理

- 明确过期的 `*.zip` 同步包。
- 明确测试用的 mp4、png、临时输出。
- 已归档且当前页面验证通过的旧 `index.html.bak.*`。
- 过期的上传/输出文件，但必须先确认服务不引用。

### Review：需要确认

- `/home/ubuntu/venv`
- `/home/ubuntu/.local`
- `/home/ubuntu/.openclaw/npm`
- `/home/ubuntu/tree_output`
- 所有 systemd 服务目录和数据目录。

### Do Not Touch：禁止随意处理

- `/home/ubuntu/.ssh`
- `/home/ubuntu/.openclaw/credentials`
- `/etc/*.env`
- `/var/lib/docker`
- `/home/ubuntu/n2-server/Server` 和 `publish`，除非有明确部署任务。
- `/opt/ai-image-service`、`/var/lib/ai-image-service`，除非按服务文档执行。

## 9. 新服务目录审批清单

新增服务前必须明确：

- 服务名：`<service-name>`
- systemd 名：`<service-name>.service`
- 代码目录：`/opt/<service-name>`
- 配置文件：`/etc/<service-name>.env`
- 数据目录：`/var/lib/<service-name>`
- 端口：是否在规划段内
- 健康检查：`/healthz`
- 日志查看：`journalctl -u <service-name>`
- 备份/回滚方式
- 文档：`cloud-ops/docs/<ServiceName>.md`
- 历史：`cloud-ops/history/YYYY-MM-DD-<service>-<action>.md`

## 10. 当前建议的下一步

1. 排查 `/var/log/syslog` 增长来源。
2. 评估并限制 journal 大小。
3. 规划 `/home/ubuntu/pipeline_*`、`/home/ubuntu/kling_*` 迁移到 `/var/lib/ai-media-tools/`。
4. 对 `/home/ubuntu/.openclaw/canvas/n2/index.html.bak.*` 做保留策略：稳定后保留最近 5 份。
5. 将有价值的一次性脚本归档到 `cloud-ops/scripts/` 或服务文档，删除无价值临时文件。

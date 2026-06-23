# 2026-06-19 云端目录治理规范

## 背景

对当前云端目录做只读盘点后，发现整体服务能正常运行，但 `/home/ubuntu` 根目录开始混入工具上传、输出、测试文件和同步包；同时 `/var/log/syslog` 与 `/var/log/journal` 占用较大，需要形成明确治理规范。

## 盘点摘要

关键目录：

- `/home/ubuntu/cloud-ops`：云端运维规范和历史归档，约 80K。
- `/home/ubuntu/n2-server`：N2 服务端目录，约 138M。
- `/home/ubuntu/.openclaw/canvas`：静态工具面板目录，含 `n2/kling/video-pipeline`。
- `/opt/ai-image-service`：AI 图片服务程序目录，约 130M。
- `/var/lib/ai-image-service`：AI 图片服务运行数据目录，约 68M。

需要关注的目录和文件：

- `/home/ubuntu/pipeline_output`
- `/home/ubuntu/pipeline_uploads`
- `/home/ubuntu/kling_output`
- `/home/ubuntu/kling_uploads`
- `/home/ubuntu/tree_output`
- `/home/ubuntu/n2-server-sync.zip`
- `/home/ubuntu/test_video.mp4`
- `/var/log/syslog` 约 3.3G
- `/var/log/journal` 约 4.0G

## 本次新增

新增文档：

- `/home/ubuntu/cloud-ops/docs/DirectoryGovernance.md`

同步更新：

- `/home/ubuntu/cloud-ops/README.md`
- `/home/ubuntu/cloud-ops/docs/ServerLayout.md`

## 文档内容

`DirectoryGovernance.md` 定义了：

- 当前目录现状快照
- 标准职责划分
- 推荐目标结构
- `/home/ubuntu` 根目录治理
- 备份文件治理
- 日志治理
- 缓存治理
- 清理分级：Safe / Review / Do Not Touch
- 新服务目录审批清单
- 当前建议的下一步

## 重要规则

- `/home/ubuntu` 是用户工作区，不作为长期服务数据目录。
- 通用服务仍按 `/opt/<service>`、`/etc/<service>.env`、`/var/lib/<service>` 分离代码、配置和数据。
- OpenClaw canvas 只放静态页面，不放大文件输出。
- 清理日志前先排查刷日志来源，不直接清空。
- `.ssh`、OpenClaw credentials、`/etc/*.env`、Docker 数据、N2 服务目录、AI Image 服务目录都列为 Do Not Touch，除非有明确任务。

## 验收

已验证：

- `DirectoryGovernance.md` 存在并包含“当前目录现状快照”“日志治理”“Do Not Touch”“当前建议的下一步”。
- `README.md` 已链接 `docs/DirectoryGovernance.md`。
- `ServerLayout.md` 已链接 `DirectoryGovernance.md`。

## 后续建议

1. 排查 `/var/log/syslog` 增长来源。
2. 限制 systemd journal 大小。
3. 规划 `pipeline_*`、`kling_*` 迁移到 `/var/lib/ai-media-tools/`。
4. N2 面板备份稳定后保留最近 5 份。
5. 清理或归档 home 根目录中的同步包、测试视频和临时输出。

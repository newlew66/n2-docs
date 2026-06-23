# 2026-06-19 N2 云端服务治理文档

## 背景

用户希望将服务器通用治理交给云端规范处理，但需要先整理 N2 相关目录和服务现状。

本次执行只读盘点，不修改服务配置、不重启服务、不清理文件。

## 只读盘点结果

N2 服务状态：

- `n2-meta-silo.service`：active running，自 2026-06-13 14:18:54 CST 起运行，`NRestarts=0`。
- `n2-gateway.service`：active running，自 2026-06-13 14:19:02 CST 起运行，`NRestarts=0`。
- Gateway `/healthz` 返回 `HTTP/1.1 200 OK` 与 `Healthy`。

基础设施：

- `n2-server-postgres`：Up / healthy
- `n2-server-redis`：Up / healthy
- `n2-server-jaeger`：Up
- `n2-server-otel-collector`：Up

目录：

- `/home/ubuntu/n2-server`：约 138M
- `/home/ubuntu/n2-server/Server`：约 105M
- `/home/ubuntu/n2-server/publish`：约 34M
- `/home/ubuntu/.openclaw/canvas/n2`：约 596K

## 发现的问题

### 1. N2 journal 日志量异常偏高

最近 1 小时 journal 行数：

- `n2-gateway`：约 151284 行
- `n2-meta-silo`：约 69840 行

日志主要是 OpenTelemetry / .NET runtime / HTTP metrics 输出，例如：

- `http.client.request.duration`
- `http.client.open_connections`
- `process.runtime.dotnet.assemblies.count`
- histogram bucket 输出

这很可能是 `/var/log/syslog` 和 journal 膨胀的重要来源。

### 2. N2 Server 目录存在历史验证产物

`/home/ubuntu/n2-server/Server` 中存在：

- `gateway-host.cloud.out.log`
- `gateway-host.cloud.err.log`
- `meta-host.cloud.out.log`
- `meta-host.cloud.err.log`
- `gateway-host.verify.out.log`
- `meta-host.verify.out.log`
- `gateway-host.pid`
- `meta-host.pid`

这些更像 systemd 正式托管前的手动 cloud/verify 启动产物，后续可归档。

### 3. N2 控制台备份较多

`/home/ubuntu/.openclaw/canvas/n2` 中存在大量 `index.html.bak.20260619-*`，属于当天频繁迭代安全备份。稳定后建议保留最近 5 份，其余归档。

## 本次新增文档

新增：

- `/home/ubuntu/cloud-ops/docs/N2ServerOperations.md`

同步更新：

- `/home/ubuntu/cloud-ops/README.md`

## 文档内容

`N2ServerOperations.md` 记录：

- 当前结论
- 目录现状
- 服务关系
- systemd 配置
- 端口
- 配置边界
- 日志现状与风险
- 日志治理建议
- N2 目录内可整理项
- N2 控制台备份治理
- Do Not Touch 清单
- 常用只读诊断命令
- 发布与回滚建议
- 当前建议下一步

## 验收

已验证：

- `N2ServerOperations.md` 存在。
- 文档包含“当前结论”“日志现状与风险”“Do Not Touch”“当前建议下一步”。
- `cloud-ops/README.md` 已链接 `docs/N2ServerOperations.md`。

## 后续建议

1. 优先排查并关闭 N2 OpenTelemetry metrics 向 stdout/journal 的高频输出。
2. 观察修复后 10 分钟 journal 行数。
3. 将 N2 旧 cloud/verify 日志和 pid 文件归档，不直接删除。
4. N2 控制台稳定后保留最近 5 份备份。
5. 后续长期测试服化时评估 releases/current 发布结构。

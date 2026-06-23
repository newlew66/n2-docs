# deploy

本目录用于放服务端部署相关资产。

当前已落地的本地开发环境基线：

- `docker-compose.yml`
  - `postgres:17-alpine`
  - `redis:7.4-alpine`
  - `otel/opentelemetry-collector-contrib:0.123.0`
  - `jaegertracing/all-in-one:1.76.0`
- `otel-collector-config.yaml`
  - 提供 OTLP gRPC / HTTP 接收
  - traces 转发到 Jaeger
  - traces / metrics / logs 同时输出到 collector debug exporter，便于本地排障

推荐启动方式：

```powershell
docker compose -f deploy/docker-compose.yml up -d
```

默认端口：

- PostgreSQL：`5432`
- Redis：`6379`
- OTLP gRPC：`4317`
- OTLP HTTP：`4318`
- OTel Collector health：`13133`
- Jaeger UI：`16686`

环境变量约定：

- 仓库根提供 `.env.example`
- 若本地需要覆盖默认端口、数据库名或密码，可在 `Server/.env` 中覆写

当前边界：

- 该 compose 仅面向本地开发，不代表生产部署拓扑
- 生产环境的 Kubernetes / Agones / Helm 清单仍留待后续阶段

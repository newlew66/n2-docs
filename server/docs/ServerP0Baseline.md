# 服务端 P0 基线冻结

> 冻结日期：2026-04-24
> 适用范围：`Server/` 的 P0 / P1 阶段

## 1. 本次冻结的目的

本文件用于把服务端从“仅有归档与 Todo”的状态推进到“可继续落项目骨架”的状态，并明确以下问题：

- 首版技术栈是否按架构归档默认方案落地
- P0 业务闭环的边界到底包含哪些能力
- `BattleHost` 在当前阶段是否只保留占位
- 命名规范、命名空间规范和依赖检查方式如何统一
- 为什么当前先不把 `Server/` 改造成顶层 monorepo

## 2. 已确认技术栈

首版技术栈按架构归档默认方案冻结，不再为 P0 反复摇摆：

- 目标运行时：`net8.0`
- Gateway 宿主：`ASP.NET Core`
- 长连接与推送：`WebSocket`
- 内部服务调用：`gRPC`
- Meta / World 单实体串行模型：`Orleans`
- 主存储：`PostgreSQL`
- 缓存 / 会话 / 辅助分布式能力：`Redis`
- 可观测性：`OpenTelemetry`
- 战斗宿主编排预留：`Agones`

补充约束：

- 当前阶段先冻结 `TargetFramework = net8.0`，不急于补 `global.json` 锁死 SDK 版本。
- 允许本地使用更高版本 SDK 构建 `net8.0` 项目；等 CI 和首批项目落地后，再单独决定是否追加 SDK 固定策略。
- 当前已冻结首批中央包版本：
  - Orleans：`10.1.0`
  - OpenTelemetry：`1.15.x` 首批接入组（`Extensions.Hosting`、`Exporter.Console`、`Exporter.OpenTelemetryProtocol`、`Instrumentation.AspNetCore`、`Instrumentation.Http`、`Instrumentation.Runtime`）
  - Npgsql：`10.0.2`
- Redis / gRPC 的具体包版本仍延后到真实接线阶段统一冻结。

## 3. P0 业务范围

P0 最小闭环冻结为：

- `auth.login`
- `auth.logout`
- `auth.heartbeat`
- `player.getData`
- `inventory.getList`
- `inventory.useItem`
- `home.getState`
- `home.saveLayout`

P0 必做横切能力：

- 所有写请求必须具备幂等入口
- 单玩家主状态写入必须收敛到统一 Meta 写入口
- 经济变化必须可落流水
- 异步推送必须走 outbox
- 关键操作必须可落审计日志

不在 P0 承诺的内容：

- 公会、排行榜、邮件的完整业务闭环
- 战斗房间调度、战斗结果回流和结算闭环
- 面向多项目复用的完整平台化运营后台

## 4. BattleHost 当前策略

`BattleHost` 当前冻结为“目录和宿主入口占位”，不作为 P0 / M1 / M2 的主要交付目标：

- P0 / P1：只保留目录与宿主命名位置
- M1：优先跑通 `GatewayHost + MetaSiloHost + auth/player/bag/home`
- M2：优先补工具链、迁移、配置发布、审计骨架
- M3：再进入 `BattleHost` 的会话、房间、结果回流和结算闭环

这样做的原因：

- 当前最关键的架构风险不在战斗宿主，而在 Meta 单写入口、状态存储和请求链路是否站得住
- 先做 BattleHost 容易把注意力拉到房间调度和实时逻辑，反而拖慢首个业务闭环
- 归档已经明确战斗结果必须回流 Meta 层结算，因此 BattleHost 不适合先于 Meta 基础能力落地

## 5. 命名与命名空间规范

### 5.1 解决方案与根目录

- 解决方案固定为 `Server.sln`
- 公共构建约束固定在仓库根：`Directory.Build.props`、`Directory.Packages.props`
- 服务端工作区目录固定为：`docs/`、`todo/`、`src/`、`tests/`、`deploy/`、`sql/`、`generated/`

### 5.2 项目命名

- 基础设施层：`GameServer.Foundation.*`
- 项目模块层：`Game.N2.*`
- 宿主层：`Game.N2.*Host`
- 工具层：`GameServer.*`
- 生成代码层：`GameProto.Network`、`GameConfig`

### 5.3 命名空间

- 默认命名空间与程序集名一致
- 子命名空间按目录延展，例如 `Game.N2.Domain.Player`
- 禁止在同一项目内混用多个根命名空间
- 文档默认中文，代码标识保持英文

### 5.4 目录命名

- 目录统一使用英文 `PascalCase`
- 一个职责一个目录，不做“Misc / Common2 / TempImpl”这类模糊命名
- 宿主项目只保留装配和入口，不在 `Hosts/` 目录下写业务规则

### 5.5 首批项目模板策略

首批项目骨架按以下策略冻结：

- 非宿主项目统一从最小 `classlib` 起步
- `Game.N2.GatewayHost` 使用空 `web` 模板
- `Game.N2.MetaSiloHost` 与 `Game.N2.BattleHost` 使用最小 `console` 模板
- 先锁定项目名、目录和引用方向，再逐步引入 Orleans / ASP.NET Core / gRPC / Npgsql / OTel 的具体包和 wiring

这样做的原因：

- 当前最需要冻结的是边界和依赖方向，而不是一次性把外部基础设施接满
- 最小模板可以尽快让解决方案进入“可编译、可引用、可迭代”的状态
- 具体外部包版本仍应结合真实实现阶段统一收敛到中央包管理

### 5.6 当前最小运行链路

当前已验证的最小运行链路为：

- `Game.N2.MetaSiloHost` 以本地 Orleans 集群模式启动
- `Game.N2.GatewayHost` 作为外部 Orleans client 接入同一本地集群
- Gateway 已提供最小 HTTP 路由：
  - `POST /auth/login`
  - `POST /auth/logout`
  - `POST /auth/heartbeat`
  - `GET /player/{playerId}`
  - `GET /inventory/{playerId}`
  - `POST /inventory/use`
  - `GET /home/{playerId}`
  - `POST /home/save`
  - `GET /healthz`
- 已完成一次本地链路验证：`health -> login -> player -> inventory -> useItem -> home.save -> home.get`
- `MetaSiloHost` 已增加以下首版基础设施入口：
  - `ConnectionStrings:ServerDb`
  - `ConnectionStrings:Redis`
  - `Persistence:PlayerStateStore`（`InMemory` / `Postgres`）
  - `Persistence:IdempotencyStore`（`InMemory` / `Postgres`）
  - `OpenTelemetry:Otlp:Endpoint`
- `GameServer.DbMigrator` 已建立，可顺序执行 `sql/migrations/*.sql`
- `deploy/docker-compose.yml` 与 `otel-collector-config.yaml` 已建立，用于本地 PostgreSQL / Redis / OTel / Jaeger 环境
- 当前 P0 写请求均已增加 `IdempotencyKey` 入口，且已完成一次本地重放验证：
  - `auth.login` 同键返回同一 `sessionId`
  - `inventory.useItem` 同键不会重复扣减
  - `home.saveLayout` 同键不会重复执行
- `GatewayHost` 已注入响应头：
  - `X-Request-Id`
  - `X-Trace-Id`

边界说明：

- 当前默认开发链路仍以 `InMemory` 模式启动，不强制依赖 PostgreSQL
- `Postgres` 玩家状态存储代码路径已落地，但当前工作区未完成容器级实机验证
- `Postgres` 幂等记录代码路径已落地，但当前工作区未完成容器级实机验证
- Redis、真实鉴权、幂等、审计和 outbox 落库仍未接通

## 6. 依赖检查清单

评审和建项目时必须按下列方向检查：

1. `Foundation.*` 不允许引用 `Game.N2.*`
2. `Game.N2.Domain` 不允许直接引用 protobuf 生成类
3. `Game.N2.Application.Contracts` 只放命令、查询、事件和稳定 DTO
4. `Game.N2.Application` 负责编排，不承担宿主启动职责
5. `Game.N2.Meta.Abstractions` 只放 grain 接口和稳定调用边界
6. `Game.N2.Meta.Grains` 负责 Meta 权威状态实现，但不能把宿主装配写进去
7. `Hosts/*` 只做配置、DI、管线装配，不写业务规则
8. `BattleHost` 不允许直接改玩家主状态
9. 所有异步通知都走 outbox，不直接在业务代码里“顺手推送”
10. 生成代码项目只能被引用，不能反向依赖业务层

推荐把依赖检查拆成两个问题：

- 这个引用是为了复用基础设施，还是把玩法模型抽错了层？
- 这个写入口是 Meta 权威写入口，还是绕过了聚合边界？

## 7. 版本控制边界

- `Server/` 使用独立 git 仓库维护
- `client/` 继续维持现有 Unity 仓库边界
- 当前阶段不把两者强行合并成顶层 monorepo

这样做的原因：

- `client/` 当前已经有独立历史和较脏工作区，立即合仓会放大迁移风险
- 服务端需要独立的忽略规则、构建物和提交粒度
- 先把服务端骨架稳定下来，再讨论顶层仓库是否真的有必要

若未来要改成 monorepo，必须单独归档：

- 迁移动机
- git 历史承接方式
- `.gitignore` / `.gitattributes` 调整策略
- 客户端与服务端提交流程的变化

## 8. P0 / P1 验收标准

完成以下内容即可认为 P0 / P1 基线成立：

- P0 冻结文档存在且与架构归档一致
- `Server.sln` 已建立
- `Directory.Build.props` 与 `Directory.Packages.props` 已建立
- `src/`、`tests/`、`deploy/`、`sql/`、`generated/` 目录已建立
- `Server/` 已处于独立 git 仓库下
- Todo 已回写为新的真实状态

## 9. 延后决策

以下内容明确延后到后续阶段处理：

- Proto 与 Luban 输出到 `Server/src/Generated` 的自动化脚本
- Redis / gRPC 的真实接线与配置收敛
- PostgreSQL 状态存储的实机联调、审计 / outbox / 经济流水落库

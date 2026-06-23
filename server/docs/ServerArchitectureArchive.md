# 服务端架构归档

> 归档日期：2026-04-23
> 适用范围：`Server/` 独立服务端工作区

## 1. 背景与决策

本轮决定正式放弃以下旧路径：

- 不再兼容客户端内嵌 `LocalMock` / `LocalGameServer`
- 不再把服务端文档归档到 `client/Assets/Editor/Tools/Doc`
- 不再以“当前本地模拟服可平滑迁移”为主要设计目标

新方案目标：

1. 建立独立的服务端目录、文档和工程结构
2. 采用业内成熟游戏服常见分层：接入层、实体状态层、房间战斗层、基础设施层
3. 基础框架可复用于其他项目，但玩法规则保持项目内聚
4. 后续本地开发也连接真实服务端宿主，不再走客户端内嵌模拟服

## 2. 总体设计目标

### 2.1 要达成的能力

- 支持长连接接入、登录鉴权、请求分发、推送
- 支持单玩家单写者模型，避免并发写穿
- 支持静态配置版本化加载
- 支持经济流水、审计日志、幂等控制
- 支持后续独立战斗宿主和战斗结算回流
- 支持将基础框架复用于其他同类型项目

### 2.2 明确不做的事

- 不做客户端本地模拟服兼容层
- 不做“一个线上进程同时跑多个游戏”的平台型超大一体化系统
- 不把 `Player/Home/Bag/BattleRule` 这类玩法对象抽进通用基础层

## 3. 架构分层

采用“四平面 + 项目模块层”的结构。

### 3.1 Gateway Plane

职责：

- 客户端接入
- 协议编解码
- 登录鉴权
- 版本校验
- 限流、封禁、会话校验
- WebSocket 推送连接管理

约束：

- 无状态
- 不直接落业务库
- 不持有玩家权威状态

### 3.2 Meta / World Plane

职责：

- 玩家、背包、家园、任务、邮件、公会、排行榜等实体型状态的唯一权威
- 以“单实体串行处理”为核心并发模型
- 负责写库、发奖、推进状态、落审计日志

约束：

- 所有玩家主状态写入都要收敛到统一入口
- 战斗宿主不能绕过此层直接改玩家主状态

### 3.3 Battle / Room Plane

职责：

- 房间分配
- 战斗会话管理
- 实时战斗或战斗校验
- 结算结果回流

约束：

- 不直接持久化玩家主状态
- 仅产生战斗结果与战斗日志，再回流 Meta 层完成正式结算

### 3.4 Foundation Plane

职责：

- 提供跨项目稳定的基础设施能力
- 不承载任何具体玩法对象

包含：

- Hosting
- Rpc
- Auth / Session
- Orleans 运行时封装
- Persistence / Outbox / 审计
- Config
- Push
- Observability
- Ops
- Battle Abstractions

### 3.5 Game Module Layer

职责：

- 放具体项目的玩法规则和业务模块
- 例如 N2 的玩家、背包、家园、成长、战斗适配都留在项目模块层

原则：

- 通用的是基础设施
- 不通用的是玩法模型

## 4. 技术选型

默认技术栈如下：

- `.NET 8 LTS`
- `ASP.NET Core`：Gateway 宿主与 HTTP 接入
- `WebSocket`：推送与长连接
- `gRPC`：内部服务调用
- `Orleans`：Meta / World 单实体串行处理模型
- `PostgreSQL`：主存储
- `Redis`：缓存、会话、辅助分布式能力
- `OpenTelemetry`：日志、指标、追踪
- `Agones`：后续战斗宿主编排预留

## 5. 目录树与项目拆分

目标目录结构如下：

```text
Server/
├── Server.sln
├── Directory.Build.props
├── Directory.Packages.props
├── README.md
├── docs/
├── deploy/
├── generated/
├── sql/
├── src/
│   ├── Generated/
│   │   ├── GameProto.Network/
│   │   └── GameConfig/
│   ├── Foundation/
│   │   ├── GameServer.Foundation.Abstractions/
│   │   ├── GameServer.Foundation.Hosting/
│   │   ├── GameServer.Foundation.Gateway/
│   │   ├── GameServer.Foundation.Rpc/
│   │   ├── GameServer.Foundation.Auth/
│   │   ├── GameServer.Foundation.Session/
│   │   ├── GameServer.Foundation.Orleans/
│   │   ├── GameServer.Foundation.Persistence/
│   │   ├── GameServer.Foundation.Config/
│   │   ├── GameServer.Foundation.Push/
│   │   ├── GameServer.Foundation.Observability/
│   │   ├── GameServer.Foundation.Ops/
│   │   └── GameServer.Foundation.Battle.Abstractions/
│   ├── GameModules/
│   │   └── N2/
│   │       ├── Game.N2.Application.Contracts/
│   │       ├── Game.N2.Domain/
│   │       ├── Game.N2.Application/
│   │       ├── Game.N2.Meta.Abstractions/
│   │       ├── Game.N2.Meta.Grains/
│   │       ├── Game.N2.Battle/
│   │       └── Game.N2.Bootstrap/
│   ├── Hosts/
│   │   ├── Game.N2.GatewayHost/
│   │   ├── Game.N2.MetaSiloHost/
│   │   └── Game.N2.BattleHost/
│   └── Tools/
│       ├── GameServer.DbMigrator/
│       ├── GameServer.ConfigPublisher/
│       └── GameServer.DevCli/
└── tests/
    ├── Foundation/
    └── N2/
```

## 6. 项目职责说明

### 6.1 Generated

- `GameProto.Network`：编译 `Proto/` 生成的 protobuf 模型
- `GameConfig`：编译 Luban 生成的配置访问代码

### 6.2 Foundation

- `Abstractions`：基元、结果模型、时间接口、上下文
- `Hosting`：宿主启动与公共装配
- `Gateway`：接入层能力
- `Rpc`：route 分发与 protobuf 管线
- `Auth`：鉴权与权限控制
- `Session`：登录态与设备会话
- `Orleans`：grain 基类与运行时封装
- `Persistence`：PostgreSQL / Redis / Outbox / 审计
- `Config`：配置版本、加载、激活
- `Push`：推送通道与投递
- `Observability`：OTel、日志、指标
- `Ops`：GM、后台操作、审计、灰度
- `Battle.Abstractions`：房间与战斗会话抽象

### 6.3 N2 项目模块

- `Game.N2.Application.Contracts`：内部命令、查询、事件
- `Game.N2.Domain`：N2 领域模型与业务规则
- `Game.N2.Application`：N2 用例编排
- `Game.N2.Meta.Abstractions`：grain 接口
- `Game.N2.Meta.Grains`：grain 实现
- `Game.N2.Battle`：N2 战斗模块
- `Game.N2.Bootstrap`：N2 模块注册

### 6.4 Hosts

- `Game.N2.GatewayHost`：客户端入口
- `Game.N2.MetaSiloHost`：Meta / World 集群宿主
- `Game.N2.BattleHost`：战斗宿主

## 7. 依赖规则

必须遵守以下规则：

1. `Foundation.*` 不允许引用 `Game.N2.*`
2. `Game.N2.Domain` 不允许直接引用 protobuf 生成类
3. `Hosts/*` 只做装配，不写业务规则
4. 所有玩家主状态写入都必须经过统一 Meta 写入口
5. `BattleHost` 不允许直接改玩家主状态
6. 所有异步通知都走 outbox
7. 所有外部写请求都必须幂等
8. 生成代码项目只能被引用，不能反向引用业务层

## 8. 状态与数据设计原则

### 8.1 并发原则

- 单玩家单写者
- 聚合内串行处理
- 禁止多处直接写玩家主状态

### 8.2 存储原则

首版采用“聚合快照 + 关键流水”并行模型：

- 玩家当前态：快照
- 经济变化：流水
- 推送：outbox
- 审计：独立日志
- 战斗：会话与结果表

### 8.3 配置原则

- `Proto/` 继续作为协议唯一源
- 静态配置继续来自 `Configs/GameConfig`
- 现有 `gen_code_bin_to_server` 后续应改为输出到新的 `Server/src/Generated/GameConfig` 路径
- 配置运行时只认“当前激活版本”

## 9. 与旧本地模拟服的关系

旧本地模拟服只保留参考价值：

- 可参考其业务边界与部分规则拆分
- 不再作为兼容目标
- 不再作为最终宿主实现

明确不延续以下内容：

- `LocalGameServer`
- `LocalLoopbackTransport`
- `LocalSaveRepository`
- 客户端本地权威存档

## 10. 里程碑建议

### M0：架构冻结

- 冻结目录树
- 冻结项目拆分
- 冻结依赖规则
- 冻结 P0 范围

### M1：基础宿主与最小闭环

- GatewayHost
- MetaSiloHost
- Orleans 集群基础
- PostgreSQL / Redis
- OTel
- `auth/player/bag/home` 最小闭环

### M2：服务端工具链

- DbMigrator
- ConfigPublisher
- DevCli
- 审计与 GM 骨架

### M3：BattleHost

- 战斗会话
- 房间分配
- 结果回流
- 结算闭环

## 11. 本归档的用途

本文件是 `Server/` 独立服务端工作的正式归档入口，用于：

- 约束后续目录结构
- 约束项目拆分与依赖方向
- 作为后续开工和评审的基线
- 为未来接入其他项目时提供可复用基础框架边界

后续所有服务端相关增量文档，应优先落到 `Server/docs/`。

## 12. 协作与版本控制边界

- `Server/` 使用独立 git 仓库维护，避免与 `client/` 当前 Unity 仓库的历史、忽略规则和提交粒度互相污染。
- 服务端文档、宿主工程、部署脚本、SQL、测试和后续工具链，均在 `Server/` 仓库内独立提交。
- `client/` 继续维持现有仓库边界；服务端推进不要求先改造成顶层 monorepo。
- 若未来需要改成顶层 monorepo，必须单独归档迁移动机、历史承接方式、忽略规则调整和协作流程变化。

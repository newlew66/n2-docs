# 2026-06-19 N2 面板改为服务与文档控制台

## 背景

原 `/n2/` 面板是通用项目分类卡片骨架，数据依赖浏览器 `localStorage`，不适合当前阶段的 N2 云端管理需求。当前更需要一个只读的服务、文档、工具入口控制台。

## 本次变更

修改文件：

```text
/home/ubuntu/.openclaw/canvas/n2/index.html
```

备份文件：

```text
/home/ubuntu/.openclaw/canvas/n2/index.html.bak.20260619-200429
```

新版定位：

```text
N2 Service Console
```

主要模块：

- 服务总览：N2 Gateway、Meta Silo、AI Image Service、Nginx、PostgreSQL/Redis、Jaeger。
- 文档中心：cloud-ops、N2 服务端、客户端网络、资源管线文档索引。
- 运维记录：cloud-ops history、HistoryCMD、ChangeLog 关键入口。
- 工具入口：AI 图片、视频处理、AI 视频、OpenClaw、Jaeger。
- 简化资源：序列帧、Enemy/Facility 规范、当前藤蔓投手动作资源提示。

## 边界

本面板是静态只读导航控制台：

- 不执行 systemctl。
- 不执行部署。
- 不写数据库。
- 不修改配置表。
- 不承载正式 GM / Admin 能力。

危险操作仍通过 SSH、明确命令和人工确认执行；未来如需动态状态，应新增只读 `/n2-api/`，写操作应进入独立 AdminHost 并带权限和审计。

## 验收

未认证访问：

```text
GET http://115.159.127.52:18082/n2/ -> 401 Unauthorized
```

认证后访问：

```text
GET http://127.0.0.1:18082/n2/ -> 200 OK
<title>N2 Service Console</title>
```

文本编码已清理为 UTF-8 without BOM。

## 后续建议

- 如文档继续增长，可新增 `/n2/docs/` 静态文档站。
- 如需要实时状态，可新增只读 `/n2-api/status`。
- 如需要玩家/GM/配置操作，应单独建设 N2 AdminHost。

## 2026-06-19 20:24 右侧内容卡片固定尺寸

- 调整 `/home/ubuntu/.openclaw/canvas/n2/index.html` 样式，固定右侧服务卡片与文档条目的基础高度。
- 服务卡片标题、描述、元信息、按钮区改为稳定尺寸，减少不同内容长度导致的卡片跳动。
- 文档/工具列表条目补充最小高度和固定标题/描述区域，保持右侧内容扫描一致。
- 验收：本机云端 `curl -u` 访问 `/n2/` 返回 `HTTP/1.1 200 OK`，页面包含 `N2 Service Console`、`Video Pipeline`、`AI Video Generator`。

## 2026-06-19 20:29 服务总览与快捷入口职责拆分

- 将原“工具入口”调整为“快捷入口”，定位为日常跳转页。
- 快捷入口只保留工具名称、用途、打开链接和复制链接，去掉端口、进程、文档路径、排查命令等运维信息。
- 服务总览继续承担服务状态、URL、systemd / compose、端口、文档路径和排查命令展示。
- 验收：云端 `curl -u` 访问 `/n2/` 返回 `HTTP/1.1 200 OK`，页面包含“快捷入口”和 `shortcut-grid`。

## 2026-06-19 20:35 文档中心细分与游戏设计页签

- 文档中心从目录堆叠调整为场景分类：云端运维规范、服务部署与拓扑、客户端接入、AI 资源生产、资源与配置规范、连接与访问。
- 新增左侧独立页签“游戏设计”，先作为占位，不混入文档中心。
- 游戏设计占位分类包含玩法设计、单位设计、关卡设计、数值设计，后续再接入具体设计文档。
- 验收：云端 `curl -u` 访问 `/n2/` 返回 `HTTP/1.1 200 OK`，页面包含“游戏设计”“云端运维规范”“连接与访问”。

## 2026-06-19 20:42 文档中心改为内部页签

- 文档中心不再一次性展开所有分类，改为顶部二级页签切换。
- 当前分类下只显示该分类文档，并显示文档数量，降低页面纵向堆叠和视觉噪音。
- 左侧主导航保持不变；游戏设计仍为独立主页签。
- 验收：云端 `curl -u` 访问 `/n2/` 返回 `HTTP/1.1 200 OK`，页面包含 `doc-tabs`、`activeDocGroup` 和“云端运维规范”。

## 2026-06-19 20:48 游戏设计页签支持本地新增与编辑

- 游戏设计页签增加内部分类页签：玩法设计、单位设计、关卡设计、数值设计。
- 支持在当前浏览器新增和编辑设计条目，字段包括标题、说明、文档路径/链接。
- 数据暂存于浏览器 `localStorage`，定位为个人草稿区；不会写入云端文件，也不会影响静态运维面板只读边界。
- 后续若需要多人共享，应升级为后端存储或正式文档归档流程。
- 验收：云端 `curl -u` 访问 `/n2/` 返回 `HTTP/1.1 200 OK`；本地 `node --check` 检查页面脚本无语法错误。

## 2026-06-19 20:55 游戏设计单位库编辑器

- 将“游戏设计 / 单位设计”从普通草稿条目升级为单位库编辑器。
- 支持单位列表、新增单位、编辑基础信息、玩法说明、美术说明、AI 图片 Prompt、Negative Prompt、资源路径和备注。
- 支持按动作维护 Prompt，默认动作包括 Idle、Move、Attack、Hit、Die；每个动作可编辑 Prompt、Negative Prompt、是否含投射物、固定镜头、保持原设计、是否循环、时长。
- 内置藤蔓投手示例，Attack Prompt 明确不包含投射物。
- 支持复制图片 Prompt、Negative Prompt、动作 Prompt，以及导出当前单位 JSON 到剪贴板。
- 数据暂存于浏览器 `localStorage`，定位为个人草稿；后续多人共享或长期归档时再升级为后端存储或文档写入流程。
- 验收：云端 `curl -u` 访问 `/n2/` 返回 `HTTP/1.1 200 OK`；本地 `node --check` 检查页面脚本无语法错误。

## 2026-06-19 20:59 修复单位库改造后的初始化错误

- 问题：刷新 `/n2/` 后出现 `Uncaught ReferenceError: renderNav is not defined`。
- 原因：单位库改造时替换脚本区间过大，误删了基础渲染函数 `show/copyText/toast/renderNav/renderDashboard/renderDocs/renderHistory/renderTools/renderResources`。
- 修复：从备份恢复基础渲染函数，并保留新的单位库编辑器代码。
- 加固：显式声明 DOM 引用 `nav/pageTitle/pageDesc/dashboard/docs/history/tools/resources/design`，不再依赖浏览器将 DOM id 自动暴露为全局变量。
- 验收：云端 `curl -u` 访问 `/n2/` 返回 `HTTP/1.1 200 OK`；本地 `node --check` 通过；最小 DOM/localStorage 运行时模拟输出 `runtime ok`。

## 2026-06-19 22:25 简化单位动作 Prompt 表单

- 移除动作编辑界面中的“不含投射物、固定镜头、保持原设计、可循环”四个布尔字段。
- 动作卡片现在只保留动作名、时长秒、Prompt、Negative Prompt、保存和复制 Prompt。
- 保存动作时保留原有 `constraints` 数据，只更新 `durationSec`，避免破坏已保存 JSON 的兼容性。
- 验收：云端 `curl -u` 访问 `/n2/` 返回 `HTTP/1.1 200 OK`；页面 HTML 不再包含 `<label>不含投射物</label>`；本地 `node --check` 与最小运行时模拟通过。

## 2026-06-19 22:37 单位库世界观身份字段调整

- 将单位库默认藤蔓投手从 `category: Enemy` 调整为世界观身份语义：`worldType: 生灵`、`spiritType: 植物型`、`combatSide: 守护方`、`resourceType: 序列帧单位`。
- 单位表单将“类型 / 阵营”改为“世界观身份 / 生灵或污秽类型 / 战斗侧 / 资源身份 / 玩法定位”。
- 保留旧字段 `category/faction` 的兼容写入，避免旧导出 JSON 断裂。
- 增加浏览器 localStorage 迁移逻辑：已保存的 `vine_thrower` 若仍为 `Enemy`，加载时自动补成“生灵 / 植物型 / 守护方”。
- 验收：云端 `/n2/` 返回 `HTTP/1.1 200 OK`；本地 `node --check` 与旧数据迁移运行时模拟通过。

## 2026-06-19 22:53 同步首章污秽单位到控制台

- 在 `/n2/` 控制台“游戏设计 / 单位设计”中新增首章污秽单位默认样板：污泥团、锈蚀爬虫、废壳沉块、毒雾囊。
- 新增 `TAINT_UNIT_PRESETS`，每个单位包含 `worldType: 污秽`、`spiritType: 腐化系`、`combatSide: 威胁方`、`resourceType: 序列帧单位`、玩法说明、美术说明、Negative Prompt、资源路径占位和备注。
- `DEFAULT_UNITS` 现在包含藤蔓投手与四个污秽单位样板。
- `normalizeUnits` 增加增量补齐逻辑：如果浏览器 localStorage 已有旧单位库，只补缺失的污秽单位，不覆盖同 ID 已编辑内容。
- 验收：云端 `/n2/` 返回 `HTTP/1.1 200 OK`；页面包含四个污秽单位；本地 `node --check` 和旧 localStorage 补齐模拟通过，最终单位数为 5。

## 2026-06-19 22:56 补充首章污秽单位 AI 图片 Prompt

- 为控制台单位库中的首章污秽单位补充 AI 图片 Prompt：污泥团、锈蚀爬虫、废壳沉块、毒雾囊。
- Prompt 统一强调：2D 游戏单位概念图、透明或纯色绿幕背景、固定可读 3/4 视角、完整居中、厚描边、适合作为序列帧动画参考图。
- 按单位风格分别强化：非生命污泥、锈铁爬行残骸、废料外壳沉重单位、半透明毒雾囊体。
- `normalizeUnits` 增加空值补齐：如果浏览器 localStorage 中同 ID 污秽单位的 `imagePrompt` 或 `negativePrompt` 为空，则从默认样板补齐；已有非空 prompt 不覆盖。
- 验收：云端 `/n2/` 返回 `HTTP/1.1 200 OK`；页面包含四组 prompt 关键文本；本地 `node --check` 和空 prompt 迁移模拟通过。

## 2026-06-19 23:01 单位 AI 图片 Prompt 改为中文风格模板

- 将控制台单位库的图片 Prompt 改为中文，风格参考“手机游戏单位立绘 / 小体型精灵感 / 扁平卡通涂鸦风 / 粗黑描边 / 墟境配色”。
- 为藤蔓投手补充生灵侧 Prompt：植物型自然灵体、爬山虎藤蔓、小体型精灵感、类似邦布与兰纳罗的圆润符号化造型。
- 为首章四个污秽单位补充同风格中文 Prompt：污泥团、锈蚀爬虫、废壳沉块、毒雾囊；保持手机游戏单位立绘和清晰轮廓，但明确不是生灵、不是人类、不是可爱宠物。
- 迁移逻辑更新：藤蔓投手和污秽单位若浏览器 localStorage 中 `imagePrompt` 为空，会从默认样板补齐；已有非空 prompt 不覆盖。
- 验收：云端 `/n2/` 返回 `HTTP/1.1 200 OK`；页面包含“手机游戏单位立绘”“植物型自然灵体”“腐化系污秽单位”“粗黑描边#2C2E33”；本地 `node --check` 与空 prompt 迁移模拟通过。


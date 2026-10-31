---
title: Atlas 知识生产 / 管理 / 消费全景
created: 2026-10-27
tags:
  - Atlas
  - 项目串讲
  - 知识生产
  - 知识管理
  - 知识消费
---

# Atlas 知识生产 / 管理 / 消费全景

> 用途：回答“现在知识是怎么进来的、怎么管的、给谁用的”
> 代码基线：`atlas` 仓 `master`（2026-10-27）
> 相关：[[项目串讲]]、[[智能模式生成-需求精读与代码地图]]、[[数平平台串讲-BSK-AtlasUlduar-Narya-Lumina]]、[[jev决策模型-学习与调用手册]]

一句话：

```
Keeper / BSK / 人工 / 外部系统
        │
        ▼
   ① 知识生产（3 条流水线 + 3 类导入 + 人工编辑）
        │
        ▼
   Atlas 知识库（tb_atlas_entity / relation / property）
        │
        ├──▶ ② 知识管理（浏览、编辑、复核、发布、冲突、域、本体、审计）
        │
        └──▶ ③ 知识消费（Narya / 对外 API / MCP / 表召回 / 知了文档）
```

---

## 一、知识生产

### 1. 按“表知识生成模式”看：2 个模式 + 1 条独立完整流程

| 模式 | 流程 | 原子数 | 产物 | 落到哪个域 | 状态 |
|---|---|---|---|---|---|
| **标准模式** | `tableLiteGen` | 3 | DT + FIELD + BIND_MEANING | 业务域（未发布） | ✅ 已上线 |
| **智能模式** | 标准模式上扩展 | 未定 | 上述 + 新 MET/DIM/SP + 确认清单 | 业务域 | ❌ 未实现 |
| **完整流程** | `tableKnowledgeGen` | 14 | 粒度、cube、指标维度清单、使用指南、表汇总等全量知识 | **私域** `privately-owned`（BPMN 硬编码） | ✅ 已上线，无业务前端入口 |

> “标准/智能”是产品层面对**轻量流程**的两种模式；完整流程是第三条独立链路，不是第三种“模式”。

完整流程 14 个原子：

```
collectFromKeeper → tableBaseExtract → etlInsight → knowledgeMatch
→ tableDataProfile → explorePlanLlm → enumExplore → grainExplore
→ cubeExplore → usageGuideLlm → fieldEnrichLlm → tableMetricDimLlm
→ tableSummaryLlm → tableArchive
```

### 2. 按“写入通道”看：3 条流水线 + 3 类导入 + 1 类人工

| 通道 | 入口 | 写什么 | 触发方式 |
|---|---|---|---|
| **完整流程** | `POST /atlas/workflow/table-gen/batches`、监控页 | 表知识全量（私域） | 手动 / 接口 |
| **轻量流程** | `POST /atlas/table-gen/lite-batches[/validate]` | 表 + 字段 + 语义绑定 | 认领 / 新增表 |
| **域资源构建** | `POST /atlas/resource/{domain}/add` | 表/文档 → 知识点 | `KnowledgeConstructSchedule` 每 10s 圈批 |
| **统一 ingest（MCP）** | `admin_import_knowledge` / `admin_validate_knowledge` | 任意本体类型 + 关系 | 外部 AI / 脚本调用 |
| **dw-kb 导入** | `DwKbImportController` | 表字段知识批量灌库 | 一次性 / 批量 |
| **Git 导入** | `GitImportController` | 知识库仓库内容 | 手动 |
| **人工编辑** | 知识管理页 `/workshop/data-assets` | 新建 BP/BE/MET/DIM/SP/BR/FAQ、改属性、加关系 | 人工点击保存 |

### 3. 增量更新通道

`TableFieldSyncSchedule`：每天 03:00，用 Keeper 元信息同步官方域**已有表**的字段变化（新增/下线字段、分区标记、类型）。**默认关闭**。

它不产出新表知识，只维护存量字段，但决定了“知识是否新鲜”。

### 4. 标准模式的三个原子（补充）

```
liteCollect → liteMatch → liteArchive
```

- `liteCollect`：Keeper `collect2` 取 `json_context` → DT + FIELD + BELONG_TAB
- `liteMatch`：本域候选（`tb_atlas_entity` MET/DIM）+ 本域历史绑定（`tb_atlas_relation`）→ HISTORY → EXACT → LLM 三级匹配 → BIND_MEANING
- `liteArchive`：适配成 ingest 请求 → 19 条规则校验 → 落库（INACTIVE / SUGGEST_REVIEW）

详解见 [[智能模式生成-需求精读与代码地图]]。

---

## 二、知识管理

### 1. 管知识内容（知识管理页）

入口：域知识管理 → 知识全景 → 知识管理 → `/workshop/data-assets`

| 能力 | 说明 |
|---|---|
| 类型大纲 + 知识卡片 + 属性面板 | 按域浏览 MET/DIM/DT/FIELD/BP/BE/... |
| 新建知识点 | 手动建 BP/BE/MET/DIM/SP/BR/FAQ |
| 编辑属性 | 只改本地，必须点「保存」才落库 |
| 关系管理 | 加/改/删关系、字段绑定语义、标记“无需绑定” |
| 跨域引用 | C-12 允许引用别域原子 |
| 复制知识点 | 派生新知识点 |
| 转移域 | 把知识点挪到另一个域 |
| 字段同步 | 手动触发单表字段增量 |
| 批量删除 / 批量复核 | 批量治理 |

### 2. 管发布状态

| 能力 | 入口 |
|---|---|
| 发布上线 / 回退待复核 | 知识管理页 |
| 公私域同名表状态对齐 | `TableEffectiveAlignSchedule` 每分钟，仅 online；公域 ACTIVE 优先 |
| 表认领 | `GET /tables/{tableName}/claim-status` |
| 详情同名择优 | 官方 ACTIVE 才顶替私域 |

### 3. 管冲突

`ConflictController` + `KnowledgeConflictFacadeService`：

```
DUP（实体重复） → ATTR（属性冲突） → REL（关系冲突）
```

- 人工裁决：合并 / 标记非重复 / 属性取值 / 关系处置
- PRD 规划：**下线冲突管理**，只保留“重复”并挪到二期「域知识体检」

### 4. 管域

`KnowledgeDomainController`：

- 域 CRUD、负责人、描述
- 环境标识 `env_type`（online/pre）
- 私域 `privately-owned` 特殊承载

### 5. 管本体（管理员）

`OntologyController` 系列：

- 实体类型注册
- 属性元数据（propmeta）与值域
- 关系类型与端点矩阵
- 校验规则开关

### 6. 管审计与运维

| 能力 | 入口 |
|---|---|
| 操作审计（WEB/MCP） | `/operation-logs` |
| Workflow 监控与终止 | `/atlas/workflow/**` |
| MCP 调用埋点看板 | `McpToolCallLogAspect` + 日志平台 |
| 表知识清理 | skill `table-knowledge-cleanup` |

### 7. 规划中的管理面

智能模式要新增「智能生成管理」页，3 个 Tab：

1. 生成确认（字段语义 / 表知识只读 / 高频查询模板）
2. 任务进展
3. 域知识体检（二期占位）

---

## 三、知识消费

### 1. 对外 API（给其他系统）

| 接口 | 用途 |
|---|---|
| `GET /atlas/open/knowledge/{domain}/snapshot/lite` | 轻量快照（表 + 字段 + 绑定） |
| `GET /atlas/open/knowledge/{domain}/snapshot/full` | 全量快照 |
| `POST /atlas/open/knowledge/{domain}/query` | 域内点查 |
| `GET /atlas/open/knowledge/{domain}/search` | 域内搜索 |

> 私域隔离是硬约束：对外接口不返回 `privately-owned`（见 issue 023）。

### 2. Narya（查数）

`POST /narya/report/knowledge/query`

- 表点查：表基本信息 + 字段清单 + 字段语义 + 关联知识点
- 字段关系透出：BIND_MEANING / REL_ATOM / FIELD_ENUM
- 消费侧答问题、写 SQL

### 3. cognee / MCP（AI 侧）

| 工具 | 背后接口 |
|---|---|
| `read_table_knowledge` | `GET /atlas/table-knowledge/tables/{tn}/markdown` |
| 表召回 | 表知识 + 向量检索 |
| 知识检索 | `/atlas/open/knowledge/**` |

表知识 Markdown 有 `KeeperTableMetadataFallback`：Atlas 没收录的表，用 Keeper 元信息兜底渲染。

### 4. 表召回链路（内部）

`TableRetrievalMode` / `RecallPipeline`：

```
TableUnderstandStage → TableRecallModuleStage → TableRerankStage
```

规划：召回链路迁 Python，Java 只保留 dept 解析，通过 `HttpRecallStage` 调 Python。

### 5. 人（前端浏览）

表知识中心 `/table-knowledge`：

- 搜索/筛选（官方/非官方、负责人、官方域、排序）
- 表卡片 → 详情页
- 基本信息 / 业务语义映射 / 字段清单 / 使用说明 / 维度组合 / 待确认问题
- **只读**；官方表可“去域内编辑”，非官方表可“认领”

### 6. 转文档推送（推向知了）

`KnowledgeDocPublishSchedule` 每小时整点，把知识点转文档推送到知了；**开关默认关闭**。

---

## 四、三条链路和三种模式的关系

```
                    ┌── 完整流程 tableKnowledgeGen（14 原子）──▶ 私域
知识生产 ───────────┤
                    ├── 轻量流程 tableLiteGen（3 原子）
                    │      ├── 标准模式（现状）──────────────▶ 业务域
                    │      └── 智能模式（规划）──────────────▶ 业务域 + 确认清单
                    └── 域资源构建 knowledgeBuild ───────────▶ 业务域（资源级）
```

最容易混的一点：

- **“标准模式”不是完整流程的简化版**，它俩是两条独立链路，产物差很多。
- 完整流程产粒度、cube、使用指南、指标维度清单；轻量流程只产表 + 字段 + 语义绑定。
- 所以智能模式要的“表知识确认区”，轻量流程本身产不出来，得借完整流程的原子。

---

## 五、明确的边界（现在做不到的）

| 能力 | 状态 |
|---|---|
| 智能模式生成 | ❌ 未实现（前端灰显） |
| 从文档生成知识 | ❌ 二期 |
| 域知识体检 | ❌ 二期 |
| 业务前端“完整流程”入口 | ❌ 没有 |
| 私域表在业务前端编辑 | ❌ 只能看 |
| 资源删除（已成功的表） | ❌ 只能加不能删 |
| 失败任务重试 | ❌ 方案未定（评审挂着） |
| `/datasets`、`/search` | ⚠️ Cognee 遗留页，不属于 Atlas 业务能力 |

---

## 代码依据

- 流程定义：`app/atlas/src/main/resources/processes/tableKnowledgeGen.bpmn20.xml`、`tableLiteGen.bpmn20.xml`、`knowledgeBuild.bpmn20.xml`
- 轻量流程三原子：`workflow/atom/impl/lite/LiteCollectAtom.java`、`LiteMatchAtom.java`、`LiteArchiveAtom.java`
- 字段匹配：`facade/FieldMeaningResolver.java`、`facade/LightweightTableKnowledgeFacadeService.java`
- 写入闸门：`service/impl/KnowledgeIngestServiceImpl.java`、`facade/ingest/rule/BusinessConsistencyRules.java`
- 调度：`schedule/BatchGenScheduleService.java`、`schedule/KnowledgeConstructSchedule.java`
- 消费：`api/http/OpenKnowledgeController.java`、`api/http/TableKnowledgeController.java`、`facade/NaryaReportFacadeService.java`

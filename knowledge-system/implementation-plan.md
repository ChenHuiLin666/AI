# 需求维度知识沉淀系统 — 实现方案

## 一、背景与问题

### 1.1 痛点

每次迭代开发过程中，大量有价值的信息散落在各处：

- **需求文档** → 投喂给 AI 后，提取的要点在一次对话后就丢失了
- **一页纸** → 存在 Confluence 上，但和实际的开发知识是割裂的
- **开发过程** → 发现的技术决策、隐藏逻辑、模块间隐式耦合，只在开发者脑子里
- **Debug 过程** → Bug 的根因分析、暴露出的隐藏逻辑，只在 Jira 评论区里
- **需求变更** → 为什么改、影响了什么，没有系统记录

到写迭代总结时，全凭回忆。下次改同一个模块时，以前踩过的坑想不起来。

### 1.2 目标

建立一个**以需求为主维度的知识沉淀系统**，覆盖完整开发流程：

```
需求文档投喂 → 一页纸生成 → 需求变更 → 开发 → Debug → 迭代总结
     │              │           │         │       │         │
     └──────────────┴───────────┴─────────┴───────┴─────────┘
                              │
                              ▼
                    需求维度知识库（持续积累）
```

### 1.3 核心设计原则

1. **从需求文档投喂就开始介入**，不是从一页纸生成之后
2. **增量更新**，不是全量重建。需求变更时保留历史
3. **以需求 ID 和模块名**作为主索引维度
4. **低摩擦捕获**：一句话就能记录，不用填表单
5. **在正确的时间点激活**：改代码前自动提示注意事项

---

## 二、系统架构

### 2.1 整体架构

```
┌─────────────────────────────────────────────────────┐
│                    5 个 Skills                        │
│  req-extract / one-pager-generate / req-sync /        │
│  insight-capture / impact-check                       │
└────────────────────┬────────────────────────────────┘
                     │ 调用
                     ▼
┌─────────────────────────────────────────────────────┐
│              knowledge-store MCP Server              │
│          10 个工具 / 文件系统读写 / frontmatter 解析   │
└────────────────────┬────────────────────────────────┘
                     │ 读写 .md 文件
                     ▼
┌─────────────────────────────────────────────────────┐
│  knowledge-store/   (Markdown 文件，按迭代分目录)      │
│  ├── requirements/  (需求条目 .md)                    │
│  ├── insights/      (知识条目 .md，含 frontmatter)     │
│  └── links.json     (关联关系索引)                     │
└────────────────────┬────────────────────────────────┘
                     │ AI 可直接读取
                     ▼
┌─────────────────────────────────────────────────────┐
│           知识文件本身就是语料                          │
│   不依赖 MCP，AI 可直接把 .md 文件作为 prompt 输入      │
└─────────────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│          已有基础设施（复用）                          │
│  Jira MCP / Confluence MCP / iteration-summary-creator│
└─────────────────────────────────────────────────────┘
```

**为什么选 Markdown 而不是数据库**：最终目的是让 AI 读懂知识图谱。.md 文件本身就是语料，可以直接喂给 AI，不依赖 MCP Server 是否启动。MCP Server 的角色是便利层（批量查、格式化输出、自动补全关联），但即使没有 MCP，知识文件仍然可读可用。

### 2.2 数据模型

知识库以 **Markdown 文件 + 轻量索引** 方式组织，核心目录结构：

```
knowledge-store/                         ← 项目根目录下
├── requirements/                        ← 需求条目
│   └── 2026-07-30/                       ← 按迭代分目录
│       ├── index.md                     ← 迭代需求索引
│       ├── req-001-审批流程.md
│       ├── req-002-计划列表.md
│       └── req-003-数据报表.md
│
├── insights/                            ← 知识条目（核心资产）
│   └── 2026-07-30/
│       ├── ins-001-审批驳回状态隐式逻辑.md
│       ├── ins-002-审批状态枚举影响通知模块.md
│       ├── ins-003-Bug-PROJ-123-竞态条件.md
│       └── ...
│
└── links.json                           ← 关联关系索引
```

**每篇需求文件的结构：**
```markdown
---
id: req-001
iteration_id: 2026-07-30
title: 审批流程
module: 审批流
description: 实现审批流的发起、审批、驳回流程
hours: 16
status: active
parent_id: null
source_page_id: "123456789"
created_at: 2026-07-29
updated_at: 2026-07-29
---

# 审批流程

## 功能描述
...（从需求文档提取的原文摘要）...

## 原始需求文本
...（保留原文片段，便于追溯）...

## 子需求
- [[req-004]] 审批发起
- [[req-005]] 审批驳回
```

**每篇 insight 文件的结构（核心）：**
```markdown
---
id: ins-001
type: hidden_logic
severity: critical
requirement_id: req-001
requirement_title: 审批流程
module: 审批流
source_type: bug_fix
source_ref: PROJ-123
affected_modules:
  - 通知模块
affected_files:
  - src/components/ApprovalFlow/index.tsx
  - src/utils/approvalStatus.ts
tags:
  - 状态机
  - 审批
created_at: 2026-07-29
cross_project_refs:        ← 跨项目引用（可选）
  - project: "通知服务"
    module: "消息模板"
    note: "审批状态枚举值和消息模板的文案映射是跨项目的数据契约"
---

# 审批驳回后状态隐式逻辑

## 现象
审批驳回后，状态不是回到"草稿"而是回到"待提交"。

## 触发条件
任何审批驳回操作都会触发。

## 原因
后端状态机定义如此，驳回 = 回到上一个"待提交"节点，不是回到初始状态。
前端没有显式文档说明这个行为。

## 涉及文件
- src/components/ApprovalFlow/index.tsx
- src/utils/approvalStatus.ts

## 关联知识
- [[ins-002]] 状态枚举变更影响通知模块
```

**links.json 结构（轻量索引，MCP Server 维护）：**
```json
{
  "links": [
    {
      "source": "ins-001",
      "target": "ins-002",
      "link_type": "relates_to",
      "description": "同一个状态机相关的问题"
    }
  ]
}
```

### 2.3 Markdown 文件即知识资产

**核心设计理念：知识文件 = AI 可直接消费的语料。**

```
对比：

SQLite 方案:
  AI → MCP tool → SQL → JSON → AI 理解
  中间必须经过 MCP，脱离 MCP 知识就无法被 AI 读取

Markdown 方案:
  AI → Read("insights/2026-07-30/ins-001-xxx.md") → AI 直接理解
  即使没有 MCP Server，知识文件仍然可读、可用、可投喂
  
  或者（批量场景）：
  AI → MCP tool(query_by_module) → 解析 frontmatter → 结构化结果 → AI 理解
  MCP 提供便利，但不是必需品
```

**实际价值**：

| 场景 | Markdown 方案能做到的 |
|------|----------------------|
| 改代码前查注意事项 | AI 直接 Read 该模块相关的 insight .md 文件，frontmatter 和正文一起消费 |
| 迭代总结 | AI 批量 Read 本迭代 insight 目录，所有知识作为 prompt 上下文 |
| 投喂给其他 AI 工具 | 直接复制 .md 文件内容，不需要迁移数据库 |
| 人工翻阅 | 直接用编辑器打开看，目录结构清晰 |
| Git 追踪 | 每次知识变更都有 diff，可 review |

### 2.4 insight 类型定义

| type | 含义 | 触发场景 |
|------|------|---------|
| `key_point` | 关键知识点 | 需求要点澄清、业务规则、边界条件、交互细节 |
| `hidden_logic` | 隐藏逻辑 | 不显式声明但实际生效的规则、隐式耦合、默认行为 |
| `change_impact` | 改动影响点 | 改 A 模块会影响 B 模块、数据契约依赖、状态联动 |
| `bug_lesson` | Bug 教训 | Bug 根因分析、暴露出的隐藏逻辑、如何预防 |
| `decision` | 技术决策 | 方案选型、架构决策、为什么这样做 |
| `note` | 一般备注 | 变更记录、历史快照、备忘 |

### 2.5 严重度定义

| severity | 含义 |
|----------|------|
| `critical` | 涉及数据安全、资金、不可逆操作、核心流程阻塞 |
| `important` | 可能导致 Bug、影响其他模块、容易遗漏 |
| `note` | 一般性记录、备忘 |

---

## 三、MCP Server：knowledge-store

### 3.1 技术选型

- **语言**：TypeScript（Node.js）
- **存储**：Markdown 文件（frontmatter + 正文） + links.json 索引
- **为什么是文件而非数据库**：最终目的是让 AI 读懂知识图谱。.md 文件本身就是语料——可以直接被 Read 工具读取、可以直接作为 prompt 上下文喂给 AI。MCP Server 的角色从"存储引擎"变为"便利层"（写入模板化文件、批量解析 frontmatter、维护 links.json 索引）。即使没有 MCP Server，知识文件仍然是可读可用的。
- **SDK**：`@modelcontextprotocol/sdk` + `zod`（参数校验），`gray-matter`（解析 frontmatter）
- **位置**：`mcpServers/knowledge-store/src/index.ts`

### 3.2 工具清单（10 个）

MCP Server 不再操作 SQL，改为**生成 .md 文件 + 解析 frontmatter + 维护 links.json**。工具接口（参数/返回值）不变，底层实现从 SQL 变为文件读写。

#### 工具 1：`init_requirement`

```
功能：创建需求 .md 文件（按模板生成，含 frontmatter）
实现：在 requirements/{iteration_id}/ 目录下创建 req-xxx-{标题}.md
```

```
功能：创建需求条目

参数：
  iteration_id   string    迭代标识，即组件上线日期，如 "2026-07-30"
  title          string    需求标题
  module         string    所属模块名
  description    string?   需求描述（尽量用原文措辞）
  hours          number?   预估工时
  parent_id      string?   父需求 ID（用于层级结构）
  source_text    string?   原始需求文本片段（保留原文，便于追溯）

返回：{ requirement_id, title, module }
```

#### 工具 2：`update_requirement`

```
功能：更新需求 .md 文件的 frontmatter 字段（增量更新）
实现：解析 frontmatter → 更新指定字段 → 写回文件

参数：
  requirement_id  string    需求 ID
  title           string?   新标题
  module          string?   新模块归属
  description     string?   新描述
  hours           number?   新工时
  status          string?   状态: active | changed | deprecated | done
  change_note     string?   变更说明（什么变了 + 为什么变）

返回：{ requirement_id, updated_fields: [...], change_recorded: true }
```

#### 工具 3：`add_insight` ⭐ 最核心、最高频

```
功能：生成一篇 insight .md 文件（按模板生成，含完整 frontmatter）
实现：在 insights/{iteration_id}/ 目录下创建 ins-xxx-{标题}.md
      frontmatter 存结构化字段，正文存详细描述
      .md 文件本身 = AI 可直接消费的知识语料

参数：
  requirement_id         string     关联的需求 ID
  type                   string     类型: key_point | hidden_logic | change_impact | bug_lesson | decision | note
  severity               string     严重度: critical | important | note
  title                  string     一句话标题
  content                string     详细内容
  source_type            string?    来源: requirement_doc | one_pager | development | code_review | bug_fix | manual
  source_ref             string?    来源引用: Jira Key / commit hash / Confluence page ID
  affected_modules       []string?  影响范围-模块列表
  affected_files         []string?  影响范围-文件路径列表
  tags                   []string?  标签
  cross_project_refs     []object?  跨项目引用 [{project, module, note}]
  iteration_id           string?    迭代标识（默认取当前迭代）

返回：{ insight_id, title, type }
```

#### 工具 4：`query_by_requirement`

```
功能：查询某个需求下的全部知识
实现：遍历 insights/{iteration_id}/ 目录 → 解析 frontmatter → 
      筛选 requirement_id 匹配的文件 → 返回结构化列表
```

参数：
  requirement_id   string    需求 ID
  type_filter      string?   类型过滤
  severity_filter  string?   严重度过滤

返回：{
  requirement: { id, title, module, description, status, hours },
  insights: [{ id, type, severity, title, content, source_type, source_ref, tags, created_at }],
  stats: { total, by_type: {...}, by_severity: {...} }
}
```

#### 工具 5：`query_by_module`

```
功能：查询某个模块关联的全部知识（跨需求）
实现：遍历 insights/ 目录 → 解析 frontmatter → 
      筛选 module 或 affected_modules 匹配的文件
```

参数：
  module_name   string    模块名
  iteration_id  string?   限定迭代范围

返回：{
  module: string,
  related_requirements: [{ id, title }],
  insights: [...],
  hidden_logic_count: number,
  impact_count: number,
  bug_count: number
}
```

#### 工具 6：`get_prerequisite` ⭐ 改代码前必读

```
功能：返回"改这个模块前你必须知道的事"

参数：
  module_name     string   目标模块名
  requirement_id  string?  关联需求 ID

返回：{
  module: string,
  checklist: {
    hidden_logic: [
      { title, content, severity, source_ref, created_at }
    ],
    change_impacts: [
      { title, content, affected_modules, coupling_type }
    ],
    past_bugs: [
      { title, content, jira_key, root_cause }
    ],
    key_decisions: [
      { title, content }
    ]
  },
  cross_project_refs: [
    { project, module, note }
  ],
  risk_summary: "一句话风险概述",
  related_modules_to_check: ["模块A", "模块B"]
}
```

#### 工具 7：`search_insights`

```
功能：全文搜索所有知识（关键词 grep + frontmatter 解析）
实现：Python glob 遍历 .md 文件 → 标题/正文关键词匹配 → 
      解析 frontmatter → 返回匹配列表
```

参数：
  keyword        string   搜索关键词
  type_filter    string?  类型过滤
  module_filter  string?  模块过滤
  limit          number   返回条数，默认 20

返回：[{ insight_id, type, title, content_snippet, requirement_title, module }]
```

#### 工具 8：`link_insights`

```
功能：在两个知识条目之间建立关联关系
实现：更新 links.json 追加关联记录 + 
      在源 insight .md 文件末尾追加 "## 关联知识\n- [[target_id]] target_title"
```

参数：
  source_id    string   源 insight ID
  target_id    string   目标 insight ID
  link_type    string   关系类型: relates_to | impacts | caused_by | depends_on | contradicts
  description  string?  关系说明

返回：{ linked: true, source_id, target_id, link_type }
```

#### 工具 9：`get_requirement_tree`

```
功能：获取某迭代的完整需求树（含每个需求下的知识统计）

参数：
  iteration_id  string   迭代标识

返回：{
  iteration_id: string,
  requirements: [
    {
      id, title, module, hours, status, parent_id,
      children: [...],
      insight_summary: {
        total, key_points, hidden_logic, change_impacts, bug_lessons, decisions, notes
      }
    }
  ]
}
```

#### 工具 10：`export_iteration_knowledge`

```
功能：导出某迭代全部知识为结构化 Markdown

参数：
  iteration_id  string   迭代标识
  group_by      string   分组方式: requirement | module，默认 requirement

返回：{ markdown_content: "..." }
```

---

## 四、Skill 设计（5 个）

### 4.1 Skill 全景

```
需求文档（你投喂）
    │
    ▼
┌──────────────┐
│ req-extract  │  提取需求要点 → 初始化需求树 → 关键点入库
└──────┬───────┘
       │
       ▼
┌────────────────────┐
│ one-pager-generate │  生成一页纸 → Story 拆分 → 回写需求树 → 写入 Confluence
└──────┬─────────────┘
       │
       ├── 需求调整时 ──▶ ┌──────────┐
       │                  │ req-sync │  增量更新 + 变更追踪 + 一页纸同步
       │                  └──────────┘
       │
       ├── 开发/Debug中 ──▶ ┌──────────────────┐
       │                    │ insight-capture  │  隐藏逻辑 / 影响点 / Bug 教训 / 决策
       │                    └──────────────────┘
       │
       ├── 改代码前 ──▶ ┌──────────────┐
       │                │ impact-check │  查询风险 + 注意事项清单
       │                └──────────────┘
       │
       ▼
┌──────────────────────────────┐
│ iteration-summary-creator    │  聚合知识库 → 补充迭代总结（已有 Skill）
└──────────────────────────────┘
```

---

### 4.2 Skill 1：`req-extract` — 需求文档投喂与知识提取

**触发时机**：你把需求文档（PRD、需求说明、会议纪要、任何形式）投喂给 AI 时

**角色定位**：需求分析专家，从非结构化文档中提取结构化信息

**核心原则**：
- 不丢信息：宁可多提取，不要遗漏
- 保留原文：关键描述保留原始措辞
- 标记不确定性：不确定的地方标为 `[待确认]`
- 按模块归类

**工作流：**

```
阶段 1：接收与理解
├── 确认迭代标识（若未提供则先问用户）
├── 确认是否有多个需求文档/来源
└── 确认涉及的产品/项目范围

阶段 2：结构化提取（四类信息）
├── A. 功能点：名称、模块、描述（原文措辞）、用户场景、优先级
├── B. 业务规则：规则描述、触发条件、关联功能点、是否隐性
├── C. 约束与依赖：技术约束、外部依赖、时间约束、数据约束
└── D. 待确认点：模糊/矛盾/歧义/遗漏的场景

阶段 3：初始化知识库
├── 对每个功能点 → init_requirement 创建需求条目
├── 对每条业务规则 → add_insight(type=key_point, severity=important)
├── 对每项约束依赖 → add_insight(type=key_point, severity=critical)
└── 对每个待确认点 → add_insight(type=note, severity=note, tags=["待确认"])

阶段 4：输出提取报告（Markdown 格式）
├── 📊 概览：功能点数 / 业务规则数 / 约束依赖数 / 待确认数
├── 🌲 需求树：模块 | 需求标题 | 工时 | 状态
├── ⚠️ 关键约束：按严重度排序
├── ❓ 待确认：逐条列出
└── 📝 已入库统计
```

---

### 4.3 Skill 2：`one-pager-generate` — 生成一页纸 + 知识库同步

**触发时机**：需求提取确认完毕，生成一页纸

**前置条件**：
- `req-extract` 已执行，需求树已初始化
- 用户已确认需求树无误
- 用户已指定 Confluence 一页纸父页面

**核心原则**：
- 一页纸内容源头是知识库中的需求条目，保持一致性
- Story 拆分必须回写到知识库
- 模块划分必须和需求树对齐
- 一页纸是技术方案化，不是重写需求

**工作流：**

```
阶段 1：加载需求上下文
└── 调用 get_requirement_tree 获取完整需求树，确认准确性

阶段 2：规划 Story 拆分
├── 原则：一个 Story = 一个可独立交付的功能单元
├── 每个 Story 定义：标题、页面范围、核心功能点、工时、技术要点
└── 每个 Story 对应一个或多个需求条目

阶段 3：生成一页纸内容
├── 迭代信息（标识、时间范围、总工时）
├── Story 列表（标题、工时、页面范围、关键功能点、技术备注）
├── 模块划分与页面映射
├── 技术方案要点
├── 风险与依赖
└── 待确认项

阶段 4：回写知识库
├── Story 拆分为子需求 → init_requirement(parent_id=原需求)
├── 工时 → update_requirement
└── 技术要点 → add_insight(type=key_point)

阶段 5：写入 Confluence
├── 在用户指定父页面下创建一页纸
├── 记录页面 ID → update_requirement(source_page_id)
└── 生成确认报告
```

---

### 4.4 Skill 3：`req-sync` — 需求变更增量同步

**触发时机**：开发过程中需求有调整（新增、修改、废弃）

**核心原则**：
- **增量更新**，不是全量重建
- **保留历史**，变更前的旧信息用 insight 快照保留
- **追踪原因**，每次变更记录"为什么改"
- **检查连锁影响**，变更后检查相关需求是否需要同步

**三种变更类型的处理：**

```
类型 A：需求新增
├── 确认所属模块
├── init_requirement 创建新条目
├── add_insight(type=note, tags=["变更记录"]) 记录新增事件
└── 如需要 → 更新一页纸（追加新 Story）

类型 B：需求修改
├── query_by_requirement 查看当前信息
├── add_insight(type=note, tags=["变更记录","历史快照"]) 保留旧值快照
├── update_requirement(change_note="变更原因") 更新
├── add_insight(type=note, tags=["变更记录"]) 记录变更事件
└── 检查是否有 insight 需要重新审视

类型 C：需求废弃
├── update_requirement(status="deprecated")
├── add_insight(type=note, tags=["变更记录","废弃"]) 记录废弃原因
├── 不删除已有 insight（历史知识仍有参考价值）
└── 一页纸中标注该 Story 已废弃

类型 D：连锁影响检查（自动）
├── query_by_module 查找同模块其他需求
├── 检查 insight_links 中被变更需求引用的其他条目
└── 如有潜在影响 → 提醒用户
```

**输出格式：**

```markdown
## 🔄 需求变更记录

### 变更摘要
- 变更类型: 新增/修改/废弃
- 影响需求: {标题}
- 变更原因: {原因}

### 知识库操作
- [x] 需求条目已更新
- [x] 变更历史已记录
- [x] 旧值快照已保留

### 一页纸同步
- [x] / [ ] 一页纸已同步更新

### ⚠️ 连锁影响
- 相关需求可能需要检查: [...]
```

---

### 4.5 Skill 4：`insight-capture` — 开发/Debug 知识捕获 ⭐ 最高频

**触发时机**：开发过程中发现了值得记录的东西（一句话触发）

**触发语**：
- "记录一下，..."
- "记个知识点：..."
- "这个 Bug 的教训是..."
- "发现一个隐藏逻辑：..."
- "备忘：..."

**自动类型识别规则：**

| 识别为 | 信号词 | 示例 |
|--------|--------|------|
| `hidden_logic` | 隐式、默认、自动、没写在文档里、坑 | "审批驳回后状态回到待提交而非草稿" |
| `change_impact` | 改了...导致、影响、联动、别忘了 | "改了审批状态枚举，通知模块文案映射也要同步" |
| `bug_lesson` | Bug、根因、复现、漏了 | "Bug PROJ-123 根因是竞态条件" |
| `decision` | 选型、决定、用...方案、最终采用 | "状态管理用 useState，场景不复杂" |
| `key_point` | （默认类型） | 边界条件、交互细节、业务规则 |

**工作流：**

```
步骤 1：解析用户输入
├── 提取核心内容
├── 推断类型
├── 推断严重度
└── 推断关联的模块/需求

步骤 2：确认关联关系（仅在无法自动确定时追问）
├── "关联到哪个需求？"
└── "涉及哪个模块？"

步骤 3：丰富内容（仅在关键信息缺失时追问）
├── hidden_logic → 触发条件？为什么被隐藏？涉及哪些文件？
├── change_impact → 影响了哪些模块？耦合类型？
└── bug_lesson → Jira Key？根因？如何预防？

步骤 4：存入知识库
└── add_insight(...)

步骤 5：检查关联
└── 若与已有 insight 存在关联 → link_insights

步骤 6：简短确认
└── ✅ 已记录 | 🔒隐藏逻辑 | 关联 [需求X] | 模块 [审批流]
```

**特别场景：修完 Bug 后的批量记录**

```
用户: "记录一下 PROJ-456 这个 Bug"

执行：
1. 调用 Jira MCP 获取 Bug 详情
2. 分析 Bug 根因
3. 若涉及隐藏逻辑 → 同时创建 hidden_logic 条目
4. 若修复涉及多个文件 → 创建 change_impact 条目
5. 创建 bug_lesson 条目
6. link_insights 连接 bug_lesson → hidden_logic → change_impact
7. 关联到对应需求
```

---

### 4.6 Skill 5：`impact-check` — 改代码前风险检查

**触发时机**：要改某个模块/功能前，先了解有什么坑

**触发语**：
- "我要改 XXX 模块，有什么要注意的？"
- "改 XXX 之前帮我查一下"
- "check 一下审批流模块"

**工作流：**

```
步骤 1：确定查询范围
├── 提取目标模块名/文件路径/需求 ID
└── 若模糊 → search_insights 先搜索匹配

步骤 2：多维度查询（并行）
├── get_prerequisite(module_name) → 模块级注意事项
├── query_by_requirement(requirement_id) → 需求级全部知识
└── search_insights(keyword, module_filter) → 补充搜索

步骤 3：风险分级输出
├── 🔴 高优先级：隐藏逻辑 + 关键影响链
├── 🟡 中优先级：历史 Bug + 技术决策
├── 🟢 参考信息：关联需求 + 相关文件
└── 📋 建议检查清单

步骤 4：补充建议
└── 根据 insight_links 推荐同时检查的模块
```

**输出模板：**

```markdown
## 🔍 改动前检查：[模块名]

### 🔴 高优先级（必须知道）
| 隐藏逻辑 | 触发条件 | 发现来源 |
|---------|---------|---------|
| ... | ... | ... |

| 改动 | 会影响 | 耦合类型 |
|------|--------|---------|
| ... | ... | ... |

### 🟡 中优先级（建议了解）
| Bug | 根因 | 教训 |
|-----|------|------|
| ... | ... | ... |

### 🟢 参考信息
- 关联需求: [...]
- 相关文件: [...]

### 📋 建议检查清单
- [ ] ...
- [ ] ...

### 🔗 建议同时检查
- **通知模块**：审批流改动历史上有 X 次影响了通知模块

### 🌐 跨项目影响
| 项目 | 模块 | 说明 |
|------|------|------|
| 通知服务 | 消息模板 | 审批状态枚举值和消息模板文案映射是跨项目数据契约 |
```

---

## 五、完整工作流串联（一个迭代的视角）

```
📥 第 1 步：你投喂需求文档
   "这是本迭代的需求文档 [文档内容]"
   → req-extract Skill
   → 提取 12 个功能点、5 个业务约束、3 个待确认点
   → 初始化需求树，关键点入库
   → 输出提取报告

📝 第 2 步：生成一页纸
   "基于刚才的需求，帮我生成一页纸"
   → one-pager-generate Skill
   → 加载需求树 → Story 拆分 → 回写知识库
   → 生成一页纸 → 写入 Confluence
   → 输出确认报告

🔄 第 3 步：需求调整（可能发生多次）
   "需求有调整，审批流程加了驳回后重新提交的场景"
   → req-sync Skill
   → 增量更新审批模块需求条目
   → 变更历史记录，旧值快照保留
   → 如需要，同步更新一页纸

💻 第 4 步：开发过程中（持续、高频）
   "记录一下，审批状态机有个隐藏逻辑：驳回后状态不是回到草稿而是回到待提交"
   → insight-capture Skill
   → 自动识别为 hidden_logic
   → 关联需求"审批流程"、模块"状态机"
   → ✅ 已记录

🐛 第 5 步：Debug（持续）
   修完 Bug PROJ-456
   "记录这个 Bug"
   → insight-capture Skill
   → 获取 Bug 详情 → 分析根因
   → 存为 bug_lesson + hidden_logic + change_impact
   → 建立关联链接
   → ✅ 已记录

🔍 第 6 步：改代码前（按需）
   "我要改状态机，有什么要注意的？"
   → impact-check Skill
   → 返回该模块的隐藏逻辑 + 影响链 + 历史Bug + 检查清单

📊 第 7 步：迭代总结
   "帮我写迭代总结"
   → iteration-summary-creator Skill（已有）
   → 内部拉取知识库中本迭代所有知识
   → 补充到复盘的"收获"和"注意事项"
```

---

## 六、Skill 之间的数据流关系

```
req-extract
  │  创建需求条目 (init_requirement)
  │  提取关键点   (add_insight: key_point)
  │
  ▼
one-pager-generate
  │  创建子需求   (init_requirement, parent_id)
  │  更新工时     (update_requirement)
  │  记录技术要点 (add_insight: key_point)
  │
  ├── req-sync ──────────────────────────────
  │    更新需求     (update_requirement)
  │    变更快照     (add_insight: note, tags=["变更记录"])
  │    变更事件     (add_insight: note, tags=["变更记录"])
  │
  ├── insight-capture ───────────────────────
  │    隐藏逻辑     (add_insight: hidden_logic)
  │    改动影响     (add_insight: change_impact)
  │    Bug 教训     (add_insight: bug_lesson)
  │    技术决策     (add_insight: decision)
  │    建立关联     (link_insights)
  │
  ├── impact-check ──────────────────────────
  │    (只读) 查询      (get_prerequisite, query_by_module, search_insights)
  │
  ▼
iteration-summary-creator (已有 Skill)
    聚合数据     (export_iteration_knowledge, get_requirement_tree)
```

---

## 七、配置方式

### 7.1 目录结构

```
personnel/AI/                           ← 以你当前项目为例
├── skills/                             ← 已有 + 新增的 5 个 Skill（复用现有目录）
│   ├── dev-summary-creator/            ← 已有
│   ├── bug-fixer                       ← 已有
│   ├── req-extract.md                  ← 新增
│   ├── one-pager-generate.md           ← 新增
│   ├── req-sync.md                     ← 新增
│   ├── insight-capture.md              ← 新增
│   └── impact-check.md                 ← 新增
│
├── mcpServers/
│   └── knowledge-store/
│       ├── src/
│       │   └── index.ts               ← MCP Server 主程序
│       ├── package.json
│       └── tsconfig.json
│
├── knowledge-system/                   ← 设计文档
│   └── implementation-plan.md
│
└── knowledge-store/                    ← 知识数据（每个项目独立一份）
    ├── requirements/
    │   └── 2026-07-30/                  ← 按迭代分目录
    │       ├── index.md
    │       └── req-001-xxx.md
    ├── insights/
    │   └── 2026-07-30/
    │       └── ins-001-xxx.md
    └── links.json

关键点：
- MCP Server 代码（mcpServers/）一份即可，配在用户级 ~/.claude/settings.json
- 知识数据（knowledge-store/）每个项目各一份，MCP Server 从当前工作目录自动发现 ./knowledge-store/
- 跨项目的模块耦合通过 insight frontmatter 的 cross_project_refs 字段记录
```

### 7.2 MCP 配置

**推荐：用户级全局配置**（所有项目都能用 `knowledge-store`）

```json
// ~/.claude/settings.json
{
  "mcpServers": {
    "knowledge-store": {
      "command": "node",
      "args": ["/Users/chenhuilin/personnel/AI/mcpServers/knowledge-store/dist/index.js"]
    }
  }
}
```

注意：
- `args` 必须用**绝对路径**指向编译后的 dist/index.js（用户级配置不知道当前在哪个项目）
- MCP Server 启动后，从当前 Claude Code 的工作目录自动发现 `./knowledge-store/` 数据目录
- 换项目时 MCP 不用重新配，但知识数据是各项目独立的

**如果只想在当前项目启用**（项目级配置）：

```json
// 项目根目录/settings.local.json 或 settings.json
{
  "mcpServers": {
    "knowledge-store": {
      "command": "node",
      "args": ["mcpServers/knowledge-store/dist/index.js"]
    }
  }
}
```

路径可用相对路径，但只对当前项目生效。

### 7.3 依赖

```json
// mcpServers/knowledge-store/package.json
{
  "name": "knowledge-store-mcp",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "build": "tsc",
    "watch": "tsc --watch"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.0",
    "gray-matter": "^4.0.0",
    "zod": "^3.0.0"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "@types/node": "^20.0.0"
  }
}
```

- `@modelcontextprotocol/sdk` — MCP Server 框架
- `zod` — 工具参数校验
- `gray-matter` — Markdown frontmatter 解析/写入

---

## 八、实施路线图

| 优先级 | 组件 | 原因 | 预估工作量 |
|--------|------|------|-----------|
| **P0** | `knowledge-store` MCP Server | 所有 Skill 的底层依赖，没有它一切跑不起来 | ~400 行 TypeScript |
| **P0** | `insight-capture` Skill | 最高频使用，当天就能开始积累知识 | ~150 行 Markdown |
| **P1** | `impact-check` Skill | 依赖 insight-capture 积累的数据，越早用价值越大 | ~130 行 Markdown |
| **P1** | `req-extract` Skill | 下个迭代启动就能用 | ~140 行 Markdown |
| **P2** | `one-pager-generate` Skill | 依赖 req-extract 的输出 | ~150 行 Markdown |
| **P2** | `req-sync` Skill | 需求变更时才触发 | ~150 行 Markdown |

### 建议实施顺序

```
第 1 步：MCP Server 开发 + 测试
  → 先让文件读写 + frontmatter 解析 + 10 个工具跑通
  → 手动测试每个工具

第 2 步：insight-capture Skill
  → 立刻就能用，开始积累数据
  → 用几天，感受一下分类和关联是否合理

第 3 步：impact-check Skill
  → 有了一些数据后，这个 Skill 才有意义
  → 验证查询结果是否实用

第 4 步：req-extract + one-pager-generate
  → 下个迭代启动时使用
  → 验证需求提取和一页纸生成流程

第 5 步：req-sync
  → 当第一次需求变更发生时自然用到
  → 验证增量更新和变更追踪

第 6 步：持续迭代优化
  → 根据实际使用反馈调整 Skill 指令
  → 优化 MCP 工具的参数和返回格式
```

---

## 九、与已有系统的关系

### 9.1 与 iteration-summary-creator 的关系

**不是替代，是上游数据源：**

```
之前：用户手动回忆 → 写总结
之后：日常持续记录 → knowledge-store → 自动聚合 → 喂给总结 Skill
```

已有的 `iteration-summary-creator` 继续负责"总结的写作风格和格式"，
新系统负责"提供更丰富的素材"——在总结流程中增加一步：
调用 `export_iteration_knowledge` 拉取本迭代全部知识，补充到复盘部分。

### 9.2 与 Jira / Confluence MCP 的关系

**复用，不是重建：**

- `insight-capture` 在记录 Bug 时会调用 Jira MCP 获取 Bug 详情
- `one-pager-generate` 会调用 Confluence MCP 写入一页纸
- `req-extract` 如果需求文档在 Confluence 上，会调用 Confluence MCP 读取

新系统只负责"知识的结构化存储和查询"，不重复造轮子。

---

## 十、关键设计决策记录

| 决策 | 选择 | 原因 |
|------|------|------|
| 存储方案 | Markdown 文件 + frontmatter | 最终目的是让 AI 读懂知识图谱。.md 文件本身就是语料，可被 AI 直接 Read，可跨工具复用。MCP Server 是便利层而非必需品 |
| MCP 语言 | TypeScript (Node.js) | 使用 @modelcontextprotocol/sdk + zod + gray-matter |
| insight 类型 | 6 种 | 覆盖核心场景，不过度设计 |
| 更新策略 | 增量 + 快照 | 需求变更时旧值存为 insight（type=note, tags=["历史快照"]），不物理删除 |
| 需求废弃处理 | 标记 deprecated，不删 insight | 隐藏逻辑和 Bug 教训在需求废弃后仍有参考价值 |
| Skill 触发方式 | 显式调用为主 | 避免 AI 误判触发时机 |
| 目录组织 | 按迭代分目录 | 每个迭代有独立的需求目录和知识目录，天然隔离，方便归档 |
| MCP 配置层级 | 用户级 ~/.claude/settings.json | 一份 MCP 代码所有项目共用，知识数据跟随项目独立 |
| 跨项目耦合 | insight 增加 cross_project_refs 字段 | 记录跨项目模块依赖，impact-check 时一并提示 |

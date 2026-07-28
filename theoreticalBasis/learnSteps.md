# AI 开发学习路线：从零到实战

## 目录

- [一、学习路线全景图](#一学习路线全景图)
- [二、第一阶段：AI 基础认知](#二第一阶段ai-基础认知)
- [三、第二阶段：提示词工程](#三第二阶段提示词工程)
- [四、第三阶段：MCP 协议](#四第三阶段mcp-协议)
- [五、第四阶段：Skills 技能系统](#五第四阶段skills-技能系统)
- [六、第五阶段：Agent 智能体开发](#六第五阶段agent-智能体开发)
- [七、第六阶段：Workflow 工作流编排](#七第六阶段workflow-工作流编排)
- [八、第七阶段：Memory 与持久化](#八第七阶段memory-与持久化)
- [九、第八阶段：Hooks 与自动化](#九第八阶段hooks-与自动化)
- [十、第九阶段：实战项目](#十第九阶段实战项目)
- [十一、工具与资源汇总](#十一工具与资源汇总)

---

## 一、学习路线全景图

```
                         AI 开发学习路线

  ┌─────────┐    ┌──────────┐    ┌──────────┐    ┌──────────────┐
  │ 1.基础   │───▶│ 2.提示词  │───▶│ 3. MCP   │───▶│ 4. Skills    │
  │ AI 认知  │    │ Prompt   │    │ 协议原理  │    │ 技能系统     │
  └─────────┘    └──────────┘    └──────────┘    └──────────────┘
                                                        │
       ┌────────────────────────────────────────────────┤
       ▼                ▼                ▼              ▼
  ┌─────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
  │5. Agent │    │6.Workflow│    │7. Memory │    │8. Hooks  │
  │ 智能体  │    │ 工作流   │    │ 持久记忆 │    │ 自动化   │
  └─────────┘    └──────────┘    └──────────┘    └──────────┘
       │                │               │              │
       └────────────────┴───────────────┴──────────────┘
                              │
                              ▼
                     ┌────────────────┐
                     │ 9. 实战项目     │
                     │ 融会贯通       │
                     └────────────────┘
```

### 各阶段核心能力

| 阶段 | 核心能力 | 学习周期 | 产出物 |
|------|---------|---------|--------|
| 1. AI 基础认知 | 理解 LLM 原理、token、上下文窗口 | 1-2 天 | 概念笔记 |
| 2. 提示词工程 | 写出高质量的 prompt、结构化输出 | 3-5 天 | Prompt 模板库 |
| 3. MCP 协议 | 手写 MCP Server、连接外部工具 | 1-2 周 | 1-2 个 MCP Server |
| 4. Skills 技能 | 封装可复用的技能模块 | 3-5 天 | 自定义 Skills |
| 5. Agent 开发 | 让 AI 自主规划执行多步任务 | 1-2 周 | 功能 Agent |
| 6. Workflow 编排 | 多 Agent 协同、流水线任务 | 1-2 周 | 工作流脚本 |
| 7. Memory 记忆 | 持久化上下文、知识管理 | 3-5 天 | 记忆系统 |
| 8. Hooks 自动化 | 事件驱动的自动触发 | 3-5 天 | 自动化配置 |
| 9. 实战项目 | 综合运用所有技能 | 2-4 周 | 完整项目 |

---

## 二、第一阶段：AI 基础认知

### 2.1 核心概念

#### LLM（大语言模型）

> LLM = 一个巨大的"下一个词预测器"。给它一段文字，它预测最可能的下一个词是什么。规模大到一定程度后，涌现出推理、编程、翻译等能力。

关键概念：

| 概念 | 说明 | 类比 |
|------|------|------|
| **Token** | 模型处理的最小文本单元 | 约等于 0.75 个英文单词，一个汉字 ≈ 1-2 个 token |
| **上下文窗口** | 模型一次能"看到"的最大 token 数 | 像工作记忆，超出就"忘记" |
| **Temperature** | 控制输出的随机性 (0-1) | 0=确定性的，1=天马行空 |
| **System Prompt** | 设定 AI 角色和行为的指令 | 像给员工的岗位说明书 |
| **Tool Use** | AI 调用外部函数的能力 | 像给人一个工具箱 |
| **Streaming** | 逐字输出而非一次性返回 | 像打字机效果 |

#### 当前主流模型（2026）

| 模型 | 特点 | 适用场景 |
|------|------|---------|
| Claude Opus 5 | 最强推理能力 | 复杂架构设计、高难度编程 |
| Claude Fable 5 | 平衡能力与可用性 | 日常开发、代码审查 |
| Claude Sonnet 5 | 快速响应 | 简单任务、实时交互 |
| Claude Haiku 4.5 | 极速轻量 | 分类、提取、简单问答 |

### 2.2 学习资源

- 阅读 [Claude API 文档](https://docs.anthropic.com)
- 理解 [Prompt Caching 机制](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- 动手在 [Claude Code](https://claude.ai/code) 中试用

### 2.3 验证标准

- [ ] 能解释 token 和上下文窗口的关系
- [ ] 知道何时用高/低 temperature
- [ ] 能区分不同模型的能力边界

---

## 三、第二阶段：提示词工程

### 3.1 Prompt 的核心原则

```
好的 Prompt = 角色 + 任务 + 约束 + 格式 + 示例
```

#### 原则一：明确角色

```
❌ 差: "帮我写代码"
✅ 好: "你是一个资深 TypeScript 后端工程师，请帮我..."
```

#### 原则二：具体任务

```
❌ 差: "优化这个函数"
✅ 好: "将这个函数的时间复杂度从 O(n²) 降到 O(n log n)，使用二分查找"
```

#### 原则三：给定约束

```
❌ 差: "写一个 API"
✅ 好: "写一个 REST API，使用 Express.js，支持分页，返回 JSON 格式"
```

#### 原则四：指定输出格式

```
✅ 好: "返回 JSON 格式：{ summary: string, steps: string[], code: string }"
```

#### 原则五：提供示例（Few-shot）

```
✅ 好:
"输入：'我创建了一个文件'
输出：{ action: 'create', target: 'file' }

输入：'我删除了三行代码'
输出："
```

### 3.2 高级技巧

#### Chain of Thought（思维链）

```
不要直接给答案，请一步步思考：
1. 先分析问题
2. 列出可能的方案
3. 比较优劣
4. 给出最终建议
```

#### Structured Output（结构化输出）

通过 JSON Schema 约束 AI 的输出格式，确保下游程序可解析：

```json
{
  "type": "object",
  "properties": {
    "summary": { "type": "string" },
    "severity": { "type": "string", "enum": ["low", "medium", "high"] },
    "file_path": { "type": "string" }
  },
  "required": ["summary", "severity"]
}
```

#### 迭代优化

```
第 1 轮：写 prompt → 看输出 → 发现遗漏 → 补充约束
第 2 轮：补充约束 → 看输出 → 发现歧义 → 加示例
第 3 轮：加示例 → 看输出 → 满意 → 保存为模板
```

### 3.3 学习路径

1. 阅读 [Anthropic Prompt Engineering Guide](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering)
2. 在 Claude Code 中反复练习写 prompt
3. 建立自己的 prompt 模板库
4. 学习用 JSON Schema 做结构化输出

### 3.4 验证标准

- [ ] 能写出带角色、约束、格式的完整 prompt
- [ ] 能用 Few-shot 示例提升输出质量
- [ ] 能使用 JSON Schema 约束结构化输出

---

## 四、第三阶段：MCP 协议

> 详见同目录下的 [mcpServers.md](mcpServers.md)（MCP 从原理到实践完整教程）

### 4.1 学习要点速览

| 主题 | 核心内容 | 参考 |
|------|---------|------|
| 什么是 MCP | AI 的 "USB-C 接口"，统一连接外部工具 | [mcpServers.md#一](mcpServers.md#一什么是-mcp) |
| 核心架构 | Client ↔ JSON-RPC 2.0 ↔ Server (stdio/HTTP) | [mcpServers.md#二](mcpServers.md#二核心架构) |
| 底层协议 | JSON-RPC 2.0 消息结构 | [mcpServers.md#三](mcpServers.md#三底层协议json-rpc) |
| 为什么不用脚本 | 自动发现、类型安全、跨工具复用 | [mcpServers.md#四](mcpServers.md#四为什么不用脚本) |
| TS 手写 Server | `@modelcontextprotocol/sdk` + Zod | [mcpServers.md#六](mcpServers.md#六手写-mcp-servertypescript-版) |
| Python 手写 Server | `mcp` 库 + `stdio_server` | [mcpServers.md#七](mcpServers.md#七手写-mcp-serverpython-版) |
| 实战：Jira 工具 | 认证、JQL、API 版本、.env 管理 | [mcpServers.md#八](mcpServers.md#八实战jira-bug-查询-mcp-工具) |
| 踩坑记录 | Node 版本、认证方式、路径问题 | [mcpServers.md#十](mcpServers.md#十踩坑记录) |

### 4.2 学习步骤

```
Step 1: 理解 MCP 概念和架构        (1-2 小时)
   └─▶ 读 mcpServers.md 前四章

Step 2: 跑通第一个示例              (2-4 小时)
   └─▶ 用 TS 或 Python 写一个简单的 MCP Server（如 echo/计算器）

Step 3: 在 Claude Code 中配置使用    (1 小时)
   └─▶ 编辑 settings.json，验证工具被正确发现和调用

Step 4: 开发实用 MCP Server        (2-3 天)
   └─▶ 连接内部 API（Jira/GitLab/数据库），实现业务工具

Step 5: 发布和复用                  (1 天)
   └─▶ 打包、文档化、团队共享
```

### 4.3 MCP Server 开发本质

```
你写函数 → 注册到 Server → Server 通过 stdio 暴露给 AI
                              ↓
AI 自动发现工具 → 用户提问 → AI 调用工具 → 返回结果
```

一个 MCP Server 就是一个**常驻的命令行进程**：

1. **启动** → 告知 AI "我有哪些工具，每个工具的参数是什么"
2. **运行** → 收到调用请求 → 执行业务逻辑 → 返回结构化结果
3. **通信** → 通过 stdio 走 JSON-RPC 格式的消息

### 4.4 验证标准

- [ ] 能手写一个功能完整的 MCP Server（TS 或 Python）
- [ ] 能在 Claude Code 中配置并使用自己写的工具
- [ ] 理解 Tools、Resources、Prompts 三种能力类型
- [ ] 能处理认证、错误、环境变量等工程问题

---

## 五、第四阶段：Skills 技能系统

### 5.1 什么是 Skill

> **Skill 是一个可复用的、打包好的指令集**，告诉 AI 如何完成特定类型的任务。
> 可以理解为给 AI 配置的"专业培训手册"——遇到对应场景，自动按手册执行。

```
普通模式：用户每次手写指令 → AI 猜测意图 → 可能遗漏细节
Skill 模式：触发 Skill → 加载完整指令 → 按流程执行 → 输出一致
```

### 5.2 Skill 的类型

| 类型 | 说明 | 示例 |
|------|------|------|
| **内置 Skill** | Claude Code 自带 | `/review`、`/init`、`/simplify` |
| **项目 Skill** | 放在项目 `.claude/` 目录 | 项目特定的部署、测试流程 |
| **用户 Skill** | 放在用户 `~/.claude/` 目录 | 个人偏好的代码风格、命名规范 |
| **插件 Skill** | 通过插件安装 | 第三方集成的能力 |

### 5.3 Skill 的结构

```
.claude/skills/
├── deploy.md          # 部署流程
├── code-review.md     # 代码审查规范
├── test-strategy.md   # 测试策略
└── release-notes.md   # 发布说明生成
```

一个 Skill 文件的典型结构：

```markdown
---
name: deploy
description: 部署项目到生产环境
---

# 部署流程

## 前置检查
1. 确认在 main 分支
2. 确认所有测试通过
3. 确认版本号已更新

## 部署步骤
1. 构建项目: `npm run build`
2. 运行 E2E 测试: `npm run test:e2e`
3. 推送镜像: `docker push ...`
4. 更新 K8s: `kubectl apply -f ...`

## 回滚方案
如果部署失败，执行: `kubectl rollout undo ...`
```

### 5.4 创建 Skill 的流程

```
Step 1: 识别可复用的任务模式
   └─▶ 什么任务反复做？什么流程容易出错？

Step 2: 编写 Skill 文件
   └─▶ 定义触发词（name）、描述（description）、执行步骤

Step 3: 放置到正确位置
   ├─▶ 项目级: .claude/skills/my-skill.md
   └─▶ 用户级: ~/.claude/skills/my-skill.md

Step 4: 测试和迭代
   └─▶ 触发 Skill → 观察执行 → 改进指令
```

### 5.5 Skill 的设计原则

1. **单一职责**：一个 Skill 只做一件事
2. **步骤明确**：每一步都可执行、可验证
3. **包含检查点**：关键步骤后加验证
4. **失败可恢复**：提供回滚或重试方案
5. **持续迭代**：根据使用反馈不断优化

### 5.6 学习路径

1. 先使用内置 Skills 理解工作方式
2. 阅读项目或社区的 Skill 示例
3. 识别自己日常工作中的重复模式
4. 编写第一个自定义 Skill
5. 团队共享和标准化

### 5.7 验证标准

- [ ] 能区分不同类型的 Skill
- [ ] 能编写并测试一个完整的自定义 Skill
- [ ] 理解 Skill 的触发和加载机制
- [ ] 能为团队建立 Skill 库

---

## 六、第五阶段：Agent 智能体开发

### 6.1 什么是 Agent

> **Agent = LLM + 工具 + 自主决策循环**。Agent 不仅能回答问题，还能自主规划步骤、调用工具、根据结果调整策略，直到完成任务。

```
普通 LLM：用户问 → AI 答
Agent：用户给目标 → AI 规划 → 调用工具 → 检查结果 → 调整 → ... → 最终交付
```

### 6.2 Agent 的核心循环

```
        ┌──────────────────────────────────────────┐
        │              Agent 循环                    │
        │                                           │
        │   ┌──────────┐                            │
        │   │ 分析任务  │                            │
        │   └────┬─────┘                            │
        │        ▼                                   │
        │   ┌──────────┐    不够                    │
        │   │ 规划步骤  │─────────────┐             │
        │   └────┬─────┘             │             │
        │        ▼                   │             │
        │   ┌──────────┐             │             │
        │   │ 调用工具  │             │             │
        │   └────┬─────┘             │             │
        │        ▼                   │             │
        │   ┌──────────┐             │             │
        │   │ 检查结果  │             │             │
        │   └────┬─────┘             │             │
        │        ▼                   │             │
        │   目标达成？─── 是 ───▶ 交付结果          │
        │        │                                   │
        │        否                                  │
        │        │                                   │
        │        └───────────────────────────────────┘
        └───────────────────────────────────────────
```

### 6.3 Agent 开发关键要素

| 要素 | 说明 | 示例 |
|------|------|------|
| **系统提示** | 定义 Agent 的角色、能力边界 | "你是一个代码审查 Agent..." |
| **工具集** | Agent 可以调用的工具 | 读文件、运行测试、搜索代码 |
| **停止条件** | 何时认为任务完成 | "所有测试通过"或"达到最大步数" |
| **错误处理** | 工具调用失败时怎么做 | 重试 3 次、降级处理、请求帮助 |
| **上下文管理** | 控制对话长度和记忆 | 摘要压缩、选择性遗忘 |

### 6.4 Agent 开发模式

#### 模式一：单 Agent + 多工具

```
Agent "代码助手"
  ├─▶ 工具: 读文件
  ├─▶ 工具: 搜索代码
  ├─▶ 工具: 运行测试
  └─▶ 工具: 提交 PR
```

适用：职责单一、工具数量可控的场景

#### 模式二：多 Agent 协作

```
主 Agent "项目经理"
  ├─▶ 子 Agent "前端开发"
  ├─▶ 子 Agent "后端开发"
  ├─▶ 子 Agent "测试"
  └─▶ 子 Agent "文档"
```

适用：复杂项目、需要并行处理的场景

#### 模式三：Agent 流水线

```
代码生成 Agent → 代码审查 Agent → 测试 Agent → 部署 Agent
```

适用：有严格顺序要求的场景

### 6.5 学习路径

1. 理解 Agent 循环（ReAct / Tool Use 模式）
2. 写一个简单的单 Agent（如代码分析 Agent）
3. 添加多个工具和错误处理
4. 尝试多 Agent 协作
5. 优化上下文管理和 token 消耗

### 6.6 验证标准

- [ ] 能实现一个带工具调用的 Agent
- [ ] 能处理工具调用失败和重试
- [ ] 能设计多 Agent 协作架构
- [ ] 理解 token 消耗和上下文管理策略

---

## 七、第六阶段：Workflow 工作流编排

### 7.1 什么是 Workflow

> **Workflow 是多 Agent 的编排层**——定义哪些 Agent 按什么顺序执行、如何传递数据、如何处理异常。它把"让 AI 干活"升级为"让 AI 团队干活"。

### 7.2 编排模式

#### Pipeline（流水线）

项目依次经过多个阶段，每个阶段的结果传给下一个：

```
[源文件] → Agent A (分析) → Agent B (优化) → Agent C (审查) → [最终结果]
```

- **特点**：各阶段独立，后面的阶段不阻塞前面的项目
- **适用**：多文件处理、多步骤变换

#### Parallel（并行）

多个 Agent 同时工作，在某一点汇合：

```
              ┌─▶ Agent A (搜索 Bug) ─┐
[任务分配] ──┼─▶ Agent B (搜索性能) ─┼──▶ [合并结果]
              └─▶ Agent C (搜索安全) ─┘
```

- **特点**：充分利用并发，缩短总时间
- **适用**：多维度分析、独立子任务

#### Barrier（关卡）

所有并行任务完成后，统一进入下一阶段：

```
[并行搜索] ──▶ 等待全部完成 ──▶ [去重合并] ──▶ [并行验证] ──▶ 等待全部完成 ──▶ [报告]
```

- **特点**：保证数据完整性，但会等待最慢的任务
- **适用**：需要完整数据才能进入下一步的场景

#### Loop-until（循环至满足条件）

```javascript
// 循环收集直到达到目标数量
while (bugs.length < 10) {
  const result = await agent("查找代码 Bug", { schema: BUGS_SCHEMA });
  bugs.push(...result.bugs);
}
```

- **特点**：动态迭代，直到满足条件
- **适用**：搜索结果收集、质量门禁

### 7.3 Workflow 脚本结构

```javascript
export const meta = {
  name: 'code-review-workflow',
  description: '全维度代码审查工作流',
  phases: [
    { title: '扫描', detail: '多维度扫描代码' },
    { title: '验证', detail: '交叉验证发现的问题' },
  ],
};

phase('扫描');
const dimensions = [
  { key: 'bugs', prompt: '查找代码中的 Bug...' },
  { key: 'perf', prompt: '分析性能问题...' },
  { key: 'security', prompt: '检查安全漏洞...' },
];

// Pipeline：每个维度独立审查→验证
const results = await pipeline(
  dimensions,
  d => agent(d.prompt, { label: `review:${d.key}`, phase: '扫描', schema: FINDINGS_SCHEMA }),
  review => parallel(
    review.findings.map(f => () =>
      agent(`验证: ${f.title}`, { label: `verify:${f.file}`, phase: '验证', schema: VERDICT_SCHEMA })
        .then(v => ({ ...f, verdict: v }))
    )
  )
);

const confirmed = results.flat().filter(Boolean).filter(f => f.verdict?.isReal);
return { confirmed };
```

### 7.4 Workflow 设计原则

1. **默认用 Pipeline**：大多数场景下 Pipeline 足够，不需要 Barrier
2. **只在真正需要完整数据时用 Barrier**：如去重、计数、汇总
3. **独立验证**：让多个 Agent 从不同角度审查同一结果
4. **循环收敛**：Loop-until-dry（连续 N 轮无新发现才停止）
5. **资源控制**：注意 token 预算和并发上限

### 7.5 学习路径

1. 理解 Pipeline vs Parallel vs Barrier 的区别
2. 写简单的单阶段 Workflow
3. 添加验证阶段（adversarial verify）
4. 尝试多维度并行审查模式
5. 实现 Loop-until 收集模式

### 7.6 验证标准

- [ ] 能写出包含多阶段的 Workflow 脚本
- [ ] 能正确选择 Pipeline/Parallel/Barrier 模式
- [ ] 能实现交叉验证和 Loop-until 模式
- [ ] 理解 token 预算和编排规模控制

---

## 八、第七阶段：Memory 与持久化

### 8.1 为什么需要 Memory

AI 对话是无状态的——每次新对话，AI "忘记"之前的事。Memory 系统让你能跨对话保存重要的上下文。

```
没有 Memory：
  对话 1: "我叫小明，喜欢用 TypeScript"
  对话 2: "帮我推荐项目" → AI 不知道你是谁

有 Memory：
  对话 1: "我叫小明，喜欢用 TypeScript" → 存入 Memory
  对话 2: "帮我推荐项目" → AI 读取 Memory → "小明，根据你的 TS 偏好..."
```

### 8.2 Memory 的类型

| 类型 | 说明 | 示例 |
|------|------|------|
| **user** | 用户身份、偏好、技能 | "小明，前端工程师，偏好 React + TS" |
| **feedback** | 用户给 AI 的行为反馈 | "每次提交前必须运行测试" |
| **project** | 项目背景、目标、约束 | "这个项目需要兼容 Node 18" |
| **reference** | 外部资源引用 | "API 文档在 https://..." |

### 8.3 Memory 文件结构

```
.claude/memory/
├── MEMORY.md                 # 索引文件（每行一条记忆）
├── user-preferences.md       # 用户偏好
├── project-context.md        # 项目背景
└── feedback-rules.md         # 反馈规则
```

Memory 文件格式：

```markdown
---
name: user-preferences
description: 用户的编程偏好和技术栈
metadata:
  type: user
---

小明偏好使用 TypeScript + React，在写代码时优先使用函数式组件和 Hooks。
**Why:** 用户有 5 年前端经验，最熟悉这套技术栈。
**How to apply:** 生成前端代码时默认使用 TS + React 函数式组件。
```

### 8.4 Memory 最佳实践

1. **原子性**：一个文件只记录一个事实
2. **可链接**：用 `[[other-memory-name]]` 关联相关记忆
3. **有原因**：记录 **Why** 和 **How to apply**
4. **可验证**：信息应该是事实，不是推测
5. **不重复代码**：已经记录在代码库里的信息不用存 memory

### 8.5 验证标准

- [ ] 理解四种 Memory 类型的使用场景
- [ ] 能正确创建和链接 Memory 文件
- [ ] 知道什么该存 Memory、什么不该存

---

## 九、第八阶段：Hooks 与自动化

### 9.1 什么是 Hooks

> **Hooks 是在特定事件发生时自动触发的动作**。比如"每次代码提交前运行测试"或"新文件创建时自动格式化"。

### 9.2 常见 Hook 事件

| 事件 | 触发时机 | 典型用途 |
|------|---------|---------|
| `PreToolUse` | 工具调用之前 | 权限检查、参数校验 |
| `PostToolUse` | 工具调用之后 | 结果格式转换、日志记录 |
| `Notification` | 收到通知时 | 自动响应事件 |
| `Stop` | AI 回答完成时 | 自动继续下一步 |
| `UserPromptSubmit` | 用户提交 prompt 时 | 上下文注入、预处理 |

### 9.3 Hook 配置示例

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'about to run command' >> /tmp/claude.log"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "osascript -e 'display notification \"Claude 回答完成\"'"
          }
        ]
      }
    ]
  }
}
```

### 9.4 学习路径

1. 了解可用的 Hook 事件类型
2. 配置一个简单的通知 Hook
3. 实现 PreToolUse 权限控制
4. 组合多个 Hook 构建自动化流水线

---

## 十、第九阶段：实战项目

### 10.1 项目一：个人代码助手

**目标**：打造一个懂你项目、帮你写代码的 AI 助手

涉及知识点：
- MCP Server：连接 Git、包管理器
- Skills：代码生成规范、测试策略
- Memory：项目架构、编码偏好
- Agent：单 Agent + 多工具

**产出**：
```
自定义 MCP Server (3-5 个工具)
  ├─▶ git-status: 仓库状态分析
  ├─▶ code-stats: 代码统计
  ├─▶ dep-check: 依赖分析
  ├─▶ test-runner: 测试管理
  └─▶ pr-helper: PR 创建辅助
```

### 10.2 项目二：自动化代码审查流水线

**目标**：从 PR 创建到合并的全自动审查

涉及知识点：
- Workflow：多维度并行审查
- MCP Server：连接 Git 平台
- Agent：Bug 搜索 + 安全扫描 + 性能分析
- Hooks：PR 事件自动触发

**流程**：
```
PR 创建 → 多维度并行扫描 → 交叉验证 → 评分 → 自动评论 → 批准/请求修改
```

### 10.3 项目三：企业内部工具集成

**目标**：让 AI 能操作企业内部系统

涉及知识点：
- MCP Server：连接 Jira、Confluence、数据库
- Skills：工作流程标准化
- Workflow：跨系统协作编排

**产出**：
```
企业工具套件
  ├─▶ Jira MCP (Bug/任务管理)
  ├─▶ Confluence MCP (文档查询)
  ├─▶ DB MCP (数据查询)
  └─▶ 工作流: Bug 分析→代码定位→生成修复→测试验证→更新 Jira
```

### 10.4 项目检查清单

- [ ] 项目一：能通过自然语言让 AI 操作你的代码仓库
- [ ] 项目二：能自动化代码审查流程，减少人工 Review 时间
- [ ] 项目三：能让 AI 打通多个企业内部系统

---

## 十一、工具与资源汇总

### 11.1 核心工具

| 工具 | 用途 | 链接 |
|------|------|------|
| Claude Code | AI 编程助手（CLI/IDE/Web） | [claude.ai/code](https://claude.ai/code) |
| Claude Desktop | 桌面版 AI 助手 | [claude.ai](https://claude.ai) |
| MCP SDK (TS) | TypeScript MCP 开发 | `@modelcontextprotocol/sdk` |
| MCP SDK (Python) | Python MCP 开发 | `pip install mcp` |
| Claude API | 程序化调用 Claude | [docs.anthropic.com](https://docs.anthropic.com) |

### 11.2 学习资源

| 资源 | 说明 |
|------|------|
| [Anthropic 官方文档](https://docs.anthropic.com) | API、MCP、Prompt Engineering |
| [MCP 官方规范](https://modelcontextprotocol.io) | MCP 协议完整规范 |
| [Claude Code 文档](https://docs.anthropic.com/en/docs/claude-code) | Claude Code 完整功能说明 |
| [MCP Servers 仓库](https://github.com/modelcontextprotocol/servers) | 官方和社区 MCP Server 集合 |
| [mcpServers.md](mcpServers.md) | 本项目 MCP 实战教程 |

### 11.3 学习建议

```
学习节奏：
  每天 1-2 小时理论学习 + 2-3 小时动手实践
  每完成一个阶段，做一个迷你项目巩固

优先级：
  必学：MCP（能写工具）、Skills（能封装流程）
  进阶：Agent（自主执行）、Workflow（多 Agent 编排）
  高级：Memory（持久化）、Hooks（全自动化）

最重要的原则：
  边学边做，不要只看不写！
  先跑通最小可用版本，再逐步加功能。
```

---

## 附录：概念速查表

| 概念 | 一句话解释 | 在哪学 |
|------|-----------|--------|
| LLM | 大语言模型，"下一个词预测器" | 第二阶段 |
| Token | 模型处理的最小文本单元 | 第二阶段 |
| Prompt | 给 AI 的指令 | 第三阶段 |
| MCP | AI 连接外部工具的标准协议 | 第四阶段 + [mcpServers.md](mcpServers.md) |
| MCP Server | 暴露工具给 AI 的后端进程 | 第四阶段 |
| MCP Client | 使用工具的 AI 应用 | 第四阶段 |
| Tool | MCP Server 暴露的可调用函数 | 第四阶段 |
| Skill | 可复用的任务指令集 | 第五阶段 |
| Agent | 能自主规划执行任务的 AI | 第六阶段 |
| Workflow | 多 Agent 的编排执行脚本 | 第七阶段 |
| Pipeline | 流水线模式，阶段间无等待 | 第七阶段 |
| Parallel | 并行模式，所有任务完成后汇合 | 第七阶段 |
| Barrier | 关卡模式，必须等前一阶段全完成 | 第七阶段 |
| Memory | 跨对话持久化的上下文 | 第八阶段 |
| Hooks | 事件驱动的自动触发动作 | 第九阶段 |

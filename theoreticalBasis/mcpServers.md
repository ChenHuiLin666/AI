# MCP（Model Context Protocol）从原理到实践

## 目录

- [一、什么是 MCP](#一什么是-mcp)
- [二、核心架构](#二核心架构)
- [三、底层协议：JSON-RPC](#三底层协议json-rpc)
- [四、为什么不用脚本？](#四为什么不用脚本)
- [五、MCP 生态与语言选择](#五mcp-生态与语言选择)
- [六、手写 MCP Server（TypeScript 版）](#六手写-mcp-servertypescript-版)
- [七、手写 MCP Server（Python 版）](#七手写-mcp-serverpython-版)
- [八、实战：Jira Bug 查询 MCP 工具](#八实战jira-bug-查询-mcp-工具)
- [九、在 Claude Code 中配置](#九在-claude-code-中配置)
- [十、踩坑记录](#十踩坑记录)
- [十一、总结](#十一总结)

---

## 一、什么是 MCP

**MCP（Model Context Protocol，模型上下文协议）** 是 Anthropic 发布的开放协议标准，为 AI 模型与外部工具、数据源之间定义了统一的通信接口。

### 一句话类比

> **MCP 就是 AI 应用的 "USB-C 接口"**——USB-C 让不同设备用同一根线连接，MCP 让不同 AI 模型以同一种方式连接各种外部工具和数据。

### 它能做什么

- 让 Claude 读取本地文件、查询数据库、调 API
- 一次开发，所有支持 MCP 的 AI 工具都能用（Claude Code、Claude Desktop、Cursor 等）
- 工具能力由 Server 自动"自我介绍"，AI 不需要在 prompt 里手动写说明书

---

## 二、核心架构

```
┌──────────────────┐                      ┌──────────────────┐
│                  │    JSON-RPC 2.0       │                  │
│   AI 客户端       │ ◄──────────────────► │   MCP Server     │
│  (Claude Code)   │    通过 stdio/HTTP     │  (你写的工具)     │
│                  │                      │                  │
└──────────────────┘                      └──────────────────┘
```

### 关键概念

| 概念 | 说明 |
|------|------|
| **MCP Client** | AI 应用本身（Claude Code、Claude Desktop 等），负责发现和调用工具 |
| **MCP Server** | 你写的工具进程，暴露 Tools、Resources、Prompts |
| **Tools** | 可执行的函数，有输入参数和返回值（类似 API 端点） |
| **Resources** | 只读数据源，如文件内容、配置信息 |
| **Transport** | 通信方式，最常用的是 **stdio**（标准输入输出），也支持 HTTP/SSE |

### 通信流程

```
1. Claude Code 启动你的 Server 进程:
   node dist/index.js  （或 python3 server.py）

2. Server 启动，通过 stdio 等待消息

3. Claude Code 询问 Server 有哪些能力:
   ──→  {"method": "tools/list"}
   ←──  [{"name": "count_lines", "inputSchema": {...}}]

4. 用户提问，Claude Code 调用工具:
   ──→  {"method": "tools/call", "params": {"name": "count_lines", "arguments": {...}}}
   ←──  {"result": "共 8342 行"}

5. Claude Code 把结果显示给用户
```

---

## 三、底层协议：JSON-RPC

MCP 底层使用 **JSON-RPC 2.0**，这是一种极简的远程过程调用协议。

### JSON-RPC 消息结构

**请求**（Client → Server）：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "count_lines",
    "arguments": { "directory": "/src", "extension": ".ts" }
  }
}
```

**响应**（Server → Client）：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": "共 8342 行"
}
```

### 和 REST API 的区别

| | REST API | JSON-RPC |
|------|----------|----------|
| 概念模型 | 操作"资源"（GET /users/1） | 调用"函数"（`getUser(id=1)`） |
| 传输层 | 必须是 HTTP | 任意（stdio、HTTP、WebSocket） |
| 数据格式 | 不固定 | 永远是 JSON |
| 错误处理 | HTTP 状态码 | `error` 字段 + 自定义错误码 |

> **MCP = JSON-RPC 的应用层协议**。底层用 JSON-RPC 通信，上层定义了 AI 场景专用语义（Tools、Resources、Prompts）。

---

## 四、为什么不用脚本？

写一个 shell 脚本让 Claude 调用确实能工作。但 MCP 解决了几个关键问题：

### 对比

| 能力 | 脚本方式 | MCP 方式 |
|------|---------|---------|
| **工具发现** | 在 prompt 里手写说明，AI 靠阅读理解 | `tools/list` 自动告知，带完整 Schema |
| **参数校验** | 自己写，出错靠重试 | JSON Schema 定义，类型安全 |
| **返回值** | 纯文本，需手动解析 | 结构化 JSON |
| **跨工具复用** | 每个 AI 工具重新写 prompt | 一次开发，所有支持 MCP 的工具通用 |
| **多工具管理** | 每个工具一个脚本文件 | 一个进程暴露 N 个工具 |

### 核心差异

> **脚本 = 你手动告诉 AI 怎么用。MCP = AI 自己发现怎么用。**

比如一个参数是 `directory`（路径），当你用脚本时 Claude 可能传 `src` 还是 `./src/` 不确定；用 MCP 时 Schema 写明类型是 `string`、描述是"要统计的目录路径"，Claude 不会猜错。

---

## 五、MCP 生态与语言选择

### 流行语言

| 语言 | 热度 | 适合场景 |
|------|------|---------|
| **TypeScript/JavaScript** | 🔥 最热 | 全栈、工具链、快速原型 |
| **Python** | 🔥 最热 | 数据分析、API 封装、脚本工具 |
| **Go** | ☀️ 热门 | 高性能、系统级工具 |
| **Rust** | 🌤️ 增长中 | 极致性能、安全敏感场景 |

### 推荐

- 新手/API 封装类工具 → **Python**（不用编译，改完就跑）
- 全栈/前端项目 → **TypeScript**（生态最完善，类型安全）

---

## 六、手写 MCP Server（TypeScript 版）

### 项目结构

```
my-mcp-server/
├── package.json
├── tsconfig.json
└── src/
    └── index.ts
```

### Step 1: 初始化

```bash
mkdir my-mcp-server && cd my-mcp-server
npm init -y
npm install @modelcontextprotocol/sdk zod
npm install -D typescript @types/node
npx tsc --init --target ES2022 --module nodenext --moduleResolution nodenext \
  --outDir dist --rootDir src --strict
```

### Step 2: 写代码

```typescript
// src/index.ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({
  name: "my-tools",
  version: "1.0.0",
});

// 注册工具
server.tool(
  "count_lines",                            // 工具名
  "统计指定目录的代码行数",                     // 描述（AI 会读到）
  {                                          // 参数 Schema
    directory: z.string().describe("目录路径"),
    extension: z.string().optional().describe("扩展名过滤，如 .ts"),
  },
  async ({ directory, extension }) => {       // 实现
    // ...你的逻辑...
    return {
      content: [{ type: "text", text: "共 8342 行" }],
    };
  },
);

// 启动
const transport = new StdioServerTransport();
await server.connect(transport);
```

### Step 3: 编译 + 运行

```bash
npx tsc
node dist/index.js
```

### Step 4: 在 Claude Code 中配置

```json
{
  "mcpServers": {
    "my-tools": {
      "command": "node",
      "args": ["./my-mcp-server/dist/index.js"]
    }
  }
}
```

---

## 七、手写 MCP Server（Python 版）

### Step 1: 安装依赖

```bash
pip install mcp httpx python-dotenv
```

### Step 2: 写代码

```python
# server.py
import os
from pathlib import Path
from dotenv import load_dotenv
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

# 加载环境变量
SCRIPT_DIR = Path(__file__).resolve().parent
load_dotenv(SCRIPT_DIR.parent / ".env")

# 创建 Server
server = Server("my-tools")

@server.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="my_tool",
            description="工具描述，AI 会读到这个来决定何时调用",
            inputSchema={
                "type": "object",
                "properties": {
                    "param1": {
                        "type": "string",
                        "description": "参数说明",
                    },
                },
                "required": ["param1"],
            },
        ),
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    if name == "my_tool":
        result = f"收到参数: {arguments['param1']}"
        return [TextContent(type="text", text=result)]
    return [TextContent(type="text", text=f"未知工具: {name}")]

async def main():
    async with stdio_server() as (read_stream, write_stream):
        await server.run(read_stream, write_stream,
                         server.create_initialization_options())

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

### Step 3: 在 Claude Code 中配置

```json
{
  "mcpServers": {
    "my-tools": {
      "command": "/opt/miniconda3/bin/python3",
      "args": ["/Users/xxx/mcp/server.py"]
    }
  }
}
```

---

## 八、实战：Jira Bug 查询 MCP 工具

### 背景

用 Python + conda base 环境，开发一个 MCP Server，在 Claude Code 中查询 Jira Bug。

### 项目结构

```
~/code/mcp/
├── .env                          # Jira 连接凭证
└── jira-tools/
    └── server.py                 # MCP Server
```

### .env 文件

```ini
JIRA_BASE_URL=https://jira.your-company.com
JIRA_EMAIL=you@company.com
JIRA_API_TOKEN=your-personal-access-token
```

### 提供的三个工具

| 工具 | 功能 | 参数 |
|------|------|------|
| `get_my_bugs` | 获取当前用户的 Bug 列表 | `status`（可选）、`max_results`（默认 20） |
| `search_jira_bugs` | 搜索 Bug（可按项目/状态/经办人过滤） | `project_key`、`status`、`assignee`、`max_results` |
| `get_jira_issue` | 获取单个 Bug 详情（含评论） | `issue_key`（必填） |

### server.py 核心结构

```
                          启动
                           │
                    ┌──────▼──────┐
                    │ 加载 .env    │  python-dotenv 读取凭证
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │ 获取当前用户  │  GET /rest/api/2/myself
                    │ (init_user) │  自动识别 jiangxuechen
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │ 注册 3 个工具 │  list_tools() 告知 Claude
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │ stdio 监听   │  等待 Claude Code 调用
                    └─────────────┘
```

### 关键实现细节

**认证方式**：私有部署版 Jira（非 atlassian.net）使用 Bearer Token，不是 Basic Auth。

```python
AUTH_HEADER = {
    "Authorization": f"Bearer {JIRA_API_TOKEN}",
    "Accept": "application/json",
}
```

**API 路径**：私有部署版用 `/rest/api/2/`，云版用 `/rest/api/3/`。

**JQL 构建**：

```python
jql_parts = ["issuetype = Bug"]
if status:
    jql_parts.append(f"status = '{status}'")
if assignee:
    jql_parts.append(f"assignee = {assignee}")
jql = " AND ".join(jql_parts) + " ORDER BY created DESC"
```

**评论并行加载**：

```python
issue, comments = await asyncio.gather(
    jira_request("GET", f"/rest/api/2/issue/{key}"),
    jira_request("GET", f"/rest/api/2/issue/{key}/comment"),
)
```

---

## 九、在 Claude Code 中配置

编辑项目或全局的 `.claude/settings.local.json`：

```json
{
  "permissions": {
    "allow": ["...已有权限..."]
  },
  "mcpServers": {
    "jira-tools": {
      "command": "/opt/miniconda3/bin/python3",
      "args": ["/Users/lateyoung/code/mcp/jira-tools/server.py"]
    }
  }
}
```

密钥在 `.env` 文件中，不在配置里暴露。重启 Claude Code 后自动生效。

---

## 十、踩坑记录

### 坑 1：Node 版本过低

- **现象**：`npm install @modelcontextprotocol/sdk` 安装失败
- **原因**：MCP SDK 需要 Node 18+，系统自带的是 v14
- **解决**：`nvm install 20 && nvm use 20`

### 坑 2：私有部署版 Jira 认证失败（HTTP 401）

- **现象**：用 `Authorization: Basic <base64(email:token)>` 始终返回 401
- **排查**：测试 Bearer Token、不同的 API 版本、查看 `WWW-Authenticate` 响应头
- **解决**：私有部署版 Jira 使用 `Authorization: Bearer <token>`，不是 Basic Auth

### 坑 3：私有部署版 API 版本不对

- **现象**：部分接口返回 404
- **原因**：私有部署版用 `/rest/api/2/`，云版用 `/rest/api/3/`
- **解决**：统一使用 `/rest/api/2/`

### 坑 4：.env 文件路径

- **问题**：MCP Server 由 Claude Code 启动，工作目录不确定
- **解决**：用 `Path(__file__).resolve().parent` 定位脚本所在目录，再向上找 `.env`

```python
SCRIPT_DIR = Path(__file__).resolve().parent
load_dotenv(SCRIPT_DIR.parent / ".env")   # ~/code/mcp/.env
```

---

## 十一、总结

### MCP 的核心思想

```
你写函数 → 注册到 Server → Server 通过 stdio 暴露给 Claude Code
                              ↓
Claude 自动发现工具 → 用户提问 → Claude 调用工具 → 返回结果
```

### 开发 MCP Server 的本质

就是写一个**常驻的命令行进程**：

1. **启动时**：告诉 Claude "我有哪些工具，每个工具的参数是什么"
2. **运行时**：收到工具调用请求 → 执行业务逻辑 → 返回结果
3. **传输层**：通过标准输入输出（stdio）走 JSON-RPC 格式的消息，不涉及网络端口

### 适合用 MCP 的场景

- 需要查询内部 API（Jira、GitLab、数据库等）
- 需要操作本地文件（代码统计、批量处理）
- 需要复杂参数和结构化返回值的工具
- 需要在多个 AI 工具间复用同一套工具

### 不适合用 MCP 的场景

- 一次性的简单脚本（直接用 Bash 更方便）
- 纯聊天问答（不需要工具）
- 没有结构化输入输出的命令行工具

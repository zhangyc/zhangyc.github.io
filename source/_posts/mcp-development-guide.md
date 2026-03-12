---
title: MCP 开发实战：给 AI 装上「手」
date: 2026-03-12
tags:
  - MCP
  - Claude
  - AI
  - 工具开发
categories:
  - AI 工程化
---

## 什么是 MCP？

Model Context Protocol（MCP）是 Anthropic 在 2024 年底提出的开放协议，目标很简单：**让 AI 模型能够标准化地调用外部工具**。

在 MCP 之前，每家 AI 产品都有自己的 function calling 格式，工具和模型强绑定，开发一套工具只能用在一个平台。MCP 把这层协议标准化了——工具开发者只要实现 MCP Server，任何支持 MCP 的客户端（Claude Desktop、Claude Code、Cursor 等）都能直接用。

类比一下：MCP 之于 AI 工具，就像 USB-C 之于充电器。统一接口，任意组合。

---

## 核心概念

MCP 的模型很简单，只有三种能力：

| 能力 | 说明 | 典型场景 |
|------|------|---------|
| **Tools** | AI 可以调用的函数 | 查数据库、发 HTTP 请求、操作文件 |
| **Resources** | AI 可以读取的数据源 | 读取文件内容、获取配置 |
| **Prompts** | 预置的提示词模板 | 标准化的任务流程 |

大部分情况下，你只需要实现 **Tools**。

---

## 架构：Client ↔ Server

```
Claude (Client)
    │
    │  JSON-RPC 2.0（stdio 或 SSE）
    ▼
MCP Server（你开发的）
    │
    ▼
外部系统（Figma API、数据库、文件系统...）
```

Claude Code 通过 `~/.claude/settings.local.json` 或 `~/.claude/settings.json` 配置 MCP Server 列表，启动时自动拉起对应进程，通过 **stdio** 通信。

---

## 传输方式

MCP 支持两种传输：

- **stdio**：最常用。Claude 启动一个子进程，通过 stdin/stdout 通信。简单可靠，适合本地工具。
- **SSE（Server-Sent Events）**：HTTP 长连接，适合远程 Server 或需要共享的服务。

本文以 stdio 为主。

---

## 动手：用 Node.js 写一个 MCP Server

以一个「读取本地 Markdown 笔记」的工具为例。

### 1. 初始化项目

```bash
mkdir my-notes-mcp && cd my-notes-mcp
npm init -y
npm install @modelcontextprotocol/sdk
```

### 2. 实现 Server

```javascript
// index.js
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  ListToolsRequestSchema,
  CallToolRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";
import fs from "fs";
import path from "path";

const NOTES_DIR = path.join(process.env.HOME, "Documents/notes");

// 1. 创建 Server 实例
const server = new Server(
  { name: "notes-mcp", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

// 2. 声明工具列表
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "list_notes",
      description: "列出所有笔记文件",
      inputSchema: {
        type: "object",
        properties: {},
      },
    },
    {
      name: "read_note",
      description: "读取指定笔记的内容",
      inputSchema: {
        type: "object",
        properties: {
          filename: {
            type: "string",
            description: "笔记文件名（含扩展名）",
          },
        },
        required: ["filename"],
      },
    },
  ],
}));

// 3. 实现工具逻辑
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  if (name === "list_notes") {
    const files = fs.readdirSync(NOTES_DIR).filter((f) => f.endsWith(".md"));
    return {
      content: [{ type: "text", text: files.join("\n") }],
    };
  }

  if (name === "read_note") {
    const filePath = path.join(NOTES_DIR, args.filename);
    if (!fs.existsSync(filePath)) {
      return {
        content: [{ type: "text", text: `文件不存在: ${args.filename}` }],
        isError: true,
      };
    }
    const content = fs.readFileSync(filePath, "utf-8");
    return {
      content: [{ type: "text", text: content }],
    };
  }

  throw new Error(`未知工具: ${name}`);
});

// 4. 启动
const transport = new StdioServerTransport();
await server.connect(transport);
```

### 3. 注册到 Claude Code

编辑 `~/.claude/settings.local.json`（或项目级的 `.claude/settings.local.json`）：

```json
{
  "mcpServers": {
    "notes": {
      "command": "node",
      "args": ["/absolute/path/to/my-notes-mcp/index.js"]
    }
  }
}
```

重启 Claude Code，输入 `/mcp` 可以看到工具已经注册。

---

## 真实案例：Figma MCP

这是我目前在用的配置：

```json
{
  "mcpServers": {
    "TalkToFigma": {
      "command": "npx",
      "args": ["@sethdouglasford/mcp-figma@latest"]
    }
  }
}
```

`@sethdouglasford/mcp-figma` 提供了一批 Figma 相关工具：读取节点树、获取设计 token、下载图片等。配置好之后，直接把 Figma URL 丢给 Claude，它就能自己去读设计稿，输出还原度极高的 UI 代码。

核心工具一览：

| 工具名 | 作用 |
|--------|------|
| `get_figma_data` | 读取文件/节点的完整设计数据 |
| `download_figma_images` | 下载图片资源到本地 |

---

## 用 Go 写 MCP Server

如果你的团队主要用 Go，也可以用 Go 实现。目前社区有 `mark3labs/mcp-go`：

```go
package main

import (
    "context"
    "github.com/mark3labs/mcp-go/mcp"
    "github.com/mark3labs/mcp-go/server"
)

func main() {
    s := server.NewMCPServer("my-server", "1.0.0")

    // 注册工具
    s.AddTool(mcp.NewTool("get_weather",
        mcp.WithDescription("获取城市天气"),
        mcp.WithString("city", mcp.Required(), mcp.Description("城市名")),
    ), func(ctx context.Context, req mcp.CallToolRequest) (*mcp.CallToolResult, error) {
        city := req.Params.Arguments["city"].(string)
        // 调用天气 API...
        return mcp.NewToolResultText("晴天，25°C"), nil
    })

    // stdio 启动
    if err := server.ServeStdio(s); err != nil {
        panic(err)
    }
}
```

---

## 实践经验

### 工具描述要写好

工具的 `description` 是 AI 决定"要不要调用这个工具"的依据，写得越清楚，AI 判断越准确。

```javascript
// 不好：太模糊
description: "处理数据"

// 好：明确输入输出和触发条件
description: "根据用户 ID 查询数据库中的用户信息，返回姓名、邮箱和注册时间"
```

### 错误处理要友好

返回 `isError: true` + 清晰的错误信息，让 AI 知道出了什么问题，它会自动尝试修正参数或告知用户。

### 安全边界

stdio 模式下，MCP Server 运行在本地，有和用户一样的文件系统权限。**不要在 MCP Server 里暴露删除/写入操作，除非你清楚风险。**

---

## 小结

MCP 的价值在于**可组合性**。你把能力封装成 Server，Claude 根据上下文自主决定调用哪个工具、何时调用。随着 Server 越来越多，AI 的能力边界也随之扩大。

下一篇聊 **Claude Code Skill 开发**，这是另一种扩展 AI 能力的方式——不是给 AI 加工具，而是给 AI 注入领域知识和编码规范。

---
title: Anthropic MCP 介绍
date: 2024-12-10 21:42:39 +0800
categories: 技术
tags:
  - AI
layout: single
toc: "true"
toc_label: 目录
toc_icon: list
toc_sticky: "true"
sidebar:
  nav: docs
comments: "true"
---
## MCP架构和原理


![](https://d2m4tio3tm4t0x.cloudfront.net/2024/12/e13f142ee2cbde7d77c63f9b6e6f6e5b.svg)
### 资源的交互方式

模型通过 MCP Resource 读取文档内容的完整流程：

1. **MCP Server 端准备工作**：

```typescript
// 服务器需要实现资源列表和读取能力
const server = new Server({
  name: "document-server",
  version: "1.0.0"
}, {
  capabilities: {
    resources: {}  // 声明支持资源能力
  }
});

// 实现列出可用资源的处理器
server.setRequestHandler(ListResourcesRequestSchema, async () => {
  return {
    resources: [{
      uri: "file:///docs/example.txt",  // 文档的唯一标识符
      name: "Example Document", 
      mimeType: "text/plain"
    }]
  };
});

// 实现读取资源的处理器
server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
  if (request.params.uri === "file:///docs/example.txt") {
    const content = await readFile("/docs/example.txt");
    return {
      contents: [{
        uri: request.params.uri,
        mimeType: "text/plain",
        text: content
      }]
    };
  }
});
```

2. **交互流程**：

```
用户 -> Claude Desktop(MCP Client) -> MCP Server -> 文件系统
   |                |                      |            |
   |  选择文档        |                      |            |
   |--------------->|                      |            |
   |                | 1.发送resources/list  |            |
   |                |--------------------->|            |
   |                |                      |            |
   |                | 2.返回可用资源列表      |            |
   |                |<---------------------|            |
   |                |                      |            |
   | 确认使用该文档    |                      |            |
   |--------------->|                      |            |
   |                | 3.发送resources/read  |            |
   |                |--------------------->|            |
   |                |                      | 读取文件     |
   |                |                      |----------->|
   |                |                      |            |
   |                | 4.返回文件内容         |            |
   |                |<---------------------|            |
   |                |                      |            |
   | 文档内容作为上下文 |                      |            |
   |<---------------|                      |            |
```

3. **具体步骤**：

a) **初始化阶段**：
- MCP Server 启动并准备好资源访问能力
- Client (Claude Desktop) 与 Server 建立连接

b) **资源发现**：
- Client 调用 `resources/list` 获取可用资源列表
- Server 返回包含该文档在内的资源列表

c) **用户交互**：
- Claude Desktop 向用户展示可用文档
- 用户选择要让模型访问的文档

d) **资源读取**：
- Client 发送 `resources/read` 请求，包含所选文档的 URI
- Server 读取文件内容并返回
- Client 将文档内容作为上下文提供给 Claude

4. **更新机制**（可选）：
```typescript
// 如果文档内容可能发生变化，服务器可以实现更新通知
server.notification({
  method: "notifications/resources/updated",
  params: {
    uri: "file:///docs/example.txt"
  }
});
```

5. **错误处理**：
```typescript
server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
  try {
    // ... 读取文件逻辑 ...
  } catch (error) {
    throw new Error({
      code: ErrorCode.InternalError,
      message: "Failed to read document"
    });
  }
});
```

这种设计确保了：
- 文档访问是受控的（用户必须明确选择）
- 内容传输是标准化的（通过 MCP 协议）
- 系统是可扩展的（可以处理不同类型的文档）
- 操作是安全的（Server 端可以实现访问控制）

这就是为什么它被称为 "application-controlled" - 整个过程都需要应用程序（Claude Desktop）和用户的明确参与，而不是由模型自主决定访问什么资源。

### Tool的交互方式

让我以一个实时股票价格查询的例子来说明如何通过 MCP Tool 来让模型访问动态数据：

1. **MCP Server 端的工具定义**：

```typescript
const server = new Server({
  name: "stock-server",
  version: "1.0.0"
}, {
  capabilities: {
    tools: {}  // 声明支持工具能力
  }
});

// 实现工具列表
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [{
      name: "get_stock_price",
      description: "Get real-time stock price for a given symbol",
      inputSchema: {
        type: "object",
        properties: {
          symbol: { 
            type: "string",
            description: "Stock symbol (e.g. AAPL)" 
          }
        },
        required: ["symbol"]
      }
    }]
  };
});

// 实现工具调用
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (request.params.name === "get_stock_price") {
    const { symbol } = request.params.arguments;
    
    try {
      const price = await fetchStockPrice(symbol);
      return {
        toolResult: {
          price: price,
          timestamp: new Date().toISOString()
        }
      };
    } catch (error) {
      return {
        isError: true,
        content: [{
          type: "text",
          text: `Error fetching price: ${error.message}`
        }]
      };
    }
  }
});
```

2. **交互流程**：

```
用户 -> Claude(MCP Client) -> MCP Server -> 外部API
  |          |                    |             |
  | 提问股价    |                    |             |
  |--------->|                    |             |
  |          | 1.发送tools/list    |             |
  |          |------------------->|             |
  |          |                    |             |
  |          | 2.返回可用工具列表     |             |
  |          |<-------------------|             |
  |          |                    |             |
  |          | 建议使用工具          |             |
  |<---------|                    |             |
  |          |                    |             |
  | 同意使用    |                    |             |
  |--------->|                    |             |
  |          | 3.发送tools/call    |             |
  |          |------------------->|             |
  |          |                    | 获取实时数据    |
  |          |                    |------------>|
  |          |                    |             |
  |          | 4.返回查询结果        |             |
  |          |<-------------------|             |
  |          |                    |             |
  | 收到股价答复 |                    |             |
  |<---------|                    |             |
```

3. **具体用户体验场景**：

```
用户：请告诉我苹果公司的当前股价？

Claude：为了获取苹果公司的实时股价，我可以使用股票价格查询工具。你是否同意我使用这个工具来查询 AAPL 的股价？

用户：好的，同意。

[Claude 调用工具]
MCP Server 执行查询并返回：{ price: 187.42, timestamp: "2024-12-18T17:05:00Z" }

Claude：根据最新数据，苹果公司(AAPL)的当前股价是 187.42 美元，这个数据的时间戳是 2024年12月18日 17:05 UTC。
```

4. **和 Resource 的关键区别**：

- **主动性**：
  - Tool：Claude 可以主动提议使用工具
  - Resource：需要用户先选择资源

- **数据特性**：
  - Tool：适合动态、实时数据
  - Resource：更适合静态或半静态数据

- **交互模式**：
  - Tool：每次使用都需要执行调用
  - Resource：读取后可以持续使用内容

- **参数化**：
  - Tool：可以接受参数来定制查询
  - Resource：通常是预定义的内容

5. **安全考虑**：

```typescript
// 工具调用前的验证
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  // 验证请求参数
  validateInput(request.params.arguments);
  
  // 验证调用频率
  checkRateLimit(request);
  
  // 验证权限
  verifyPermissions(request);
  
  // ... 工具执行逻辑 ...
});
```

这种设计的优势：
- 模型可以根据对话上下文主动提议使用工具
- 用户保持最终控制权（需要同意工具使用）
- 数据始终保持最新（每次调用获取实时数据）
- 可以通过参数自定义查询
- 便于实现访问控制和安全限制

这就是为什么它被称为 "model-controlled" - 模型可以主动提议使用工具，但仍需要用户批准，这样既保持了灵活性又确保了安全性。
## MCP调试
### uvx命令的MCP Server无法启动

在Mac OS中，通过命令安装UV

```bash
curl -LsSf [https://astral.sh/uv/install.sh](https://astral.sh/uv/install.sh) | sh
```

安装的路径为：/Users/{username}/.local/bin  ，Claude MCP Server运行的Path环境变量中，不包含这个路径。所以需要在MCP配置文件中，uvx命令需要写成绝对路径。

这里注意，为了安全和隔离性考虑，Claude APP运行在一个独立的容器或沙盒环境中，这个环境和你的实际终端环境是分开的。

### Log日志的路径

~/Library/logs/Claude/mcp*.log
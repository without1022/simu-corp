# 文档六：MCP Server 开发模板

**文件**：`docs/mcp-server-template.md`  
**状态**：Phase 2 前需要

---

# 模拟公司 (SimuCorp) — MCP Server 开发模板

## 1. 概述

本文档定义了模拟公司 (SimuCorp) 中所有 MCP Server 的标准开发规范。MCP Server 是 Agent 调用外部工具的唯一通道，必须遵守本模板以确保一致性、安全性和可维护性。

## 2. MCP Server 项目结构

```
mcp-servers/<server-name>/
├── package.json              # Node.js 项目配置
├── tsconfig.json             # TypeScript 配置
├── src/
│   ├── index.ts              # 入口：启动 MCP Server
│   ├── server.ts             # MCP Server 实例创建
│   ├── handlers/             # 工具处理函数（每个工具一个文件）
│   │   ├── list-items.ts
│   │   ├── create-item.ts
│   │   └── ...
│   ├── tools.ts              # 工具定义（名称、描述、参数Schema、风险等级）
│   ├── types.ts              # TypeScript 类型定义
│   ├── config.ts             # 配置管理（环境变量读取）
│   └── utils/                # 工具函数
│       ├── logger.ts         # 日志
│       └── validator.ts      # 参数校验
├── tests/                    # 单元测试
│   ├── handlers/
│   └── tools.test.ts
└── README.md                 # 本 Server 的使用说明
```

## 3. 必须实现的标准接口

每个 MCP Server 必须实现以下 MCP 标准方法：

### 3.1 `tools/list`

返回该 Server 提供的所有工具列表。

```typescript
// src/tools.ts
export const tools = [
  {
    name: "tool_name",
    description: "工具的中文描述，说明功能和使用场景",
    inputSchema: {
      type: "object",
      properties: {
        param1: {
          type: "string",
          description: "参数1的描述"
        },
        param2: {
          type: "number",
          description: "参数2的描述"
        }
      },
      required: ["param1"]
    },
    metadata: {
      risk_level: "low",           // low | medium | high | critical
      category: "data",            // code | data | communication | system | custom
      sandbox_recommended: false,  // 是否建议在沙盒中调用
      rate_limit: {
        max_calls_per_minute: 60,
        max_calls_per_hour: 1000
      }
    }
  }
  // ... 更多工具
];
```

### 3.2 `tools/call`

处理工具调用请求。

```typescript
// src/handlers/example-handler.ts
export async function handleToolCall(
  toolName: string,
  args: Record<string, unknown>
): Promise<{ content: Array<{ type: string; text: string }> }> {
  // 1. 参数校验
  // 2. 执行逻辑
  // 3. 错误处理
  // 4. 返回结果
  return {
    content: [{ type: "text", text: JSON.stringify(result) }]
  };
}
```

## 4. 元数据声明规范

每个工具必须在 `metadata` 中声明以下字段：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `risk_level` | enum | 是 | 风险等级：`low`/`medium`/`high`/`critical` |
| `category` | enum | 是 | 工具分类：`code`/`data`/`communication`/`system`/`custom` |
| `sandbox_recommended` | boolean | 是 | 是否建议在沙盒中调用 |
| `rate_limit` | object | 是 | 调用频率限制 |
| `required_permissions` | array | 否 | 所需权限：`file_read`/`file_write`/`network_outbound`/`database_read`/`database_write` |
| `parameters_require_approval` | array | 否 | 哪些参数的值需要额外审批 |

## 5. 错误处理规范

所有工具调用必须返回标准的错误格式：

```typescript
// 成功响应
{
  content: [{ type: "text", text: JSON.stringify({ success: true, data: ... }) }]
}

// 错误响应
{
  content: [{ type: "text", text: JSON.stringify({
    success: false,
    error: {
      code: "ERROR_CODE",
      message: "人类可读的错误描述",
      details: { ... }  // 可选的详细信息
    }
  })]
}
```

**常见错误码**：
- `INVALID_PARAMS` — 参数校验失败
- `AUTH_FAILED` — 认证失败
- `RATE_LIMITED` — 调用频率超限
- `UPSTREAM_ERROR` — 上游服务错误
- `TIMEOUT` — 操作超时
- `PERMISSION_DENIED` — 权限不足

## 6. 日志规范

```typescript
// src/utils/logger.ts
export const logger = {
  info: (msg: string, data?: any) => {
    console.error(JSON.stringify({
      level: "INFO",
      timestamp: new Date().toISOString(),
      server: "server-name",
      message: msg,
      ...data
    }));
  },
  warn: (msg: string, data?: any) => {
    console.error(JSON.stringify({
      level: "WARN",
      timestamp: new Date().toISOString(),
      server: "server-name",
      message: msg,
      ...data
    }));
  },
  error: (msg: string, error?: Error) => {
    console.error(JSON.stringify({
      level: "ERROR",
      timestamp: new Date().toISOString(),
      server: "server-name",
      message: msg,
      error: error?.message,
      stack: error?.stack
    }));
  }
};
```

注意：MCP Server 使用 `stderr` 输出日志（`stdout` 被用于 MCP 协议通信）。

## 7. 配置管理

所有配置通过环境变量读取，不支持硬编码：

```typescript
// src/config.ts
export const config = {
  apiKey: process.env.API_KEY || "",
  baseUrl: process.env.BASE_URL || "https://default.example.com",
  timeout: parseInt(process.env.TIMEOUT || "30000", 10),
  maxRetries: parseInt(process.env.MAX_RETRIES || "3", 10)
};

// 启动时校验必填配置
export function validateConfig(): void {
  const required = ["API_KEY"];
  for (const key of required) {
    if (!process.env[key]) {
      throw new Error(`Missing required environment variable: ${key}`);
    }
  }
}
```

## 8. 注册到 Gateway

开发完成后，在 Gateway 的 `mcp.json` 中注册：

```json
{
  "mcpServers": {
    "your-server-name": {
      "type": "stdio",
      "command": "node",
      "args": ["mcp-servers/your-server-name/dist/index.js"],
      "env": {
        "API_KEY": "${YOUR_API_KEY}",
        "BASE_URL": "https://api.example.com"
      },
      "metadata": {
        "risk_level": "medium",
        "description": "该 Server 的功能描述",
        "maintainer": "开发团队名称"
      }
    }
  }
}
```

## 9. 单元测试模板

```typescript
// tests/handlers/example-handler.test.ts
import { describe, it, expect, vi } from 'vitest';
import { handleToolCall } from '../../src/handlers/example-handler';

describe('example-handler', () => {
  it('should return success for valid params', async () => {
    const result = await handleToolCall('tool_name', { param1: 'test' });
    expect(result.content[0].text).toContain('success');
  });

  it('should return error for missing required param', async () => {
    const result = await handleToolCall('tool_name', {});
    const parsed = JSON.parse(result.content[0].text);
    expect(parsed.success).toBe(false);
    expect(parsed.error.code).toBe('INVALID_PARAMS');
  });
});
```

## 10. 开发Checklist

| 检查项 | 通过标准 |
|--------|---------|
| 实现 `tools/list` | 返回完整的工具定义数组，含 metadata |
| 实现 `tools/call` | 能正确处理有效和无效参数 |
| 参数校验 | 必填参数缺失时返回 `INVALID_PARAMS` 错误 |
| 错误处理 | 上游错误时返回 `UPSTREAM_ERROR`，不暴露内部细节 |
| 日志输出 | 使用 `stderr` 输出结构化 JSON 日志 |
| 环境变量配置 | 所有配置通过环境变量读取，启动时校验 |
| 单元测试 | 核心逻辑覆盖率 > 80% |
| README | 包含功能说明、环境变量列表、使用示例 |
| 注册到 Gateway | `mcp.json` 中添加配置，Gateway 启动后验证连接 |

---


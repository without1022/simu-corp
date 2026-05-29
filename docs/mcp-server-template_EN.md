# Document 6: MCP Server Development Template

**File**: `docs/mcp-server-template.md`  
**Status**: Required before Phase 2

---

# SimuCorp — MCP Server Development Template

## 1. Overview

This document defines the standard development specification for all MCP Servers within SimuCorp. The MCP Server is the sole channel for Agents to invoke external tools and must adhere to this template to ensure consistency, security, and maintainability.

## 2. MCP Server Project Structure

```
mcp-servers/<server-name>/
├── package.json              # Node.js project configuration
├── tsconfig.json             # TypeScript configuration
├── src/
│   ├── index.ts              # Entry point: Start MCP Server
│   ├── server.ts             # MCP Server instance creation
│   ├── handlers/             # Tool handler functions (one file per tool)
│   │   ├── list-items.ts
│   │   ├── create-item.ts
│   │   └── ...
│   ├── tools.ts              # Tool definitions (name, description, parameter schema, risk level)
│   ├── types.ts              # TypeScript type definitions
│   ├── config.ts             # Configuration management (environment variable reading)
│   └── utils/                # Utility functions
│       ├── logger.ts         # Logging
│       └── validator.ts      # Parameter validation
├── tests/                    # Unit tests
│   ├── handlers/
│   └── tools.test.ts
└── README.md                 # Usage instructions for this Server
```

## 3. Mandatory Standard Interfaces

Each MCP Server must implement the following MCP standard methods:

### 3.1 `tools/list`

Returns the list of all tools provided by this Server.

```typescript
// src/tools.ts
export const tools = [
  {
    name: "tool_name",
    description: "Chinese description of the tool, explaining its function and usage scenarios",
    inputSchema: {
      type: "object",
      properties: {
        param1: {
          type: "string",
          description: "Description of parameter 1"
        },
        param2: {
          type: "number",
          description: "Description of parameter 2"
        }
      },
      required: ["param1"]
    },
    metadata: {
      risk_level: "low",           // low | medium | high | critical
      category: "data",            // code | data | communication | system | custom
      sandbox_recommended: false,  // Whether it is recommended to call in a sandbox
      rate_limit: {
        max_calls_per_minute: 60,
        max_calls_per_hour: 1000
      }
    }
  }
  // ... more tools
];
```

### 3.2 `tools/call`

Handles tool invocation requests.

```typescript
// src/handlers/example-handler.ts
export async function handleToolCall(
  toolName: string,
  args: Record<string, unknown>
): Promise<{ content: Array<{ type: string; text: string }> }> {
  // 1. Parameter validation
  // 2. Execute logic
  // 3. Error handling
  // 4. Return result
  return {
    content: [{ type: "text", text: JSON.stringify(result) }]
  };
}
```

## 4. Metadata Declaration Specification

Each tool must declare the following fields in its `metadata`:

| Field | Type | Required | Description |
|------|------|------|------|
| `risk_level` | enum | Yes | Risk level: `low`/`medium`/`high`/`critical` |
| `category` | enum | Yes | Tool category: `code`/`data`/`communication`/`system`/`custom` |
| `sandbox_recommended` | boolean | Yes | Whether it is recommended to call in a sandbox |
| `rate_limit` | object | Yes | Call frequency limit |
| `required_permissions` | array | No | Required permissions: `file_read`/`file_write`/`network_outbound`/`database_read`/`database_write` |
| `parameters_require_approval` | array | No | Which parameter values require additional approval |

## 5. Error Handling Specification

All tool calls must return a standard error format:

```typescript
// Success response
{
  content: [{ type: "text", text: JSON.stringify({ success: true, data: ... }) }]
}

// Error response
{
  content: [{ type: "text", text: JSON.stringify({
    success: false,
    error: {
      code: "ERROR_CODE",
      message: "Human-readable error description",
      details: { ... }  // Optional detailed information
    }
  })]
}
```

**Common Error Codes**:
- `INVALID_PARAMS` — Parameter validation failed
- `AUTH_FAILED` — Authentication failed
- `RATE_LIMITED` — Call frequency exceeded
- `UPSTREAM_ERROR` — Upstream service error
- `TIMEOUT` — Operation timed out
- `PERMISSION_DENIED` — Insufficient permissions

## 6. Logging Specification

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

Note: The MCP Server outputs logs via `stderr` (`stdout` is used for MCP protocol communication).

## 7. Configuration Management

All configuration is read from environment variables; hardcoding is not supported:

```typescript
// src/config.ts
export const config = {
  apiKey: process.env.API_KEY || "",
  baseUrl: process.env.BASE_URL || "https://default.example.com",
  timeout: parseInt(process.env.TIMEOUT || "30000", 10),
  maxRetries: parseInt(process.env.MAX_RETRIES || "3", 10)
};

// Validate required configuration at startup
export function validateConfig(): void {
  const required = ["API_KEY"];
  for (const key of required) {
    if (!process.env[key]) {
      throw new Error(`Missing required environment variable: ${key}`);
    }
  }
}
```

## 8. Registration with Gateway

After development, register in the Gateway's `mcp.json`:

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
        "description": "Description of this Server's functionality",
        "maintainer": "Development team name"
      }
    }
  }
}
```

## 9. Unit Test Template

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

## 10. Development Checklist

| Check Item | Acceptance Criteria |
|--------|---------|
| Implement `tools/list` | Returns a complete array of tool definitions, including metadata |
| Implement `tools/call` | Can correctly handle valid and invalid parameters |
| Parameter validation | Returns `INVALID_PARAMS` error when required parameters are missing |
| Error handling | Returns `UPSTREAM_ERROR` for upstream errors, does not expose internal details |
| Log output | Outputs structured JSON logs using `stderr` |
| Environment variable configuration | All configuration read from environment variables, validated at startup |
| Unit tests | Core logic coverage > 80% |
| README | Includes feature description, environment variable list, usage examples |
| Registration with Gateway | Configuration added to `mcp.json`, connection verified after Gateway startup |

---
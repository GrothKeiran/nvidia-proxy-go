# nvidia-free

免费使用 NVIDIA NIM API 的代理工具，支持 OpenAI Codex CLI 原生接入。

将 OpenAI Responses API 格式转换为 NVIDIA Chat Completions API，通过多 key 轮询 + 全局速率控制规避限流。

## 功能

- **Responses API 转换**：Codex CLI 发送的 `/v1/responses` 请求自动转换为 NVIDIA Chat Completions 格式
- **Tool Calling 支持**：自动转换 Responses API 扁平格式和 Chat Completions 嵌套格式的工具定义
- **流式 SSE 转换**：将 NVIDIA 的 Chat Completions SSE 流转换为 Responses API 标准事件流
- **多 key 轮询**：支持多个 NVIDIA API key，自动轮换，单 key 限流时自动切换
- **全局速率控制**：令牌桶限流器，防止滑动窗口内请求过多触发 429
- **429 自动重试**：限流时自动换 key 重试

## 快速开始

```bash
# 1. 克隆
git clone https://github.com/wuyiliu391-hub/nvidia-free.git
cd nvidia-free

# 2. 配置
cp config.json.example config.json
# 编辑 config.json，填入你的 NVIDIA API key（从 https://build.nvidia.com/ 获取）

# 3. 编译运行
go build -o nvidia-proxy.exe .
./nvidia-proxy.exe
```

## Codex CLI 配置

```bash
export OPENAI_BASE_URL=http://localhost:9099/v1
export OPENAI_API_KEY=any-value
codex
```

## API 端点

| 端点 | 方法 | 说明 |
|------|------|------|
| `/v1/responses` | POST | Responses API（Codex CLI 主路由） |
| `/v1/chat/completions` | POST | Chat Completions（备用） |
| `/v1/models` | GET | 模型列表 |
| `/health` | GET | 健康检查 |
| `/stats` | GET | 代理状态（key 数量、请求数等） |

## 配置说明

`config.json`：
```json
{
  "keys": [
    "nvapi-key1",
    "nvapi-key2",
    "nvapi-key3"
  ]
}
```

- 支持任意数量的 key，来自不同 NVIDIA 账号
- 每个 key 限制 38 次/分钟（留 2 次余量）
- 全局速率 1 次/秒（60 次/分钟）

## 技术细节

### 限流策略

NVIDIA 使用滑动窗口限流，同一秒内大量请求会触发 429。代理通过令牌桶算法控制全局发送速率，确保请求均匀分布。

### 格式转换

```
Codex CLI                    代理                    NVIDIA
  │                           │                       │
  │ POST /v1/responses        │                       │
  │ (Responses API)           │                       │
  │──────────────────────────>│                       │
  │                           │ POST /v1/chat/        │
  │                           │ completions           │
  │                           │ (Chat Completions)    │
  │                           │──────────────────────>│
  │                           │                       │
  │                           │<──────────────────────│
  │                           │ SSE stream            │
  │<──────────────────────────│                       │
  │ SSE stream                │                       │
  │ (Responses API events)    │                       │
```

### Tool Calling 转换

Codex CLI 使用 Responses API 扁平格式的工具定义：
```json
{"type":"function","name":"read_file","description":"...","parameters":{...}}
```

代理将其转换为 Chat Completions 嵌套格式：
```json
{"type":"function","function":{"name":"read_file","description":"...","parameters":{...}}}
```

同时清理 NVIDIA 不支持的字段（`additionalProperties`、`strict`）。

## 许可证

MIT

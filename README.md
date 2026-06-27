# nvidia-free

**免费使用 NVIDIA NIM API 接入 OpenAI Codex CLI 的代理工具。**

完美适配 Codex CLI，支持 Responses API 和原生 Chat Completions API 双接口。

## 功能

- **双 API 支持**：Responses API + 原生 Chat Completions API
- **Codex CLI 完美适配**：自动转换 Responses API 格式
- **Tool Calling 完整支持**：扁平/嵌套两种工具格式，流式/非流式工具调用
- **SSE 流式转换**：NVIDIA Chat Completions SSE → Responses API 标准事件流
- **多 key 轮询**：多个 NVIDIA API key 自动轮换，429 时自动切换
- **全局速率控制**：令牌桶限流器，防止滑动窗口限流
- **沙盒模式修复**：自动解除 Codex 的只读限制
- **429/503 自动重试**：过载时自动等待重试并换 key

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

编辑 `~/.codex/config.toml`：

```toml
model = "z-ai/glm-5.1"
api_base = "http://localhost:9099/v1"
wire_api = "responses"  # 或 "chat"
```

或使用环境变量：

```bash
export OPENAI_BASE_URL=http://localhost:9099/v1
export OPENAI_API_KEY=any-value
codex
```

## API 端点

| 端点 | 方法 | 说明 |
|------|------|------|
| `/v1/responses` | POST | Responses API（Codex 默认） |
| `/v1/chat/completions` | POST | Chat Completions API（原生） |
| `/v1/models` | GET | 模型列表 |
| `/health` | GET | 健康检查 |
| `/stats` | GET | 代理状态（key 数量、请求数等） |

## 支持的模型

| 模型 | 说明 |
|------|------|
| `z-ai/glm-5.1` | 智谱 GLM-5.1（默认） |
| `moonshotai/kimi-k2.5` | 月之暗面 Kimi |
| `deepseek/deepseek-r1` | DeepSeek R1 |
| `qwen/qwen3-235b-a22b` | 通义千问 3 |

## 配置说明

`config.json`：
```json
{
  "keys": [
    "nvapi-key1",
    "nvapi-key2",
    "nvapi-key3"
  ],
  "default_model": "z-ai/glm-5.1"
}
```

- 支持任意数量的 key，来自不同 NVIDIA 账号
- 每个 key 限制 30 次/分钟
- 全局速率 2 次/秒（120 次/分钟）

## 速率限制说明

| Key 数量 | 总容量 | 429 概率 |
|---------|--------|---------|
| 1 个 | 30/分钟 | 高 |
| 5 个 | 150/分钟 | 低 |
| 10 个 | 300/分钟 | 很低 |
| 20 个 | 600/分钟 | 几乎没有 |

## 工作原理

### Responses API 模式（Codex 默认）
```
Codex CLI → Responses API → 代理转换 → Chat Completions API → NVIDIA NIM
    ↑                                                              ↓
    ←────── Responses API ←── 代理转换 ←── Chat Completions ←─────┘
```

### Chat Completions API 模式（原生）
```
客户端 → Chat Completions API → 代理转发 → NVIDIA NIM
    ↑                                          ↓
    ←────── Chat Completions ←─────────────────┘
```

## 更新历史

### v1.2 (2026-06-27)

- ✨ 新增原生 Chat Completions API 端点
- 🔧 优化速率限制（全局 2/秒，避免 429）
- 🔧 429/503 自动重试并换 key
- 🔧 thinking/reasoning_effort 参数自动剥离
- 🔧 sandbox_mode 自动提升为 elevated

### v1.1 (2026-06-27)

**核心修复：Codex 工具调用不工作**

- ✅ 修复 `arguments` 字段读取错误
- ✅ 修复沙盒模式限制
- ✅ 修复工具格式兼容性
- ✅ 修复模型名映射缺失
- ✅ 新增 503 自动重试

### v1.0 (2026-06-26)

初始版本：基本的 Responses API ↔ Chat Completions 转换。

## 许可证

MIT

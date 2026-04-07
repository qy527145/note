# 开发工具链配置指南
## 一、基础工具安装
### 1.1 bun（JavaScript 运行时）
**安装命令（PowerShell）**：
```powershell
powershell -c "irm bun.sh/install.ps1 | iex"
```

### 1.2 uv（Python 包管理工具）
**安装命令（PowerShell）**：
```powershell
# 安装 uv 核心工具
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
# 安装指定版本 Python（3.13）
uv python install 3.13
```

## 二、API 接口前置说明
### 2.1 OpenAI Chat Completions API
- **请求方式**：POST
- **请求地址**：`<base_url>/v1/chat/completions`
- **请求头**：
  ```
  Authorization: Bearer <your api_key>
  Content-Type: application/json
  ```
- **请求体示例**：
```json
{
    "model": "gpt-5",
    "messages": [
        {
            "role": "user",
            "content": "hi"
        }
    ]
}
```

### 2.2 OpenAI Responses API
- **请求方式**：POST
- **请求地址**：`<base_url>/v1/responses`
- **请求头**：
  ```
  Authorization: Bearer <your api_key>
  Content-Type: application/json
  ```
- **请求体示例**：
```json
{
    "model": "gpt-5",
    "input": "你能干什么？",
    "text": {
        "verbosity": "low"
    },
    "reasoning": {
        "effort": "low",
        "summary": "auto"
    },
    "temperature": 1,
    "top_p": 1
}
```

### 2.3 Anthropic Messages API
- **请求方式**：POST
- **请求地址**：`<base_url>/v1/messages`
- **请求头**：
  ```
  Authorization: Bearer <your api_key>
  Content-Type: application/json
  ```
- **请求体示例**：
```json
{
    "model": "claude-haiku-4-5",
    "messages": [
        {
            "role": "user",
            "content": "你好"
        }
    ]
}
```

## 三、OpenCode 配置
### 3.1 安装
```powershell
bun i -g opencode-ai
```

### 3.2 核心配置
#### 配置文件 1：`~/.config/opencode/opencode.jsonc`
```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "disabled_providers": [],
  "provider": {
    "newapi": {
      "name": "newapi",
      // 使用OpenAI Chat Completions API
      "npm": "@ai-sdk/openai-compatible",
      "models": {
        "gpt-4o": {
          "name": "gpt-4o"
        }
      },
      "options": {
        "baseURL": "<your base_url>/v1" // 替换为实际 API 地址
      }
    },
    "azure": {
      "name": "azure",
      // 使用OpenAI Responses API
      "npm": "@ai-sdk/openai",
      "models": {
        "gpt-5.3-codex": {
          "name": "gpt-5.3-codex"
        },
        "gpt-5.2": {
          "name": "gpt-5.2"
        }
      },
      "options": {
        "baseURL": "<your base_url>/v1" // 替换为实际 API 地址
      }
    },
    "anthropic": {
      "name": "anthropic",
      "options": {
        "baseURL": "<your base_url>/v1" // 替换为实际 API 地址
      }
    }
  }
}
```

#### 配置文件 2：`~/.local/share/opencode/auth.json`
```json
{
  "newapi": {
    "type": "api",
    "key": "<your api_key>" // 替换为实际 API Key
  },
  "azure": {
    "type": "api",
    "key": "<your api_key>" // 替换为实际 API Key
  }
}
```

### 3.3 JetBrains 集成配置
配置文件：`~/.jetbrains/acp.json`
```json
{
  "agent_servers": {
    "OpenCode": {
      "command": "opencode",
      "args": ["acp"]
    }
  }
}
```

## 四、Codex 配置
### 4.1 安装
```powershell
bun i -g @openai/codex
```

### 4.2 核心配置
配置文件：`~/.codex/config.toml`
```toml
profile = "profile1"

[profiles.profile1]
model_provider = "newapi"
model = "gpt-5.4"
windows_wsl_setup_acknowledged = true
model_reasoning_effort = "medium"
disable_response_storage = false
personality = "pragmatic"

[model_providers.newapi]
name = "New API"
base_url = "<your base_url>/v1" # 替换为实际 API 地址
wire_api = "responses"
# env_key = "NEW_API_KEY"
experimental_bearer_token="<your api_key>"

[windows]
sandbox = "elevated"
```

**重要说明**：
- 需先配置环境变量 `NEW_API_KEY=<your api_key>` 后再使用 Codex
- Windows 可通过 `set NEW_API_KEY=<your api_key>` 临时设置，或在系统环境变量中永久配置
- 临时使用（不推荐）：直接将api_key设置到experimental_bearer_token属性

## 五、Claude 配置
### 5.1 安装（VS Code 扩展）
```powershell
# 1、在vscode中安装claude插件
# 2、进入 Claude 扩展的原生二进制目录
cd ~/.vscode/extensions/anthropic.claude-code-2.1.76-win32-x64/resources/native-binary
# 3、执行安装
./claude.exe install
```

### 5.2 核心配置
#### 配置文件 1：`~/.claude/settings.json`
```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "apiKeyHelper": "echo <your api_key>", // 替换为实际 API Key
  "env": {
    "ANTHROPIC_BASE_URL": "<your base_url>", // 替换为实际 API 地址
    "ANTHROPIC_MODEL": "claude-opus-4-5-20251101",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "claude-opus-4-5-20251101",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-4-5-20250929",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "claude-haiku-4-5-20251001",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1"
  },
  "permissions": {
    "allow": [
      "mcp__acemcp",
      "mcp__cunzhi"
    ],
    "deny": [],
    "defaultMode": "bypassPermissions"
  },
  "enableAllProjectMcpServers": true,
  "skipDangerousModePermissionPrompt": true
}
```

#### 配置文件 2：`~/.claude.json`
```json
{
  "mcpServers": {
    "acemcp": {
      "command": "uvx",
      "args": [
        "acemcp",
        "--web-port",
        "8888"
      ]
    },
    "cunzhi": {
      "command": "寸止"
    }
  }
}
```

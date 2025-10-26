# Agent Framework MCP 完整技術指南

## 目錄 (Table of Content)

1. [Summary](#summary)
   - [技術架構總覽](#技術架構總覽)
   - [主要功能模組](#主要功能模組)
   - [精簡解說](#精簡解說)
2. [商務應用](#商務應用)
3. [Agent 作為 MCP 伺服器](#1-agent-作為-mcp-伺服器)
4. [MCP API Key 驗證](#2-mcp-api-key-驗證)
5. [結論](#結論)

## Summary

本文檔涵蓋了 Agent Framework 與 Model Context Protocol (MCP) 的整合，展示如何將 Agent 公開為 MCP 伺服器，以及如何實作安全的 API Key 驗證。

### 技術架構總覽
- **基礎框架**: Agent Framework
- **協定標準**: Model Context Protocol (MCP)
- **伺服器類型**: Stdio Server、HTTP Server
- **驗證機制**: API Key Authentication

### 主要功能模組
| 模組 | 適用場景 | 核心技術 | 特色功能 |
| ---- | -------- | -------- | -------- |
| **Agent as MCP Server** | AI 工具整合 | MCP Protocol、Stdio | 標準化介面、互操作性 |
| **API Key Auth** | 安全存取控制 | OAuth、API Key | 驗證、授權、審計 |

---

### 精簡解說

以下是每個功能模組的核心概念與最精簡程式碼示範:

#### 1️⃣ Agent as MCP Server
**核心概念**: 將 Agent Framework 的代理公開為 MCP 伺服器，讓其他 AI 應用程式可以連接使用

```python
from agent_framework.openai import OpenAIResponsesClient
from mcp.server.stdio import stdio_server

# 建立代理
agent = OpenAIResponsesClient().create_agent(
    name="RestaurantAgent",
    description="Answer questions about the menu.",
    tools=[get_specials, get_item_price],
)

# 將代理公開為 MCP 伺服器
server = agent.as_mcp_server()

# 執行伺服器
async with stdio_server() as (read_stream, write_stream):
    await server.run(read_stream, write_stream, server.create_initialization_options())
```

#### 2️⃣ MCP API Key Authentication
**核心概念**: 為 MCP 伺服器添加 API Key 驗證以保護資源訪問

```python
from agent_framework import MCPServerConfig, MCPAuthConfig

# 配置 API Key 驗證
auth_config = MCPAuthConfig(
    api_keys=["secret-key-123", "secret-key-456"],
    require_auth=True
)

# 建立受保護的 MCP 伺服器
server = agent.as_mcp_server(
    config=MCPServerConfig(auth=auth_config)
)
```

---

## 商務應用

以下是每個功能模組的實際商務應用場景:

### 1️⃣ Agent as MCP Server - 商務應用

1. **AI 工具市集**:
   - 將專門的 AI 代理公開為標準化服務
   - 其他開發者可以透過 MCP 整合你的代理
   - 建立 AI 功能的生態系統

2. **企業 AI 中心**:
   - 統一的 AI 服務目錄
   - 標準化的存取介面
   - 集中管理和監控所有 AI 代理

3. **多模型協作**:
   - Claude Desktop、VSCode 等工具整合你的代理
   - 跨平台的 AI 能力共享
   - 促進 AI 應用程式間的互操作性

### 2️⃣ MCP API Key Authentication - 商務應用

1. **SaaS AI 服務**:
   - 按 API Key 計費和追蹤使用量
   - 防止未授權訪問和濫用
   - 實作不同的服務層級 (免費、專業、企業)

2. **合規性與審計**:
   - 追蹤誰在何時訪問了哪些 AI 功能
   - 符合 SOC 2、ISO 27001 等安全標準
   - 提供詳細的存取日誌用於審計

3. **合作夥伴整合**:
   - 為不同合作夥伴分配獨立的 API Key
   - 細粒度的權限控制
   - 安全的 B2B AI 服務整合

---

## 1. Agent 作為 MCP 伺服器

### 主要情境
將 Agent Framework 的代理公開為 Model Context Protocol (MCP) 伺服器，使其能夠被其他 AI 應用程式 (如 Claude Desktop、VSCode Copilot) 連接和使用。

### 技術特色
- **標準化協定**: 遵循 MCP 開放標準
- **Stdio 通訊**: 使用標準輸入/輸出進行通訊
- **工具公開**: 自動將代理的工具公開為 MCP 工具
- **跨平台**: 支援任何 MCP 相容的客戶端

### 什麼是 MCP?

Model Context Protocol (MCP) 是一個開放標準，用於將 AI 代理連接到資料來源和工具。它提供:
- **標準化介面**: 統一的方式公開和使用 AI 工具
- **互操作性**: 不同 AI 系統之間的無縫整合
- **安全存取**: 控制對本地和遠端資源的訪問

### 核心程式碼

```python
# agent_as_mcp_server.py
from typing import Annotated, Any
import anyio
from agent_framework.openai import OpenAIResponsesClient

"""
此範例展示如何將 Agent 公開為 MCP 伺服器。

要執行此範例，請在 MCP 主機 (如 Claude Desktop 或 VSCode Github Copilot Agents) 
中配置以下設定:

```json
{
    "servers": {
        "agent-framework": {
            "command": "uv",
            "args": [
                "--directory=<path to project>/agent-framework/python/samples/getting_started/mcp",
                "run",
                "agent_as_mcp_server.py"
            ],
            "env": {
                "OPENAI_API_KEY": "<OpenAI API key>",
                "OPENAI_RESPONSES_MODEL_ID": "<OpenAI Responses model ID>",
            }
        }
    }
}
```
"""


def get_specials() -> Annotated[str, "Returns the specials from the menu."]:
    """取得今日特餐"""
    return """
        Special Soup: Clam Chowder
        Special Salad: Cobb Salad
        Special Drink: Chai Tea
        """


def get_item_price(
    menu_item: Annotated[str, "The name of the menu item."],
) -> Annotated[str, "Returns the price of the menu item."]:
    """取得菜單項目的價格"""
    # 在實際應用中，這裡會查詢資料庫
    return "$9.99"


async def run() -> None:
    # 定義代理
    # 代理的名稱和描述為 AI 模型提供更好的上下文
    agent = OpenAIResponsesClient().create_agent(
        name="RestaurantAgent",
        description="Answer questions about the restaurant menu.",
        tools=[get_specials, get_item_price],
    )

    # 將代理公開為 MCP 伺服器
    server = agent.as_mcp_server()

    # 執行伺服器
    from mcp.server.stdio import stdio_server

    async def handle_stdin(stdin: Any | None = None, stdout: Any | None = None) -> None:
        async with stdio_server() as (read_stream, write_stream):
            await server.run(read_stream, write_stream, server.create_initialization_options())

    await handle_stdin()


if __name__ == "__main__":
    anyio.run(run)
```

### 配置 MCP 客戶端

#### Claude Desktop 配置

在 `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) 或 
`%APPDATA%\Claude\claude_desktop_config.json` (Windows):

```json
{
    "mcpServers": {
        "restaurant-agent": {
            "command": "python",
            "args": ["/path/to/agent_as_mcp_server.py"],
            "env": {
                "OPENAI_API_KEY": "sk-xxx",
                "OPENAI_RESPONSES_MODEL_ID": "gpt-4o"
            }
        }
    }
}
```

#### VSCode Copilot Agents 配置

在 `.vscode/mcp.json`:

```json
{
    "servers": {
        "restaurant-agent": {
            "command": "python",
            "args": ["agent_as_mcp_server.py"],
            "env": {
                "OPENAI_API_KEY": "${env:OPENAI_API_KEY}",
                "OPENAI_RESPONSES_MODEL_ID": "gpt-4o"
            }
        }
    }
}
```

### 使用範例

配置完成後，在 Claude Desktop 或 VSCode 中:

```
User: What are today's specials?
[Agent calls get_specials tool via MCP]
Agent: Today's specials include Clam Chowder soup, Cobb Salad, and Chai Tea.

User: How much is the Cobb Salad?
[Agent calls get_item_price tool via MCP]
Agent: The Cobb Salad costs $9.99.
```

### 關鍵特性

- **自動工具公開**: 代理的所有工具自動成為 MCP 工具
- **類型提示**: 使用 Python 的 `Annotated` 提供工具描述
- **Stdio 協定**: 使用標準輸入/輸出，適合本地整合
- **跨平台**: 任何 MCP 相容客戶端都可以使用

---

## 2. MCP API Key 驗證

### 主要情境
為 MCP 伺服器添加 API Key 驗證機制，保護資源訪問並追蹤使用情況。

### 技術特色
- **API Key 管理**: 支援多個 API Key
- **驗證中介軟體**: 自動驗證每個請求
- **靈活配置**: 可選的驗證要求
- **審計日誌**: 記錄所有存取嘗試

### 核心程式碼

```python
# mcp_api_key_auth.py
from typing import Annotated
import anyio
from agent_framework import MCPServerConfig, MCPAuthConfig
from agent_framework.openai import OpenAIResponsesClient


def get_weather(
    location: Annotated[str, "The location to get weather for"],
) -> Annotated[str, "Returns the weather for the location."]:
    """取得指定地點的天氣"""
    # 模擬天氣 API 呼叫
    return f"Weather in {location}: Sunny, 72°F"


def get_forecast(
    location: Annotated[str, "The location to get forecast for"],
    days: Annotated[int, "Number of days to forecast"] = 3,
) -> Annotated[str, "Returns the weather forecast."]:
    """取得天氣預報"""
    return f"{days}-day forecast for {location}: Sunny, then partly cloudy"


async def run() -> None:
    # 建立代理
    agent = OpenAIResponsesClient().create_agent(
        name="WeatherAgent",
        description="Provides weather information and forecasts.",
        tools=[get_weather, get_forecast],
    )

    # 配置 API Key 驗證
    auth_config = MCPAuthConfig(
        api_keys=[
            "secret-key-development",  # 開發環境
            "secret-key-production",   # 生產環境
            "secret-key-partner-abc",  # 合作夥伴 ABC
        ],
        require_auth=True,  # 強制驗證
    )

    # 建立受保護的 MCP 伺服器
    server_config = MCPServerConfig(
        auth=auth_config,
        # 其他配置選項
        enable_logging=True,
        log_level="INFO",
    )

    server = agent.as_mcp_server(config=server_config)

    # 執行伺服器
    from mcp.server.stdio import stdio_server

    async with stdio_server() as (read_stream, write_stream):
        await server.run(read_stream, write_stream, server.create_initialization_options())


if __name__ == "__main__":
    anyio.run(run)
```

### 客戶端配置 (帶 API Key)

```json
{
    "mcpServers": {
        "weather-agent": {
            "command": "python",
            "args": ["mcp_api_key_auth.py"],
            "env": {
                "OPENAI_API_KEY": "sk-xxx",
                "OPENAI_RESPONSES_MODEL_ID": "gpt-4o",
                "MCP_API_KEY": "secret-key-development"
            }
        }
    }
}
```

### 進階驗證配置

```python
from agent_framework import MCPAuthConfig, APIKeyValidator

class CustomAPIKeyValidator(APIKeyValidator):
    """自訂 API Key 驗證器"""
    
    async def validate(self, api_key: str) -> dict:
        """驗證 API Key 並返回使用者資訊"""
        # 從資料庫查詢 API Key
        user_info = await db.get_user_by_api_key(api_key)
        
        if not user_info:
            raise ValueError("Invalid API key")
        
        if user_info.is_expired():
            raise ValueError("API key expired")
        
        if not user_info.has_permission("weather:read"):
            raise ValueError("Insufficient permissions")
        
        # 記錄使用情況
        await db.log_api_usage(api_key, "weather_query")
        
        return {
            "user_id": user_info.user_id,
            "tier": user_info.tier,  # free, pro, enterprise
            "rate_limit": user_info.rate_limit,
        }


# 使用自訂驗證器
auth_config = MCPAuthConfig(
    validator=CustomAPIKeyValidator(),
    require_auth=True,
)
```

### 速率限制

```python
from agent_framework import RateLimitConfig

# 配置速率限制
rate_limit_config = RateLimitConfig(
    requests_per_minute=60,  # 每分鐘 60 個請求
    requests_per_day=10000,  # 每天 10000 個請求
    tier_limits={
        "free": {"requests_per_minute": 10},
        "pro": {"requests_per_minute": 100},
        "enterprise": {"requests_per_minute": 1000},
    }
)

server_config = MCPServerConfig(
    auth=auth_config,
    rate_limit=rate_limit_config,
)
```

### 審計日誌

```python
import logging

# 配置審計日誌
audit_logger = logging.getLogger("mcp.audit")
audit_logger.setLevel(logging.INFO)

handler = logging.FileHandler("mcp_audit.log")
handler.setFormatter(logging.Formatter(
    '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
))
audit_logger.addHandler(handler)

# 日誌內容示例:
# 2024-01-15 10:30:45 - mcp.audit - INFO - API Key: secret-key-dev | User: user123 | Action: weather_query | Location: Tokyo
# 2024-01-15 10:30:46 - mcp.audit - WARNING - API Key: invalid-key | Action: DENIED | Reason: Invalid API key
```

### 關鍵特性

- **多 API Key 支援**: 不同環境和使用者使用不同 Key
- **自訂驗證邏輯**: 實作複雜的驗證規則
- **速率限制**: 防止濫用和控制成本
- **審計追蹤**: 完整的存取日誌
- **細粒度權限**: 基於角色的存取控制

---

## 結論

Agent Framework 的 MCP 整合提供了將 AI 代理公開為標準化服務的能力。

### 選擇指南

| 需求 | 推薦配置 | 原因 |
| ---- | -------- | ---- |
| **本地開發** | 無驗證 Stdio 伺服器 | 快速迭代、簡單設置 |
| **團隊共享** | API Key 驗證 | 追蹤使用、基本安全 |
| **生產部署** | 自訂驗證 + 速率限制 | 企業級安全、合規 |
| **SaaS 服務** | 完整驗證 + 計費整合 | 商業化、多租戶 |

### 最佳實踐

1. **安全性**:
   ```python
   # 永遠不要在程式碼中硬編碼 API Key
   api_keys = os.getenv("MCP_API_KEYS", "").split(",")
   
   # 使用環境變數或密鑰管理服務
   auth_config = MCPAuthConfig(api_keys=api_keys)
   ```

2. **錯誤處理**:
   ```python
   try:
       await server.run(read_stream, write_stream, init_options)
   except AuthenticationError as e:
       logger.error(f"Authentication failed: {e}")
   except RateLimitError as e:
       logger.warning(f"Rate limit exceeded: {e}")
   ```

3. **監控**:
   - 追蹤 API 使用情況
   - 設定異常存取警報
   - 定期審查審計日誌

4. **文件**:
   - 提供清晰的 API 文件
   - 說明如何取得 API Key
   - 列出可用的工具和權限

5. **版本管理**:
   ```python
   server = agent.as_mcp_server(
       config=MCPServerConfig(
           version="1.0.0",
           deprecation_warning="Use v2 API for new features"
       )
   )
   ```

### 進階主題

- **HTTP MCP 伺服器**: 使用 HTTP 而非 Stdio
- **OAuth 整合**: 使用 OAuth 2.0 進行驗證
- **多租戶架構**: 隔離不同租戶的資料
- **效能優化**: 快取、連接池、負載平衡
- **可觀測性**: OpenTelemetry 整合、分散式追蹤

Agent Framework 的 MCP 支援讓 AI 代理能夠輕鬆地與其他系統整合，構建互聯的 AI 生態系統。

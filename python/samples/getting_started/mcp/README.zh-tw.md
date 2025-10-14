# MCP (Model Context Protocol) 範例

此資料夾包含展示如何使用 Agent Framework 搭配 MCP 的範例。

## 什麼是 MCP？

Model Context Protocol (MCP) 是一個開放標準，用於將 AI 代理連接到資料來源和工具。它透過標準化協定實現對本地和遠端資源的安全、可控訪問。

## 範例

| 範例 | 檔案 | 描述 |
|--------|------|-------------|
| **Agent 作為 MCP 伺服器** | [`agent_as_mcp_server.py`](agent_as_mcp_server.py) | 展示如何將 Agent Framework 代理公開為 MCP 伺服器，以便其他 AI 應用程式可以連接 |
| **API 金鑰驗證** | [`mcp_api_key_auth.py`](mcp_api_key_auth.py) | 展示與 MCP 伺服器的 API 金鑰驗證 |

## 先決條件

- `OPENAI_API_KEY` 環境變數
- `OPENAI_RESPONSES_MODEL_ID` 環境變數

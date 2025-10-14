# Agent 範例

此資料夾包含展示如何使用 Agent Framework 的不同聊天客戶端建立和使用代理的範例。每個子資料夾專注於特定的提供者和客戶端類型，展示各種功能，如函式工具、程式碼直譯器、執行緒管理、結構化輸出、影像處理、網頁搜尋、Model Context Protocol (MCP) 整合等。

## 依提供者分類的範例

### Azure AI Foundry 範例

| 資料夾 | 描述 |
|--------|-------------|
| **[`azure_ai/`](azure_ai/)** | 使用 Azure AI Foundry Agent Service 建立代理，具有各種工具，包括函式工具、程式碼直譯器、MCP 整合和執行緒管理 |

### Microsoft Copilot Studio 範例

| 資料夾 | 描述 |
|--------|-------------|
| **[`copilotstudio/`](copilotstudio/)** | 使用 Microsoft Copilot Studio 建立代理，具有串流和非串流回應、驗證處理和明確配置選項 |

### Azure OpenAI 範例

| 資料夾 | 描述 |
|--------|-------------|
| **[`azure_openai/`](azure_openai/)** | 使用 Azure OpenAI API 建立代理，支援多種客戶端類型（Assistants、Chat 和 Responses 客戶端），支援函式工具、程式碼直譯器、執行緒管理等 |

### OpenAI 範例

| 資料夾 | 描述 |
|--------|-------------|
| **[`openai/`](openai/)** | 使用 OpenAI API 建立代理，具有全面的範例，包括 Assistants、Chat 和 Responses 客戶端，具備函式工具、程式碼直譯器、檔案搜尋、網頁搜尋、MCP 整合、影像分析/生成、結構化輸出、推理和執行緒管理 |

### Anthropic 範例

| 資料夾 | 描述 |
|--------|-------------|
| **[`anthropic/`](anthropic/)** | 透過 OpenAI Chat Client 配置使用 Anthropic 模型建立代理，展示工具呼叫功能 |

### 自訂實作範例

| 資料夾 | 描述 |
|--------|-------------|
| **[`custom/`](custom/)** | 透過擴展基礎框架類別建立自訂代理和聊天客戶端，展示對代理行為和後端整合的完全控制 |

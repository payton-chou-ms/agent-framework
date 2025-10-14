# DevUI 範例

此資料夾包含設計用於與 Agent Framework DevUI 搭配使用的範例代理和工作流程 - 一個用於互動式執行和測試代理的輕量級網頁介面。

## 什麼是 DevUI？

DevUI 是一個範例應用程式，提供：

- 用於測試代理和工作流程的網頁介面
- OpenAI 相容的 API 端點
- 基於目錄的實體發現
- 記憶體內實體註冊
- 範例實體庫

> **注意**：DevUI 是用於開發和測試的範例應用程式。對於生產使用，請使用 Agent Framework SDK 建構您自己的自訂介面。

## 快速開始

### 選項 1：記憶體模式（最簡單）

直接執行單一範例。這展示如何以程式化方式包裝代理和工作流程，而無需目錄結構：

```bash
cd python/samples/getting_started/devui
python in_memory_mode.py
```

這會在 http://localhost:8090 開啟您的瀏覽器，其中包含預先配置的代理和基本工作流程。

### 選項 2：目錄發現

啟動 DevUI 以發現此資料夾中的所有範例：

```bash
cd python/samples/getting_started/devui
devui
```

這會在 http://localhost:8080 啟動伺服器，並提供所有可用的代理和工作流程。

## 範例結構

每個代理/工作流程遵循 DevUI 發現系統所需的嚴格結構：

```
agent_name/
├── __init__.py      # 必須匯出：agent = ChatAgent(...)
├── agent.py         # 代理實作
└── .env.example     # 範例環境變數
```

## 可用範例

### 代理

| 範例 | 描述 | 功能 | 所需環境變數 |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| [**weather_agent_azure/**](weather_agent_azure/) | 使用 Azure OpenAI 和 API 金鑰驗證的天氣代理 | Azure OpenAI 整合、函式呼叫、模擬天氣工具 | `AZURE_OPENAI_API_KEY`、`AZURE_OPENAI_CHAT_DEPLOYMENT_NAME`、`AZURE_OPENAI_ENDPOINT` |
| [**foundry_agent/**](foundry_agent/) | 使用 Azure AI Agent (Foundry) 和 Azure CLI 驗證的天氣代理（先執行 `az login`） | Azure AI Agent 整合、Azure CLI 驗證、模擬天氣工具 | `AZURE_AI_PROJECT_ENDPOINT`、`FOUNDRY_MODEL_DEPLOYMENT_NAME` |
| [**openai_agent/**](openai_agent/) | 使用 OpenAI API 的數學解題代理 | OpenAI 整合、函式呼叫、計算器工具 | `OPENAI_API_KEY` |

### 工作流程

| 範例 | 描述 | 功能 | 所需環境變數 |
| ---------------------------------------------------- | ---------------------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| [**basic_workflow/**](basic_workflow/) | 使用 Azure OpenAI 的簡單寫作者-編輯者工作流程 | 工作流程編排、代理鏈接、Azure OpenAI 整合 | `AZURE_OPENAI_ENDPOINT`、`AZURE_OPENAI_CHAT_DEPLOYMENT_NAME` |

## 環境設置

1. 從對應的 `.env.example` 檔案建立 `.env` 檔案：
   ```bash
   cp weather_agent_azure/.env.example weather_agent_azure/.env
   ```

2. 使用您的實際值填寫 `.env` 檔案

3. 根據需要對其他範例重複此操作

## 需求

安裝 DevUI 套件：

```bash
pip install agent-framework-devui
```

## 進階用法

### 在記憶體模式中自訂

編輯 `in_memory_mode.py` 以：
- 新增您自己的代理
- 建立自訂工作流程
- 更改伺服器配置

### 建立您自己的範例

1. 建立一個新資料夾，遵循必要的結構
2. 實作您的代理或工作流程
3. 新增 `.env.example` 作為範本
4. 執行 `devui` 以自動發現

## 故障排除

**問題**：代理未出現在 DevUI 中
- 檢查資料夾結構是否正確
- 確保 `__init__.py` 匯出 `agent` 變數
- 驗證所有必需的環境變數已設置

**問題**：驗證錯誤
- 對於 Azure：執行 `az login`
- 檢查 API 金鑰是否有效
- 驗證端點 URL 是否正確

## 參考資料

- [Agent Framework 文件](../../../README.md)
- [DevUI 套件文件](../../../../packages/devui/README.md)

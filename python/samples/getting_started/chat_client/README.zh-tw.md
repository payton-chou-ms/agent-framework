# Chat Client 範例

此資料夾包含展示直接使用各種聊天客戶端的簡單範例。

## 範例

| 檔案 | 描述 |
|------|-------------|
| [`azure_assistants_client.py`](azure_assistants_client.py) | 直接使用 Azure Assistants Client 與 Azure OpenAI assistants 進行基本聊天互動。 |
| [`azure_chat_client.py`](azure_chat_client.py) | 直接使用 Azure Chat Client 與 Azure OpenAI 模型進行聊天互動。 |
| [`azure_responses_client.py`](azure_responses_client.py) | 直接使用 Azure Responses Client 與 Azure OpenAI 模型進行結構化回應生成。 |
| [`chat_response_cancellation.py`](chat_response_cancellation.py) | 展示如何在串流期間取消聊天回應，展示適當的取消處理和清理。 |
| [`azure_ai_chat_client.py`](azure_ai_chat_client.py) | 直接使用 Azure AI Chat Client 與 Azure AI 模型進行聊天互動。 |
| [`openai_assistants_client.py`](openai_assistants_client.py) | 直接使用 OpenAI Assistants Client 與 OpenAI assistants 進行基本聊天互動。 |
| [`openai_chat_client.py`](openai_chat_client.py) | 直接使用 OpenAI Chat Client 與 OpenAI 模型進行聊天互動。 |
| [`openai_responses_client.py`](openai_responses_client.py) | 直接使用 OpenAI Responses Client 與 OpenAI 模型進行結構化回應生成。 |

## 環境變數

根據您使用的客戶端，設置適當的環境變數：

**對於 Azure 客戶端：**
- `AZURE_OPENAI_ENDPOINT`：您的 Azure OpenAI 端點
- `AZURE_OPENAI_CHAT_DEPLOYMENT_NAME`：您的 Azure OpenAI 聊天部署名稱
- `AZURE_OPENAI_RESPONSES_DEPLOYMENT_NAME`：您的 Azure OpenAI responses 部署名稱

**對於 Azure AI 客戶端：**
- `AZURE_AI_PROJECT_ENDPOINT`：您的 Azure AI 專案端點
- `AZURE_AI_MODEL_DEPLOYMENT_NAME`：您的模型部署名稱

**對於 OpenAI 客戶端：**
- `OPENAI_API_KEY`：您的 OpenAI API 金鑰
- `OPENAI_CHAT_MODEL_ID`：用於聊天客戶端的 OpenAI 模型（例如 `gpt-4o`、`gpt-4o-mini`、`gpt-3.5-turbo`）
- `OPENAI_RESPONSES_MODEL_ID`：用於 responses 客戶端的 OpenAI 模型（例如 `gpt-4o`、`gpt-4o-mini`、`gpt-3.5-turbo`）

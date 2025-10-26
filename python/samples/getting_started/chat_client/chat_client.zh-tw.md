# Agent Framework Chat Client 完整技術指南

## 目錄 (Table of Content)

1. [Summary](#summary)
   - [技術架構總覽](#技術架構總覽)
   - [主要功能模組](#主要功能模組)
   - [精簡解說](#精簡解說)
2. [商務應用](#商務應用)
3. [Azure Assistants Client](#1-azure-assistants-client)
4. [Azure Chat Client](#2-azure-chat-client)
5. [Azure Responses Client](#3-azure-responses-client)
6. [Azure AI Chat Client](#4-azure-ai-chat-client)
7. [OpenAI Assistants Client](#5-openai-assistants-client)
8. [OpenAI Chat Client](#6-openai-chat-client)
9. [OpenAI Responses Client](#7-openai-responses-client)
10. [Chat Response Cancellation](#8-chat-response-cancellation)
11. [結論](#結論)

## Summary

本文檔涵蓋了 Agent Framework 中各種聊天客戶端 (Chat Client) 的直接使用方式，從 Azure OpenAI 到 OpenAI，以及 Assistants、Chat 和 Responses 三種不同的客戶端類型。

### 技術架構總覽
- **基礎框架**: Agent Framework
- **支援平台**: Azure OpenAI、Azure AI、OpenAI
- **客戶端類型**: Assistants Client、Chat Client、Responses Client
- **驗證方式**: Azure CLI Credential、API Key

### 主要功能模組
| 模組 | 適用場景 | 核心技術 | 特色功能 |
| ---- | -------- | -------- | -------- |
| **Assistants Client** | 需要持久化對話 | 執行緒管理、Assistant API | 自動對話歷史、工具整合 |
| **Chat Client** | 即時對話 | 串流、函式呼叫 | 靈活、直接控制 |
| **Responses Client** | 結構化輸出 | Schema 驗證、Pydantic | 類型安全、資料驗證 |
| **Azure AI** | Azure AI Foundry | 整合式 AI 服務 | 企業級安全、管理 |
| **OpenAI** | OpenAI API | 原生 OpenAI 功能 | 最新模型、廣泛支援 |

---

### 精簡解說

以下是每個客戶端類型的核心概念與最精簡程式碼示範:

#### 1️⃣ Assistants Client
**核心概念**: 使用 Assistant API 管理持久化對話，自動處理執行緒和對話歷史

```python
from agent_framework.azure import AzureOpenAIAssistantsClient
from azure.identity import AzureCliCredential

async def main():
    client = AzureOpenAIAssistantsClient(credential=AzureCliCredential())
    response = await client.get_response(messages="Hello!")
    print(response.text)
```

#### 2️⃣ Chat Client
**核心概念**: 直接與聊天模型互動，完全控制訊息歷史和對話流程

```python
from agent_framework.azure import AzureOpenAIChatClient
from agent_framework import ChatMessage

async def main():
    client = AzureOpenAIChatClient(credential=AzureCliCredential())
    messages = [ChatMessage(role="user", text="What is AI?")]
    response = await client.get_response(messages=messages)
    print(response.text)
```

#### 3️⃣ Responses Client
**核心概念**: 取得結構化、類型安全的輸出，使用 Pydantic 模型驗證

```python
from agent_framework.azure import AzureOpenAIResponsesClient
from pydantic import BaseModel

class WeatherResponse(BaseModel):
    temperature: float
    condition: str

async def main():
    client = AzureOpenAIResponsesClient(credential=AzureCliCredential())
    result = await client.get_response(
        messages="What's the weather?",
        response_format=WeatherResponse
    )
    print(result.value.temperature)
```

#### 4️⃣ Response Cancellation
**核心概念**: 在串流過程中優雅地取消請求

```python
import asyncio

async def main():
    client = AzureOpenAIChatClient(credential=AzureCliCredential())
    
    async def cancel_after_delay(task, delay):
        await asyncio.sleep(delay)
        task.cancel()
    
    stream_task = asyncio.create_task(
        client.get_response_stream(messages="Write a long story...")
    )
    
    cancel_task = asyncio.create_task(cancel_after_delay(stream_task, 2.0))
    
    try:
        async for chunk in await stream_task:
            print(chunk.text, end="", flush=True)
    except asyncio.CancelledError:
        print("\n[Cancelled]")
```

---

## 商務應用

以下是每個客戶端類型的實際商務應用場景:

### 1️⃣ Assistants Client - 商務應用

1. **長期客戶關係管理**:
   - 使用 Assistant API 維護持久化的客戶對話執行緒
   - 自動追蹤客戶偏好和歷史互動
   - 適合需要長期關係的 B2B 銷售或諮詢服務

2. **多會話專案管理**:
   - 為每個專案建立獨立的 Assistant 執行緒
   - 自動保存專案討論和決策歷史
   - 團隊成員可以在不同時間繼續相同對話

3. **教育輔導系統**:
   - 為每個學生建立個人化的學習助手
   - 追蹤學習進度和已討論的主題
   - 提供連續性的教學體驗

### 2️⃣ Chat Client - 商務應用

1. **即時客戶支援**:
   - 快速回應客戶查詢，無需持久化儲存
   - 完全控制對話流程和上下文
   - 適合一次性問題或簡單的資訊查詢

2. **內容生成服務**:
   - 直接調用模型生成文案、標題、摘要
   - 批次處理多個請求
   - 靈活的輸入輸出格式

3. **對話式分析**:
   - 允許使用者透過自然語言查詢資料
   - 即時生成資料洞察和視覺化建議
   - 無需長期儲存對話歷史

### 3️⃣ Responses Client - 商務應用

1. **資料萃取與驗證**:
   - 從非結構化文字中提取結構化資料
   - 自動驗證資料格式和類型
   - 適用於發票處理、表單填寫、資料錄入

2. **決策支援系統**:
   - AI 生成結構化的決策建議
   - 包含信心分數、替代方案、風險評估
   - 確保輸出符合業務邏輯的 schema

3. **API 整合與自動化**:
   - AI 生成符合 API 規範的請求參數
   - 類型安全的系統間通訊
   - 減少人工錯誤和資料不一致

### 4️⃣ Azure AI Chat Client - 商務應用

1. **企業級 AI 部署**:
   - 利用 Azure AI Foundry 的安全性和治理功能
   - 整合 Azure 生態系統 (Key Vault、Monitor、RBAC)
   - 符合企業合規要求

2. **混合雲 AI 服務**:
   - 結合 Azure 和本地部署
   - 統一的 AI 服務管理介面
   - 適合大型企業的複雜架構

3. **多租戶 SaaS 應用**:
   - 使用 Azure AI 的租戶隔離功能
   - 細粒度的訪問控制和計費
   - 可擴展的企業 AI 服務

### 5️⃣ Response Cancellation - 商務應用

1. **成本控制**:
   - 使用者取消請求時立即停止 AI 處理
   - 避免不必要的 API 費用
   - 適合按使用量計費的服務

2. **使用者體驗優化**:
   - 允許使用者中斷不相關或過長的回應
   - 提高應用程式響應性
   - 減少等待時間和挫折感

3. **資源管理**:
   - 在高負載時優先處理活躍請求
   - 取消逾時或放棄的請求
   - 優化伺服器資源使用

---

## 1. Azure Assistants Client

### 主要情境
使用 Azure OpenAI Assistants API 進行持久化對話，自動管理執行緒和對話歷史。

### 技術特色
- **自動執行緒管理**: Assistant API 自動處理對話執行緒
- **持久化**: 對話歷史儲存在 Azure 中
- **工具整合**: 支援函式呼叫、程式碼直譯器、檔案搜尋
- **非同步執行**: 長時間運行的任務可以非同步處理

### 核心程式碼

```python
# azure_assistants_client.py
import asyncio
from agent_framework.azure import AzureOpenAIAssistantsClient
from azure.identity.aio import AzureCliCredential

async def main():
    """使用 Azure Assistants Client 進行基本聊天互動"""
    
    # 使用 Azure CLI 憑證驗證
    async with AzureCliCredential() as credential:
        # 建立 Assistants Client
        client = AzureOpenAIAssistantsClient(credential=credential)
        
        # 發送訊息並取得回應
        # Assistant API 會自動建立和管理執行緒
        response = await client.get_response(messages="Hello! Can you help me with Python?")
        
        print(f"Assistant: {response.text}")
        
        # 後續訊息會在相同的執行緒中繼續
        response = await client.get_response(messages="What are decorators?")
        print(f"Assistant: {response.text}")

if __name__ == "__main__":
    asyncio.run(main())
```

**環境變數**:
```bash
AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com/
AZURE_OPENAI_ASSISTANT_ID=asst_xxx  # 可選，使用現有 Assistant
```

**關鍵特性**:
- 自動建立和重用執行緒
- 對話歷史自動保存
- 支援多輪對話
- 可以與現有 Assistant 整合

---

## 2. Azure Chat Client

### 主要情境
直接與 Azure OpenAI Chat Completions API 互動，完全控制訊息歷史。

### 技術特色
- **直接控制**: 完全控制訊息列表和對話流程
- **靈活性**: 可以任意修改或重組訊息歷史
- **串流支援**: 支援串流回應
- **函式呼叫**: 支援工具和函式呼叫

### 核心程式碼

```python
# azure_chat_client.py
import asyncio
from agent_framework.azure import AzureOpenAIChatClient
from agent_framework import ChatMessage, Role
from azure.identity.aio import AzureCliCredential

async def main():
    """使用 Azure Chat Client 進行直接聊天互動"""
    
    async with AzureCliCredential() as credential:
        # 建立 Chat Client
        client = AzureOpenAIChatClient(credential=credential)
        
        # 建立訊息列表
        messages = [
            ChatMessage(role=Role.SYSTEM, text="You are a helpful coding assistant."),
            ChatMessage(role=Role.USER, text="How do I read a file in Python?"),
        ]
        
        # 取得回應
        response = await client.get_response(messages=messages)
        print(f"Assistant: {response.text}")
        
        # 繼續對話 - 手動管理訊息歷史
        messages.extend(response.messages)
        messages.append(ChatMessage(role=Role.USER, text="What about writing to a file?"))
        
        # 使用串流
        async for chunk in await client.get_response_stream(messages=messages):
            print(chunk.text, end="", flush=True)
        print()

if __name__ == "__main__":
    asyncio.run(main())
```

**環境變數**:
```bash
AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com/
AZURE_OPENAI_CHAT_DEPLOYMENT_NAME=gpt-4o
```

**關鍵特性**:
- 完全控制訊息順序和內容
- 支援系統訊息設定
- 串流和非串流模式
- 可以實作自訂對話邏輯

---

## 3. Azure Responses Client

### 主要情境
取得結構化、類型安全的回應，使用 Pydantic 模型進行資料驗證。

### 技術特色
- **結構化輸出**: 使用 Pydantic 模型定義輸出格式
- **類型安全**: 編譯時類型檢查
- **自動驗證**: 自動驗證 AI 輸出
- **Schema 整合**: 將 schema 傳遞給模型以確保格式正確

### 核心程式碼

```python
# azure_responses_client.py
import asyncio
from agent_framework.azure import AzureOpenAIResponsesClient
from agent_framework import ChatMessage
from azure.identity.aio import AzureCliCredential
from pydantic import BaseModel

class PersonInfo(BaseModel):
    """定義結構化輸出的 schema"""
    name: str
    age: int
    occupation: str

async def main():
    """使用 Azure Responses Client 取得結構化輸出"""
    
    async with AzureCliCredential() as credential:
        # 建立 Responses Client
        client = AzureOpenAIResponsesClient(credential=credential)
        
        # 建立查詢
        messages = [
            ChatMessage(
                role="user",
                text="Extract information: John is a 30-year-old software engineer."
            )
        ]
        
        # 取得結構化回應
        result = await client.get_response(
            messages=messages,
            response_format=PersonInfo
        )
        
        # 類型安全的訪問
        person = result.value
        print(f"Name: {person.name}")
        print(f"Age: {person.age}")
        print(f"Occupation: {person.occupation}")
        
        # 也可以使用串流
        async for chunk in await client.get_response_stream(
            messages=messages,
            response_format=PersonInfo
        ):
            if chunk.value:
                print(f"Streaming: {chunk.value}")

if __name__ == "__main__":
    asyncio.run(main())
```

**環境變數**:
```bash
AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com/
AZURE_OPENAI_RESPONSES_DEPLOYMENT_NAME=gpt-4o
```

**關鍵特性**:
- Pydantic 模型定義輸出格式
- 自動 JSON Schema 生成
- 資料驗證和類型轉換
- 支援巢狀和複雜類型

---

## 4. Azure AI Chat Client

### 主要情境
使用 Azure AI Foundry 的整合式 AI 服務進行企業級部署。

### 技術特色
- **企業級**: 整合 Azure 安全性和治理
- **統一管理**: Azure AI Foundry 統一介面
- **多模型**: 支援多個 AI 模型
- **監控整合**: 與 Azure Monitor 整合

### 核心程式碼

```python
# azure_ai_chat_client.py
import asyncio
from agent_framework.azure import AzureAIChatClient
from agent_framework import ChatMessage
from azure.identity.aio import AzureCliCredential

async def main():
    """使用 Azure AI Chat Client 進行聊天互動"""
    
    async with AzureCliCredential() as credential:
        # 建立 Azure AI Chat Client
        client = AzureAIChatClient(credential=credential)
        
        # 建立訊息
        messages = [ChatMessage(role="user", text="Explain machine learning.")]
        
        # 取得回應
        response = await client.get_response(messages=messages)
        print(f"Response: {response.text}")
        
        # 使用串流
        async for chunk in await client.get_response_stream(messages=messages):
            print(chunk.text, end="", flush=True)
        print()

if __name__ == "__main__":
    asyncio.run(main())
```

**環境變數**:
```bash
AZURE_AI_PROJECT_ENDPOINT=https://your-project.api.azureml.ms
AZURE_AI_MODEL_DEPLOYMENT_NAME=gpt-4o
```

**關鍵特性**:
- Azure AI Foundry 整合
- 企業級安全性和合規性
- 統一的多模型介面
- 細粒度的訪問控制

---

## 5. OpenAI Assistants Client

### 主要情境
使用 OpenAI Assistants API 進行持久化對話。

### 技術特色
- **OpenAI 原生**: 直接使用 OpenAI API
- **最新功能**: 支援 OpenAI 最新功能
- **工具整合**: 程式碼直譯器、檔案搜尋
- **執行緒管理**: 自動管理對話執行緒

### 核心程式碼

```python
# openai_assistants_client.py
import asyncio
from agent_framework.openai import OpenAIAssistantsClient

async def main():
    """使用 OpenAI Assistants Client 進行基本聊天"""
    
    # 使用 API key 驗證
    client = OpenAIAssistantsClient()
    
    # 發送訊息
    response = await client.get_response(messages="What is the capital of France?")
    print(f"Assistant: {response.text}")
    
    # 繼續對話
    response = await client.get_response(messages="What about Italy?")
    print(f"Assistant: {response.text}")

if __name__ == "__main__":
    asyncio.run(main())
```

**環境變數**:
```bash
OPENAI_API_KEY=sk-xxx
OPENAI_ASSISTANT_ID=asst_xxx  # 可選
```

---

## 6. OpenAI Chat Client

### 主要情境
直接與 OpenAI Chat Completions API 互動。

### 技術特色
- **直接控制**: 完全控制訊息和參數
- **最新模型**: 支援 GPT-4、GPT-4o、o1 等
- **函式呼叫**: 原生函式呼叫支援
- **串流**: 高效的串流回應

### 核心程式碼

```python
# openai_chat_client.py
import asyncio
from agent_framework.openai import OpenAIChatClient
from agent_framework import ChatMessage, Role

async def main():
    """使用 OpenAI Chat Client 進行聊天"""
    
    client = OpenAIChatClient()
    
    messages = [
        ChatMessage(role=Role.USER, text="Explain quantum computing in simple terms.")
    ]
    
    # 非串流回應
    response = await client.get_response(messages=messages)
    print(f"Response: {response.text}")
    
    # 串流回應
    print("\nStreaming response:")
    async for chunk in await client.get_response_stream(messages=messages):
        print(chunk.text, end="", flush=True)
    print()

if __name__ == "__main__":
    asyncio.run(main())
```

**環境變數**:
```bash
OPENAI_API_KEY=sk-xxx
OPENAI_CHAT_MODEL_ID=gpt-4o
```

---

## 7. OpenAI Responses Client

### 主要情境
從 OpenAI 取得結構化輸出。

### 技術特色
- **結構化輸出**: Pydantic 模型驗證
- **類型安全**: 編譯時檢查
- **JSON Mode**: 原生 JSON 模式支援
- **Schema 驗證**: 自動 schema 驗證

### 核心程式碼

```python
# openai_responses_client.py
import asyncio
from agent_framework.openai import OpenAIResponsesClient
from pydantic import BaseModel

class MovieInfo(BaseModel):
    title: str
    year: int
    director: str
    rating: float

async def main():
    """使用 OpenAI Responses Client 取得結構化輸出"""
    
    client = OpenAIResponsesClient()
    
    result = await client.get_response(
        messages="Extract info: The Shawshank Redemption, 1994, Frank Darabont, 9.3/10",
        response_format=MovieInfo
    )
    
    movie = result.value
    print(f"Title: {movie.title}")
    print(f"Year: {movie.year}")
    print(f"Director: {movie.director}")
    print(f"Rating: {movie.rating}")

if __name__ == "__main__":
    asyncio.run(main())
```

**環境變數**:
```bash
OPENAI_API_KEY=sk-xxx
OPENAI_RESPONSES_MODEL_ID=gpt-4o
```

---

## 8. Chat Response Cancellation

### 主要情境
在串流過程中優雅地取消 AI 回應。

### 技術特色
- **優雅取消**: 使用 asyncio 取消機制
- **資源清理**: 自動清理連接和資源
- **成本控制**: 停止不需要的處理
- **使用者控制**: 允許使用者中斷長回應

### 核心程式碼

```python
# chat_response_cancellation.py
import asyncio
from agent_framework.azure import AzureOpenAIChatClient
from agent_framework import ChatMessage
from azure.identity.aio import AzureCliCredential

async def cancel_after_delay(task: asyncio.Task, delay: float):
    """在指定延遲後取消任務"""
    await asyncio.sleep(delay)
    task.cancel()
    print("\n[Task cancelled]")

async def main():
    """展示聊天回應取消"""
    
    async with AzureCliCredential() as credential:
        client = AzureOpenAIChatClient(credential=credential)
        
        messages = [
            ChatMessage(
                role="user",
                text="Write a very long story about a space adventure..."
            )
        ]
        
        # 建立串流任務
        stream_task = asyncio.create_task(
            client.get_response_stream(messages=messages)
        )
        
        # 建立取消任務 - 2 秒後取消
        cancel_task = asyncio.create_task(
            cancel_after_delay(stream_task, 2.0)
        )
        
        try:
            # 處理串流回應
            async for chunk in await stream_task:
                print(chunk.text, end="", flush=True)
        except asyncio.CancelledError:
            print("\n[Streaming cancelled gracefully]")
        finally:
            # 清理
            if not cancel_task.done():
                cancel_task.cancel()

if __name__ == "__main__":
    asyncio.run(main())
```

**關鍵特性**:
- 使用 `asyncio.Task.cancel()` 取消
- 捕獲 `asyncio.CancelledError`
- 自動資源清理
- 適用於所有支援串流的客戶端

**實作最佳實踐**:
1. 使用 try-except 捕獲 CancelledError
2. 在 finally 中清理資源
3. 提供使用者取消按鈕
4. 記錄取消事件用於分析

---

## 結論

Agent Framework 提供了多種聊天客戶端，滿足不同的需求和場景。

### 選擇指南

| 需求 | 推薦客戶端 | 原因 |
| ---- | ---------- | ---- |
| **持久化對話** | Assistants Client | 自動執行緒管理 |
| **完全控制** | Chat Client | 靈活的訊息處理 |
| **結構化輸出** | Responses Client | 類型安全、驗證 |
| **企業部署** | Azure AI Chat Client | 安全性、治理 |
| **最新功能** | OpenAI Client | 原生 OpenAI 支援 |
| **成本控制** | 任何 + Cancellation | 優雅取消 |

### 最佳實踐

1. **驗證管理**:
   - Azure: 使用 Azure CLI Credential 或 Managed Identity
   - OpenAI: 安全儲存 API Key (環境變數或 Key Vault)

2. **錯誤處理**:
   ```python
   try:
       response = await client.get_response(messages=messages)
   except Exception as e:
       logger.error(f"API call failed: {e}")
       # 實作重試或降級邏輯
   ```

3. **串流優化**:
   - 對長回應使用串流
   - 提供取消選項
   - 顯示進度指示器

4. **成本優化**:
   - 實作請求快取
   - 使用較小模型處理簡單任務
   - 監控和限制 token 使用

5. **類型安全**:
   - Responses Client 使用 Pydantic 模型
   - 定義清晰的 schema
   - 處理驗證錯誤

### 進階主題

- **自訂客戶端**: 擴展基礎客戶端類別
- **中介軟體整合**: 添加日誌、監控、快取
- **多模型策略**: 根據任務選擇最佳模型
- **效能優化**: 批次處理、並行請求
- **安全性**: API Key 輪換、請求驗證

Agent Framework 的聊天客戶端提供了構建強大 AI 應用的基礎，無論是簡單的聊天機器人還是複雜的企業 AI 系統。

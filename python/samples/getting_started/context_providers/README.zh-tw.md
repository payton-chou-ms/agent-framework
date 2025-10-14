# Context Providers 範例

## 摘要

此資料夾展示如何使用 Context Providers (上下文提供者) 搭配 Agent Framework，為代理提供持久化記憶能力。Context Providers 允許代理在多次互動中儲存和檢索對話上下文、使用者偏好以及工具輸出。這些範例展示三種不同的記憶體後端：簡單的自訂實作、Mem0（託管記憶體服務）以及 Redis（具有向量搜尋功能的強大記憶體資料儲存）。

Context Providers 讓代理能夠：
- 在對話中記住使用者偏好和詳細資訊
- 回憶過去的互動和工具輸出
- 在不同執行緒和會話中保持連續性
- 根據累積的知識提供個人化回應

## 範例

### 簡單的 Context Provider

**檔案**：`simple_context_provider.py`

**主要功能**：
- 自訂實作 ContextProvider 介面
- 展示如何提取和儲存使用者資訊（姓名、年齡）
- 使用結構化輸出從對話中解析使用者詳細資訊
- 展示 context providers 的基本生命週期（在每次代理呼叫後被調用）
- 適合學習和原型設計的最小實作

**使用情境**：了解 context providers 的基礎知識，並創建針對特定需求量身定制的自訂記憶體解決方案。

---

### Mem0 整合

Mem0 是一個託管記憶體服務，提供智能上下文儲存，配置最少。

#### 基本用法

**檔案**：`mem0/mem0_basic.py`

**主要功能**：
- 與 Mem0 雲端記憶體服務整合
- 使用 user_id 自動關聯記憶體
- 展示跨多個查詢的記憶體持久化
- 展示代理如何學習使用者偏好（公司代碼、報告格式）
- 工具與記憶體整合（retrieve_company_report 函式）

**使用情境**：快速設置生產環境代理的持久化記憶體，無需管理基礎設施。

#### 開源本地 Mem0

**檔案**：`mem0/mem0_oss.py`

**主要功能**：
- 使用本地 Mem0 OSS 客戶端（AsyncMemory）而非雲端服務
- 自託管記憶體儲存，適用於隱私敏感應用
- OpenAI 嵌入整合（可配置為其他 LLM 提供者）
- 與雲端版本相同的記憶體功能，但在本地運行

**使用情境**：自託管部署，其中資料必須保留在本地，或當您需要完全控制記憶體基礎設施時。

#### Mem0 執行緒範圍

**檔案**：`mem0/mem0_threads.py`

**主要功能**：
- **全域執行緒範圍**：在所有執行緒和操作間共享記憶體
- **單一操作執行緒範圍**：將記憶體隔離到個別執行緒
- 展示 scope_to_per_operation_thread_id 配置
- 展示如何創建和管理獨立的對話執行緒
- 多使用者或多會話場景的記憶體隔離策略

**使用情境**：需要對記憶體可見性進行細粒度控制的應用程式，例如多租戶系統或具有明確會話邊界的對話代理。

---

### Redis 整合

Redis 提供強大的自託管記憶體解決方案，具有全文搜尋和向量相似性搜尋等進階功能。

#### 基本用法

**檔案**：`redis/redis_basics.py`

**主要功能**：
- 三個漸進式場景：獨立提供者、代理整合、工具記憶體
- 全文和混合向量搜尋，用於檢索相關上下文
- RediSearch 整合，實現快速查詢
- 工具輸出持久化（search_flights 函式範例）
- 展示如何在記憶體中捕獲工具的結構化輸出

**使用情境**：需要快速、可擴展記憶體且具備進階搜尋功能的應用程式，特別是當工具輸出需要可搜尋時。

#### 對話儲存

**檔案**：`redis/redis_conversation.py`

**主要功能**：
- 使用 RedisChatMessageStore 實現持久化對話歷史
- 與 RedisProvider 整合，實現雙重功能（訊息 + 上下文）
- 使用 OpenAI 嵌入的混合搜尋，實現語義檢索
- 展示 agent.get_chat_history() 的使用
- 展示如何配置 Redis 向量化器和嵌入快取

**使用情境**：建構需要對話歷史和智能上下文檢索的聊天應用程式，例如客戶支援機器人或知識助手。

#### Redis 執行緒範圍

**檔案**：`redis/redis_threads.py`

**主要功能**：
- **全域執行緒範圍**：固定的 thread_id 在所有操作間共享記憶體
- **單一操作執行緒範圍**：將提供者綁定到單一執行緒生命週期
- **多代理記憶體隔離**：不同的 agent_id 值用於獨立角色
- 展示如何在不同層級範圍化記憶體可見性
- 展示執行緒和代理隔離的最佳實踐

**使用情境**：複雜的多代理系統，其中不同代理需要隔離的記憶體，或具有多個並發使用者會話需要記憶體分離的應用程式。

## 核心概念

### 什麼是 Context Provider？

Context Provider 是一個攔截代理互動並管理上下文資訊的元件。它：
- 實作 `ContextProvider` 介面
- 透過 `invoked()` 方法在每次代理調用後被呼叫
- 可以讀取請求/回應訊息並儲存相關資訊
- 在後續互動期間向代理提供上下文

### 記憶體關聯

Context providers 通常將記憶體與識別符關聯：
- **user_id**：將記憶體連結到特定使用者
- **agent_id**：將記憶體連結到特定代理實例
- **thread_id**：將記憶體連結到對話執行緒
- **application_id**：將記憶體連結到應用程式（用於多租戶場景）

### 記憶體範圍

不同的範圍策略控制記憶體可見性：
- **全域範圍**：所有執行緒看到相同的記憶體
- **單一執行緒範圍**：每個執行緒具有隔離的記憶體
- **單一代理範圍**：每個代理維護獨立的記憶體

## 安裝

### 基本需求

```bash
pip install agent-framework --pre
```

### Mem0 整合

```bash
pip install mem0ai
# 為雲端服務設置 MEM0_API_KEY 環境變數
# 或使用本地 OSS 版本（需要 OpenAI API key）
```

### Redis 整合

```bash
pip install "agent-framework-redis"
# 需要 Redis Stack（含 RediSearch 模組）
# 可選設置 OPENAI_API_KEY 用於向量嵌入
```

## 環境變數

### Azure 驗證

```bash
# 對於 Azure AI 代理，透過 Azure CLI 驗證
az login
# 或設置這些變數：
# AZURE_OPENAI_ENDPOINT
# AZURE_OPENAI_DEPLOYMENT_NAME
# AZURE_OPENAI_API_KEY
```

### Mem0 配置

```bash
# 雲端服務
MEM0_API_KEY=your_mem0_api_key

# 本地 OSS（需要 LLM 用於嵌入）
OPENAI_API_KEY=your_openai_api_key
```

### Redis 配置

```bash
# Redis 連接
REDIS_URL=redis://localhost:6379  # 預設值

# 用於向量搜尋
OPENAI_API_KEY=your_openai_api_key  # 如果使用 OpenAI 嵌入
OPENAI_CHAT_MODEL_ID=gpt-4o-mini    # 可選，用於聊天範例
```

## 執行範例

1. 導航到 context_providers 資料夾：
   ```bash
   cd python/samples/getting_started/context_providers
   ```

2. 設置所需的環境變數（見上文）

3. 執行任何範例：
   ```bash
   python simple_context_provider.py
   python mem0/mem0_basic.py
   python redis/redis_basics.py
   ```

## 選擇正確的 Provider

| Provider | 最適合 | 複雜度 | 託管方式 |
|----------|----------|------------|---------|
| **簡單自訂** | 學習、原型、自訂邏輯 | 低 | N/A |
| **Mem0 雲端** | 快速生產設置、託管服務 | 低 | 雲端 |
| **Mem0 OSS** | 自託管、隱私敏感應用 | 中 | 自託管 |
| **Redis** | 高效能、進階搜尋、大規模 | 高 | 自託管 |

## 常見模式

### 提取和儲存使用者資訊

```python
class UserInfoMemory(ContextProvider):
    async def invoked(self, request_messages, response_messages, **kwargs):
        # 從對話中提取結構化資料
        user_info = await self._chat_client.get_response(
            messages=request_messages,
            chat_options=ChatOptions(response_format=UserInfo)
        )
```

### 向代理提供上下文

```python
# Mem0 自動檢索相關記憶體
agent = client.create_agent(
    name="Assistant",
    context_providers=Mem0Provider(user_id=user_id)
)
```

### 將記憶體範圍限定到執行緒

```python
# 全域範圍（跨執行緒共享）
provider = Mem0Provider(
    user_id=user_id,
    thread_id=global_thread_id,
    scope_to_per_operation_thread_id=False
)

# 單一操作範圍（隔離執行緒）
provider = Mem0Provider(
    user_id=user_id,
    scope_to_per_operation_thread_id=True
)
```

## 參考資料

- [Mem0 文件](https://docs.mem0.ai/)
- [Redis 文件](https://redis.io/docs/)
- [RediSearch](https://redis.io/docs/stack/search/)
- [Agent Framework 核心文件](../../../README.md)

# 記憶體與對話狀態 - 程式碼摘要

本文件提供與圖表中顯示的記憶體與對話狀態架構相關的程式碼實作全面摘要。

## 架構概述

Microsoft Agent Framework 提供兩種類型的記憶體來維護對話情境：

1. **短期記憶體（會話範圍）**：透過 `AgentThread` 管理即時對話情境
2. **長期記憶體（持久性）**：透過 `ContextProvider` 整合（例如 Mem0、Redis）管理使用者偏好設定、事實和歷史資料

## 核心元件

### 1. AgentThread - 短期記憶體

**位置**：`python/packages/core/agent_framework/_threads.py`

`AgentThread` 類別代表對話執行緒，在單一會話的回合中維護會話範圍的對話歷史記錄。

#### 主要類別

```python
class AgentThread:
    """
    Agent thread 類別代表本機管理和服務管理的執行緒。
    維護代理程式互動的對話狀態和訊息歷史記錄。
    """
    def __init__(
        self,
        *,
        service_thread_id: str | None = None,
        message_store: ChatMessageStoreProtocol | None = None,
        context_provider: AggregateContextProvider | None = None,
    ):
        # service_thread_id 或 message_store 可以設定，但不能同時設定
        self._service_thread_id = service_thread_id
        self._message_store = message_store
        self.context_provider = context_provider
```

#### 儲存選項

框架支援執行緒的兩種儲存機制：

1. **服務管理的執行緒**（`service_thread_id`）：
   - 由 AI 服務提供者管理的執行緒（例如 OpenAI Assistants API）
   - 本機儲存執行緒 ID，遠端儲存訊息
   - 範例：`thread = agent.get_new_thread(service_thread_id="thread_abc123")`

2. **本機訊息存放區**（`ChatMessageStore`）：
   - 訊息在本機記憶體或自訂儲存中儲存
   - 完全控制訊息持久性
   - 範例：`agent = ChatAgent(chat_message_store_factory=ChatMessageStore)`

#### 程式碼範例：執行緒管理

```python
# 來源：python/samples/getting_started/threads/suspend_resume_thread.py

# 建立執行緒並進行對話
thread = agent.get_new_thread()
await agent.run("Tell me a joke about a pirate.", thread=thread)

# 序列化執行緒狀態以便持久化
serialized_thread = await thread.serialize()
# 儲存到資料庫、檔案等

# 稍後：反序列化並繼續
resumed_thread = await agent.deserialize_thread(serialized_thread)
await agent.run("Tell the same joke in pirate voice", thread=resumed_thread)
```

#### 主要方法

```python
class AgentThread:
    async def on_new_messages(self, new_messages: ChatMessage | Sequence[ChatMessage]):
        """新訊息新增到對話時呼叫"""
        
    async def serialize(self) -> dict[str, Any]:
        """序列化執行緒狀態以便持久化"""
        
    @classmethod
    async def deserialize(cls, serialized_thread_state, message_store=None):
        """從字典反序列化狀態到新的 AgentThread"""
```

---

### 2. ChatMessageStore - 訊息持久性

**位置**：`python/packages/core/agent_framework/_threads.py`

```python
class ChatMessageStore:
    """
    ChatMessageStoreProtocol 的記憶體內實作。
    在清單中儲存訊息，支援序列化。
    """
    def __init__(self, messages: Sequence[ChatMessage] | None = None):
        self.messages = list(messages) if messages else []
    
    async def add_messages(self, messages: Sequence[ChatMessage]):
        """將訊息新增到存放區"""
        
    async def list_messages(self) -> list[ChatMessage]:
        """按時間順序取得所有訊息"""
        
    async def serialize(self) -> dict[str, Any]:
        """序列化以便持久化"""
        
    @classmethod
    async def deserialize(cls, serialized_store_state):
        """從序列化狀態建立存放區"""
```

---

### 3. ContextProvider - 長期記憶體

**位置**：`python/packages/core/agent_framework/_memory.py`

`ContextProvider` 抽象基底類別透過在每次呼叫前提供額外情境給 AI 模型來啟用長期記憶體功能。

```python
class ContextProvider(ABC):
    """
    情境提供者的基底類別。
    用於透過長期記憶體增強 AI 的情境管理。
    """
    
    async def thread_created(self, thread_id: str | None):
        """建立新執行緒時呼叫 - 載入相關的長期資料"""
        
    @abstractmethod
    async def invoking(self, messages, **kwargs) -> Context:
        """
        模型呼叫前呼叫。
        傳回包含指示、訊息和工具的 Context。
        """
        
    async def invoked(self, request_messages, response_messages, **kwargs):
        """模型回應後呼叫 - 更新長期記憶體"""
```

#### Context 物件

```python
class Context:
    """包含 ContextProvider 提供給 AI 模型的情境"""
    def __init__(
        self,
        instructions: str | None = None,
        messages: Sequence[ChatMessage] | None = None,
        tools: Sequence[ToolProtocol] | None = None,
    ):
        self.instructions = instructions  # 額外指示
        self.messages = messages or []    # 額外訊息
        self.tools = tools or []          # 額外工具
```

---

### 4. AggregateContextProvider - 多個記憶體來源

**位置**：`python/packages/core/agent_framework/_memory.py`

```python
class AggregateContextProvider(ContextProvider):
    """
    包含多個情境提供者的 ContextProvider。
    將事件委派給多個情境提供者，並在傳回前彙總回應。
    """
    def __init__(self, context_providers):
        self.providers = context_providers
    
    async def invoking(self, messages, **kwargs) -> Context:
        # 從所有提供者收集情境
        contexts = await asyncio.gather(
            *[provider.invoking(messages) for provider in self.providers]
        )
        # 結合所有指示、訊息和工具
        return Context(
            instructions=combined_instructions,
            messages=combined_messages,
            tools=combined_tools
        )
```

---

## 實作範例

### 範例 1：Mem0 Provider - 長期記憶體

**位置**：`python/samples/getting_started/context_providers/mem0/mem0_threads.py`

Mem0 提供跨對話會話的持久性記憶體，具有不同的範圍策略：

```python
# 全域範圍：跨所有執行緒共用的記憶體
agent = AzureAIAgentClient().create_agent(
    name="Assistant",
    context_providers=Mem0Provider(
        user_id="user123",
        thread_id=global_thread_id,
        scope_to_per_operation_thread_id=False  # 跨執行緒共用
    )
)

# 儲存偏好設定
await agent.run("Remember that I prefer technical responses")

# 新執行緒 - 但記憶體持久化
new_thread = agent.get_new_thread()
await agent.run("What are my preferences?", thread=new_thread)
# 代理程式記住來自先前執行緒的偏好設定
```

```python
# 執行緒範圍：每個執行緒隔離的記憶體
agent = AzureAIAgentClient().create_agent(
    name="Assistant",
    context_providers=Mem0Provider(
        user_id="user123",
        scope_to_per_operation_thread_id=True  # 每個執行緒隔離
    )
)

thread = agent.get_new_thread()
await agent.run("I'm working on a Python project", thread=thread)
# 記憶體僅在此特定執行緒內可存取
```

### 範例 2：Redis Provider - 可搜尋記憶體

**位置**：`python/samples/getting_started/context_providers/redis/redis_threads.py`

Redis provider 透過全文檢索啟用持久性、可搜尋記憶體：

```python
from agent_framework.redis import RedisProvider

redis_provider = RedisProvider(
    user_id="user123",
    redis_url="redis://localhost:6379"
)

agent = ChatAgent(
    client=chat_client,
    context_providers=redis_provider
)

# 記憶體儲存在 Redis 中，可以搜尋
await agent.run("I prefer dark mode interfaces")
await agent.run("My favorite color is blue")

# 稍後的會話可以檢索這些記憶體
await agent.run("What do you know about my preferences?")
```

---

## 記憶體流程圖對應

根據提供的圖像，以下是程式碼如何對應到架構概念：

### Thread（旅行規劃）
- **程式碼**：`AgentThread` 類別
- **用途**：在會話內維護對話情境
- **儲存**：`ChatMessageStore` 或服務管理

### Run 1 & Run 2（執行 1 和執行 2）
- **程式碼**：`agent.run()` 方法呼叫
- **用途**：代理程式的每次呼叫
- **流程**：
  1. ContextProvider.invoking() - 載入長期記憶體
  2. 代理程式使用短期情境（執行緒訊息）處理
  3. ContextProvider.invoked() - 更新長期記憶體

### Memory（右側）
- **程式碼**：`ContextProvider` 實作（Mem0、Redis）
- **用途**：長期持久性儲存
- **範例**：
  - 透過 Mem0Provider 儲存「紐約之旅」
  - 透過 RedisProvider 儲存「每日餐費補貼」

### 短期記憶體
- **程式碼**：`AgentThread` 內的 `ChatMessageStore`
- **範圍**：會話範圍、即時情境
- **內容**：當前對話中的使用者和代理程式訊息

### 長期記憶體
- **程式碼**：`ContextProvider` 整合（Mem0、Redis、自訂）
- **範圍**：跨會話和執行緒持久化
- **內容**：使用者偏好設定、事實、歷史資料

---

## 程式碼架構摘要

```
┌─────────────────────────────────────────────────────────────┐
│                        Agent                                 │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           AgentThread（短期記憶體）                     │  │
│  │  - ChatMessageStore（對話訊息）                        │  │
│  │  - AggregateContextProvider（長期存取）                │  │
│  └──────────────────────────────────────────────────────┘  │
│                           │                                  │
│                           │ agent.run()                      │
│                           ▼                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         1. ContextProvider.invoking()                │  │
│  │            （載入長期記憶體）                           │  │
│  └──────────────────────────────────────────────────────┘  │
│                           │                                  │
│                           ▼                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │    2. 代理程式使用組合情境處理：                        │  │
│  │       - 短期：執行緒訊息                               │  │
│  │       - 長期：提供者指示/訊息                          │  │
│  └──────────────────────────────────────────────────────┘  │
│                           │                                  │
│                           ▼                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         3. ContextProvider.invoked()                 │  │
│  │            （更新長期記憶體）                           │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘

外部儲存：
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   Mem0 API      │  │  Redis Server   │  │  Custom DB      │
│ （長期）         │  │ （長期）         │  │ （長期）         │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

---

## 主要檔案參考

### 核心實作
- `python/packages/core/agent_framework/_threads.py` - AgentThread、ChatMessageStore
- `python/packages/core/agent_framework/_memory.py` - ContextProvider、Context
- `python/packages/core/agent_framework/exceptions.py` - AgentThreadException

### 情境提供者實作
- `python/packages/mem0/agent_framework/mem0/_provider.py` - Mem0Provider
- `python/packages/redis/agent_framework/redis/_provider.py` - RedisProvider

### 範例
- `python/samples/getting_started/threads/suspend_resume_thread.py` - 執行緒序列化
- `python/samples/getting_started/threads/custom_chat_message_store_thread.py` - 自訂儲存
- `python/samples/getting_started/context_providers/mem0/mem0_threads.py` - 長期記憶體範圍
- `python/samples/getting_started/context_providers/redis/redis_threads.py` - 可搜尋記憶體

---

## 摘要

Microsoft Agent Framework 提供全面的記憶體架構：

1. **AgentThread** 管理短期、會話範圍的對話歷史記錄
2. **ChatMessageStore** 為執行緒訊息提供可插拔儲存
3. **ContextProvider** 啟用跨會話的長期持久性記憶體
4. **AggregateContextProvider** 結合多個記憶體來源

這個架構允許代理程式：
- 在會話內維護即時對話情境（短期）
- 跨會話記住使用者偏好設定和事實（長期）
- 結合多個記憶體來源以獲得豐富的情境
- 無縫序列化和繼續對話
- 從簡單的記憶體內擴展到生產資料庫（Redis、Mem0、自訂）

圖表說明了這個雙層記憶體系統，其中：
- **Thread** 包含對話流程（短期）
- **Memory** 包含持久性事實（長期）
- **Runs** 代表結合兩種記憶體類型的代理程式呼叫

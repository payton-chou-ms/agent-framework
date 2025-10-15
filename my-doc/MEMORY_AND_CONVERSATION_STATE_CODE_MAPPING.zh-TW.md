# 記憶體與對話狀態 - 程式碼對應

本文件提供架構圖與實際程式碼實作之間的視覺化對應。

## 圖表到程式碼的對應

### 圖表元件：「Thread（旅行規劃）」

**對應到**：`_threads.py` 中的 `AgentThread` 類別

```python
# 位置：python/packages/core/agent_framework/_threads.py:292

class AgentThread:
    """
    Agent thread 類別代表圖表中顯示的對話執行緒。
    它維護代理程式互動的對話狀態和訊息歷史記錄。
    """
    def __init__(
        self,
        *,
        service_thread_id: str | None = None,
        message_store: ChatMessageStoreProtocol | None = None,
        context_provider: AggregateContextProvider | None = None,
    ):
        self._service_thread_id = service_thread_id
        self._message_store = message_store
        self.context_provider = context_provider
```

**使用範例**：
```python
# 建立執行緒（如圖表中的「旅行規劃」）
thread = agent.get_new_thread()
```

---

### 圖表元件：「使用者的訊息」和「代理程式的訊息」

**對應到**：儲存在 `ChatMessageStore` 中的 `ChatMessage` 物件

```python
# 位置：python/packages/core/agent_framework/_threads.py:185

class ChatMessageStore:
    """儲存圖表中顯示的對話訊息"""
    def __init__(self, messages: Sequence[ChatMessage] | None = None):
        self.messages = list(messages) if messages else []
    
    async def add_messages(self, messages: Sequence[ChatMessage]) -> None:
        """將使用者和代理程式訊息新增到對話"""
        self.messages.extend(messages)
    
    async def list_messages(self) -> list[ChatMessage]:
        """按時間順序取得所有訊息（最舊的在前）"""
        return self.messages
```

**對話流程**：
```python
# 使用者的訊息：「I need to book a hotel in New York for 2 stays.」
await agent.run("I need to book a hotel in New York for 2 stays.", thread=thread)

# 代理程式的訊息：「Here are some suggestions:」
# （自動儲存在 thread's message_store 中）

# 使用者的訊息：「What's the daily meal allowance for the business trip?」
await agent.run("What's the daily meal allowance for the business trip?", thread=thread)

# 所有訊息都保留在 thread.message_store.messages 中
```

---

### 圖表元件：「Run 1」和「Run 2」

**對應到**：使用情境提供者呼叫的 `agent.run()` 方法呼叫

```python
# 圖表中的每個「Run」對應於一次 agent.run() 呼叫

# Run 1：使用 Tripadvisor API 搜尋最近的飯店
result1 = await agent.run(
    "I need to book a hotel in New York for 2 stays.", 
    thread=thread
)
# 幕後處理：
# 1. context_provider.invoking() - 從記憶體載入「紐約之旅」
# 2. 代理程式使用執行緒情境 + 長期記憶體處理
# 3. context_provider.invoked() - 如果需要，更新記憶體

# Run 2：使用 Microsoft SharePoint 查詢公司差旅政策
result2 = await agent.run(
    "What's the daily meal allowance for the business trip?", 
    thread=thread
)
# 幕後處理：
# 1. context_provider.invoking() - 載入相關記憶體
# 2. 代理程式使用累積的執行緒情境 + 長期記憶體處理
# 3. context_provider.invoked() - 將「每日餐費補貼」儲存到記憶體
```

---

### 圖表元件：「Memory」（右側）

**對應到**：`ContextProvider` 實作

```python
# 位置：python/packages/core/agent_framework/_memory.py:74

class ContextProvider(ABC):
    """
    長期記憶體提供者的基底類別。
    圖表中的「Memory」方塊代表透過 ContextProvider 
    實作存取的持久性儲存。
    """
    
    async def thread_created(self, thread_id: str | None) -> None:
        """
        建立執行緒時呼叫。
        載入相關的長期記憶體（例如「紐約之旅」）
        """
        pass
    
    @abstractmethod
    async def invoking(self, messages, **kwargs) -> Context:
        """
        在每次 Run（Run 1、Run 2 等）之前呼叫。
        從長期記憶體傳回額外情境。
        """
        pass
    
    async def invoked(
        self, 
        request_messages, 
        response_messages, 
        **kwargs
    ) -> None:
        """
        在每次 Run 之後呼叫。
        使用新資訊更新長期記憶體
        （例如儲存「每日餐費補貼」）。
        """
        pass
```

**具體實作**：

1. **Mem0Provider**（用於「紐約之旅」、「每日餐費補貼」）：
```python
# 位置：python/packages/mem0/agent_framework/mem0/_provider.py

agent = ChatAgent(
    client=chat_client,
    context_providers=Mem0Provider(
        user_id="user123",
        scope_to_per_operation_thread_id=False  # 全域記憶體
    )
)

# Run 1 儲存：「紐約之旅」
# Run 2 儲存：「每日餐費補貼」
# 兩者都持久化在 Mem0 中，並在未來的執行中檢索
```

2. **RedisProvider**（替代的長期儲存）：
```python
# 位置：python/packages/redis/agent_framework/redis/_provider.py

agent = ChatAgent(
    client=chat_client,
    context_providers=RedisProvider(
        user_id="user123",
        redis_url="redis://localhost:6379"
    )
)
```

---

### 圖表元件：Memory 箭頭（⬇️）

**對應到**：ContextProvider 生命週期中的資料流

```python
# 從 Run 1 到 Memory 的箭頭（「紐約之旅」）
# 這發生在 ContextProvider.invoked() 中

class Mem0Provider(ContextProvider):
    async def invoked(
        self, 
        request_messages, 
        response_messages, 
        **kwargs
    ):
        """
        Run 1 完成後，此方法：
        1. 提取資訊：「紐約之旅」
        2. 將它儲存在 Mem0 API 中
        3. 使它可供未來的執行使用
        """
        await self._client.add_memories(
            messages=[{"role": "user", "content": "Trip to New York"}],
            user_id=self._user_id
        )

# 從 Memory 到 Run 2 的箭頭（檢索「每日餐費補貼」）
# 這發生在 ContextProvider.invoking() 中

class Mem0Provider(ContextProvider):
    async def invoking(self, messages, **kwargs) -> Context:
        """
        Run 2 開始前，此方法：
        1. 查詢 Mem0 以取得相關記憶體
        2. 檢索：「紐約之旅」
        3. 將它新增到代理程式的情境中
        """
        memories = await self._client.search_memories(
            query="travel, trip",
            user_id=self._user_id
        )
        
        # 將記憶體格式化為額外指示
        instructions = "## Memories\n"
        for memory in memories:
            instructions += f"- {memory['text']}\n"
        
        return Context(instructions=instructions)
```

---

## 完整的程式碼流程

以下是圖表如何轉換為實際程式碼執行：

### 設定階段

```python
# 步驟 1：建立具有長期記憶體提供者的代理程式
from agent_framework.mem0 import Mem0Provider

mem0_provider = Mem0Provider(user_id="user123")

agent = ChatAgent(
    client=chat_client,
    name="TravelAssistant",
    context_providers=mem0_provider
)

# 步驟 2：建立執行緒（代表「Thread: Travel Planning」）
thread = agent.get_new_thread()
```

### Run 1 執行

```python
# 使用者的訊息：「I need to book a hotel in New York for 2 stays.」
result1 = await agent.run(
    "I need to book a hotel in New York for 2 stays.",
    thread=thread
)

# 幕後處理：
# 1. thread.on_new_messages() - 將使用者訊息新增到短期記憶體
# 2. mem0_provider.invoking() - 載入長期記憶體（第一次執行時為空）
# 3. 代理程式處理：
#    - 短期：[「User: I need to book a hotel...」]
#    - 長期：[]
# 4. 代理程式回應：「Here are some suggestions:」
# 5. thread.on_new_messages() - 將代理程式訊息新增到短期記憶體
# 6. mem0_provider.invoked() - 將「紐約之旅」儲存在長期記憶體

# 執行緒現在包含：
# - 使用者的訊息
# - 代理程式的訊息
# 記憶體現在包含：
# - 「紐約之旅」
```

### Run 2 執行

```python
# 使用者的訊息：「What's the daily meal allowance for the business trip?」
result2 = await agent.run(
    "What's the daily meal allowance for the business trip?",
    thread=thread
)

# 幕後處理：
# 1. thread.on_new_messages() - 將使用者訊息新增到短期記憶體
# 2. mem0_provider.invoking() - 載入長期記憶體：
#    → 檢索：「紐約之旅」
# 3. 代理程式處理：
#    - 短期：[
#        「User: I need to book a hotel...」,
#        「Agent: Here are some suggestions...」,
#        「User: What's the daily meal allowance...」
#      ]
#    - 長期：[「紐約之旅」]
# 4. 代理程式回應：「The daily allowance for your business trip is $75...」
# 5. thread.on_new_messages() - 將代理程式訊息新增到短期記憶體
# 6. mem0_provider.invoked() - 將「每日餐費補貼」儲存在長期記憶體

# 執行緒現在包含：
# - 使用者的訊息（Run 1）
# - 代理程式的訊息（Run 1）
# - 使用者的訊息（Run 2）
# - 代理程式的訊息（Run 2）
# 記憶體現在包含：
# - 「紐約之旅」
# - 「每日餐費補貼」
```

---

## 程式碼檔案位置

### 核心元件

| 圖表元件 | 程式碼位置 | 主要類別/函式 |
|---------|-----------|-------------|
| Thread | `python/packages/core/agent_framework/_threads.py` | `AgentThread`（第 292 行） |
| 使用者/代理程式訊息 | `python/packages/core/agent_framework/_threads.py` | `ChatMessageStore`（第 185 行） |
| Memory（基底） | `python/packages/core/agent_framework/_memory.py` | `ContextProvider`（第 74 行） |
| Memory（Mem0） | `python/packages/mem0/agent_framework/mem0/_provider.py` | `Mem0Provider` |
| Memory（Redis） | `python/packages/redis/agent_framework/redis/_provider.py` | `RedisProvider` |
| 情境彙總 | `python/packages/core/agent_framework/_memory.py` | `AggregateContextProvider`（第 189 行） |

### 範例實作

| 範例 | 位置 | 說明 |
|------|------|------|
| 執行緒暫停/繼續 | `python/samples/getting_started/threads/suspend_resume_thread.py` | 顯示執行緒序列化（短期持久性） |
| 自訂訊息存放區 | `python/samples/getting_started/threads/custom_chat_message_store_thread.py` | 執行緒訊息的自訂儲存 |
| Mem0 全域範圍 | `python/samples/getting_started/context_providers/mem0/mem0_threads.py` | 跨執行緒共用的長期記憶體 |
| Mem0 執行緒範圍 | `python/samples/getting_started/context_providers/mem0/mem0_threads.py` | 每個執行緒隔離的長期記憶體 |
| Redis 儲存 | `python/samples/getting_started/context_providers/redis/redis_threads.py` | 可搜尋的長期記憶體 |

---

## 短期 vs 長期記憶體

### 短期記憶體（會話範圍、即時情境）

**特性**：
- 儲存在 `AgentThread.message_store` 中
- 包含當前會話的對話訊息
- 執行緒結束時清除（除非序列化）
- 用於維護對話流程

**程式碼**：
```python
# 位置：python/packages/core/agent_framework/_threads.py:401

class AgentThread:
    async def on_new_messages(self, new_messages: ChatMessage | Sequence[ChatMessage]):
        """將訊息儲存在短期記憶體中（僅此會話）"""
        if self._message_store is None:
            self._message_store = ChatMessageStore()
        
        if isinstance(new_messages, ChatMessage):
            new_messages = [new_messages]
        
        await self._message_store.add_messages(new_messages)
```

### 長期記憶體（持久性、使用者事實）

**特性**：
- 透過 `ContextProvider`（Mem0、Redis 等）儲存
- 包含使用者偏好設定、事實、歷史資料
- 跨會話和執行緒持久化
- 用於個人化和情境延續性

**程式碼**：
```python
# 位置：python/packages/core/agent_framework/_memory.py:119-138

class ContextProvider(ABC):
    async def invoked(
        self,
        request_messages: ChatMessage | Sequence[ChatMessage],
        response_messages: ChatMessage | Sequence[ChatMessage] | None = None,
        **kwargs
    ):
        """
        代理程式回應後呼叫。
        實作使用此方法將資訊儲存在長期記憶體中。
        
        範例：Run 1 後，Mem0Provider 提取「紐約之旅」
        並將它儲存在 Mem0 API 中以供未來檢索。
        """
        pass
```

---

## 摘要圖

```
┌─────────────────────────────────────────────────────────────────────┐
│                    圖表 → 程式碼對應                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Thread（旅行規劃）      →    AgentThread                            │
│  - 使用者的訊息          →    ChatMessageStore 中的 ChatMessage      │
│  - 代理程式的訊息        →    ChatMessageStore 中的 ChatMessage      │
│                                                                      │
│  Run 1                 →    await agent.run(...)                    │
│  - 步驟 1：使用 API     →    ContextProvider.invoking()             │
│  - 步驟 2：建立訊息     →    代理程式處理                            │
│                                                                      │
│  Run 2                 →    await agent.run(...)                    │
│  - 步驟 1：使用 SharePoint →    ContextProvider.invoking()          │
│  - 步驟 2：建立訊息     →    代理程式處理                            │
│                                                                      │
│  Memory                →    ContextProvider 實作                     │
│  - 紐約之旅            →    Mem0Provider / RedisProvider             │
│  - 每日餐費補貼        →    透過 invoked() 方法儲存                  │
│                                                                      │
│  Memory 箭頭（⬇️）      →    資料流：                                │
│  - 到 Memory           →    ContextProvider.invoked()               │
│  - 從 Memory           →    ContextProvider.invoking()              │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 主要要點

1. **Thread = AgentThread**：具有短期訊息歷史記錄的對話容器
2. **Messages = ChatMessageStore**：執行緒內的使用者和代理程式訊息清單
3. **Run = agent.run()**：結合短期 + 長期情境的每次執行
4. **Memory = ContextProvider**：透過 invoking() 存取並透過 invoked() 更新的長期儲存
5. **短期**：當前會話中的對話訊息（AgentThread）
6. **長期**：跨會話持久化的使用者事實（ContextProvider 實作）

架構使代理程式能夠：
- 在會話內記住對話流程（短期）
- 跨會話記住使用者事實和偏好設定（長期）
- 結合兩種類型的記憶體以實現智慧、情境感知的回應

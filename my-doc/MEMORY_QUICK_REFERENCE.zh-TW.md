# 記憶體與對話狀態 - 快速參考指南

這是理解和使用 Microsoft Agent Framework 中記憶體與對話狀態功能的快速參考指南。

## 快速概述

框架提供**兩種類型的記憶體**：

| 記憶體類型 | 實作 | 範圍 | 使用案例 | 範例 |
|-----------|------|------|---------|------|
| **短期** | `AgentThread` + `ChatMessageStore` | 會話範圍 | 對話流程 | 「告訴我關於 X」→「還有什麼關於 X？」 |
| **長期** | `ContextProvider`（Mem0、Redis） | 持久性 | 使用者偏好設定、事實 | 「我喜歡深色模式」（永久記住） |

## 主要類別與位置

```
AgentThread               → python/packages/core/agent_framework/_threads.py:292
ChatMessageStore          → python/packages/core/agent_framework/_threads.py:185
ContextProvider           → python/packages/core/agent_framework/_memory.py:74
Mem0Provider             → python/packages/mem0/agent_framework/mem0/_provider.py
RedisProvider            → python/packages/redis/agent_framework/redis/_provider.py
AggregateContextProvider → python/packages/core/agent_framework/_memory.py:189
```

## 常見模式

### 模式 1：基本執行緒（僅短期記憶體）

```python
from agent_framework.openai import OpenAIChatClient

# 建立代理程式
agent = OpenAIChatClient().create_agent(
    name="Assistant",
    instructions="You are a helpful assistant"
)

# 建立執行緒
thread = agent.get_new_thread()

# 對話（訊息儲存在執行緒中）
await agent.run("What is Python?", thread=thread)
await agent.run("Show me an example", thread=thread)  # 記住情境

# 序列化以便持久化
state = await thread.serialize()
# 將狀態儲存到資料庫...

# 稍後：反序列化並繼續
thread = await agent.deserialize_thread(state)
await agent.run("Can you explain more?", thread=thread)
```

**何時使用**：
- 單一會話內的簡單對話
- 不需要跨會話記憶體
- 想要完全控制訊息歷史記錄

---

### 模式 2：執行緒 + 長期記憶體（建議）

```python
from agent_framework.openai import OpenAIChatClient
from agent_framework.mem0 import Mem0Provider

# 建立具有長期記憶體的代理程式
agent = OpenAIChatClient().create_agent(
    name="Assistant",
    instructions="You are a helpful assistant",
    context_providers=Mem0Provider(
        user_id="user123",
        scope_to_per_operation_thread_id=False  # 跨執行緒共用記憶體
    )
)

# 會話 1
thread1 = agent.get_new_thread()
await agent.run("I prefer dark mode interfaces", thread=thread1)
# → 儲存在 Mem0 長期記憶體

# 會話 2（不同執行緒，但記憶體持久化）
thread2 = agent.get_new_thread()
await agent.run("What are my preferences?", thread=thread2)
# → 代理程式從 Mem0 檢索「深色模式偏好設定」
```

**何時使用**：
- 多會話對話
- 使用者偏好設定和個人化
- 跨執行緒記憶體共用
- 生產應用程式

---

### 模式 3：每個執行緒的長期記憶體

```python
from agent_framework.mem0 import Mem0Provider

# 建立具有執行緒範圍記憶體的代理程式
agent = OpenAIChatClient().create_agent(
    name="Assistant",
    context_providers=Mem0Provider(
        user_id="user123",
        scope_to_per_operation_thread_id=True  # 每個執行緒隔離記憶體
    )
)

# 工作執行緒
work_thread = agent.get_new_thread()
await agent.run("I'm working on Project X", thread=work_thread)
# → 使用 work_thread.id 儲存在 Mem0

# 個人執行緒（不同的記憶體）
personal_thread = agent.get_new_thread()
await agent.run("I like hiking", thread=personal_thread)
# → 使用 personal_thread.id 儲存在 Mem0

# 執行緒之間的記憶體是隔離的
```

**何時使用**：
- 分離的情境（工作與個人）
- 多租戶應用程式
- 執行緒特定的持久性

---

### 模式 4：多個記憶體來源

```python
from agent_framework.mem0 import Mem0Provider
from agent_framework.redis import RedisProvider

# 結合多個情境提供者
agent = OpenAIChatClient().create_agent(
    name="Assistant",
    context_providers=[
        Mem0Provider(user_id="user123"),      # 使用者偏好設定
        RedisProvider(user_id="user123"),     # 可搜尋的事實
    ]
)

# 兩個提供者都會在每次執行前被呼叫
# 兩個提供者都可以在每次執行後儲存記憶體
await agent.run("I prefer Python and like cats", thread=thread)
# → 儲存在 Mem0 和 Redis
```

**何時使用**：
- 多個記憶體後端
- 混合記憶體策略
- 進階生產案例

---

## 程式碼片段

### 儲存和載入執行緒

```python
# 將執行緒儲存到資料庫
thread = agent.get_new_thread()
await agent.run("Tell me about cats", thread=thread)

serialized = await thread.serialize()
# serialized = {
#   "chat_message_store_state": {
#     "messages": [
#       {"role": "user", "content": "Tell me about cats"},
#       {"role": "assistant", "content": "Cats are..."}
#     ]
#   }
# }

# 儲存到資料庫
db.save("thread_123", serialized)

# 稍後：從資料庫載入
serialized = db.load("thread_123")
thread = await agent.deserialize_thread(serialized)
await agent.run("What else?", thread=thread)  # 情境保留
```

### 自訂訊息存放區

```python
class DatabaseMessageStore:
    """將訊息儲存在資料庫中"""
    
    def __init__(self, db_connection, thread_id):
        self.db = db_connection
        self.thread_id = thread_id
    
    async def list_messages(self) -> list[ChatMessage]:
        # 從資料庫載入
        rows = await self.db.query(
            "SELECT * FROM messages WHERE thread_id = ?", 
            self.thread_id
        )
        return [ChatMessage.from_dict(row) for row in rows]
    
    async def add_messages(self, messages: Sequence[ChatMessage]):
        # 儲存到資料庫
        for msg in messages:
            await self.db.insert(
                "INSERT INTO messages VALUES (?, ?, ?)",
                self.thread_id, msg.role, msg.content
            )
    
    async def serialize(self) -> dict:
        messages = await self.list_messages()
        return {"messages": [msg.to_dict() for msg in messages]}
    
    @classmethod
    async def deserialize(cls, serialized_state):
        # 從序列化狀態還原
        messages = [ChatMessage.from_dict(m) for m in serialized_state["messages"]]
        store = cls(db_connection=get_db(), thread_id="restored")
        await store.add_messages(messages)
        return store

# 使用自訂存放區
agent = ChatAgent(
    client=chat_client,
    chat_message_store_factory=DatabaseMessageStore
)
```

### 自訂情境提供者

```python
from agent_framework import ContextProvider, Context

class CustomMemoryProvider(ContextProvider):
    """從資料庫載入使用者偏好設定"""
    
    def __init__(self, user_id: str):
        self.user_id = user_id
        self.db = get_database_connection()
    
    async def thread_created(self, thread_id: str | None):
        """建立新執行緒時呼叫"""
        print(f"New thread created: {thread_id}")
    
    async def invoking(self, messages, **kwargs) -> Context:
        """代理程式呼叫前呼叫 - 載入記憶體"""
        # 查詢資料庫以取得使用者偏好設定
        preferences = await self.db.query(
            "SELECT * FROM user_preferences WHERE user_id = ?",
            self.user_id
        )
        
        # 格式化為指示
        instructions = "## 使用者偏好設定\n"
        for pref in preferences:
            instructions += f"- {pref['key']}: {pref['value']}\n"
        
        # 傳回包含指示的情境
        return Context(instructions=instructions)
    
    async def invoked(self, request_messages, response_messages, **kwargs):
        """代理程式回應後呼叫 - 儲存新記憶體"""
        # 提取並儲存新的偏好設定
        for msg in response_messages or []:
            if "remember" in msg.content.lower():
                # 提取並儲存偏好設定
                await self.db.insert(
                    "INSERT INTO user_preferences VALUES (?, ?, ?)",
                    self.user_id, "extracted_key", "extracted_value"
                )

# 使用自訂提供者
agent = ChatAgent(
    client=chat_client,
    context_providers=CustomMemoryProvider(user_id="user123")
)
```

---

## 生命週期圖

```
使用者：「我喜歡 Python」
       │
       ▼
   agent.run(message, thread=thread)
       │
       ├─→ 1. thread.on_new_messages(user_message)
       │      └─→ 新增到 ChatMessageStore（短期）
       │
       ├─→ 2. context_provider.invoking(messages)
       │      └─→ 從 Mem0/Redis 載入長期記憶體
       │          傳回：Context(instructions="使用者喜歡 Python")
       │
       ├─→ 3. 代理程式使用組合情境處理：
       │      - 短期：來自執行緒的對話歷史記錄
       │      - 長期：來自情境提供者的指示
       │
       ├─→ 4. 代理程式產生回應
       │
       ├─→ 5. thread.on_new_messages(agent_message)
       │      └─→ 新增到 ChatMessageStore（短期）
       │
       └─→ 6. context_provider.invoked(request, response)
              └─→ 將「喜歡 Python」儲存在 Mem0/Redis（長期）
```

---

## 常見問題

### 問：我應該何時使用短期與長期記憶體？

**短期**（`AgentThread`）：
- ✅ 會話內的對話流程
- ✅ 立即後續問題的情境
- ✅ 臨時工作記憶體
- ❌ 跨會話持久性
- ❌ 使用者偏好設定

**長期**（`ContextProvider`）：
- ✅ 使用者偏好設定和事實
- ✅ 跨會話記憶體
- ✅ 個人化
- ✅ 歷史資料
- ❌ 對話流程（使用執行緒）

### 問：如何在 Mem0 和 Redis 之間選擇？

**Mem0Provider**：
- ✅ 受管理的服務（無基礎結構）
- ✅ 自動記憶體提取
- ✅ 易於入門
- ✅ 自我改進記憶體
- ❌ 需要 API 金鑰/外部服務

**RedisProvider**：
- ✅ 自行託管（完全控制）
- ✅ 全文檢索
- ✅ 向量搜尋支援
- ✅ 高效能
- ❌ 需要 Redis 基礎結構

### 問：我可以同時使用服務管理和本機執行緒嗎？

不可以。`AgentThread` 可以有：
- `service_thread_id`（由 AI 服務管理，如 OpenAI Assistants）
- `message_store`（在本機管理）

但**不能同時擁有兩者**。根據您的需求選擇：

**服務管理**（`service_thread_id`）：
```python
# 訊息由 OpenAI 儲存
agent = OpenAIChatClient().create_agent(...)
thread = agent.get_new_thread(service_thread_id="thread_abc123")
```

**本機**（`message_store`）：
```python
# 訊息在本機儲存
agent = ChatAgent(
    client=chat_client,
    chat_message_store_factory=ChatMessageStore
)
thread = agent.get_new_thread()
```

---

## 使用案例範例

### 電子商務助理
```python
# 跨會話記住使用者偏好設定
agent = ChatAgent(
    client=chat_client,
    context_providers=Mem0Provider(
        user_id="customer_123",
        scope_to_per_operation_thread_id=False  # 全域記憶體
    )
)

# 會話 1：使用者分享偏好設定
thread1 = agent.get_new_thread()
await agent.run("I prefer size M and blue colors", thread=thread1)

# 會話 2：代理程式記住
thread2 = agent.get_new_thread()
await agent.run("Show me shirts", thread=thread2)
# → 代理程式建議 M 號藍色襯衫
```

### 客戶支援
```python
# 記住每個執行緒的使用者問題和情境
agent = ChatAgent(
    client=chat_client,
    context_providers=Mem0Provider(
        user_id="customer_123",
        scope_to_per_operation_thread_id=True  # 每個執行緒記憶體
    )
)

# 工單 1 執行緒
ticket1_thread = agent.get_new_thread()
await agent.run("My order is delayed", thread=ticket1_thread)

# 工單 2 執行緒（不同問題，不同情境）
ticket2_thread = agent.get_new_thread()
await agent.run("I need to return an item", thread=ticket2_thread)

# 工單之間的記憶體是隔離的
```

### 個人助理
```python
# 結合短期對話與長期使用者設定檔
agent = ChatAgent(
    client=chat_client,
    context_providers=[
        Mem0Provider(user_id="user_123"),     # 個人偏好設定
        RedisProvider(user_id="user_123"),    # 行事曆、聯絡人等
    ]
)

thread = agent.get_new_thread()

# 代理程式可以存取：
# - 短期：當前對話
# - 長期（Mem0）：使用者偏好設定
# - 長期（Redis）：使用者資料（行事曆等）

await agent.run("Schedule a meeting tomorrow", thread=thread)
# → 代理程式知道使用者的偏好設定和行事曆可用性
```

---

## 相關檔案

### 文件
- `/my-doc/MEMORY_AND_CONVERSATION_STATE_SUMMARY.zh-TW.md` - 全面摘要
- `/my-doc/MEMORY_AND_CONVERSATION_STATE_CODE_MAPPING.zh-TW.md` - 圖表到程式碼的對應

### 範例
- `python/samples/getting_started/threads/suspend_resume_thread.py`
- `python/samples/getting_started/threads/custom_chat_message_store_thread.py`
- `python/samples/getting_started/context_providers/mem0/mem0_threads.py`
- `python/samples/getting_started/context_providers/redis/redis_threads.py`

### 核心程式碼
- `python/packages/core/agent_framework/_threads.py`
- `python/packages/core/agent_framework/_memory.py`
- `python/packages/mem0/agent_framework/mem0/_provider.py`
- `python/packages/redis/agent_framework/redis/_provider.py`

# 記憶體與對話狀態文件

本目錄包含 Microsoft Agent Framework 中記憶體與對話狀態架構的完整文件。

## 概述

Microsoft Agent Framework 提供雙層記憶體架構來建構智慧型代理程式：

1. **短期記憶體**（會話範圍）：在會話內管理即時對話情境
2. **長期記憶體**（持久性）：跨會話儲存使用者偏好設定、事實和歷史資料

## 文件檔案

### 1. [MEMORY_AND_CONVERSATION_STATE_SUMMARY.zh-TW.md](./MEMORY_AND_CONVERSATION_STATE_SUMMARY.zh-TW.md)
**記憶體架構的全面技術摘要**

本文件提供：
- 核心元件的詳細說明（`AgentThread`、`ChatMessageStore`、`ContextProvider`）
- 來自儲存庫的程式碼範例
- 架構圖
- Mem0 和 Redis 提供者的實作細節
- 完整的程式碼流程範例

**何時使用**：當您需要深入了解框架中記憶體運作方式的技術細節時。

---

### 2. [MEMORY_AND_CONVERSATION_STATE_CODE_MAPPING.zh-TW.md](./MEMORY_AND_CONVERSATION_STATE_CODE_MAPPING.zh-TW.md)
**架構圖與實際程式碼之間的視覺化對應**

本文件提供：
- 從圖表元件到程式碼類別/方法的直接對應
- 圖表中每個「執行」的詳細程式碼流程
- 所有相關程式碼的檔案位置
- 完整的執行生命週期
- 短期與長期記憶體的比較

**何時使用**：當您想了解架構圖如何轉換為實際程式碼實作時。

---

### 3. [MEMORY_QUICK_REFERENCE.zh-TW.md](./MEMORY_QUICK_REFERENCE.zh-TW.md)
**包含常見模式和程式碼片段的快速參考指南**

本文件提供：
- 記憶體類型的快速概覽表
- 常見使用模式（4 種主要模式）
- 可複製貼上的程式碼片段
- 生命週期圖
- 常見問題解答部分
- 使用案例範例（電子商務、支援、個人助理）

**何時使用**：當您想快速在應用程式中實作記憶體功能時。

---

## 快速入門

### 初學者：從這裡開始

1. 首先閱讀**快速參考指南**：[MEMORY_QUICK_REFERENCE.zh-TW.md](./MEMORY_QUICK_REFERENCE.zh-TW.md)
2. 嘗試**模式 1**（基本執行緒）以了解短期記憶體
3. 嘗試**模式 2**（執行緒 + 長期記憶體）以新增持久性

### 中級使用者

1. 檢閱**程式碼對應**文件：[MEMORY_AND_CONVERSATION_STATE_CODE_MAPPING.zh-TW.md](./MEMORY_AND_CONVERSATION_STATE_CODE_MAPPING.zh-TW.md)
2. 研究完整的程式碼流程範例
3. 探索自訂實作（自訂訊息存放區、自訂情境提供者）

### 進階使用者

1. 研讀**技術摘要**：[MEMORY_AND_CONVERSATION_STATE_SUMMARY.zh-TW.md](./MEMORY_AND_CONVERSATION_STATE_SUMMARY.zh-TW.md)
2. 檢閱核心實作檔案：
   - `python/packages/core/agent_framework/_threads.py`
   - `python/packages/core/agent_framework/_memory.py`
3. 針對您的特定使用案例建構自訂情境提供者

---

## 架構圖參考

文件基於以下架構圖：

![記憶體與對話狀態架構](https://github.com/user-attachments/assets/f62b33ee-4b3c-4bb5-b9ca-eaa1e35f1b0f)

**圖表中的主要概念**：

- **Thread（執行緒）**：代表具有訊息歷史記錄的對話（短期記憶體）
- **Run 1、Run 2（執行 1、執行 2）**：結合短期和長期情境的個別代理程式呼叫
- **Memory（記憶體）**：透過情境提供者存取的長期持久性儲存
- **Arrows（箭頭）**：執行與記憶體之間的資料流（儲存和檢索資訊）

---

## 程式碼位置

### 核心實作檔案

```
python/packages/core/agent_framework/
├── _threads.py                    # AgentThread、ChatMessageStore
├── _memory.py                     # ContextProvider、Context
└── exceptions.py                  # AgentThreadException

python/packages/mem0/agent_framework/mem0/
└── _provider.py                   # Mem0Provider（長期記憶體）

python/packages/redis/agent_framework/redis/
└── _provider.py                   # RedisProvider（可搜尋記憶體）
```

### 範例檔案

```
python/samples/getting_started/
├── threads/
│   ├── suspend_resume_thread.py              # 執行緒序列化
│   ├── custom_chat_message_store_thread.py   # 自訂儲存
│   └── redis_chat_message_store_thread.py    # Redis 支援的執行緒
└── context_providers/
    ├── mem0/
    │   ├── mem0_basic.py                     # 基本 Mem0 使用
    │   └── mem0_threads.py                   # 記憶體範圍策略
    └── redis/
        └── redis_threads.py                  # Redis 記憶體範例
```

---

## 主要概念

### 短期記憶體（會話範圍）

**實作**：`AgentThread` + `ChatMessageStore`

**特性**：
- 儲存當前會話的對話訊息
- 執行緒結束時清除（除非序列化）
- 用於維護對話流程
- 快速的記憶體內存取

**範例**：
```python
thread = agent.get_new_thread()
await agent.run("What is Python?", thread=thread)
await agent.run("Show me an example", thread=thread)  # 記住「Python」
```

---

### 長期記憶體（持久性）

**實作**：`ContextProvider`（Mem0、Redis、自訂）

**特性**：
- 儲存使用者偏好設定、事實、歷史資料
- 跨會話和執行緒持久化
- 用於個人化和情境延續性
- 與外部儲存整合（Mem0 API、Redis、資料庫）

**範例**：
```python
agent = ChatAgent(
    client=chat_client,
    context_providers=Mem0Provider(user_id="user123")
)

# 會話 1
await agent.run("I prefer dark mode")

# 會話 2（不同執行緒）
await agent.run("What are my preferences?")  # 記住「深色模式」
```

---

## 常見模式

### 模式 1：基本執行緒（僅短期）
```python
thread = agent.get_new_thread()
await agent.run("Message 1", thread=thread)
await agent.run("Message 2", thread=thread)  # 具有訊息 1 的情境
```

### 模式 2：執行緒 + 長期記憶體（建議）
```python
agent = ChatAgent(
    client=chat_client,
    context_providers=Mem0Provider(user_id="user123")
)
thread = agent.get_new_thread()
await agent.run("I like Python", thread=thread)  # 儲存在 Mem0

# 稍後的會話
thread2 = agent.get_new_thread()
await agent.run("What do I like?", thread=thread2)  # 從 Mem0 檢索
```

### 模式 3：多個記憶體來源
```python
agent = ChatAgent(
    client=chat_client,
    context_providers=[
        Mem0Provider(user_id="user123"),
        RedisProvider(user_id="user123"),
    ]
)
```

---

## API 參考

### AgentThread

```python
class AgentThread:
    async def on_new_messages(messages: ChatMessage | Sequence[ChatMessage])
    async def serialize() -> dict[str, Any]
    
    @classmethod
    async def deserialize(serialized_thread_state, message_store=None) -> AgentThread
    
    @property
    def service_thread_id -> str | None
    
    @property
    def message_store -> ChatMessageStoreProtocol | None
```

### ContextProvider

```python
class ContextProvider(ABC):
    async def thread_created(thread_id: str | None) -> None
    async def invoking(messages, **kwargs) -> Context
    async def invoked(request_messages, response_messages, **kwargs) -> None
```

### ChatMessageStore

```python
class ChatMessageStore:
    async def add_messages(messages: Sequence[ChatMessage]) -> None
    async def list_messages() -> list[ChatMessage]
    async def serialize() -> dict[str, Any]
    
    @classmethod
    async def deserialize(serialized_store_state) -> ChatMessageStore
```

---

## 常見問題

### 問：Thread 和 Memory 有什麼區別？

**Thread（執行緒）**（`AgentThread`）：
- 短期對話歷史記錄
- 會話範圍
- 包含使用者和代理程式訊息
- 暫時性（除非序列化）

**Memory（記憶體）**（`ContextProvider`）：
- 長期事實和偏好設定
- 跨會話持久化
- 包含提取的知識
- 永久性

### 問：我應該使用 Mem0 還是 Redis？

**Mem0Provider**：
- ✅ 受管理的服務（無需基礎結構）
- ✅ 自動記憶體提取
- ✅ 易於入門

**RedisProvider**：
- ✅ 自行託管（完全控制）
- ✅ 全文檢索和向量搜尋
- ✅ 高效能

### 問：我可以同時使用兩者嗎？

是的！使用情境提供者陣列：
```python
context_providers=[Mem0Provider(...), RedisProvider(...)]
```

### 問：如何持久化執行緒？

```python
# 儲存
serialized = await thread.serialize()
db.save(serialized)

# 載入
thread = await agent.deserialize_thread(db.load())
```

---

## 相關資源

### Microsoft Learn 文件
- [Agent Framework 概述](https://learn.microsoft.com/agent-framework/overview/agent-framework-overview)
- [使用者指南](https://learn.microsoft.com/agent-framework/user-guide/overview)

### 儲存庫連結
- [Python 範例](../../python/samples/getting_started/)
- [.NET 範例](../../dotnet/samples/GettingStarted/)

### 外部服務
- [Mem0 平台](https://mem0.ai/)
- [Redis](https://redis.io/)

---

## 意見反應和貢獻

如果您有問題、意見反應或改善本文件的建議：
- 提交 [GitHub 問題](https://github.com/microsoft/agent-framework/issues)
- 加入 [Microsoft Azure AI Foundry Discord](https://discord.gg/b5zjErwbQM)

---

## 文件歷史記錄

- **2025-10-14**：基於架構圖建立初始文件
  - 建立全面的技術摘要
  - 建立圖表到程式碼的對應
  - 建立快速參考指南
  - 根據儲存庫程式碼驗證所有文件

---

## 摘要

本文件提供了您在 Microsoft Agent Framework 中理解和實作記憶體功能所需的一切：

1. **三份全面的文件**涵蓋不同層級的細節
2. **程式碼範例**直接從儲存庫提取
3. 架構與程式碼之間的**視覺化對應**
4. 快速實作的**常見模式**
5. 常見問題的**常見問題解答和疑難排解**

從快速參考指南開始以便立即實作，或深入研究技術摘要以獲得完整的理解。

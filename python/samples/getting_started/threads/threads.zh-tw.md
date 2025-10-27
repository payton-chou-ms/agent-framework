# Agent Framework Threads 完整技術指南

## 目錄 (Table of Content)

1. [Summary](#summary)
   - [技術架構總覽](#技術架構總覽)
   - [主要功能模組](#主要功能模組)
   - [精簡解說](#精簡解說)
2. [商務應用](#商務應用)
3. [自訂 Chat Message Store](#1-自訂-chat-message-store)
4. [暫停與恢復執行緒](#2-暫停與恢復執行緒)
5. [Redis Chat Message Store](#3-redis-chat-message-store)
6. [結論](#結論)

## Summary

本文檔涵蓋了 Agent Framework 中執行緒管理 (Thread Management) 的完整功能，包括對話歷史持久化、執行緒暫停/恢復、以及使用 Redis 進行可擴展的訊息儲存。

### 技術架構總覽
- **基礎框架**: Agent Framework
- **核心概念**: Thread (執行緒)、ChatMessageStore (訊息儲存)
- **儲存後端**: In-Memory、Custom、Redis
- **持久化**: 序列化/反序列化、跨會話恢復

### 主要功能模組
| 模組 | 適用場景 | 核心技術 | 特色功能 |
| ---- | -------- | -------- | -------- |
| **Custom Message Store** | 自訂儲存邏輯 | ChatMessageStore 介面 | 靈活的持久化方案 |
| **Suspend/Resume** | 長時程對話 | Thread 序列化 | 跨會話持久化 |
| **Redis Store** | 生產環境、大規模 | RedisChatMessageStore | 分散式、高效能 |

---

### 精簡解說

以下是每個功能模組的核心概念與最精簡程式碼示範:

#### 1️⃣ Custom Chat Message Store
**核心概念**: 實作 ChatMessageStore 介面，自訂訊息儲存邏輯

```python
from agent_framework import ChatMessageStore, ChatMessage

class FileChatMessageStore(ChatMessageStore):
    """將訊息儲存到檔案的自訂實作"""
    
    def __init__(self, file_path: str):
        self._file_path = file_path
        self._messages: list[ChatMessage] = []
    
    async def add(self, message: ChatMessage) -> None:
        """添加訊息並儲存到檔案"""
        self._messages.append(message)
        await self._save()
    
    async def get_messages(self) -> list[ChatMessage]:
        """取得所有訊息"""
        return self._messages.copy()
    
    async def _save(self) -> None:
        """儲存訊息到檔案"""
        import json
        with open(self._file_path, 'w') as f:
            json.dump([msg.model_dump() for msg in self._messages], f)
```

#### 2️⃣ Suspend and Resume Thread
**核心概念**: 序列化執行緒狀態，支援跨會話恢復對話

```python
# 建立帶有訊息儲存的執行緒
thread = agent.get_new_thread(message_store=store)

# 進行對話
await agent.run("Hello", thread=thread)

# 暫停 - 序列化執行緒
thread_data = thread.serialize()
with open("thread_state.json", "w") as f:
    f.write(thread_data)

# 恢復 - 從序列化資料重建
with open("thread_state.json", "r") as f:
    thread_data = f.read()
restored_thread = agent.get_thread_from_serialized(thread_data)

# 繼續對話
await agent.run("Continue our conversation", thread=restored_thread)
```

#### 3️⃣ Redis Chat Message Store
**核心概念**: 使用 Redis 進行分散式、高效能的訊息儲存

```python
from agent_framework_redis import RedisChatMessageStore
import redis.asyncio as redis

# 建立 Redis 連接
redis_client = await redis.from_url("redis://localhost:6379")

# 建立 Redis 訊息儲存
store = RedisChatMessageStore(
    redis_client=redis_client,
    key_prefix="chat:user123:",
    ttl_seconds=86400  # 24 小時過期
)

# 使用儲存建立執行緒
thread = agent.get_new_thread(message_store=store)

# 訊息自動儲存到 Redis
await agent.run("What's the weather?", thread=thread)
```

---

## 商務應用

以下是每個功能模組的實際商務應用場景:

### 1️⃣ Custom Chat Message Store - 商務應用

1. **客製化資料庫整合**:
   - 整合現有的資料庫系統 (PostgreSQL、MongoDB、DynamoDB)
   - 符合企業資料治理政策
   - 統一的資料儲存架構

2. **稽核與合規**:
   - 實作詳細的訊息日誌和追蹤
   - 符合 GDPR、HIPAA 等法規要求
   - 加密敏感對話內容

3. **分析與優化**:
   - 儲存額外的元資料 (時間戳、使用者資訊、意圖)
   - 支援商業智慧和分析查詢
   - 識別常見問題和改進機會

### 2️⃣ Suspend and Resume Thread - 商務應用

1. **客戶服務移交**:
   - 客服人員交接時保留完整對話上下文
   - 跨班次繼續客戶對話
   - 確保服務連續性和品質

2. **長期諮詢服務**:
   - 法律、醫療、財務諮詢跨越多天的對話
   - 保存諮詢歷程用於後續參考
   - 支援間歇性的深度對話

3. **應用程式升級與維護**:
   - 系統維護期間保存活躍對話
   - 無縫的服務中斷恢復
   - 零資料損失的系統遷移

### 3️⃣ Redis Chat Message Store - 商務應用

1. **大規模聊天應用**:
   - 支援數百萬並發使用者
   - 快速的訊息讀寫效能
   - 水平擴展能力

2. **多伺服器部署**:
   - 跨伺服器共享對話狀態
   - 負載平衡和容錯
   - 無狀態應用伺服器設計

3. **即時協作平台**:
   - 團隊成員同時訪問相同對話
   - 即時訊息同步
   - 支援多人 AI 協作場景

---

## 1. 自訂 Chat Message Store

### 主要情境
實作自訂的 `ChatMessageStore` 以符合特定的儲存需求，例如使用特定資料庫或添加自訂邏輯。

### 技術特色
- **介面實作**: 實作 `ChatMessageStore` 協議
- **靈活性**: 完全控制儲存邏輯
- **整合性**: 輕鬆整合現有系統
- **可擴展**: 添加自訂功能如加密、壓縮

### 核心程式碼

```python
# custom_chat_message_store_thread.py
import asyncio
import json
from collections.abc import Sequence
from pathlib import Path

from agent_framework import ChatAgent, ChatMessage, ChatMessageStore
from agent_framework.openai import OpenAIChatClient


class FileChatMessageStore(ChatMessageStore):
    """將聊天訊息儲存到檔案系統的自訂實作"""
    
    def __init__(self, file_path: str | Path):
        self._file_path = Path(file_path)
        self._messages: list[ChatMessage] = []
        self._load()
    
    async def add(self, message: ChatMessage) -> None:
        """添加新訊息並持久化"""
        self._messages.append(message)
        await self._save()
    
    async def get_messages(
        self,
        before: str | None = None,
        limit: int | None = None
    ) -> Sequence[ChatMessage]:
        """取得訊息，支援分頁"""
        messages = self._messages.copy()
        
        if before:
            # 找到指定 ID 之前的訊息
            try:
                idx = next(i for i, m in enumerate(messages) if m.id == before)
                messages = messages[:idx]
            except StopIteration:
                pass
        
        if limit:
            messages = messages[-limit:]
        
        return messages
    
    async def clear(self) -> None:
        """清空所有訊息"""
        self._messages.clear()
        await self._save()
    
    def _load(self) -> None:
        """從檔案載入訊息"""
        if self._file_path.exists():
            with open(self._file_path, 'r') as f:
                data = json.load(f)
                self._messages = [
                    ChatMessage.model_validate(msg) for msg in data
                ]
    
    async def _save(self) -> None:
        """儲存訊息到檔案"""
        self._file_path.parent.mkdir(parents=True, exist_ok=True)
        with open(self._file_path, 'w') as f:
            json.dump(
                [msg.model_dump() for msg in self._messages],
                f,
                indent=2
            )
    
    def serialize(self) -> str:
        """序列化為 JSON 字串"""
        return json.dumps([msg.model_dump() for msg in self._messages])
    
    @classmethod
    def deserialize(cls, data: str) -> "FileChatMessageStore":
        """從序列化資料重建"""
        messages = [ChatMessage.model_validate(msg) for msg in json.loads(data)]
        store = cls(file_path="temp.json")
        store._messages = messages
        return store


async def main():
    """展示自訂訊息儲存的使用"""
    
    # 建立客戶端和代理
    client = OpenAIChatClient()
    agent = ChatAgent(
        chat_client=client,
        instructions="You are a helpful assistant.",
    )
    
    # 建立自訂訊息儲存
    store = FileChatMessageStore(file_path="./chat_history/conversation.json")
    
    # 建立使用自訂儲存的執行緒
    thread = agent.get_new_thread(message_store=store)
    
    print("Session 1: Starting conversation")
    response = await agent.run("Hello! My name is Alice.", thread=thread)
    print(f"Assistant: {response.text}")
    
    response = await agent.run("What's the capital of France?", thread=thread)
    print(f"Assistant: {response.text}")
    
    # 檢查儲存的訊息
    messages = await store.get_messages()
    print(f"\nStored {len(messages)} messages in {store._file_path}")
    
    # 模擬應用程式重啟 - 建立新的代理實例
    print("\n--- Application Restart ---\n")
    
    new_agent = ChatAgent(
        chat_client=OpenAIChatClient(),
        instructions="You are a helpful assistant.",
    )
    
    # 載入相同的儲存
    restored_store = FileChatMessageStore(file_path="./chat_history/conversation.json")
    restored_thread = new_agent.get_new_thread(message_store=restored_store)
    
    # 繼續對話 - 代理會記住之前的上下文
    print("Session 2: Continuing conversation")
    response = await new_agent.run("What's my name?", thread=restored_thread)
    print(f"Assistant: {response.text}")


if __name__ == "__main__":
    asyncio.run(main())
```

**關鍵實作細節**:
- 實作 `add()`, `get_messages()`, `clear()` 方法
- 支援序列化/反序列化用於持久化
- 檔案系統儲存，易於除錯和檢視
- 支援分頁查詢 (before, limit)

---

## 2. 暫停與恢復執行緒

### 主要情境
在應用程式重啟或長時間中斷後，能夠暫停和恢復對話執行緒。

### 技術特色
- **狀態保存**: 完整保存執行緒狀態
- **序列化**: JSON 格式易於儲存和傳輸
- **無縫恢復**: 重建完整的對話上下文
- **跨會話**: 支援長時程對話

### 核心程式碼

```python
# suspend_resume_thread.py
import asyncio
import json
from pathlib import Path

from agent_framework import ChatAgent
from agent_framework.openai import OpenAIChatClient


async def save_thread(agent: ChatAgent, thread_id: str, file_path: str) -> None:
    """暫停執行緒並儲存狀態"""
    thread = agent.get_thread(thread_id)
    thread_data = thread.serialize()
    
    Path(file_path).parent.mkdir(parents=True, exist_ok=True)
    with open(file_path, 'w') as f:
        f.write(thread_data)
    
    print(f"Thread suspended and saved to {file_path}")


async def load_thread(agent: ChatAgent, file_path: str):
    """從檔案載入並恢復執行緒"""
    with open(file_path, 'r') as f:
        thread_data = f.read()
    
    thread = agent.get_thread_from_serialized(thread_data)
    print(f"Thread resumed from {file_path}")
    return thread


async def main():
    """展示暫停和恢復執行緒"""
    
    client = OpenAIChatClient()
    agent = ChatAgent(
        chat_client=client,
        instructions="You are a helpful travel assistant.",
    )
    
    print("=== Session 1: Starting conversation ===\n")
    
    # 建立新執行緒
    thread = agent.get_new_thread()
    thread_id = thread.thread_id
    
    # 進行對話
    response = await agent.run(
        "I'm planning a trip to Japan. What cities should I visit?",
        thread=thread
    )
    print(f"Assistant: {response.text}\n")
    
    response = await agent.run(
        "How many days should I spend in Tokyo?",
        thread=thread
    )
    print(f"Assistant: {response.text}\n")
    
    # 暫停執行緒
    thread_file = "./saved_threads/travel_conversation.json"
    await save_thread(agent, thread_id, thread_file)
    
    print("\n=== Application closed ===\n")
    print("--- Time passes ---\n")
    print("=== Application restarted ===\n")
    
    # 模擬應用程式重啟
    new_agent = ChatAgent(
        chat_client=OpenAIChatClient(),
        instructions="You are a helpful travel assistant.",
    )
    
    # 恢復執行緒
    restored_thread = await load_thread(new_agent, thread_file)
    
    print("\n=== Session 2: Continuing conversation ===\n")
    
    # 繼續對話 - 代理記住之前的上下文
    response = await new_agent.run(
        "What about transportation between cities?",
        thread=restored_thread
    )
    print(f"Assistant: {response.text}\n")
    
    # 檢視執行緒內容
    history = await new_agent.get_chat_history(restored_thread)
    print(f"\nTotal messages in thread: {len(history)}")
    for i, msg in enumerate(history):
        print(f"{i+1}. [{msg.role}]: {msg.text[:50]}...")


if __name__ == "__main__":
    asyncio.run(main())
```

**使用場景**:
- 客服系統交接
- 長期諮詢服務
- 應用程式維護和升級
- 跨裝置繼續對話

**實作要點**:
- 使用 `thread.serialize()` 序列化
- 使用 `agent.get_thread_from_serialized()` 反序列化
- 保存執行緒狀態到持久化儲存
- 支援跨應用程式實例恢復

---

## 3. Redis Chat Message Store

### 主要情境
使用 Redis 作為訊息儲存後端，支援大規模、分散式、高效能的聊天應用。

### 技術特色
- **分散式**: 跨多個應用伺服器共享狀態
- **高效能**: Redis 快速的讀寫能力
- **可擴展**: 水平擴展支援大量使用者
- **TTL 支援**: 自動過期舊對話
- **訊息修剪**: 限制每個執行緒的訊息數量

### 核心程式碼

```python
# redis_chat_message_store_thread.py
import asyncio
from typing import Optional

from agent_framework import ChatAgent
from agent_framework.openai import OpenAIChatClient
from agent_framework_redis import RedisChatMessageStore
import redis.asyncio as redis


async def basic_usage():
    """基本的 Redis 訊息儲存使用"""
    
    print("=== Basic Redis Chat Message Store ===\n")
    
    # 建立 Redis 連接
    redis_client = await redis.from_url("redis://localhost:6379")
    
    # 建立 Redis 訊息儲存
    store = RedisChatMessageStore(
        redis_client=redis_client,
        key_prefix="chat:basic:",
    )
    
    # 建立代理和執行緒
    client = OpenAIChatClient()
    agent = ChatAgent(
        chat_client=client,
        instructions="You are a helpful assistant.",
    )
    
    thread = agent.get_new_thread(message_store=store)
    
    # 進行對話 - 訊息自動儲存到 Redis
    response = await agent.run("Hello! How are you?", thread=thread)
    print(f"Assistant: {response.text}\n")
    
    # 檢查 Redis 中的訊息
    messages = await store.get_messages()
    print(f"Messages in Redis: {len(messages)}")
    
    await redis_client.close()


async def user_session_management():
    """使用 Redis 進行使用者會話管理"""
    
    print("\n=== User Session Management ===\n")
    
    redis_client = await redis.from_url("redis://localhost:6379")
    
    # 為不同使用者建立獨立的訊息儲存
    user_id = "user123"
    store = RedisChatMessageStore(
        redis_client=redis_client,
        key_prefix=f"chat:user:{user_id}:",
        ttl_seconds=3600,  # 1 小時後過期
    )
    
    client = OpenAIChatClient()
    agent = ChatAgent(
        chat_client=client,
        instructions="You are a personal shopping assistant.",
    )
    
    thread = agent.get_new_thread(message_store=store)
    thread_id = thread.thread_id
    
    # 使用者對話
    await agent.run("I'm looking for a laptop", thread=thread)
    await agent.run("Budget is $1000", thread=thread)
    
    print(f"User {user_id} thread {thread_id} created in Redis")
    
    # 稍後恢復相同使用者的會話
    # (可能在不同的伺服器實例上)
    new_store = RedisChatMessageStore(
        redis_client=redis_client,
        key_prefix=f"chat:user:{user_id}:",
    )
    
    new_thread = agent.get_new_thread(
        thread_id=thread_id,
        message_store=new_store
    )
    
    response = await agent.run(
        "What did I say about my budget?",
        thread=new_thread
    )
    print(f"Assistant: {response.text}\n")
    
    await redis_client.close()


async def message_trimming():
    """自動訊息修剪以限制記憶體使用"""
    
    print("\n=== Message Trimming ===\n")
    
    redis_client = await redis.from_url("redis://localhost:6379")
    
    # 設定最大訊息數量
    store = RedisChatMessageStore(
        redis_client=redis_client,
        key_prefix="chat:trimmed:",
        max_messages=10,  # 只保留最近 10 條訊息
    )
    
    client = OpenAIChatClient()
    agent = ChatAgent(
        chat_client=client,
        instructions="You are a helpful assistant.",
    )
    
    thread = agent.get_new_thread(message_store=store)
    
    # 發送多條訊息
    for i in range(15):
        await agent.run(f"Message {i+1}", thread=thread)
    
    # 檢查訊息數量
    messages = await store.get_messages()
    print(f"Total messages after 15 rounds: {len(messages)}")
    print("(Older messages automatically trimmed)\n")
    
    await redis_client.close()


async def cross_application_persistence():
    """跨應用程式重啟的對話持久化"""
    
    print("\n=== Cross-Application Persistence ===\n")
    
    redis_client = await redis.from_url("redis://localhost:6379")
    
    session_id = "session_abc123"
    store = RedisChatMessageStore(
        redis_client=redis_client,
        key_prefix=f"chat:session:{session_id}:",
        ttl_seconds=86400,  # 24 小時
    )
    
    client = OpenAIChatClient()
    agent = ChatAgent(
        chat_client=client,
        instructions="You are a financial advisor.",
    )
    
    thread = agent.get_new_thread(message_store=store)
    
    print("Application Instance 1:")
    await agent.run("I want to invest $10000", thread=thread)
    print("Conversation saved to Redis\n")
    
    # 模擬應用程式重啟
    print("--- Application Restart ---\n")
    
    # 新的應用程式實例
    new_client = OpenAIChatClient()
    new_agent = ChatAgent(
        chat_client=new_client,
        instructions="You are a financial advisor.",
    )
    
    # 使用相同的 Redis key 重建執行緒
    new_store = RedisChatMessageStore(
        redis_client=redis_client,
        key_prefix=f"chat:session:{session_id}:",
    )
    
    # 重建執行緒
    new_thread = new_agent.get_new_thread(
        thread_id=thread.thread_id,
        message_store=new_store
    )
    
    print("Application Instance 2:")
    response = await new_agent.run(
        "What investment amount did I mention?",
        thread=new_thread
    )
    print(f"Assistant: {response.text}")
    print("(Conversation restored from Redis)\n")
    
    await redis_client.close()


async def main():
    """執行所有範例"""
    
    try:
        await basic_usage()
        await user_session_management()
        await message_trimming()
        await cross_application_persistence()
    except redis.ConnectionError:
        print("Error: Cannot connect to Redis.")
        print("Make sure Redis is running on localhost:6379")
        print("Start Redis with: redis-server")


if __name__ == "__main__":
    asyncio.run(main())
```

**環境需求**:
```bash
pip install agent-framework-redis redis
redis-server  # 啟動 Redis 伺服器
```

**環境變數**:
```bash
REDIS_URL=redis://localhost:6379
OPENAI_API_KEY=sk-xxx
```

**關鍵功能**:
- **Key Prefix**: 用於組織和隔離不同使用者/會話的資料
- **TTL**: 自動過期舊對話，節省儲存空間
- **Max Messages**: 限制每個執行緒的訊息數量
- **分散式**: 支援多伺服器共享狀態
- **持久化**: 跨應用程式重啟保留對話

---

## 結論

Agent Framework 的執行緒管理提供了靈活且強大的對話持久化能力。

### 選擇指南

| 需求 | 推薦方案 | 原因 |
| ---- | -------- | ---- |
| **簡單應用** | In-Memory Store | 內建、無需設置 |
| **自訂需求** | Custom Store | 完全控制、整合現有系統 |
| **檔案儲存** | File Store | 簡單、易於除錯 |
| **資料庫整合** | Custom + DB | 企業級、符合規範 |
| **大規模應用** | Redis Store | 高效能、可擴展 |
| **分散式系統** | Redis Store | 跨伺服器共享狀態 |

### 最佳實踐

1. **選擇合適的儲存**:
   - 開發/測試: In-Memory 或 File Store
   - 生產環境: Redis 或資料庫
   - 企業應用: Custom Store + 企業資料庫

2. **訊息管理**:
   ```python
   # 設定訊息限制避免無限增長
   store = RedisChatMessageStore(
       redis_client=redis_client,
       max_messages=100,  # 限制訊息數量
       ttl_seconds=86400,  # 設定過期時間
   )
   ```

3. **執行緒識別**:
   ```python
   # 使用有意義的執行緒 ID
   thread_id = f"user:{user_id}:session:{session_id}"
   thread = agent.get_new_thread(thread_id=thread_id)
   ```

4. **錯誤處理**:
   ```python
   try:
       thread = await load_thread(agent, thread_file)
   except FileNotFoundError:
       # 如果執行緒不存在，建立新的
       thread = agent.get_new_thread()
   ```

5. **效能優化**:
   - 使用訊息分頁避免載入全部歷史
   - 實作快取層減少儲存訪問
   - 定期清理過期執行緒

6. **安全性**:
   - 加密敏感對話內容
   - 實作訪問控制檢查執行緒所有權
   - 遵守資料保留政策

### 進階主題

- **多租戶架構**: 使用 key prefix 隔離租戶資料
- **訊息搜尋**: 整合全文搜尋引擎
- **分析與報告**: 從訊息歷史提取洞察
- **備份與恢復**: 實作定期備份策略
- **效能監控**: 追蹤儲存效能指標

Agent Framework 的執行緒管理為構建可靠、可擴展的對話應用提供了堅實的基礎。

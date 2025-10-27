# Agent Framework Context Providers 完整技術指南

## 目錄 (Table of Content)

1. [Summary](#summary)
2. [商務應用](#商務應用)
3. [簡單 Context Provider](#1-簡單-context-provider)
4. [Mem0 整合](#2-mem0-整合)
5. [Redis 整合](#3-redis-整合)
6. [結論](#結論)

## Summary

本文檔涵蓋 Agent Framework 中上下文提供者 (Context Providers) 的使用，為代理提供持久化記憶能力。

### 技術架構總覽
- **基礎框架**: Agent Framework
- **記憶體後端**: Custom、Mem0、Redis
- **功能**: 對話記憶、使用者偏好、工具輸出

### 主要功能模組
| 模組 | 適用場景 | 核心技術 | 特色功能 |
| ---- | -------- | -------- | -------- |
| **Custom Provider** | 學習、原型 | ContextProvider 介面 | 靈活自訂 |
| **Mem0** | 快速部署 | 託管服務 | 零配置 |
| **Redis** | 生產環境 | 向量搜尋 | 高效能、可擴展 |

### 精簡解說

#### 1️⃣ Custom Context Provider
**核心概念**: 實作自訂記憶體邏輯

```python
from agent_framework import ContextProvider, Context

class UserInfoMemory(ContextProvider):
    def __init__(self):
        self.user_info = {}
    
    async def invoking(self, messages, **kwargs) -> Context:
        # 在代理執行前提供上下文
        instructions = f"User info: {self.user_info}"
        return Context(instructions=instructions)
    
    async def invoked(self, request_messages, response_messages, **kwargs):
        # 在代理執行後提取資訊
        # 更新 self.user_info
        pass
```

#### 2️⃣ Mem0 Integration
**核心概念**: 使用 Mem0 託管記憶體服務

```python
from mem0 import MemoryClient
from agent_framework.mem0 import Mem0Provider

memory_client = MemoryClient(api_key="xxx")
provider = Mem0Provider(
    memory_client=memory_client,
    user_id="user123"
)

agent = client.create_agent(
    instructions="You are a helpful assistant.",
    context_providers=provider
)
```

#### 3️⃣ Redis Integration
**核心概念**: 使用 Redis 進行高效能記憶體儲存

```python
from agent_framework_redis import RedisProvider
import redis.asyncio as redis

redis_client = await redis.from_url("redis://localhost:6379")
provider = RedisProvider(
    redis_client=redis_client,
    user_id="user123"
)

agent = client.create_agent(
    instructions="You are a helpful assistant.",
    context_providers=provider
)
```

## 商務應用

### 1️⃣ 個人化服務
- 記住使用者偏好
- 提供客製化建議
- 累積使用者知識

### 2️⃣ 長期關係
- 跨會話連續性
- 客戶歷史追蹤
- 關係深化

### 3️⃣ 知識管理
- 組織記憶
- 團隊知識共享
- 專家系統

## 結論

Context Providers 讓 AI 代理具備記憶能力，大幅提升使用者體驗。

### 選擇指南

| 需求 | 推薦方案 |
| ---- | -------- |
| **學習** | Custom Provider |
| **快速部署** | Mem0 |
| **生產環境** | Redis |
| **隱私敏感** | Custom + 本地儲存 |

### 最佳實踐

1. **選擇合適的記憶體範圍**
2. **實作資料過期策略**
3. **保護敏感資訊**
4. **優化記憶體檢索**
5. **測試記憶體一致性**

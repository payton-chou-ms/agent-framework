# Agent Framework Middleware 完整技術指南

## 目錄 (Table of Content)

1. [Summary](#summary)
2. [商務應用](#商務應用)
3. [函式與類別中介軟體](#1-函式與類別中介軟體)
4. [中介軟體終止](#2-中介軟體終止)
5. [結果覆寫](#3-結果覆寫)
6. [聊天中介軟體](#4-聊天中介軟體)
7. [結論](#結論)

## Summary

本文檔涵蓋 Agent Framework 中介軟體 (Middleware) 的完整功能，包括代理中介軟體、函式中介軟體和聊天中介軟體。

### 技術架構總覽
- **基礎框架**: Agent Framework
- **中介軟體類型**: Agent Middleware、Function Middleware、Chat Middleware
- **實作方式**: 函式、類別、裝飾器

### 主要功能模組
| 模組 | 適用場景 | 核心技術 | 特色功能 |
| ---- | -------- | -------- | -------- |
| **Agent Middleware** | 代理執行攔截 | @agent_middleware | 日誌、驗證、轉換 |
| **Function Middleware** | 工具呼叫攔截 | @function_middleware | 參數驗證、結果修改 |
| **Chat Middleware** | 聊天請求攔截 | ChatMiddleware | 輸入/輸出轉換 |

### 精簡解說

#### 1️⃣ Function-Based Middleware
**核心概念**: 使用簡單函式實作中介軟體

```python
from agent_framework import agent_middleware, AgentMiddlewareContext

@agent_middleware
async def logging_middleware(context: AgentMiddlewareContext, next):
    print(f"Before: {context.messages}")
    result = await next(context)
    print(f"After: {result}")
    return result
```

#### 2️⃣ Middleware Termination
**核心概念**: 使用中介軟體提前終止執行

```python
@agent_middleware
async def security_middleware(context: AgentMiddlewareContext, next):
    if contains_sensitive_info(context.messages):
        context.terminate = True
        return "Request blocked for security reasons"
    return await next(context)
```

#### 3️⃣ Result Override
**核心概念**: 修改工具或代理的執行結果

```python
@function_middleware
async def result_filter_middleware(context: FunctionMiddlewareContext, next):
    result = await next(context)
    # 過濾敏感資訊
    return filter_sensitive_data(result)
```

## 商務應用

### 1️⃣ 日誌與監控
- 追蹤所有代理執行和工具呼叫
- 效能監控和瓶頸分析
- 異常檢測和警報

### 2️⃣ 安全與合規
- 輸入驗證和清理
- 輸出過濾和審查
- 存取控制和審計追蹤

### 3️⃣ 成本控制
- Token 使用追蹤
- 速率限制實作
- 預算管理

### 4️⃣ 品質保證
- 自動測試和驗證
- A/B 測試支援
- 結果品質評估

## 結論

中介軟體提供了強大的橫切關注點處理能力，是建構生產級 AI 應用的關鍵組件。

### 最佳實踐

1. **保持中介軟體簡單專注**
2. **合理排序中介軟體鏈**
3. **處理好異步執行**
4. **記錄中介軟體行為**
5. **測試中介軟體隔離性**

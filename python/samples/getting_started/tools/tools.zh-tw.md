# Agent Framework Tools 完整技術指南

## 目錄 (Table of Content)

1. [Summary](#summary)
2. [商務應用](#商務應用)
3. [AI Tool 核准機制](#1-ai-tool-核准機制)
4. [失敗的工具處理](#2-失敗的工具處理)
5. [依賴注入工具](#3-依賴注入工具)
6. [結論](#結論)

## Summary

本文檔涵蓋 Agent Framework 中工具 (Tools) 的進階使用模式，包括核准工作流程、錯誤處理和依賴注入。

### 技術架構總覽
- **基礎框架**: Agent Framework
- **核心概念**: Tools/Functions、Approval Mode、Error Handling
- **整合方式**: @ai_function 裝飾器、依賴注入

### 主要功能模組
| 模組 | 適用場景 | 核心技術 | 特色功能 |
| ---- | -------- | -------- | -------- |
| **Tool Approval** | 敏感操作控制 | approval_mode | 使用者核准工作流程 |
| **Error Handling** | 穩健性 | Exception Handling | 優雅的失敗恢復 |
| **Dependency Injection** | 靈活配置 | Runtime Injection | 動態工具配置 |

### 精簡解說

#### 1️⃣ Tool Approval
**核心概念**: 在執行敏感工具前要求使用者核准

```python
from agent_framework import ai_function

@ai_function(approval_mode="always_require")
def send_email(to: str, subject: str, body: str) -> str:
    """發送電子郵件 - 需要使用者核准"""
    # 執行郵件發送
    return f"Email sent to {to}"

# 處理核准請求
result = await agent.run("Send an email to john@example.com")
if result.user_input_requests:
    for request in result.user_input_requests:
        approval = request.create_response(approved=True)
        result = await agent.run(ChatMessage(contents=[approval]), thread=thread)
```

#### 2️⃣ Error Handling
**核心概念**: 優雅處理工具執行失敗

```python
@ai_function
def risky_operation(value: int) -> str:
    """可能失敗的操作"""
    if value == 0:
        raise ValueError("Invalid value: 0")
    return f"Result: {100 / value}"

# Agent 會接收錯誤並可以恢復
result = await agent.run("Perform risky operation with value 0")
# Agent 可以處理錯誤並重試或通知使用者
```

#### 3️⃣ Dependency Injection
**核心概念**: 在執行時注入依賴到工具中

```python
from agent_framework import inject

def get_user_data(user_id: str, db = inject("database")) -> str:
    """使用注入的資料庫連接取得使用者資料"""
    return db.query(f"SELECT * FROM users WHERE id = {user_id}")

# 提供依賴
dependencies = {"database": DatabaseConnection()}
agent = ChatAgent(
    chat_client=client,
    tools=[get_user_data],
    dependencies=dependencies
)
```

## 商務應用

### 1️⃣ Tool Approval - 商務應用

1. **金融交易審核**: 在執行轉帳或支付前要求核准
2. **資料修改保護**: 刪除或更新重要資料前確認
3. **外部API呼叫**: 控制成本高昂的API使用
4. **合規性要求**: 滿足需要人工監督的監管要求

### 2️⃣ Error Handling - 商務應用

1. **API容錯**: 處理外部服務暫時不可用
2. **使用者友好**: 轉換技術錯誤為可理解的訊息
3. **自動重試**: 智能重試失敗的操作
4. **降級處理**: 主要功能失敗時提供替代方案

### 3️⃣ Dependency Injection - 商務應用

1. **多環境配置**: 開發、測試、生產環境使用不同配置
2. **測試便利**: 輕鬆注入模擬物件進行測試
3. **第三方整合**: 動態切換不同的服務提供者
4. **資源管理**: 集中管理資料庫連接、API客戶端等

## 結論

Agent Framework 的工具系統提供了強大且靈活的方式來擴展 AI 代理的能力，同時保持安全性和可維護性。

### 最佳實踐

1. **使用核准模式保護敏感操作**
2. **實作完整的錯誤處理**
3. **使用依賴注入提高可測試性**
4. **記錄所有工具執行用於審計**
5. **為工具提供清晰的文檔字符串**

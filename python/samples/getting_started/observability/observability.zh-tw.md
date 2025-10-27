# Agent Framework Observability 完整技術指南

## 目錄 (Table of Content)

1. [Summary](#summary)
2. [商務應用](#商務應用)
3. [基礎可觀測性設置](#1-基礎可觀測性設置)
4. [Azure AI Agent 可觀測性](#2-azure-ai-agent-可觀測性)
5. [Workflow 可觀測性](#3-workflow-可觀測性)
6. [結論](#結論)

## Summary

本文檔涵蓋 Agent Framework 中可觀測性 (Observability) 的完整功能，包括追蹤、日誌、指標和監控。

### 技術架構總覽
- **基礎框架**: Agent Framework + OpenTelemetry
- **追蹤後端**: Application Insights、Jaeger、Zipkin
- **整合方式**: 環境變數、程式碼配置

### 主要功能模組
| 模組 | 適用場景 | 核心技術 | 特色功能 |
| ---- | -------- | -------- | -------- |
| **Agent Tracing** | 代理執行追蹤 | OpenTelemetry | 分散式追蹤 |
| **Workflow Tracing** | 工作流程監控 | WorkflowTracer | 執行器追蹤 |
| **Azure Monitor** | Azure 整合 | Application Insights | 企業級監控 |

### 精簡解說

#### 1️⃣ Quick Setup with Environment Variables
**核心概念**: 使用環境變數啟用可觀測性

```python
# 設定環境變數
import os
os.environ["AZURE_APP_INSIGHTS_CONNECTION_STRING"] = "InstrumentationKey=xxx"

# 自動啟用追蹤
from agent_framework import ChatAgent
agent = ChatAgent(chat_client=client)
# 所有執行自動追蹤
```

#### 2️⃣ Programmatic Setup
**核心概念**: 在程式碼中配置可觀測性

```python
from agent_framework import enable_observability, ObservabilityConfig

config = ObservabilityConfig(
    connection_string="InstrumentationKey=xxx",
    enable_content_tracing=True,  # 追蹤訊息內容
    enable_token_tracing=True,    # 追蹤 token 使用
)

enable_observability(config)
```

#### 3️⃣ Workflow Tracing
**核心概念**: 追蹤工作流程執行

```python
from agent_framework import WorkflowTracer

class CustomTracer(WorkflowTracer):
    async def on_executor_start(self, executor_id, input_data):
        print(f"Executor {executor_id} started")
    
    async def on_executor_complete(self, executor_id, output_data):
        print(f"Executor {executor_id} completed")

workflow = WorkflowBuilder().with_tracer(CustomTracer()).build()
```

## 商務應用

### 1️⃣ 效能監控
- 識別慢速代理或工具
- 優化 token 使用
- 容量規劃

### 2️⃣ 除錯與診斷
- 追蹤請求流程
- 識別錯誤根因
- 重現問題場景

### 3️⃣ 成本管理
- 追蹤 API 使用量
- 識別成本高昂的操作
- 優化資源使用

### 4️⃣ 合規與審計
- 完整的執行記錄
- 資料存取追蹤
- 符合監管要求

## 結論

可觀測性是生產級 AI 應用的關鍵，提供深入的系統洞察和問題診斷能力。

### 最佳實踐

1. **從開發初期就啟用追蹤**
2. **設定適當的採樣率控制成本**
3. **保護敏感資料不被記錄**
4. **建立警報和儀表板**
5. **定期審查追蹤資料優化效能**

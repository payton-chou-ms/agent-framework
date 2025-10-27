# Agent Framework Agents 完整技術指南

## 目錄 (Table of Content)

1. [Summary](#summary)
2. [商務應用](#商務應用)
3. [Azure AI Agents](#1-azure-ai-agents)
4. [Azure OpenAI Agents](#2-azure-openai-agents)
5. [OpenAI Agents](#3-openai-agents)
6. [Custom Agents](#4-custom-agents)
7. [結論](#結論)

## Summary

本文檔涵蓋 Agent Framework 中各種代理 (Agents) 的建立和使用，包括 Azure AI、Azure OpenAI、OpenAI 和自訂代理。

### 技術架構總覽
- **基礎框架**: Agent Framework
- **支援平台**: Azure AI、Azure OpenAI、OpenAI、Anthropic、Ollama
- **功能**: 工具呼叫、檔案搜尋、程式碼直譯器、執行緒管理

### 主要功能模組
| 模組 | 適用場景 | 核心技術 | 特色功能 |
| ---- | -------- | -------- | -------- |
| **Azure AI** | 企業級 AI | Agent Service | 整合式工具、安全 |
| **Azure OpenAI** | Azure 部署 | Assistants API | 持久化執行緒 |
| **OpenAI** | 快速原型 | 最新模型 | 豐富功能 |
| **Custom** | 特殊需求 | 自訂實作 | 完全控制 |

### 精簡解說

#### 1️⃣ Basic Agent
**核心概念**: 建立簡單的對話代理

```python
from agent_framework.azure import AzureAIAgentClient

client = AzureAIAgentClient(credential=credential)
agent = client.create_agent(
    instructions="You are a helpful assistant.",
    name="assistant"
)

response = await agent.run("Hello!")
print(response.text)
```

#### 2️⃣ Agent with Tools
**核心概念**: 為代理添加函式工具

```python
def get_weather(location: str) -> str:
    return f"Weather in {location}: Sunny"

agent = client.create_agent(
    instructions="You are a weather assistant.",
    tools=[get_weather]
)

response = await agent.run("What's the weather in Tokyo?")
```

#### 3️⃣ Agent with Threads
**核心概念**: 使用執行緒管理長期對話

```python
thread = agent.get_new_thread()

# 多輪對話
await agent.run("My name is Alice", thread=thread)
await agent.run("What's my name?", thread=thread)  # 記住上下文
```

## 商務應用

### 1️⃣ 客戶服務
- 24/7 自動化支援
- 多語言客服
- 問題分類和路由

### 2️⃣ 內容創作
- 文案生成
- 翻譯和本地化
- 內容優化

### 3️⃣ 資料分析
- 自然語言查詢
- 報表生成
- 洞察提取

### 4️⃣ 流程自動化
- 工作流程協調
- 決策支援
- 任務執行

## 結論

Agent Framework 支援多種代理平台，讓開發者可以根據需求選擇最適合的方案。

### 選擇指南

| 需求 | 推薦平台 |
| ---- | -------- |
| **企業部署** | Azure AI |
| **Azure 生態** | Azure OpenAI |
| **快速原型** | OpenAI |
| **特殊需求** | Custom |

### 最佳實踐

1. **選擇適合的平台和模型**
2. **合理設計代理指令**
3. **測試工具整合**
4. **管理執行緒和狀態**
5. **監控和優化效能**

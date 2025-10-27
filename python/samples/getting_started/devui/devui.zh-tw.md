# Agent Framework DevUI 完整技術指南

## 目錄 (Table of Content)

1. [Summary](#summary)
2. [商務應用](#商務應用)
3. [In-Memory Mode](#1-in-memory-mode)
4. [Weather Agent](#2-weather-agent)
5. [Workflow Integration](#3-workflow-integration)
6. [結論](#結論)

## Summary

本文檔涵蓋 Agent Framework DevUI 的使用，提供視覺化的開發和除錯介面。

### 技術架構總覽
- **基礎框架**: Agent Framework + DevUI
- **執行模式**: In-Memory、Persistent
- **功能**: 視覺化對話、除錯、測試

### 主要功能模組
| 模組 | 適用場景 | 核心技術 | 特色功能 |
| ---- | -------- | -------- | -------- |
| **DevUI** | 開發除錯 | Web UI | 視覺化、即時測試 |

### 精簡解說

#### 1️⃣ Enable DevUI
**核心概念**: 為代理啟用開發介面

```python
from agent_framework import ChatAgent
from agent_framework.devui import enable_devui

agent = ChatAgent(chat_client=client)

# 啟用 DevUI
enable_devui(agent, port=5000)

# 訪問 http://localhost:5000 進行測試
```

## 商務應用

### 1️⃣ 快速原型
- 即時測試代理行為
- 調整指令和參數
- 驗證工具整合

### 2️⃣ 除錯
- 視覺化對話流程
- 檢查代理決策
- 識別問題

### 3️⃣ 展示
- 向利害關係人展示功能
- 收集反饋
- 驗證需求

## 結論

DevUI 是開發和除錯 AI 代理的強大工具。

### 最佳實踐

1. **在開發環境使用 DevUI**
2. **不要在生產環境暴露 DevUI**
3. **使用 DevUI 快速迭代**
4. **記錄有趣的測試案例**

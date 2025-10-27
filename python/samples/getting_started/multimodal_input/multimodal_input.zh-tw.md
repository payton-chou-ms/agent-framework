# Agent Framework Multimodal Input 完整技術指南

## 目錄 (Table of Content)

1. [Summary](#summary)
2. [商務應用](#商務應用)
3. [Azure Chat Multimodal](#1-azure-chat-multimodal)
4. [Azure Responses Multimodal](#2-azure-responses-multimodal)
5. [OpenAI Chat Multimodal](#3-openai-chat-multimodal)
6. [結論](#結論)

## Summary

本文檔涵蓋 Agent Framework 中多模態輸入 (Multimodal Input) 的支援，包括圖片、音訊和影片處理。

### 技術架構總覽
- **基礎框架**: Agent Framework
- **支援模態**: 圖片、音訊 (未來)、影片 (未來)
- **平台**: Azure OpenAI、OpenAI

### 主要功能模組
| 模組 | 適用場景 | 核心技術 | 特色功能 |
| ---- | -------- | -------- | -------- |
| **Image Analysis** | 圖片理解 | Vision API | OCR、物體檢測、場景描述 |
| **Image + Text** | 混合輸入 | Multimodal Chat | 圖文結合查詢 |

### 精簡解說

#### 1️⃣ Image Input
**核心概念**: 傳送圖片給 AI 進行分析

```python
from agent_framework import ChatMessage, ImageContent

# 從 URL 載入圖片
image = ImageContent(url="https://example.com/image.jpg")

# 或從本地檔案
with open("image.jpg", "rb") as f:
    image = ImageContent(data=f.read(), mime_type="image/jpeg")

# 發送給代理
message = ChatMessage(role="user", contents=[image, "What's in this image?"])
response = await agent.run([message])
```

#### 2️⃣ Structured Image Analysis
**核心概念**: 從圖片中提取結構化資料

```python
from pydantic import BaseModel

class ImageAnalysis(BaseModel):
    objects: list[str]
    scene: str
    text: str | None

result = await responses_client.get_response(
    messages=[ChatMessage(contents=[image, "Analyze this image"])],
    response_format=ImageAnalysis
)
```

## 商務應用

### 1️⃣ 文件處理
- 發票、收據OCR和資料提取
- 身分證件驗證
- 表單自動填寫

### 2️⃣ 電子商務
- 商品圖片分析和標籤
- 視覺搜尋
- 品質檢查

### 3️⃣ 內容審核
- 不當內容檢測
- 品牌合規檢查
- 圖片品質評估

### 4️⃣ 客戶支援
- 視覺問題診斷
- 產品識別
- 安裝指導

## 結論

多模態支援大大擴展了 AI 代理的應用範圍，讓它們能夠理解和處理視覺資訊。

### 最佳實踐

1. **優化圖片大小以控制成本**
2. **使用結構化輸出提取資料**
3. **實作圖片驗證和清理**
4. **考慮隱私和安全性**
5. **處理多語言OCR需求**

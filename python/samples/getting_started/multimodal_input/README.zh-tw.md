# 多模態輸入範例

此資料夾包含展示如何使用 Agent Framework 向 AI 代理發送多模態內容（影像、音訊、PDF 檔案）的範例。

## 範例

### OpenAI Chat Client

- **檔案**：`openai_chat_multimodal.py`
- **描述**：展示如何向 OpenAI 的 Chat Completions API 發送影像、音訊和 PDF 檔案
- **支援格式**：PNG/JPEG 影像、WAV/MP3 音訊、PDF 文件

### Azure OpenAI Chat Client

- **檔案**：`azure_chat_multimodal.py`
- **描述**：展示如何向 Azure OpenAI Chat Completions API 發送影像
- **支援格式**：PNG/JPEG 影像（Chat Completions API 不支援 PDF 檔案）

### Azure OpenAI Responses Client

- **檔案**：`azure_responses_multimodal.py`
- **描述**：展示如何向 Azure OpenAI Responses API 發送影像和 PDF 檔案
- **支援格式**：PNG/JPEG 影像、PDF 文件（完整的多模態支援）

## 環境變數

在執行範例前設置以下環境變數：

**對於 OpenAI：**
- `OPENAI_API_KEY`：您的 OpenAI API 金鑰

**對於 Azure OpenAI：**

- `AZURE_OPENAI_ENDPOINT`：您的 Azure OpenAI 端點
- `AZURE_OPENAI_CHAT_DEPLOYMENT_NAME`：您的 Azure OpenAI 聊天模型部署名稱
- `AZURE_OPENAI_RESPONSES_DEPLOYMENT_NAME`：您的 Azure OpenAI responses 模型部署名稱

Azure OpenAI 的可選變數：
- `AZURE_OPENAI_API_VERSION`：要使用的 API 版本（預設為 `2024-10-21`）
- `AZURE_OPENAI_API_KEY`：您的 Azure OpenAI API 金鑰（如果不使用 `AzureCliCredential`）

**注意：**您也可以直接在程式碼中提供配置，而不使用環境變數：
```python
# 範例：直接傳遞 deployment_name
client = AzureOpenAIChatClient(
    credential=AzureCliCredential(),
    deployment_name="your-deployment-name",
    endpoint="https://your-resource.openai.azure.com"
)
```

## 驗證

Azure 範例使用 `AzureCliCredential` 進行驗證。在執行範例前，在終端機中執行 `az login`，或將 `AzureCliCredential` 替換為您偏好的驗證方法（例如提供 `api_key` 參數）。

## 執行範例

```bash
# 執行 OpenAI 範例
python openai_chat_multimodal.py

# 執行 Azure OpenAI Chat 範例
python azure_chat_multimodal.py

# 執行 Azure OpenAI Responses 範例
python azure_responses_multimodal.py
```

## 使用您自己的檔案

範例包含用於示範的小型嵌入式測試檔案。要使用您自己的檔案：

### 方法 1：Data URIs（推薦）

```python
import base64

# 載入並編碼您的檔案
with open("path/to/your/image.jpg", "rb") as f:
    image_data = f.read()
    image_base64 = base64.b64encode(image_data).decode('utf-8')
    image_uri = f"data:image/jpeg;base64,{image_base64}"

# 在 DataContent 中使用
DataContent(
    uri=image_uri,
    media_type="image/jpeg"
)
```

### 方法 2：原始位元組

```python
# 載入原始位元組
with open("path/to/your/image.jpg", "rb") as f:
    image_bytes = f.read()

# 在 DataContent 中使用
DataContent(
    data=image_bytes,
    media_type="image/jpeg"
)
```

## 支援的檔案類型

| 類型 | 格式 | 注意事項 |
| --------- | -------------------- | ------------------------------ |
| 影像 | PNG、JPEG、GIF、WebP | 最常見的影像格式 |
| 音訊 | WAV、MP3 | 用於轉錄和分析 |
| 文件 | PDF | 文字提取和分析 |

## API 差異

- **OpenAI Chat Completions API**：支援影像、音訊和 PDF 檔案
- **Azure OpenAI Chat Completions API**：僅支援影像（不支援 PDF/音訊檔案類型）
- **Azure OpenAI Responses API**：支援影像和 PDF 檔案（完整的多模態支援）

根據您的多模態需求和可用的 API 選擇適當的客戶端。

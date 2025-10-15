# 工具範例

## 摘要

此資料夾包含展示 Agent Framework 進階工具使用模式的範例。工具（也稱為函式）允許代理與外部系統互動、執行計算或代表使用者執行操作。這些範例展示關鍵模式，包括核准工作流程、錯誤處理、依賴注入和基於執行緒的工具管理。

工具讓代理能夠：
- 執行外部函式並與 API 互動
- 在執行敏感操作前需要使用者核准
- 優雅地處理錯誤並從失敗中恢復
- 在對話執行緒中維護工具呼叫歷史
- 在執行時注入依賴項以實現靈活的工具配置

## 範例

### 帶核准的 AI 工具

**檔案**：`ai_tool_with_approval.py`

**主要功能**：
- **核准模式**：使用 `approval_mode="always_require"` 來控制工具執行
- **無狀態核准處理**：展示無執行緒的核准工作流程
- **手動訊息管理**：展示不使用執行緒時如何手動傳遞訊息
- **多個工具**：包含已核准（`get_weather_detail`）和未核准（`get_weather`）的工具
- **核准請求迴圈**：迭代處理核准請求直到完成
- **使用者輸入請求**：展示如何處理和回應使用者輸入請求

**使用情境**：需要在執行敏感操作（例如購買、發送電子郵件或修改資料）前獲得使用者同意的代理，無需持久化執行緒儲存。

**關鍵模式**：
```python
@ai_function(approval_mode="always_require")
def get_weather_detail(location: str) -> str:
    """取得詳細天氣 - 需要使用者核准。"""
    return f"{location} 的天氣詳情..."

# 手動處理核准
result = await agent.run(query)
while result.user_input_requests:
    # 收集核准
    approval_response = request.create_response(approved=True)
    result = await agent.run([query, result, approval_response])
```

---

### 帶核准和執行緒的 AI 工具

**檔案**：`ai_tool_with_approval_and_threads.py`

**主要功能**：
- **基於執行緒的核准**：使用執行緒自動管理對話歷史
- **簡化的核准流程**：無需手動追蹤訊息 - 執行緒會處理
- **行事曆工具範例**：展示行事曆事件建立的核准
- **核准請求處理**：展示處理核准請求的簡潔模式
- **執行緒狀態管理**：執行緒自動儲存核准請求和回應

**使用情境**：生產環境應用程式，其中工具核准需要在對話的多個回合中追蹤，例如排程會議、建立任務或預訂的助手。

**關鍵模式**：
```python
@ai_function(approval_mode="always_require")
def add_to_calendar(event_name: str, date: str) -> str:
    """新增行事曆事件 - 需要核准。"""
    return f"已在 {date} 將 '{event_name}' 新增到行事曆"

# 使用執行緒，核准更簡單
thread = agent.get_new_thread()
result = await agent.run(query, thread=thread)

if result.user_input_requests:
    for request in result.user_input_requests:
        approval = request.create_response(approved=True)
        result = await agent.run(ChatMessage(contents=[approval]), thread=thread)
```

---

### 失敗的工具

**檔案**：`failing_tools.py`

**主要功能**：
- **例外處理**：展示優雅處理工具執行失敗
- **錯誤恢復**：展示代理如何從失敗的工具呼叫中恢復
- **ZeroDivisionError 範例**：使用除以零來觸發可預測的失敗
- **繼續對話**：證明工具失敗後對話仍能繼續
- **LLM 的錯誤上下文**：代理接收例外詳情以做出恢復決策
- **對話重播**：展示失敗的工具呼叫如何出現在訊息歷史中

**使用情境**：建構能夠處理工具失敗而不崩潰的穩健代理、重試操作，或以對話方式通知使用者錯誤（例如 API 超時、無效輸入、外部服務失敗）。

**關鍵模式**：
```python
def safe_divide(a: int, b: int) -> str:
    """除法 - 可能因 ZeroDivisionError 失敗。"""
    try:
        result = a / b
    except ZeroDivisionError as exc:
        print(f"工具失敗：{exc}")
        raise  # 代理接收錯誤並可以恢復

# 代理自動處理錯誤
response = await agent.run("除以 10 除以 0", thread=thread)
# 代理：「抱歉，無法執行除以零的操作...」
# 對話正常繼續
```

---

### 帶注入函式的工具

**檔案**：`tool_with_injected_func.py`

**主要功能**：
- **依賴注入**：展示如何在執行時注入函式實作
- **工具序列化**：展示從字典定義建立工具
- **延遲實作**：工具結構與實作分開定義
- **執行時配置**：實現無需更改程式碼的動態工具配置
- **靈活的工具建立**：適用於從配置檔案或資料庫載入工具

**使用情境**：需要根據執行時條件動態配置工具、從外部來源載入工具配置，或支援類似插件的工具系統（實作與結構分開提供）的應用程式。

**關鍵模式**：
```python
# 工具定義（可能來自配置/資料庫）
definition = {
    "type": "ai_function",
    "name": "add_numbers",
    "description": "將兩個數字相加。",
    "input_model": { ... }
}

# 在執行時注入實際實作
def func(a, b) -> int:
    return a + b

tool = AIFunction.from_dict(
    definition,
    dependencies={
        "ai_function": {
            "name:add_numbers": {"func": func}
        }
    }
)
```

**注意**：此功能正在積極開發中，API 可能在未來版本中演變。請始終參考最新文件以獲取更新。

---

## 核心概念

### 工具核准模式

工具可以配置不同的核准模式：
- **無核准**：工具立即執行（預設）
- **`always_require`**：使用者必須明確核准每次工具呼叫
- **條件核准**：自訂邏輯決定何時需要核准

### 核准工作流程

1. 代理識別需要工具呼叫
2. 框架攔截並建立 `UserInputRequest`
3. 應用程式提示使用者核准
4. 使用者以 `approval_response.create_response(approved=True/False)` 回應
5. 如果核准，工具執行；如果拒絕，代理接收拒絕訊息

### 執行緒與無狀態核准

**使用執行緒**（推薦）：
- 執行緒自動儲存對話歷史
- 程式碼更簡單 - 無需手動訊息管理
- 更適合多回合對話

**不使用執行緒**（進階）：
- 必須手動傳遞所有先前的訊息
- 更複雜但更多控制
- 適用於無狀態或無伺服器場景

### 錯誤處理理念

框架遵循「優雅失敗」方法：
- 工具例外被捕獲並報告給代理
- 代理接收錯誤詳情並可以決定如何回應
- 失敗後對話繼續
- 使用者接收自然語言解釋

### 依賴注入的好處

- **靈活性**：更改工具實作而無需更改代理程式碼
- **測試**：注入模擬函式用於單元測試
- **配置**：從外部來源載入工具定義
- **插件系統**：支援第三方工具實作

## 環境變數

```bash
# 用於 OpenAI 範例
OPENAI_API_KEY=your_openai_api_key

# 用於 Azure OpenAI 範例
AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com/
AZURE_OPENAI_DEPLOYMENT_NAME=your-deployment-name
AZURE_OPENAI_API_KEY=your-api-key  # 或使用 Azure CLI 驗證
```

## 執行範例

1. **設置環境變數**（見上文）

2. **導航到工具資料夾**：
   ```bash
   cd python/samples/getting_started/tools
   ```

3. **執行個別範例**：
   ```bash
   # 無執行緒的核准
   python ai_tool_with_approval.py

   # 有執行緒的核准（更簡單的模式）
   python ai_tool_with_approval_and_threads.py

   # 錯誤處理
   python failing_tools.py

   # 依賴注入
   python tool_with_injected_func.py
   ```

## 最佳實踐

### 1. 使用執行緒進行核准工作流程
執行緒通過自動管理對話歷史來簡化核准處理：
```python
thread = agent.get_new_thread()
result = await agent.run(query, thread=thread)
# 執行緒自動追蹤所有訊息
```

### 2. 始終處理工具例外
讓代理優雅地處理錯誤而不是崩潰：
```python
def my_tool(param: str) -> str:
    try:
        return risky_operation(param)
    except Exception as exc:
        raise  # 讓框架處理
```

### 3. 提供清晰的核准上下文
為使用者提供足夠的資訊以做出核准決策：
```python
@ai_function(approval_mode="always_require")
def send_email(to: str, subject: str, body: str) -> str:
    """發送電子郵件 - 需要核准。
    
    Args:
        to: 收件人電子郵件地址
        subject: 電子郵件主旨
        body: 電子郵件訊息內容
    """
    # 清晰的參數幫助使用者了解他們正在核准什麼
```

### 4. 測試工具失敗
確保您的代理優雅地處理失敗：
```python
# 使用無效輸入測試
await agent.run("除以 10 除以 0")
# 驗證代理提供有用的錯誤訊息
```

### 5. 使用型別註解
註解型別為 LLM 提供更好的工具描述：
```python
from typing import Annotated

def get_weather(
    location: Annotated[str, "城市和州，例如 '舊金山, 加州'"]
) -> str:
    """取得某地點的天氣。"""
```

### 6. 考慮安全影響
- 對破壞性操作始終需要核准
- 在執行前驗證工具輸入
- 記錄工具執行以進行稽核追蹤
- 對外部 API 呼叫實施速率限制

## 常見模式

### 帶執行緒的核准

```python
@ai_function(approval_mode="always_require")
def sensitive_operation(param: str) -> str:
    """需要核准的操作。"""
    return f"使用 {param} 執行"

thread = agent.get_new_thread()
result = await agent.run("執行某些敏感操作", thread=thread)

for request in result.user_input_requests:
    # 呈現給使用者，獲取核准
    approved = get_user_approval(request)
    
    response = request.create_response(approved=approved)
    result = await agent.run(
        ChatMessage(contents=[response]),
        thread=thread
    )
```

### 錯誤恢復

```python
def api_call(endpoint: str) -> str:
    """呼叫外部 API - 可能失敗。"""
    try:
        return call_external_api(endpoint)
    except requests.RequestException as exc:
        # 代理接收錯誤並可以重試或通知使用者
        raise

# 代理自動處理失敗
result = await agent.run("呼叫使用者 API")
# 代理：「抱歉，API 目前無法使用...」
```

### 動態工具載入

```python
# 從配置載入工具定義
tool_configs = load_from_database()

tools = []
for config in tool_configs:
    impl = get_implementation(config["name"])
    tool = AIFunction.from_dict(
        config,
        dependencies={
            "ai_function": {
                f"name:{config['name']}": {"func": impl}
            }
        }
    )
    tools.append(tool)

agent = client.create_agent(tools=tools)
```

## 故障排除

### 核准無法運作
- 驗證已設置 `approval_mode="always_require"`
- 檢查您是否正在處理 `user_input_requests`
- 確保核准回應格式正確

### 工具例外未被處理
- 確保例外被引發（而不是被捕獲並以字串返回）
- 驗證代理具有適當的錯誤處理指令
- 檢查工具是否正確註冊到代理

### 依賴注入失敗
- 驗證依賴字典結構與模式匹配
- 確保工具名稱在定義和依賴項之間匹配
- 檢查函式簽名是否與輸入模型匹配

### 訊息未持久化
- 使用執行緒進行自動訊息持久化
- 如果不使用執行緒，手動傳遞所有先前的訊息
- 驗證執行緒正確初始化並傳遞給 `run()`

## 參考資料

- [執行緒管理](../threads/README.md)
- [Agent Framework 核心文件](../../../README.md)
- [中介軟體範例](../middleware/README.md) - 更進階的工具攔截模式

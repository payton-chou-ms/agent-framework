# TAU2 Module - Technical Documentation

## Summary

TAU2 (Task-oriented Agent Understanding 2) benchmark 模組是一個專門用於評估客戶服務場景下 AI Agent 的基準測試框架。本模組基於航空公司客戶服務領域，提供了完整的多代理對話系統，包括客服助理和用戶模擬器的協調執行。主要功能包括：

- **多代理工作流**：協調客服助理 Agent 和用戶模擬器之間的對話
- **環境集成**：與 tau2-bench 環境集成，支持工具調用和狀態管理
- **多維度評估**：使用 tau2 官方評估器，提供全面的性能指標
- **消息管理**：智能的滑動窗口機制，自動管理對話上下文的 token 限制
- **兼容性層**：提供與 tau2-bench 環境的無縫集成

本模組適用於需要評估 Agent Framework 在客戶服務任務場景中表現的應用。

---

## 程式功能介紹

### 1. `__init__.py`
**模組初始化與導出**

**主要功能：**
- 導出 TAU2 模組的核心類和工具函數
- 管理版本信息
- 提供統一的模組接口

**導出項目：**
- `TaskRunner`: 主要的任務執行協調器類
- `ASSISTANT_AGENT_ID`: 客服助理 Agent 的 ID 常量
- `USER_SIMULATOR_ID`: 用戶模擬器的 ID 常量
- `ORCHESTRATOR_ID`: 協調器的 ID 常量
- `patch_env_set_state`: 環境兼容性補丁函數
- `unpatch_env_set_state`: 移除環境補丁函數

**使用場景：**
```python
from agent_framework.lab.tau2 import TaskRunner, ASSISTANT_AGENT_ID
```

---

### 2. `runner.py`
**TAU2 任務執行協調器**

本文件實現了 TAU2 基準測試的核心執行邏輯，是模組的主要組件。

#### `TaskRunner` 類
**多代理對話系統協調器**

**主要功能：**
- 管理客服助理和用戶模擬器之間的對話流程
- 處理工具調用和環境交互
- 監控對話狀態和終止條件
- 執行 tau2 多維度評估
- 收集完整的對話歷史

**架構特點：**
- 使用 Agent Framework 的 Workflow 系統
- 雙 Agent 架構：助理 Agent + 用戶模擬器
- Orchestrator 模式：協調兩個 Agent 的交互
- 滑動窗口消息管理：自動處理 token 限制

**狀態追蹤：**
- `step_count`: 當前對話步驟數
- `full_conversation`: 完整對話歷史
- `termination_reason`: 終止原因（完成、超時等）
- `full_reward_info`: 完整的評估獎勵信息
- `_final_user_message`: 最終用戶消息
- `_assistant_executor`: 助理 Agent 執行器
- `_user_executor`: 用戶模擬器執行器

**配置參數：**
- `max_steps`: 最大對話步驟數（默認需要設置）
- `assistant_sampling_temperature`: 助理 Agent 的採樣溫度（默認 0.0）
- `assistant_window_size`: 助理 Agent 的上下文窗口大小（默認 32768）

**初始化方法：**
```python
def __init__(self, max_steps: int, 
             assistant_sampling_temperature: float = 0.0,
             assistant_window_size: int = 32768)
```

**核心方法：**

##### `reinit()`
重置所有狀態以執行新任務
- 清空對話歷史
- 重置步驟計數
- 清除終止狀態
- 返回 self 以支持鏈式調用

##### `async run_task(task, assistant_client, user_client)`
執行單個 TAU2 任務的主要方法

**參數：**
- `task`: tau2 Task 對象，包含任務描述和環境信息
- `assistant_client`: 客服助理使用的 LLM 客戶端
- `user_client`: 用戶模擬器使用的 LLM 客戶端

**執行流程：**
1. 初始化 tau2 環境和工具
2. 構建雙 Agent 工作流（助理 + 用戶）
3. 創建協調器工作流
4. 執行對話循環直到終止
5. 評估對話結果
6. 返回詳細的評估信息

**返回：**
包含以下字段的字典：
- `evaluation`: RewardInfo 對象（tau2 評估結果）
- `termination_reason`: TerminationReason 枚舉值
- `messages`: 完整對話消息列表
- `config`: 使用的模型配置信息

**內部工作流程：**

1. **環境初始化**
   ```python
   env = get_environment(domain="airline")
   env.init(task=task)
   ```

2. **構建助理 Agent 工作流**
   - 使用 ChatAgent 和 ASSISTANT_AGENT_INSTRUCTION
   - 配置滑動窗口消息存儲
   - 添加環境工具（如查詢航班、修改預訂等）
   - 設置採樣溫度

3. **構建用戶模擬器工作流**
   - 使用 ChatAgent 和用戶模擬器指令
   - 配置消息翻轉（助理 -> 用戶視角）
   - 不提供工具訪問

4. **創建協調器**
   - 交替執行助理和用戶
   - 處理終止條件（STOP、TRANSFER、OUT_OF_SCOPE）
   - 監控步驟限制

5. **執行對話循環**
   ```python
   while not self.termination_reason and self.step_count < self.max_steps:
       response = await orchestrator.run(...)
       self.step_count += 1
   ```

6. **評估結果**
   ```python
   simulation_run = SimulationRun(messages=tau2_messages)
   self.full_reward_info = evaluate_simulation(task, env, simulation_run)
   ```

##### `_handle_agent_executor_request(request)`
處理 Agent 執行器請求的回調函數
- 檢查終止條件（STOP、TRANSFER、OUT_OF_SCOPE）
- 更新對話歷史
- 記錄最終用戶消息

##### `_handle_agent_executor_response(response)`
處理 Agent 執行器響應的回調函數
- 記錄助理響應
- 處理工具調用結果
- 更新對話上下文

**常量定義：**

```python
ASSISTANT_AGENT_ID = "assistant_agent"
USER_SIMULATOR_ID = "user_simulator"
ORCHESTRATOR_ID = "orchestrator"

ASSISTANT_AGENT_INSTRUCTION = """
You are a customer service agent that helps the user according to the <policy> provided below.
In each turn you can either:
- Send a message to the user.
- Make a tool call.
You cannot do both at the same time.
Try to be helpful and always follow the policy. Always make sure you generate valid JSON only.
"""

DEFAULT_FIRST_AGENT_MESSAGE = "Hi! How can I help you today?"
```

**使用場景：**
```python
runner = TaskRunner(max_steps=20)
result = await runner.run_task(
    task=tau2_task,
    assistant_client=assistant_llm,
    user_client=user_llm
)
print(f"Reward: {result['evaluation'].reward}")
```

---

### 3. `_tau2_utils.py`
**TAU2 環境集成工具**

**主要功能：**
提供 Agent Framework 與 tau2-bench 環境之間的轉換和兼容性函數。

#### 核心函數：

##### `convert_tau2_tool_to_ai_function(tau2_tool)`
**將 tau2 Tool 轉換為 AIFunction**

**主要功能：**
- 包裝 tau2 工具為 Agent Framework 的 AIFunction
- 保留工具的名稱、描述和參數模式
- 深拷貝結果防止意外修改

**參數：**
- `tau2_tool`: tau2 的 Tool 對象

**返回：**
- `AIFunction`: Agent Framework 兼容的函數對象

**實現細節：**
```python
def wrapped_func(**kwargs: Any) -> Any:
    result = tau2_tool(**kwargs)
    # 深拷貝以防止修改返回的數據
    return result.model_copy(deep=True) if isinstance(result, BaseModel) else deepcopy(result)
```

**使用場景：**
```python
tau2_tools = env.get_tools()
ai_functions = [convert_tau2_tool_to_ai_function(tool) for tool in tau2_tools]
```

##### `convert_agent_framework_messages_to_tau2_messages(messages)`
**將 Agent Framework 消息轉換為 tau2 消息**

**主要功能：**
- 映射消息角色（system、user、assistant、tool）
- 提取文本內容和函數調用
- 轉換函數結果為 ToolMessage
- 處理多種內容類型

**參數：**
- `messages`: Agent Framework 的 ChatMessage 列表

**返回：**
- `list[Message]`: tau2 的 Message 對象列表

**處理邏輯：**
1. **角色映射**
   - system -> SystemMessage
   - user -> UserMessage
   - assistant -> AssistantMessage
   - tool results -> ToolMessage

2. **內容提取**
   - 提取所有文本類型內容
   - 提取函數調用並轉換為 ToolCall
   - 提取函數結果並創建獨立的 ToolMessage

3. **特殊處理**
   - 函數結果單獨創建為 ToolMessage
   - 保留工具調用的請求者信息

**使用場景：**
```python
tau2_messages = convert_agent_framework_messages_to_tau2_messages(conversation)
simulation_run = SimulationRun(messages=tau2_messages)
```

##### `patch_env_set_state()` / `unpatch_env_set_state()`
**環境狀態管理補丁**

**主要功能：**
- 修改 tau2 Environment 的 set_state 方法
- 避免重複初始化導致的狀態丟失
- 支持測試和調試場景

**使用場景：**
```python
# 在需要時應用補丁
patch_env_set_state()

# 使用環境
env.set_state(...)

# 可選：移除補丁
unpatch_env_set_state()
```

**注意事項：**
- 僅在特定集成場景下需要
- 謹慎使用，可能影響環境行為

---

### 4. `_message_utils.py`
**消息處理工具函數**

**主要功能：**
提供消息處理和日志記錄的工具函數。

#### 核心函數：

##### `flip_messages(messages)`
**翻轉消息角色**

**主要功能：**
- 將助理消息轉換為用戶消息
- 將用戶消息轉換為助理消息
- 用於角色扮演場景（如用戶模擬器）

**轉換邏輯：**
- assistant -> user（過濾掉函數調用）
- user -> assistant
- tool 消息被跳過
- 其他角色保持不變

**使用場景：**
```python
# 在用戶模擬器中使用
user_perspective = flip_messages(assistant_messages)
```

**注意事項：**
- 函數調用在轉換為用戶消息時會被移除
- 保留消息的 author_name 和 message_id

##### `log_messages(messages)`
**彩色消息日志記錄**

**主要功能：**
- 按角色和內容類型彩色輸出消息
- 提供視覺化的調試信息
- 轉義 HTML 字符防止格式問題

**顏色映射：**
- 🔵 SYSTEM: cyan
- 🟢 USER: green
- 🔵 ASSISTANT: blue
- 🟡 TOOL: yellow
- 🟡 TOOL_CALL: yellow with 🔧
- 🟡 TOOL_RESULT: yellow with 🔨
- 🟣 OTHER: magenta

**使用場景：**
```python
log_messages(conversation)
```

**輸出示例：**
```
[USER] I need to change my flight
[ASSISTANT] I'd be happy to help you change your flight
[TOOL_CALL] 🔧 search_flights({"origin": "LAX", "destination": "JFK"})
[TOOL_RESULT] 🔨 ID:call_123 -> [{"flight_id": "AA123", ...}]
```

---

### 5. `_sliding_window.py`
**滑動窗口消息存儲**

**主要功能：**
實現一個 token 感知的滑動窗口消息存儲，自動管理對話上下文的長度限制。

#### `SlidingWindowChatMessageStore` 類

**核心特性：**
- 維護完整歷史和截斷窗口兩份消息列表
- 使用 tiktoken 進行 token 計數
- 超過限制時自動移除最舊消息
- 移除前導工具消息以確保對話流程有效

**初始化參數：**
```python
def __init__(self,
    messages: Sequence[ChatMessage] | None = None,
    max_tokens: int = 3800,
    system_message: str | None = None,
    tool_definitions: Any | None = None
)
```

**核心方法：**

##### `async add_messages(messages)`
添加新消息並自動截斷
- 調用父類添加消息
- 更新截斷消息列表
- 執行自動截斷邏輯

##### `async list_messages()`
獲取當前消息列表（可能已截斷）
- 返回：`list[ChatMessage]` - 截斷後的消息

##### `async list_all_messages()`
獲取所有消息包括被截斷的
- 返回：`list[ChatMessage]` - 完整歷史

##### `truncate_messages()`
執行消息截斷邏輯
- 當 token 數超過 max_tokens 時移除最舊消息
- 移除前導工具消息（工具結果不能是第一條消息）

##### `get_token_count()`
**估算當前消息的 token 數**

**計算邏輯：**
1. 計算系統消息 token（如有）+ 4
2. 每條消息基礎成本 + 4
3. 按內容類型計算：
   - text: 直接編碼計算
   - function_call: 4 + 序列化後計算
   - function_result: 4 + 序列化後計算
   - 其他: 序列化後計算

**警告機制：**
- 超過 50% 時黃色警告
- 超過 100% 時紅色警告

##### `estimate_any_object_token_count(obj)`
估算任意對象的 token 數
- 嘗試 JSON 序列化
- 失敗則使用字符串表示
- 使用 tiktoken 編碼計算

**使用場景：**
```python
store = SlidingWindowChatMessageStore(
    max_tokens=4000,
    system_message="You are a helpful assistant"
)

# 自動管理 token 限制
await store.add_messages(new_messages)
current_messages = await store.list_messages()  # 可能已截斷
full_history = await store.list_all_messages()  # 完整歷史
```

**技術細節：**
- 使用 `o200k_base` 編碼（tiktoken）
- 保守估算以避免超出限制
- 保留完整歷史供評估使用

---

### 6. `samples/run_benchmark.py`
**TAU2 基準測試執行腳本**

**主要功能：**
提供完整的 TAU2 基準測試執行框架，包括批量處理、結果記錄和統計分析。

#### 核心函數：

##### `to_dumpable(result)`
**將結果轉換為 JSONL 可序列化格式**

**主要功能：**
- 處理成功和錯誤兩種情況
- 轉換 Pydantic 模型為字典
- 提取枚舉值為字符串
- 確保 JSON 兼容性

**錯誤情況結構：**
```json
{
  "id": "task_id",
  "error": "error message",
  "evaluation": {"reward": 0.0},
  "config": {...},
  "task": {...}
}
```

**成功情況結構：**
```json
{
  "id": "task_id",
  "evaluation": {...},  // 詳細評估指標
  "config": {...},
  "termination_reason": "completed",
  "messages": [...],  // 完整對話
  "task": {...}
}
```

##### `async run_benchmark(assistant_model, user_model, debug_task_id, max_steps)`
**執行完整基準測試的主函數**

**執行步驟：**

**步驟 1: 配置輸出處理**
- 全量模式：創建時間戳結果文件
- 調試模式：單任務執行，無文件輸出

**步驟 2: 加載數據集**
```python
tasks = get_tasks()  # 加載所有 tau2 航空任務
```

**步驟 3: 配置 LLM 客戶端**
```python
assistant_client = OpenAIChatClient(model=assistant_model)
user_client = OpenAIChatClient(model=user_model)
```

**步驟 4: 執行任務循環**
- 應用環境補丁（如需要）
- 為每個任務初始化 TaskRunner
- 執行任務並捕獲錯誤
- 記錄結果到 JSONL

**步驟 5: 匯總統計**
- 計算總獎勵
- 計算平均獎勵
- 計算成功率
- 打印彩色統計信息

**參數：**
- `assistant_model`: 客服模型 ID（如 "gpt-4o"）
- `user_model`: 用戶模擬器模型 ID
- `debug_task_id`: 可選的單任務 ID（調試模式）
- `max_steps`: 最大對話步驟數

**輸出：**
- 創建時間戳 JSONL 文件（格式：`MMDDHHMM`）
- 打印控制台統計信息

**使用示例：**
```bash
# 完整基準測試
python run_benchmark.py --assistant-model gpt-4o --user-model gpt-4o

# 調試單個任務
python run_benchmark.py --debug-task-id task_001 --max-steps 10

# 自定義模型和步驟
python run_benchmark.py \
  --assistant-model gpt-4-turbo \
  --user-model gpt-4o \
  --max-steps 30
```

**命令行接口：**
```python
if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Run TAU2 benchmark")
    parser.add_argument("--assistant-model", default="gpt-4o")
    parser.add_argument("--user-model", default="gpt-4o")
    parser.add_argument("--debug-task-id", default=None)
    parser.add_argument("--max-steps", type=int, default=20)
    parser.add_argument("--patch-env", action="store_true")
```

**特性：**
- ✅ 批量處理所有任務
- ✅ 單任務調試模式
- ✅ 自動錯誤處理和記錄
- ✅ 詳細的日志輸出
- ✅ 彩色統計報告
- ✅ JSONL 結果保存
- ✅ 環境兼容性補丁

---

## 典型工作流程

### 基本評估流程

```python
from agent_framework.lab.tau2 import TaskRunner
from agent_framework.openai import OpenAIChatClient
from tau2.domains.airline.environment import get_tasks

# 1. 初始化運行器
runner = TaskRunner(max_steps=20, assistant_sampling_temperature=0.0)

# 2. 配置客戶端
assistant_llm = OpenAIChatClient(model="gpt-4o")
user_llm = OpenAIChatClient(model="gpt-4o")

# 3. 加載任務
tasks = get_tasks()

# 4. 執行任務
for task in tasks:
    runner.reinit()
    result = await runner.run_task(task, assistant_llm, user_llm)
    print(f"Task {task.id}: Reward = {result['evaluation'].reward}")
```

### 完整基準測試

```bash
# 使用默認設置運行完整基準測試
python run_benchmark.py

# 查看結果
cat results/gpt-4o_user-gpt-4o_10151430.jsonl
```

---

## 性能考慮

- **Token 管理**：使用滑動窗口避免超出上下文限制
- **步驟限制**：設置合理的 max_steps 避免無限循環
- **錯誤處理**：所有任務執行都有完整的異常捕獲
- **並行化**：當前實現為順序執行，可根據需要添加並行支持
- **內存管理**：完整對話歷史保留在內存中，大規模測試需要注意

---

## 最佳實踐

1. **設置環境變量**：配置 OpenAI API 密鑰
   ```bash
   export OPENAI_API_KEY="sk-..."
   ```

2. **調試單個任務**：開發時使用 `--debug-task-id` 快速測試
   ```bash
   python run_benchmark.py --debug-task-id task_001 --max-steps 10
   ```

3. **監控 Token 使用**：注意滑動窗口的警告信息

4. **結果分析**：使用 JSONL 格式便於後續分析
   ```python
   import json
   with open("results.jsonl") as f:
       results = [json.loads(line) for line in f]
   ```

5. **溫度設置**：評估時使用 `temperature=0.0` 確保可重現性

6. **錯誤追蹤**：檢查輸出中的 traceback 快速定位問題

---

## 與 tau2-bench 的集成

本模組設計為與官方 tau2-bench 庫無縫集成：

- **環境使用**：直接使用 `tau2.domains.airline.environment`
- **工具轉換**：自動轉換 tau2 工具為 Agent Framework 函數
- **消息轉換**：雙向轉換 Agent Framework 和 tau2 消息格式
- **評估器**：使用官方 `tau2.evaluator.evaluator.evaluate_simulation`
- **任務加載**：使用官方 `tau2.domains.airline.environment.get_tasks`

這確保了評估結果與官方基準測試完全一致。

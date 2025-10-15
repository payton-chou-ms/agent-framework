# Lightning Module - Technical Documentation

## Summary

Lightning 模組是 Agent Framework 的強化學習（RL）訓練擴展，專門用於訓練和優化 AI Agent。本模組基於 agentlightning 庫，提供了完整的 RL 訓練流程，包括數據集準備、rollout 執行、獎勵計算和模型優化。主要功能包括：

- **RL 訓練支持**：集成 VERL（Verified Reinforcement Learning）算法進行 Agent 訓練
- **靈活的訓練模式**：支持函數裝飾器和類繼承兩種訓練方式
- **多 Agent 訓練**：支持複雜多 Agent 場景的訓練和 Agent 過濾
- **資源管理**：統一管理 LLM、數據庫等資源
- **可觀測性集成**：啟用 OpenTelemetry 追蹤訓練過程
- **基準測試集成**：支持與 GAIA、TAU2 等基準測試的訓練集成

本模組適用於需要通過強化學習優化 Agent 行為的場景，特別是需要任務導向學習的應用。

**注意**：Lightning 模組目前處於開發階段（in development），API 可能會發生變化。

---

## 程式功能介紹

### 1. `__init__.py`
**RL 模組初始化**

**主要功能：**
- 重新導出 agentlightning 的所有功能
- 提供初始化函數以配置 Agent Framework
- 管理版本信息

**導出內容：**
```python
from agentlightning import *  # 導出所有 agentlightning 組件
```

包括但不限於：
- `LLM`: LLM 資源管理
- `Dataset`: 數據集類型
- `Trainer`: 訓練器類
- `rollout`: rollout 裝飾器
- `LitAgent`: Agent 類繼承基類
- `NamedResources`: 資源管理容器
- `Rollout`: rollout 上下文對象
- `VERL`: VERL 算法實現

**核心函數：**

#### `init()`
**初始化 Lightning 訓練環境**

**主要功能：**
- 啟用 Agent Framework 的 OpenTelemetry 支持
- 配置可觀測性設置

**實現：**
```python
def init() -> None:
    """Initialize the agent-framework-lab-lightning for training."""
    OBSERVABILITY_SETTINGS.enable_otel = True
```

**使用場景：**
```python
from agent_framework_lab_lightning import init as lightning_init

# 在訓練開始前調用
lightning_init()
```

**注意事項：**
- 應在創建任何 Agent 或 Trainer 之前調用
- 啟用後會記錄詳細的訓練追蹤信息

---

### 2. `samples/train_math_agent.py`
**基礎數學 Agent 訓練示例**

**主要功能：**
展示使用 Lightning 模組訓練簡單數學問題解決 Agent 的完整流程。

**示例特點：**
- ✅ 使用 `@rollout` 裝飾器（簡單場景推薦）
- ✅ 集成 MCP calculator 工具
- ✅ 強健的答案評估邏輯
- ✅ 數據集加載和處理
- ✅ VERL 算法訓練
- ✅ 訓練結果保存

**硬件需求：**
- 一個 40GB 顯存的 GPU 即可運行

#### 數據結構

##### `MathProblem` (TypedDict)
**訓練樣本結構定義**

```python
class MathProblem(TypedDict):
    id: str            # 問題唯一標識
    question: str      # 數學問題描述
    chain: str         # 逐步解答過程（訓練時不使用）
    result: str        # 標準答案（用於評估）
    source: str        # 數據來源
```

**設計考慮：**
- 包含評估所需的所有信息
- TypedDict 可選，但有助於類型安全
- 可根據任務需求自定義結構

#### 核心函數

##### `_load_jsonl(file_path)`
**加載 JSONL 格式數據集**

**主要功能：**
- 讀取 JSONL 文件
- 解析為 MathProblem 列表
- 轉換為 Dataset 類型

**使用場景：**
```python
train_dataset = _load_jsonl("data/math/train.jsonl")
val_dataset = _load_jsonl("data/math/val.jsonl")
```

##### 評估函數

**數學答案評估邏輯鏈：**

###### `_normalize_option(option)`
標準化選項答案（移除空格和括號）

###### `_is_option_result(result)`
判斷是否為選項類型答案（A、B、C 等）

###### `_float_eval(input_str)`
評估數學表達式為浮點數
- 處理 "= around" 格式
- 使用 sympy 解析和計算

###### `_scalar_are_results_same(pred_result, true_result, rel_tol)`
**比較標量結果是否相同**

**主要功能：**
- 支持選項答案精確匹配
- 支持數值答案相對誤差匹配
- 處理各種邊界情況

**參數：**
- `pred_result`: 預測答案
- `true_result`: 標準答案
- `rel_tol`: 相對誤差容差（默認 0.01）

**邏輯：**
1. 標準化選項格式
2. 選項答案：精確匹配
3. 數值答案：
   - 計算相對誤差
   - 容差內視為正確

###### `_evaluate_math_result(pred_result, true_result)`
**評估數學結果的主函數**

**主要功能：**
- 處理字符串、數值和容器類型
- 支持多結果比較
- 返回布爾值（正確/錯誤）

**支持格式：**
- 單一答案："42"
- 選項答案："A"
- 數值答案："3.14159"
- 表達式："sqrt(2)"

#### Rollout 實現

##### `@rollout` 裝飾器方式

```python
@rollout(
    model=LLM(name="openai/gpt-4o-mini-2024-07-18"),
    tokenizer=LLM(name="openai/gpt-4o-mini-2024-07-18", tokenizer_only=True),
)
async def run_task(task: MathProblem, llm, tokenizer) -> float:
    """執行單個數學任務並返回獎勵"""
    # 1. 創建帶工具的 Agent
    agent = ChatAgent(
        name="Math Agent",
        instruction="Solve math problems using the calculator tool.",
        llm_client=llm,
        tools=[calculator_tool],
    )
    
    # 2. 運行 Agent
    result = await agent.run(task["question"])
    
    # 3. 提取答案
    predicted_answer = extract_answer(result.text)
    
    # 4. 評估並返回獎勵
    is_correct = _evaluate_math_result(predicted_answer, task["result"])
    return 1.0 if is_correct else 0.0
```

**關鍵點：**
- `@rollout` 裝飾器自動管理資源
- `llm` 和 `tokenizer` 參數由框架注入
- 返回值為獎勵分數（0.0-1.0）
- 適合單 Agent 簡單場景

#### 工具定義

##### Calculator Tool (MCP)
**數學計算器工具**

**主要功能：**
- 使用 MCP stdio 協議
- 提供基本數學計算功能
- 支持表達式求值

**配置：**
```python
calculator_tool = MCPStdioTool(
    name="calculator",
    command="uvx",
    args=["mcp-server-calc"],
)
```

**可用操作：**
- 加減乘除
- 指數和對數
- 三角函數
- 等等

#### 訓練配置

##### `main()` 函數
**訓練流程主函數**

```python
async def main():
    # 1. 初始化
    lightning_init()
    
    # 2. 加載數據
    train_dataset = _load_jsonl("data/math/train.jsonl")
    val_dataset = _load_jsonl("data/math/val.jsonl")
    
    # 3. 創建訓練器
    trainer = Trainer(
        algorithm=VERL(),
        num_epochs=3,
        train_dataset=train_dataset,
        val_dataset=val_dataset,
        output_dir="./outputs/math_agent",
    )
    
    # 4. 開始訓練
    await trainer.train(run_task)
    
    # 5. 保存模型（自動）
```

**訓練參數：**
- `algorithm`: 使用 VERL 算法
- `num_epochs`: 訓練輪數
- `train_dataset`: 訓練集
- `val_dataset`: 驗證集
- `output_dir`: 輸出目錄

**輸出：**
- 訓練好的模型檢查點
- 訓練日誌和指標
- 評估結果

**命令行執行：**
```bash
python train_math_agent.py
```

---

### 3. `samples/train_tau2_agent.py`
**高級多 Agent 訓練示例**

**主要功能：**
展示使用 Lightning 模組訓練複雜多 Agent 系統的高級技術。

**示例特點：**
- ✅ 使用 `LitAgent` 類（高級場景推薦）
- ✅ 多 Agent 場景與 Agent 過濾
- ✅ 資源管理（NamedResources）
- ✅ 與 TAU2 基準測試集成
- ✅ 複雜任務序列化處理
- ✅ 確定性訓練/驗證分割

**硬件需求：**
- 至少一個 80GB 顯存的 GPU

#### 數據結構

##### `SerializedTask` (TypedDict)
**序列化的 TAU2 任務**

```python
class SerializedTask(TypedDict):
    id: str         # 任務 ID
    data: str       # JSON 序列化的任務數據
```

**設計原因：**
- TAU2 任務對象複雜
- 防止 HuggingFace 轉換問題
- 確保分布式訓練兼容性

#### 核心函數

##### `_load_dataset()`
**加載和準備 TAU2 數據集**

**主要功能：**
- 從環境變量讀取數據路徑
- 序列化複雜任務對象
- 確定性訓練/驗證分割（50/50）

**實現細節：**
```python
def _load_dataset() -> tuple[Dataset[SerializedTask], Dataset[SerializedTask]]:
    # 1. 讀取任務
    data_dir = os.getenv("TAU2_DATA_DIR")
    tasks_path = Path(data_dir) / "tau2/domains/airline/tasks.json"
    
    # 2. 序列化
    dataset = [{"id": task["id"], "data": json.dumps(task)} for task in dataset]
    
    # 3. 確定性分割
    random_state = random.Random(42)
    indices = list(range(len(dataset)))
    random_state.shuffle(indices)
    train_indices = indices[:int(len(dataset) * 0.5)]
    val_indices = indices[int(len(dataset) * 0.5):]
    
    return train_dataset, val_dataset
```

**環境要求：**
```bash
export TAU2_DATA_DIR="/path/to/tau2-bench"
```

#### LitAgent 類實現

##### `Tau2Agent` 類
**基於類的 Agent 實現**

**為什麼使用 LitAgent：**
- ✅ Agent 過濾（只訓練特定 Agent）
- ✅ 資源管理（多個 LLM、數據庫等）
- ✅ 複雜初始化邏輯
- ✅ 更好的代碼組織

**類結構：**
```python
class Tau2Agent(LitAgent):
    """類繼承式 Agent，支持高級資源管理和 Agent 過濾"""
    
    async def rollout_async(
        self, 
        task: SerializedTask, 
        resources: NamedResources, 
        rollout: Rollout
    ) -> float:
        """主 rollout 方法，類似 @rollout 但控制更精細"""
```

**核心方法實現：**

###### `rollout_async(task, resources, rollout)`
**執行單次 rollout**

**參數：**
- `task`: SerializedTask 對象
- `resources`: 命名資源容器
- `rollout`: Rollout 上下文對象

**返回：**
- `float`: 獎勵分數

**實現流程：**

```python
async def rollout_async(self, task, resources, rollout):
    # 1. 獲取資源
    llm = resources.get("main_llm")
    
    # 2. 反序列化任務
    tau2_task = Tau2Task(**json.loads(task["data"]))
    
    # 3. 應用環境補丁
    patch_env_set_state()
    
    # 4. 創建 TAU2 TaskRunner
    runner = Tau2TaskRunner(
        max_steps=20,
        assistant_sampling_temperature=1.0
    )
    
    # 5. 執行任務
    result = await runner.run_task(
        task=tau2_task,
        assistant_client=llm,
        user_client=llm  # 用戶模擬器使用固定模型
    )
    
    # 6. 提取獎勵
    reward = result["evaluation"].reward
    
    # 7. 標記可訓練的 Agent
    rollout.mark_agent_as_trainable(ASSISTANT_AGENT_ID)
    
    return reward
```

**Agent 過濾關鍵：**
```python
rollout.mark_agent_as_trainable(ASSISTANT_AGENT_ID)
```
- 只訓練客服助理 Agent
- 用戶模擬器保持固定
- 提高訓練效率和穩定性

#### 訓練配置

##### 資源定義
```python
resources = NamedResources({
    "main_llm": LLM(
        name="openai/gpt-4o-mini-2024-07-18",
        # 訓練配置
    ),
})
```

**NamedResources 優勢：**
- 統一管理所有資源
- 自動生命週期管理
- 支持複雜資源依賴

##### `main()` 函數
**高級訓練流程**

```python
async def main():
    # 1. 初始化
    lightning_init()
    
    # 2. 加載數據
    train_dataset, val_dataset = _load_dataset()
    
    # 3. 定義資源
    resources = NamedResources({
        "main_llm": LLM(name="openai/gpt-4o-mini-2024-07-18"),
    })
    
    # 4. 創建 Agent 實例
    agent = Tau2Agent()
    
    # 5. 創建訓練器
    trainer = Trainer(
        algorithm=VERL(),
        num_epochs=2,
        train_dataset=train_dataset,
        val_dataset=val_dataset,
        resources=resources,
        output_dir="./outputs/tau2_agent",
    )
    
    # 6. 開始訓練
    await trainer.train(agent)
```

**關鍵差異：**
- 傳入 `agent` 實例而非函數
- 使用 `resources` 參數
- Agent 過濾在 rollout 中處理

**執行：**
```bash
export TAU2_DATA_DIR="/path/to/tau2-bench"
python train_tau2_agent.py
```

---

## 訓練模式比較

### @rollout 裝飾器（簡單場景）

**適用場景：**
- ✅ 單 Agent 系統
- ✅ 簡單資源需求
- ✅ 快速原型開發

**優點：**
- 代碼簡潔
- 快速上手
- 自動資源管理

**示例：**
```python
@rollout(model=LLM(...), tokenizer=LLM(...))
async def run_task(task, llm, tokenizer):
    agent = ChatAgent(llm_client=llm, ...)
    result = await agent.run(task["question"])
    return compute_reward(result)
```

### LitAgent 類（高級場景）

**適用場景：**
- ✅ 多 Agent 系統
- ✅ Agent 過濾需求
- ✅ 複雜資源管理
- ✅ 需要狀態管理

**優點：**
- 更好的代碼組織
- 精細的控制
- 支持 Agent 過濾
- 資源共享

**示例：**
```python
class MyAgent(LitAgent):
    async def rollout_async(self, task, resources, rollout):
        llm = resources.get("main_llm")
        # 複雜邏輯
        rollout.mark_agent_as_trainable("agent1")
        return reward
```

---

## 典型工作流程

### 基礎訓練流程

```python
from agent_framework_lab_lightning import init as lightning_init
from agentlightning import Trainer, VERL, Dataset, LLM, rollout

# 1. 初始化
lightning_init()

# 2. 準備數據
train_data = load_your_dataset()
val_data = load_validation_dataset()

# 3. 定義 rollout
@rollout(model=LLM(name="openai/gpt-4o-mini-2024-07-18"))
async def run_task(task, llm):
    # Agent 邏輯
    reward = evaluate_result(result)
    return reward

# 4. 訓練
trainer = Trainer(
    algorithm=VERL(),
    num_epochs=3,
    train_dataset=train_data,
    val_dataset=val_data,
)
await trainer.train(run_task)
```

### 高級多 Agent 訓練

```python
from agent_framework_lab_lightning import init as lightning_init
from agentlightning import (
    Trainer, VERL, LitAgent, NamedResources, Rollout, LLM
)

# 1. 初始化
lightning_init()

# 2. 定義 Agent 類
class MyMultiAgent(LitAgent):
    async def rollout_async(self, task, resources, rollout):
        # 多 Agent 協調邏輯
        rollout.mark_agent_as_trainable("assistant")
        return reward

# 3. 配置資源
resources = NamedResources({
    "main_llm": LLM(name="openai/gpt-4o-mini-2024-07-18"),
    "eval_llm": LLM(name="openai/gpt-4o"),
})

# 4. 訓練
trainer = Trainer(
    algorithm=VERL(),
    num_epochs=2,
    train_dataset=train_data,
    val_dataset=val_data,
    resources=resources,
)
await trainer.train(MyMultiAgent())
```

---

## 獎勵設計最佳實踐

### 1. 清晰的成功標準
```python
# ✅ 好的獎勵設計
def compute_reward(prediction, ground_truth):
    if exact_match(prediction, ground_truth):
        return 1.0
    elif partial_match(prediction, ground_truth):
        return 0.5
    else:
        return 0.0
```

### 2. 數值穩定性
```python
# ✅ 使用相對誤差而非絕對差值
def compute_numeric_reward(pred, true, rel_tol=0.01):
    rel_error = abs(pred - true) / max(abs(true), 1e-10)
    return 1.0 if rel_error < rel_tol else 0.0
```

### 3. 強健的評估
```python
# ✅ 處理各種邊界情況
def robust_evaluate(pred, true):
    try:
        pred = normalize(pred)
        true = normalize(true)
        return compare(pred, true)
    except Exception as e:
        logger.warning(f"Evaluation failed: {e}")
        return 0.0  # 失敗時返回 0
```

---

## 性能考慮

- **GPU 內存**：根據模型大小選擇合適的 GPU
- **批次大小**：通過 VERL 配置調整批次大小
- **檢查點**：定期保存檢查點以防訓練中斷
- **驗證頻率**：平衡訓練速度和評估頻率
- **並行 Rollout**：VERL 自動處理並行化

---

## 最佳實踐

1. **初始化在最開始**
   ```python
   lightning_init()  # 在任何其他代碼之前
   ```

2. **數據集分割**
   - 使用確定性隨機種子
   - 訓練/驗證比例合理（如 80/20 或 50/50）

3. **獎勵函數**
   - 保持獎勵範圍在 0.0-1.0
   - 提供有意義的中間獎勵
   - 處理所有邊界情況

4. **Agent 過濾**（多 Agent 場景）
   - 只訓練需要優化的 Agent
   - 保持其他 Agent 固定
   - 使用 `rollout.mark_agent_as_trainable()`

5. **資源管理**
   - 使用 NamedResources 統一管理
   - 複用資源避免重複創建
   - 正確清理資源

6. **監控訓練**
   - 查看訓練日誌
   - 監控驗證集性能
   - 檢查獎勵分佈

7. **漸進式開發**
   - 從簡單任務開始
   - 使用小數據集測試
   - 逐步增加複雜度

---

## 故障排除

### 常見問題

**問題：GPU 內存不足**
```
解決：
- 減少批次大小
- 使用更小的模型
- 啟用梯度檢查點
```

**問題：訓練不收斂**
```
解決：
- 檢查獎勵函數邏輯
- 調整學習率
- 增加訓練數據
- 檢查數據質量
```

**問題：Agent 過濾不生效**
```
解決：
- 確保使用 LitAgent 類
- 正確調用 mark_agent_as_trainable()
- 檢查 Agent ID 是否正確
```

**問題：資源未正確初始化**
```
解決：
- 在開始前調用 lightning_init()
- 檢查 NamedResources 配置
- 確保資源名稱匹配
```

---

## 未來發展方向

- 更多算法支持（PPO、SAC 等）
- 更好的分布式訓練支持
- 改進的檢查點管理
- 更豐富的評估指標
- 更多示例和教程

**注意**：Lightning 模組仍在積極開發中，API 可能會有變化。請關注官方文檔獲取最新信息。

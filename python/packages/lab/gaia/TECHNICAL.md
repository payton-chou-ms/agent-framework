# GAIA Module - Technical Documentation

## Summary

GAIA (General AI Assistant) benchmark 模組是一個用於評估通用 AI 助理的完整框架。本模組提供了標準化的基準測試工具，讓開發者能夠評估其 Agent 在各種任務上的表現。主要功能包括：

- **標準化評估流程**：提供完整的任務加載、執行、評估和結果記錄流程
- **靈活的評估器**：支持自定義評估邏輯或使用內建的 GAIA 官方評分器
- **可觀測性支持**：集成 OpenTelemetry 進行詳細的追蹤和性能分析
- **並行執行**：支持多任務並行執行以提高評估效率
- **結果可視化**：提供命令行工具查看和分析評估結果

本模組適用於需要評估 Agent Framework 構建的智能助理在通用任務上表現的場景。

---

## 程式功能介紹

### 1. `__init__.py`
**模組初始化與導出**

**主要功能：**
- 導出 GAIA 模組的核心類型和函數
- 管理版本信息
- 提供統一的模組接口

**導出項目：**
- `GAIA`: 主要的基準測試運行器類
- `GAIATelemetryConfig`: 遙測和追蹤配置類
- `Evaluation`, `Evaluator`, `Prediction`, `Task`, `TaskResult`, `TaskRunner`: 核心類型定義
- `gaia_scorer`: GAIA 官方評分函數
- `viewer_main`: 結果查看器入口函數

**使用場景：**
```python
from agent_framework.lab.gaia import GAIA, Task, Prediction
```

---

### 2. `_types.py`
**核心類型定義**

**主要功能：**
定義評估系統中使用的所有核心數據結構，確保類型安全和清晰的接口。

**核心類型：**

#### `Task` (dataclass)
表示單個評估任務
- `task_id`: 任務唯一標識符
- `question`: 任務問題
- `answer`: 標準答案（可選）
- `level`: 難度等級（可選）
- `file_name`: 關聯文件名（可選）
- `metadata`: 額外的元數據（可選）

#### `Prediction` (dataclass)
表示 Agent 對任務的預測結果
- `prediction`: 預測答案字符串
- `messages`: 對話消息列表（可選）
- `metadata`: 預測過程的元數據（可選）

#### `Evaluation` (dataclass)
表示對預測的評估結果
- `is_correct`: 是否正確
- `score`: 得分（0.0-1.0）
- `details`: 評估詳細信息（可選）

#### `TaskResult` (dataclass)
完整的任務執行結果
- `task_id`: 任務 ID
- `task`: 原始任務對象
- `prediction`: 預測結果
- `evaluation`: 評估結果
- `runtime_seconds`: 執行時間（可選）
- `error`: 錯誤信息（可選）

#### `TaskRunner` (Protocol)
任務運行器協議，定義執行任務的接口
- `__call__(task: Task) -> Prediction`: 異步執行任務並返回預測

#### `Evaluator` (Protocol)
評估器協議，定義評估邏輯的接口
- `__call__(task: Task, prediction: Prediction) -> Evaluation`: 異步評估預測結果

**使用場景：**
- 實現自定義任務運行器
- 實現自定義評估邏輯
- 類型提示和靜態分析

---

### 3. `gaia.py`
**GAIA 基準測試核心實現**

本文件包含 GAIA 基準測試的完整實現，是模組的核心。

#### `GAIATelemetryConfig` 類
**遙測和追蹤配置**

**主要功能：**
- 配置 OpenTelemetry 追蹤
- 支持多種導出方式（OTLP、Azure Monitor、本地文件）
- 自動設置觀測性基礎設施

**初始化參數：**
- `enable_tracing`: 是否啟用追蹤（默認 False）
- `otlp_endpoint`: OTLP 端點 URL（可選）
- `applicationinsights_connection_string`: Azure Monitor 連接字符串（可選）
- `trace_to_file`: 是否導出到本地文件（默認 False）
- `file_path`: 本地文件路徑（默認 "gaia_traces.json"）

**核心方法：**
- `setup_observability()`: 設置 OpenTelemetry 基礎設施
- `_setup_file_export()`: 配置本地文件導出器

**使用場景：**
```python
telemetry_config = GAIATelemetryConfig(
    enable_tracing=True,
    trace_to_file=True,
    file_path="gaia_traces.jsonl"
)
```

#### `GAIA` 類
**GAIA 基準測試運行器**

**主要功能：**
- 自動下載和緩存 GAIA 數據集
- 管理任務加載和過濾
- 協調任務執行和評估
- 支持並行執行和超時控制
- 記錄詳細的執行追蹤

**初始化參數：**
- `evaluator`: 自定義評估器（可選，默認使用 GAIA 官方評分器）
- `data_dir`: 數據緩存目錄（可選）
- `hf_token`: Hugging Face 訪問令牌（可選）
- `telemetry_config`: 遙測配置（可選）

**核心方法：**

##### `async run(task_runner, level, max_n, parallel, timeout, out)`
執行基準測試的主要方法
- `task_runner`: 任務運行器函數或對象
- `level`: 難度等級（1, 2, 3 或列表）（可選）
- `max_n`: 每個等級最多執行的任務數（可選）
- `parallel`: 並行執行的任務數（默認 1）
- `timeout`: 每個任務的超時時間（秒）（可選）
- `out`: 輸出文件路徑（可選）
- 返回：`list[TaskResult]` - 所有任務的執行結果

##### `_ensure_data()`
確保 GAIA 數據集可用
- 檢查本地緩存
- 如需要則從 Hugging Face 下載數據集

##### `_default_evaluator(task, prediction)`
默認評估器實現
- 使用 GAIA 官方評分函數進行評估

##### `_run_single_task(task, task_runner, semaphore, timeout)`
執行單個任務
- 包含錯誤處理
- 記錄執行時間
- 添加追蹤 span

**使用場景：**
```python
async def run_task(task: Task) -> Prediction:
    # 自定義任務執行邏輯
    return Prediction(prediction="answer")

runner = GAIA()
results = await runner.run(
    run_task,
    level=1,
    max_n=5,
    parallel=2,
    timeout=60
)
```

#### 工具函數

##### `gaia_scorer(model_answer, ground_truth)`
**GAIA 官方評分函數**

**主要功能：**
- 標準化字符串比較
- 處理數字答案的近似匹配
- 支持逗號分隔的多答案匹配

**邏輯：**
1. 標準化兩個字符串（去除標點、小寫化）
2. 如果是數字，進行數值比較（容差 1e-3）
3. 否則進行精確字符串匹配

**使用場景：**
```python
is_correct = gaia_scorer(prediction, ground_truth)
```

##### `_normalize_str(s, remove_punct)`
標準化字符串用於比較
- 轉小寫
- 去除多餘空格
- 可選去除標點符號

##### `_normalize_number_str(number_str)`
標準化數字字符串並轉換為浮點數
- 處理逗號分隔符
- 處理分數表示

##### `_split_string(s, chars)`
按多個分隔符分割字符串

##### `_read_jsonl(path)`
讀取 JSONL 文件並逐行解析

##### `_load_gaia_local(repo_dir, wanted_levels, max_n)`
從本地目錄加載 GAIA 任務
- 遍歷所有 metadata.jsonl 文件
- 提取任務信息
- 按等級過濾
- 隨機打亂並限制數量

##### `viewer_main()`
**GAIA 結果查看器命令行工具**

**主要功能：**
- 從 JSONL 文件加載評估結果
- 支持多種過濾選項
- 顯示匯總統計信息
- 可選詳細模式查看每個任務

**命令行參數：**
- `results_file`: 結果文件路徑
- `--detailed`: 顯示詳細信息
- `--level`: 按等級過濾
- `--correct-only`: 只顯示正確的結果
- `--incorrect-only`: 只顯示錯誤的結果

**使用場景：**
```bash
uv run gaia_viewer "gaia_results_20231015.jsonl" --detailed --level 1
```

---

### 4. `samples/gaia_sample.py`
**GAIA 使用示例**

**主要功能：**
展示如何使用 GAIA 模組進行基準測試評估。

**示例內容：**

1. **自定義評估器實現**
   ```python
   def evaluate_task(task: Task, prediction: Prediction) -> Evaluation:
       is_correct = (task.answer or "").lower() in prediction.prediction.lower()
       return Evaluation(is_correct=is_correct, score=1 if is_correct else 0)
   ```

2. **配置遙測**
   - 啟用追蹤
   - 配置本地文件導出

3. **創建和重用 Agent**
   - 使用 Azure AI Agent Client
   - 使用異步上下文管理器管理生命週期

4. **實現任務運行器**
   ```python
   async def run_task(task: Task) -> Prediction:
       input_message = f"Task: {task.question}"
       result = await agent.run(input_message)
       return Prediction(prediction=result.text, messages=result.messages)
   ```

5. **執行基準測試**
   - 指定難度等級
   - 限制任務數量
   - 配置並行度和超時

6. **處理結果**
   - 打印詳細結果
   - 顯示任務 ID、問題、預測和評估

**使用場景：**
- 學習如何使用 GAIA 模組
- 快速開始基準測試
- 自定義評估邏輯的參考

**執行方式：**
```bash
uv run python run_gaia.py
```

---

## 典型工作流程

1. **初始化配置**
   ```python
   telemetry_config = GAIATelemetryConfig(enable_tracing=True)
   runner = GAIA(telemetry_config=telemetry_config)
   ```

2. **實現任務運行器**
   ```python
   async def run_task(task: Task) -> Prediction:
       # 使用您的 Agent 處理任務
       result = await my_agent.process(task.question)
       return Prediction(prediction=result)
   ```

3. **執行基準測試**
   ```python
   results = await runner.run(
       run_task,
       level=[1, 2],
       max_n=10,
       parallel=3,
       timeout=120,
       out="results.jsonl"
   )
   ```

4. **分析結果**
   ```bash
   uv run gaia_viewer results.jsonl --detailed
   ```

---

## 性能考慮

- **並行度**：根據 API 限制和資源調整 `parallel` 參數
- **超時設置**：根據任務複雜度設置合適的 `timeout` 值
- **數據緩存**：首次運行會下載數據集，後續運行使用緩存
- **遙測開銷**：啟用詳細追蹤會增加一些性能開銷

---

## 最佳實踐

1. **使用環境變量**：將 HF_TOKEN 設置為環境變量而非硬編碼
2. **錯誤處理**：任務運行器應妥善處理異常並返回有意義的錯誤信息
3. **資源管理**：使用異步上下文管理器確保資源正確釋放
4. **結果保存**：始終指定 `out` 參數以保存評估結果供後續分析
5. **增量測試**：開發時使用 `max_n` 參數限制任務數量進行快速測試

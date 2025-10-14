# Workflows 入門範例

## 安裝

Microsoft Agent Framework Workflows 支援隨核心 `agent-framework` 或 `agent-framework-core` 套件一起提供，因此無需額外的安裝步驟。

要安裝視覺化支援：

```bash
pip install agent-framework[viz] --pre
```

要匯出視覺化影像，您還需要[安裝 GraphViz](https://graphviz.org/download/)。

## 範例概覽

## 基礎概念 - 從這裡開始

按順序從 `_start-here` 資料夾開始。這三個範例介紹執行器、邊緣、工作流程中的代理和串流的核心概念。

| 範例 | 檔案 | 概念 |
|--------|------|----------|
| 執行器和邊緣 | [_start-here/step1_executors_and_edges.py](./_start-here/step1_executors_and_edges.py) | 具有基本執行器和邊緣的最小工作流程 |
| 工作流程中的代理 | [_start-here/step2_agents_in_a_workflow.py](./_start-here/step2_agents_in_a_workflow.py) | 介紹將代理新增為節點；在工作流程中呼叫代理 |
| 串流（基礎） | [_start-here/step3_streaming.py](./_start-here/step3_streaming.py) | 使用事件串流擴展工作流程 |

熟悉這些後，探索以下其餘範例。

---

## 範例概覽（依目錄）

### agents

| 範例 | 檔案 | 概念 |
|---|---|---|
| Azure Chat Agents（串流） | [agents/azure_chat_agents_streaming.py](./agents/azure_chat_agents_streaming.py) | 將 Azure Chat 代理新增為邊緣並處理串流事件 |
| Azure AI Chat Agents（串流） | [agents/azure_ai_agents_streaming.py](./agents/azure_ai_agents_streaming.py) | 將 Azure AI 代理新增為邊緣並處理串流事件 |
| Azure Chat Agents（函式橋接） | [agents/azure_chat_agents_function_bridge.py](./agents/azure_chat_agents_function_bridge.py) | 使用注入外部上下文的函式執行器鏈接兩個代理 |
| Azure Chat Agents（工具 + HITL） | [agents/azure_chat_agents_tool_calls_with_feedback.py](./agents/azure_chat_agents_tool_calls_with_feedback.py) | 具有透過 RequestInfoExecutor 進行人工反饋控制的工具啟用寫作者/編輯者管道 |
| 自訂代理執行器 | [agents/custom_agent_executors.py](./agents/custom_agent_executors.py) | 建立執行器以處理代理執行方法 |
| 工作流程作為代理（反思模式） | [agents/workflow_as_agent_reflection_pattern.py](./agents/workflow_as_agent_reflection_pattern.py) | 包裝工作流程使其表現得像代理（反思模式） |
| 工作流程作為代理 + HITL | [agents/workflow_as_agent_human_in_the_loop.py](./agents/workflow_as_agent_human_in_the_loop.py) | 使用人工在環功能擴展工作流程作為代理 |

### checkpoint

| 範例 | 檔案 | 概念 |
|---|---|---|
| 檢查點與恢復 | [checkpoint/checkpoint_with_resume.py](./checkpoint/checkpoint_with_resume.py) | 建立檢查點、檢查它們並恢復執行 |
| 檢查點與 HITL 恢復 | [checkpoint/checkpoint_with_human_in_the_loop.py](./checkpoint/checkpoint_with_human_in_the_loop.py) | 將檢查點與人工核准結合並恢復待處理的 HITL 請求 |
| 檢查點子工作流程 | [checkpoint/sub_workflow_checkpoint.py](./checkpoint/sub_workflow_checkpoint.py) | 儲存並恢復暫停等待人工核准的子工作流程 |

### composition

| 範例 | 檔案 | 概念 |
|---|---|---|
| 子工作流程（基礎） | [composition/sub_workflow_basics.py](./composition/sub_workflow_basics.py) | 將工作流程包裝為執行器並編排子工作流程 |
| 子工作流程：請求攔截 | [composition/sub_workflow_request_interception.py](./composition/sub_workflow_request_interception.py) | 使用 @handler 針對 RequestInfoMessage 子類別攔截和轉發子工作流程請求 |
| 子工作流程：並行請求 | [composition/sub_workflow_parallel_requests.py](./composition/sub_workflow_parallel_requests.py) | 多個專門的攔截器處理來自同一子工作流程的不同請求類型 |

### control-flow

| 範例 | 檔案 | 概念 |
|---|---|---|
| 順序執行器 | [control-flow/sequential_executors.py](./control-flow/sequential_executors.py) | 具有明確執行器設置的順序工作流程 |
| 順序（串流） | [control-flow/sequential_streaming.py](./control-flow/sequential_streaming.py) | 從簡單的順序執行中串流事件 |
| 邊緣條件 | [control-flow/edge_condition.py](./control-flow/edge_condition.py) | 基於代理分類的條件路由 |
| Switch-Case 邊緣群組 | [control-flow/switch_case_edge_group.py](./control-flow/switch_case_edge_group.py) | 使用分類器輸出的 Switch-case 分支 |
| 多選邊緣群組 | [control-flow/multi_selection_edge_group.py](./control-flow/multi_selection_edge_group.py) | 動態選擇一個或多個目標（子集扇出） |
| 簡單迴圈 | [control-flow/simple_loop.py](./control-flow/simple_loop.py) | 代理判斷 ABOVE/BELOW/MATCHED 的反饋迴圈 |

### human-in-the-loop

| 範例 | 檔案 | 概念 |
|---|---|---|
| 人工在環（猜謎遊戲） | [human-in-the-loop/guessing_game_with_human_input.py](./human-in-the-loop/guessing_game_with_human_input.py) | 與人類的互動式請求/回應提示 |
| Azure Agents 工具反饋迴圈 | [agents/azure_chat_agents_tool_calls_with_feedback.py](./agents/azure_chat_agents_tool_calls_with_feedback.py) | 串流工具呼叫並在兩次傳遞之間暫停以獲得人工指導的雙代理工作流程 |

### observability

| 範例 | 檔案 | 概念 |
|---|---|---|
| 追蹤（基礎） | [observability/tracing_basics.py](./observability/tracing_basics.py) | 使用基本追蹤進行工作流程遙測。參考此[目錄](../observability/)以了解更多關於可觀測性概念的資訊。 |

### orchestration

| 範例 | 檔案 | 概念 |
|---|---|---|
| 順序代理鏈 | [orchestration/sequential_agent_chain.py](./orchestration/sequential_agent_chain.py) | 三代理順序管道（產生、編輯、評論） |
| 並行代理合併 | [orchestration/parallel_agent_merge.py](./orchestration/parallel_agent_merge.py) | 並行執行多個代理並合併結果 |
| 多模型代理組合 | [orchestration/multi_model_agent_composition.py](./orchestration/multi_model_agent_composition.py) | 在一個工作流程中組合來自不同提供者（OpenAI、Azure、Anthropic）的代理 |

### parallelism

| 範例 | 檔案 | 概念 |
|---|---|---|
| 並行執行 | [parallelism/concurrent_execution.py](./parallelism/concurrent_execution.py) | 扇出/扇入模式與同步合併 |

### state-management

| 範例 | 檔案 | 概念 |
|---|---|---|
| 傳遞中間結果 | [state-management/passing_intermediate_results.py](./state-management/passing_intermediate_results.py) | 執行器之間傳遞訊息 |
| 傳遞中間結果（自訂） | [state-management/passing_intermediate_results_custom.py](./state-management/passing_intermediate_results_custom.py) | 使用 Pydantic 模型處理自訂狀態 |
| 傳遞狀態（單邊緣） | [state-management/passing_state_single_edge.py](./state-management/passing_state_single_edge.py) | 在單一邊緣上的狀態更新 |
| 單一代理工作流程 | [state-management/single_agent_workflow.py](./state-management/single_agent_workflow.py) | 迴圈迭代的最小單代理工作流程 |
| 狀態減速器 | [state-management/state_reducer.py](./state-management/state_reducer.py) | 使用減速器模式處理來自多個來源的傳入狀態 |

### visualization

| 範例 | 檔案 | 概念 |
|---|---|---|
| 並行與視覺化 | [visualization/concurrent_with_visualization.py](./visualization/concurrent_with_visualization.py) | 扇出/扇入工作流程與圖表匯出 |

### resources

此資料夾包含工作流程範例中引用的共享助手模組。

## 環境變數

大多數範例需要設置環境變數以用於 Azure 或 OpenAI 驗證。建立一個 `.env` 檔案或設置系統環境變數：

```bash
# Azure OpenAI
AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com/
AZURE_OPENAI_CHAT_DEPLOYMENT_NAME=gpt-4o
AZURE_OPENAI_API_KEY=your-key  # 或使用 Azure CLI（執行 az login）

# Azure AI Foundry
AZURE_AI_PROJECT_ENDPOINT=https://your-project.cognitiveservices.azure.com/

# OpenAI
OPENAI_API_KEY=your-openai-key

# Anthropic
ANTHROPIC_API_KEY=your-anthropic-key
```

## 執行範例

1. 導航到工作流程資料夾：
   ```bash
   cd python/samples/getting_started/workflows
   ```

2. 設置所需的環境變數（見上文）

3. 執行任何範例：
   ```bash
   python _start-here/step1_executors_and_edges.py
   python agents/azure_chat_agents_streaming.py
   python checkpoint/checkpoint_with_resume.py
   ```

## 核心概念

### 執行器

執行器是工作流程中的節點。它們：
- 處理傳入訊息
- 執行工作（呼叫代理、轉換資料）
- 發出傳出訊息

### 邊緣

邊緣連接執行器並定義訊息流。它們可以是：
- **無條件**：始終傳遞訊息
- **條件**：基於邏輯路由
- **分組**：Switch-case 或多選模式

### 串流

工作流程可以在執行期間發出事件：
- AgentResponseChunk：代理回應令牌
- FunctionCallChunk：工具呼叫進度
- WorkflowOutputEvent：最終輸出

### 檢查點

將工作流程狀態儲存到儲存：
- 恢復中斷的執行
- 實施人工在環核准
- 除錯和時間旅行

### 人工在環

使用 RequestInfoExecutor 暫停以獲取人工輸入：
- 核准/拒絕決策
- 提供額外的上下文
- 覆寫代理決策

## 最佳實踐

### 1. 從基礎開始
從 `_start-here` 資料夾中的三個基礎範例開始。

### 2. 使用視覺化進行除錯
安裝 `agent-framework[viz]` 並使用 `workflow.visualize()` 來了解圖結構。

### 3. 適當使用檢查點
對於長時間執行的工作流程或人工在環場景，使用檢查點。

### 4. 處理串流事件
訂閱相關事件以提供即時反饋和進度更新。

### 5. 構建模組化工作流程
使用子工作流程建立可重複使用的元件。

### 6. 測試邊緣條件
驗證條件路由和錯誤處理路徑。

## 參考資料

- [Workflows 核心概念](../../../docs/workflows.md)
- [檢查點指南](../../../docs/checkpoints.md)
- [Agent Framework 文件](../../../README.md)

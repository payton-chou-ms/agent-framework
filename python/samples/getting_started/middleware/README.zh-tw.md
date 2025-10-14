# 中介軟體範例

此資料夾包含展示 Agent Framework 的各種中介軟體模式的範例。中介軟體允許您在不同執行階段攔截和修改行為，包括代理執行、函式呼叫和聊天互動。

## 範例

| 檔案 | 描述 |
|------|-------------|
| [`function_based_middleware.py`](function_based_middleware.py) | 展示如何使用簡單的非同步函式而非類別來實作中介軟體。展示安全驗證、日誌記錄和效能監控中介軟體。基於函式的中介軟體適合簡單、無狀態的操作，並提供輕量級方法。 |
| [`class_based_middleware.py`](class_based_middleware.py) | 展示如何透過繼承 `AgentMiddleware` 和 `FunctionMiddleware` 基礎類別來使用基於類別的方法實作中介軟體。包括敏感資訊的安全檢查和具有計時的詳細函式執行日誌記錄。 |
| [`decorator_middleware.py`](decorator_middleware.py) | 展示如何使用 `@agent_middleware` 和 `@function_middleware` 裝飾器來明確標記中介軟體函式，而無需類型註解。展示不同的中介軟體偵測場景和明確的裝飾器使用。 |
| [`middleware_termination.py`](middleware_termination.py) | 展示中介軟體如何使用 `context.terminate` 旗標終止執行。包括預終止（防止代理處理）和後終止（允許處理但停止進一步執行）的範例。適用於安全檢查、速率限制或早期退出條件。 |
| [`exception_handling_with_middleware.py`](exception_handling_with_middleware.py) | 展示如何使用中介軟體進行函式呼叫中的集中式例外處理。展示如何捕獲函式的例外、提供優雅的錯誤回應，以及在發生錯誤時覆寫函式結果以提供使用者友好的訊息。 |
| [`override_result_with_middleware.py`](override_result_with_middleware.py) | 展示如何使用中介軟體在執行後攔截和修改函式結果，支援常規和串流代理回應。展示結果過濾、格式化、增強和自訂串流回應生成。 |
| [`shared_state_middleware.py`](shared_state_middleware.py) | 展示如何在類別內實作基於函式的中介軟體，以在多個中介軟體函式之間共享狀態。展示中介軟體如何透過共享狀態協同工作，包括呼叫計數和結果增強。 |
| [`agent_and_run_level_middleware.py`](agent_and_run_level_middleware.py) | 解釋代理級中介軟體（應用於代理的所有執行）和執行級中介軟體（僅應用於特定執行）之間的區別。展示安全驗證、效能監控和上下文特定的中介軟體模式。 |
| [`chat_middleware.py`](chat_middleware.py) | 展示如何使用聊天中介軟體觀察和覆寫發送到 AI 模型的輸入。展示如何攔截聊天請求、記錄和修改輸入訊息，以及在回應到達底層 AI 服務之前覆寫整個回應。 |

## 核心概念

### 中介軟體類型

- **代理中介軟體**：攔截代理執行，允許您修改請求和回應
- **函式中介軟體**：攔截代理內的函式呼叫，實現日誌記錄、驗證和結果修改
- **聊天中介軟體**：攔截發送到 AI 模型的聊天請求，允許輸入/輸出轉換

### 實作方法

- **基於函式**：用於輕量級、無狀態操作的簡單非同步函式
- **基於類別**：繼承基礎中介軟體類別以進行複雜、有狀態的操作
- **基於裝飾器**：使用裝飾器進行明確的中介軟體標記

### 常見使用案例

- **安全性**：驗證請求、封鎖敏感資訊、實施訪問控制
- **日誌記錄**：追蹤執行時間、記錄參數和結果、監控效能
- **錯誤處理**：捕獲例外、提供優雅的後備方案、實施重試邏輯
- **結果轉換**：過濾、格式化或增強函式輸出
- **狀態管理**：在中介軟體函式之間共享資料、維護執行上下文

### 執行控制

- **終止**：使用 `context.terminate` 提前停止執行
- **結果覆寫**：修改或替換函式/代理結果
- **串流支援**：處理常規和串流回應

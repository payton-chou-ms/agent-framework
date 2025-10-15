# Agent Framework Python 可觀測性

此範例資料夾展示如何配置 Python 應用程式，以基於 OpenTelemetry 標準將 Agent Framework 可觀測性資料發送到您選擇的應用程式效能管理 (APM) 供應商。

在此範例中，我們提供將遙測發送到 [Application Insights](https://learn.microsoft.com/zh-tw/azure/azure-monitor/app/app-insights-overview)、[Aspire Dashboard](https://learn.microsoft.com/zh-tw/dotnet/aspire/fundamentals/dashboard/overview?tabs=bash) 和控制台的選項。

> **快速開始**：對於無需 Azure 設置的本地開發，您可以使用透過 Docker 在本地執行的 [Aspire Dashboard](https://learn.microsoft.com/zh-tw/dotnet/aspire/fundamentals/dashboard/standalone)，它為 OpenTelemetry 資料提供出色的遙測檢視體驗。或者您可以使用 [AI Toolkit for VS Code](https://marketplace.visualstudio.com/items?itemName=ms-windows-ai-studio.windows-ai-studio) 的內建追蹤模組。

> 請注意，也可以使用其他應用程式效能管理 (APM) 供應商。一個範例是 [Prometheus](https://prometheus.io/docs/introduction/overview/)。請參考此[頁面](https://opentelemetry.io/docs/languages/python/exporters/)以了解更多關於匯出器的資訊。

更多資訊，請參考以下資源：

1. [Azure Monitor OpenTelemetry Exporter](https://github.com/Azure/azure-sdk-for-python/tree/main/sdk/monitor/azure-monitor-opentelemetry-exporter)
2. [Aspire Dashboard for Python Apps](https://learn.microsoft.com/zh-tw/dotnet/aspire/fundamentals/dashboard/standalone-for-python?tabs=flask%2Cwindows)
3. [AI Toolkit for VS Code](https://marketplace.visualstudio.com/items?itemName=ms-windows-ai-studio.windows-ai-studio)
4. [Python Logging](https://docs.python.org/3/library/logging.html)
5. [Observability in Python](https://www.cncf.io/blog/2022/04/22/opentelemetry-and-python-a-complete-instrumentation-guide/)

## 預期效果

Agent Framework Python SDK 旨在在代理/模型調用和工具執行流程中有效地生成全面的日誌、追蹤和指標。這使您能夠有效監控 AI 應用程式的效能並準確追蹤令牌消耗。它基於 OpenTelemetry 定義的 GenAI 語義約定，工作流程會發出自己的 span 以提供端到端的可見性。

## 配置

### 必需資源

1. OpenAI 或 [Azure OpenAI](https://learn.microsoft.com/zh-tw/azure/ai-services/openai/how-to/create-resource?pivots=web-portal)
2. 一個 [Azure AI 專案](https://ai.azure.com/doc/azure/ai-foundry/what-is-azure-ai-foundry)

### 可選資源

如果您想將遙測資料發送到以下資源，則需要它們：

1. [Application Insights](https://learn.microsoft.com/zh-tw/azure/azure-monitor/app/create-workspace-resource)
2. [Aspire Dashboard](https://learn.microsoft.com/zh-tw/dotnet/aspire/fundamentals/dashboard/standalone-for-python?tabs=flask%2Cwindows#start-the-aspire-dashboard)

### 依賴項

啟用遙測不需要額外的依賴項。必要的套件包含在 `agent-framework` 套件中。除非您想使用不同的 APM 供應商，在這種情況下，您需要安裝適當的 OpenTelemetry 匯出器套件。

### 環境變數

以下環境變數用於開啟/關閉 Agent Framework 的可觀測性：

- ENABLE_OTEL=true
- ENABLE_SENSITIVE_DATA=true

當上述環境變數之一設置為 true 時，框架將發出可觀測性資料。

> **注意**：敏感資訊包括提示、回應等，應僅在開發或測試環境中啟用。不建議在生產環境中啟用此功能，因為它可能暴露敏感資料。

### 配置匯出器和提供者

開啟可觀測性只是第一步，您還需要配置將可觀測性資料發送到何處（即控制台、Application Insights）。預設情況下，未配置任何匯出器或提供者。

#### 手動設置匯出器和提供者

請參考範例 [advanced_manual_setup_console_output.py](./advanced_manual_setup_console_output.py)，了解如何手動設置追蹤、日誌和指標的匯出器和提供者，這些資料將發送到控制台的全面範例。

#### 使用 `setup_observability()` 設置匯出器和提供者

為了讓開發人員更容易上手，`agent_framework.observability` 模組提供了一個 `setup_observability()` 函式，該函式將根據環境變數設置追蹤、日誌和指標的匯出器和提供者。您可以在應用程式開始時呼叫此函式以啟用遙測。

```python
from agent_framework.observability import setup_observability

setup_observability()
```

#### `setup_observability()` 的環境變數

`setup_observability()` 函式將尋找以下環境變數以確定如何設置匯出器和提供者：

- OTLP_ENDPOINT="..."
- APPLICATIONINSIGHTS_CONNECTION_STRING="..."

透過提供上述環境變數，`setup_observability()` 函式將自動為您配置適當的匯出器和提供者。如果未提供環境變數，該函式將不設置任何匯出器或提供者。

如果您想自訂匯出器或在透過環境變數配置的匯出器之外添加額外的匯出器，您也可以直接將匯出器清單傳遞給 `setup_observability()` 函式。

```python
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from agent_framework.observability import setup_observability

exporter = OTLPSpanExporter(endpoint="another-otlp-endpoint")
setup_observability(exporters=[exporter])
```

> 使用此方法隱式啟用遙測，因此您不需要設置 `ENABLE_OTEL` 環境變數。您仍然可以設置 `ENABLE_SENSITIVE_DATA` 來控制遙測中是否包含敏感資料，或使用 `enable_sensitive_data` 參數設置為 `True` 來呼叫 `setup_observability()` 函式。

#### 日誌記錄

您可以控制日誌記錄的層級，從而控制匯出哪些日誌，您可以這樣做：

```python
import logging

logger = logging.getLogger()
logger.setLevel(logging.NOTSET)
```

這會獲取根日誌記錄器並設置其層級，其他日誌記錄器會自動從該日誌記錄器繼承，您將在遙測中獲得詳細的日誌。

## 範例

此資料夾包含展示如何在各種場景中使用遙測的不同範例。

| 範例 | 描述 |
|--------|-------------|
| [setup_observability_with_parameters.py](./setup_observability_with_parameters.py) | 一個簡單的範例，展示如何透過將參數傳遞給 `setup_observability()` 函式來設置遙測。 |
| [setup_observability_with_env_var.py](./setup_observability_with_env_var.py) | 一個簡單的範例，展示如何使用環境變數透過 `setup_observability()` 函式設置遙測。 |
| [agent_observability.py](./agent_observability.py) | 一個簡單的範例，展示如何為代理應用程式設置遙測。 |
| [azure_ai_agent_observability.py](./azure_ai_agent_observability.py) | 一個簡單的範例，展示如何使用 Azure AI 專案為代理應用程式設置遙測。 |
| [advanced_manual_setup_console_output.py](./advanced_manual_setup_console_output.py) | 一個進階範例，展示如何手動設置追蹤、日誌和指標的匯出器和提供者。 |

### 執行範例

1. 開啟終端機並導航到此資料夾：`python/samples/getting_started/observability/`。這對於正確讀取 `.env` 檔案是必要的。
2. 如果此資料夾中尚未存在 `.env` 檔案，請建立一個。請參考[範例檔案](./.env.example)。
    > 請注意，`APPLICATIONINSIGHTS_CONNECTION_STRING` 和 `OTLP_ENDPOINT` 是可選的。如果您不配置它們，一切都將輸出到控制台。
3. 啟動您的 python 虛擬環境，然後執行 `python setup_observability_with_env_vars.py` 或其他範例。

> 這也會列印 Operation/Trace ID，稍後可以在 Application Insights 或 Aspire Dashboard 中用於過濾日誌和追蹤。

## Application Insights 中的遙測

如果您配置了 Application Insights，您可以在 Azure 入口網站中查看遙測資料：

1. 導航到您的 Application Insights 資源
2. 使用 Transaction Search 或 Application Map 查看追蹤
3. 使用 Logs 查看詳細的日誌條目
4. 使用 Metrics 查看效能指標

## Aspire Dashboard 中的遙測

如果您配置了 Aspire Dashboard：

1. 啟動 Aspire Dashboard（透過 Docker 或直接）
2. 在瀏覽器中開啟儀表板（通常是 `http://localhost:18888`）
3. 查看結構化追蹤、日誌和指標
4. 使用內建過濾器和搜尋功能

## 最佳實踐

### 1. 在開發中啟用敏感資料，在生產中停用

```python
import os
# 僅在非生產環境中啟用
os.environ["ENABLE_SENSITIVE_DATA"] = "true" if os.getenv("ENV") != "production" else "false"
```

### 2. 使用適當的日誌層級

```python
import logging
# 開發環境：詳細日誌
logger.setLevel(logging.DEBUG)
# 生產環境：僅警告和錯誤
logger.setLevel(logging.WARNING)
```

### 3. 監控令牌消耗

可觀測性資料包括令牌使用情況指標，這對於成本追蹤和最佳化至關重要。

### 4. 使用追蹤 ID 進行除錯

每次執行都會發出一個追蹤 ID，用於關聯日誌、追蹤和指標。

### 5. 定期檢查指標

監控：
- 回應時間
- 錯誤率
- 令牌消耗
- 工具執行時間

## 故障排除

### 未發出遙測資料

- 驗證 `ENABLE_OTEL=true` 已設置
- 檢查匯出器配置
- 驗證網路連接到 OTLP 端點或 Application Insights

### 遺漏敏感資料

- 設置 `ENABLE_SENSITIVE_DATA=true`（僅在非生產環境中）
- 或在呼叫 `setup_observability(enable_sensitive_data=True)` 時傳遞參數

### Application Insights 中無資料

- 驗證連接字串正確
- 檢查 Azure 訂閱權限
- 允許最多 5 分鐘的延遲

### Aspire Dashboard 連接失敗

- 確保 Docker 容器正在執行
- 驗證 OTLP 端點正確（通常是 `http://localhost:4317`）
- 檢查防火牆設置

## 參考資料

- [OpenTelemetry Python 文件](https://opentelemetry.io/docs/languages/python/)
- [Azure Monitor OpenTelemetry](https://learn.microsoft.com/zh-tw/azure/azure-monitor/app/opentelemetry-enable)
- [Agent Framework 核心文件](../../../README.md)

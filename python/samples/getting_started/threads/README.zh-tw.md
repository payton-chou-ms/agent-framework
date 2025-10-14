# 執行緒管理範例

此資料夾包含展示使用 Agent Framework 管理對話執行緒和聊天訊息儲存的不同方法的範例。

## 範例

| 檔案 | 描述 |
|------|-------------|
| [`custom_chat_message_store_thread.py`](custom_chat_message_store_thread.py) | 展示如何實作自訂的 `ChatMessageStore` 以持久化對話歷史。展示如何建立具有序列化/反序列化功能的自訂儲存，並將其與代理整合，以在多個會話間進行執行緒管理。 |
| [`suspend_resume_thread.py`](suspend_resume_thread.py) | 展示如何暫停和恢復對話執行緒，允許您儲存對話狀態並稍後繼續。這對於長時間執行的對話或當您需要在應用程式重啟之間持久化對話狀態時很有用。 |
| [`redis_chat_message_store_thread.py`](redis_chat_message_store_thread.py) | 使用 Redis 支援的 `RedisChatMessageStore` 進行持久化對話儲存的全面範例。涵蓋基本用法、使用者會話管理、跨應用程式重啟的對話持久化、執行緒序列化和自動訊息修剪。需要 Redis 伺服器，並展示適用於可擴展聊天應用程式的生產就緒模式。 |

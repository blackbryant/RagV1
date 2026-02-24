# Role
你是一位精通 .NET 8/9、雲端原生架構與 AI 應用的資深 C# 軟體架構師。你非常熟悉 .NET Aspire 的微服務編排，以及使用 RabbitMQ 進行非同步訊息處理。

## Context
我正在開發一個 RAG (Retrieval-Augmented Generation) 的資料上傳平台。為了避免使用者上傳文件時卡住，我決定採用非同步架構：
1. **Web API 服務**：負責接收使用者上傳的文件，將檔案暫存，並發送一則「文件已上傳」的訊息到 RabbitMQ。
2. **Worker 服務**：監聽 RabbitMQ 的佇列，收到訊息後，進行後續的背景處理（文件讀取、Chunking、Embedding 呼叫等）。
3. **AppHost**：使用 .NET Aspire 來負責編排 API、Worker 以及 RabbitMQ 容器。

## Task
請幫我實作這個 .NET Aspire 專案的基礎骨架與 RabbitMQ 整合程式碼，具體包含：
1. `AppHost` 的設定檔：如何啟動 RabbitMQ 容器並綁定給 API 和 Worker。
2. `Web API` 端：如何注入 RabbitMQ Client，並撰寫一個簡單的發送訊息方法 (Publisher)。
3. `Worker` 端：如何使用 `BackgroundService` 監聽 RabbitMQ 佇列，並實作接收訊息的邏輯 (Consumer)。

## Constraints
- 必須使用 .NET Aspire 官方的 `Aspire.RabbitMQ.Client` 相關套件。
- 實作請符合 Dependency Injection (DI) 最佳實踐。
- 暫時不需要實作真實的 RAG Chunking 和 Embedding 邏輯，只要用 `logger.LogInformation` 模擬 Worker 收到任務即可。
- 請加上正體中文註解。

## Output Format
1. 首先，簡短說明需要的 NuGet 套件安裝指令。
2. 接著，依序給出 `Program.cs` (AppHost)、API 發送端、Worker 接收端的三段完整 C# 程式碼區塊。
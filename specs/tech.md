使用最新的net 9.0+aspire開發，使用者介面 (UI)：Open WebUI (透過 Python Pipeline 攔截與組裝 Prompt)。

API 閘道器與流量管控：LiteLLM (負責 LLM 負載均衡與 Fallback 備援)。

身分驗證與授權 (IdP)：.NET OpenIddict (OIDC 協定，發放包含 Multi-Role 的 JWT)。

後端與微服務編排：.NET Aspire (C# 12 / .NET 8+)。

非同步訊息匯流排：RabbitMQ (搭配 MassTransit 框架)。

資料庫與儲存：

關聯與向量儲存：PostgreSQL + PGVector (搭配 Entity Framework Core)。

物件儲存 (Data Lake)：MinIO (存放原始 PDF 與音檔)。

AI 模型與 ETL 工具：

文件解析：Unstructured.io API (輸出 Markdown 格式)。

文本向量化：BAAI/bge-m3 模型。

這個平台主要是使用C#開發，提供各種文件的匯入進向量資料庫，並可以使用繁體、簡體、英文、泰文來查詢，專案範圍只要檔案可以匯入並利用Rabbitmq來做處理檔案任務，Rabbitmq處理許多連線斷線、重試機制（Retry）與死信佇列（DLQ）
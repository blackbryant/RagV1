# 快速啟動指南：RagV1 企業級 RAG 平台

**分支**: `001-enterprise-rag-platform` | **日期**: 2026-02-25  
**目的**: 協助開發者快速建立本地開發環境並開始貢獻

---

## 前置需求

### 必要工具

| 工具 | 最低版本 | 用途 | 安裝連結 |
|------|----------|------|---------|
| .NET SDK | 9.0.100 | 編譯與執行 | [下載](https://dotnet.microsoft.com/download) |
| Docker Desktop | 4.25+ | 容器化服務 | [下載](https://www.docker.com/products/docker-desktop) |
| Visual Studio 2022 | 17.9+ | IDE（建議） | [下載](https://visualstudio.microsoft.com/) |
| Git | 2.40+ | 版本控制 | [下載](https://git-scm.com/) |

**可選工具**:
- **Rider** 或 **VS Code**（替代 Visual Studio）
- **Postman** 或 **Insomnia**（API 測試）
- **pgAdmin**（PostgreSQL 管理）
- **Conductor**（RabbitMQ 監控）

### 硬體建議

- **CPU**: 4 核心以上
- **記憶體**: 16GB 以上（Aspire + Docker 容器需求）
- **硬碟**: 20GB 可用空間

---

## 第一步：取得程式碼

```bash
# 克隆儲存庫
git clone https://github.com/your-org/RagV1.git
cd RagV1

# 切換至功能分支（本期開發）
git checkout 001-enterprise-rag-platform

# 還原套件
dotnet restore
```

---

## 第二步：啟動基礎設施（Docker）

本專案使用 .NET Aspire 管理基礎設施，但首次啟動需要準備容器映像：

```bash
# 啟動 Aspire AppHost（會自動拉取並啟動所有相依容器）
cd src/RagV1.AppHost
dotnet run
```

**Aspire 會自動啟動**:
- PostgreSQL 16 + PGVector（端口 5432）
- RabbitMQ 3.12 + Management Plugin（端口 5672, 15672）
- MinIO（端口 9000, 9001）
- Embedding Service（Python FastAPI，端口 8000）

**Aspire Dashboard**:
- 網址：`http://localhost:15000`
- 功能：即時監控服務狀態、日誌、分散式追蹤

---

## 第三步：初始化資料庫

```bash
# 進入 DocumentIngestion 專案目錄
cd src/RagV1.DocumentIngestion/Api

# 建立 Migration（首次執行）
dotnet ef migrations add InitialCreate --context DocumentDbContext

# 套用 Migration 至資料庫
dotnet ef database update --context DocumentDbContext
```

**驗證資料庫**:
```bash
# 連線至 PostgreSQL
docker exec -it aspire-postgres psql -U postgres -d ragv1db

# 檢查資料表
\dt

# 驗證 PGVector 擴充
SELECT * FROM pg_extension WHERE extname = 'vector';

# 離開
\q
```

---

## 第四步：啟動微服務

### 方式一：使用 Aspire AppHost（推薦）

```bash
# 在專案根目錄執行
cd src/RagV1.AppHost
dotnet run
```

所有微服務會自動啟動：
- Identity Service（端口 5001）
- DocumentIngestion API（端口 5002）
- DocumentIngestion Worker（背景任務）

### 方式二：手動啟動個別服務（除錯用）

**終端機 1 - Identity Service**:
```bash
cd src/RagV1.Identity
dotnet run
```

**終端機 2 - DocumentIngestion API**:
```bash
cd src/RagV1.DocumentIngestion/Api
dotnet run
```

**終端機 3 - DocumentIngestion Worker**:
```bash
cd src/RagV1.DocumentIngestion/Worker
dotnet run
```

---

## 第五步：驗證環境

### 檢查服務健康狀態

```bash
# Identity Service
curl http://localhost:5001/health

# DocumentIngestion API
curl http://localhost:5002/health

# RabbitMQ Management UI
# 瀏覽器開啟 http://localhost:15672
# 預設帳密：guest / guest

# MinIO Console
# 瀏覽器開啟 http://localhost:9001
# 預設帳密：minioadmin / minioadmin
```

### 測試文件上傳流程

**1. 取得 JWT Token（開發環境簡化流程）**:
```bash
curl -X POST http://localhost:5001/connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=password" \
  -d "username=testuser@ragv1.local" \
  -d "password=TestPassword123!" \
  -d "client_id=test-client" \
  -d "client_secret=test-secret" \
  -d "scope=documents.write"
```

**2. 上傳測試文件**:
```bash
TOKEN="<從上一步取得的 access_token>"

curl -X POST http://localhost:5002/v1/documents \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@test-data/sample.pdf" \
  -F "roles=[\"RD_General\"]"
```

**3. 觀察處理進度**:
- 開啟 Aspire Dashboard: `http://localhost:15000`
- 切換至 "Traces" 頁籤
- 查看 DocumentUploaded → ParseDocument → EmbedChunks 完整鏈路

**4. 查詢文件狀態**:
```bash
DOCUMENT_ID="<從上傳回應取得的 documentId>"

curl http://localhost:5002/v1/documents/$DOCUMENT_ID \
  -H "Authorization: Bearer $TOKEN"
```

---

## 開發工作流程

### 1. 建立功能分支

```bash
# 從 001-enterprise-rag-platform 分支出功能分支
git checkout -b 001-enterprise-rag-platform-api-endpoints
```

### 2. 撰寫測試（TDD）

```bash
# 建立測試檔案
cd tests/RagV1.DocumentIngestion.Tests/Unit/Services
touch UnstructuredIoParserTests.cs
```

**測試範例**:
```csharp
public class UnstructuredIoParserTests
{
    [Fact]
    public async Task ParseAsync_ValidPdf_ReturnsMarkdown()
    {
        // Arrange
        var parser = new UnstructuredIoParser(_httpClient, _logger);
        var pdfStream = File.OpenRead("test-data/sample.pdf");

        // Act
        var result = await parser.ParseAsync(pdfStream, CancellationToken.None);

        // Assert
        Assert.NotNull(result.Markdown);
        Assert.NotEmpty(result.Chunks);
        Assert.All(result.Chunks, chunk => Assert.InRange(chunk.TokenCount, 450, 600));
    }
}
```

### 3. 執行測試

```bash
# 執行所有測試
dotnet test

# 執行特定測試專案
dotnet test tests/RagV1.DocumentIngestion.Tests

# 執行特定測試類別
dotnet test --filter "FullyQualifiedName~UnstructuredIoParserTests"

# 產生涵蓋率報告
dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=opencover
```

### 4. 實作功能

```csharp
// src/RagV1.DocumentIngestion/Services/UnstructuredIoParser.cs
public class UnstructuredIoParser : IDocumentParser
{
    public async Task<ParsedDocument> ParseAsync(Stream pdfStream, CancellationToken ct)
    {
        // 實作邏輯
    }
}
```

### 5. 執行 Linter 與 Formatter

```bash
# 格式化程式碼（使用 .editorconfig 規則）
dotnet format

# 檢查風格違規
dotnet format --verify-no-changes
```

### 6. 提交與推送

```bash
git add .
git commit -m "feat(document-ingestion): implement UnstructuredIoParser"
git push origin 001-enterprise-rag-platform-api-endpoints
```

### 7. 建立 Pull Request

- 開啟 GitHub / Azure DevOps
- 建立 PR：`001-enterprise-rag-platform-api-endpoints` → `001-enterprise-rag-platform`
- 填寫 PR 範本（描述、測試、架構原則檢查）
- 等待 CI 通過（Linter、測試、涵蓋率）

---

## 常見問題排解

### Q: Aspire AppHost 啟動失敗「無法連線至 Docker」

**解決方式**:
```bash
# 確認 Docker Desktop 已執行
docker ps

# 重啟 Docker Desktop
# Windows: 右鍵系統匣圖示 → Restart
# macOS: Docker 選單 → Restart
```

### Q: PostgreSQL Migration 失敗「database "ragv1db" does not exist」

**解決方式**:
```bash
# 手動建立資料庫
docker exec -it aspire-postgres psql -U postgres -c "CREATE DATABASE ragv1db;"

# 重新執行 Migration
dotnet ef database update --context DocumentDbContext
```

### Q: RabbitMQ Consumer 無法消費訊息

**診斷步驟**:
1. 開啟 RabbitMQ Management UI: `http://localhost:15672`
2. 檢查 Queues 頁籤，確認隊列已建立
3. 檢查 Consumer 是否已註冊（Consumers 欄位 > 0）
4. 查看 Aspire Dashboard 日誌，搜尋 "MassTransit"

### Q: 文件解析一直停在 "Parsing" 狀態

**診斷步驟**:
1. 檢查 Unstructured.io API 金鑰是否配置正確（`appsettings.json`）
2. 查看 Worker 日誌：Aspire Dashboard → Logs → DocumentIngestion.Worker
3. 檢查 DLQ：RabbitMQ Management UI → Queues → `ragv1-document-ingestion-dlq`
4. 手動重試：`POST /v1/documents/{id}/retry`

### Q: 測試涵蓋率低於 80% 導致 CI 失敗

**解決方式**:
```bash
# 查看涵蓋率報告
dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=opencover

# 使用 ReportGenerator 產生 HTML 報告
reportgenerator -reports:coverage.opencover.xml -targetdir:coverage-report

# 開啟報告查看未涵蓋的程式碼
open coverage-report/index.html
```

---

## 有用的命令速查

```bash
# 清除所有容器與資料（重新開始）
docker compose down -v

# 查看 Aspire 日誌
dotnet run --project src/RagV1.AppHost --no-launch-profile

# 重新產生 OpenAPI 規格
dotnet swagger tofile --output swagger.json src/RagV1.DocumentIngestion/Api/bin/Debug/net9.0/RagV1.DocumentIngestion.Api.dll v1

# 監控 RabbitMQ 訊息流量
rabbitmqadmin list queues name messages consumers -f table

# 備份開發資料庫
docker exec aspire-postgres pg_dump -U postgres ragv1db > backup.sql
```

---

## 後續步驟

1. **閱讀架構文件**: `docs/architecture.md`（待建立）
2. **查看實體關聯圖**: `specs/001-enterprise-rag-platform/data-model.md`
3. **理解訊息流程**: `specs/001-enterprise-rag-platform/contracts.md`
4. **挑選任務**: 執行 `/speckit.tasks` 命令產生 `tasks.md`
5. **加入開發團隊頻道**: Slack / Teams（內部）

---

## 需要協助？

- **技術問題**: 在 GitHub Issues 建立問題並標註 `help-wanted`
- **架構疑問**: 聯繫架構師（內部聯絡方式）
- **CI/CD 問題**: 查看 `.github/workflows/` 或聯繫 DevOps 團隊

**祝開發愉快！🚀**

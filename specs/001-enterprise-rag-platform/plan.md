# 實作計畫：企業級多語系 RAG 與會議智慧分析平台

**分支**: `001-enterprise-rag-platform` | **日期**: 2026-02-25 | **規格**: [spec.md](./spec.md)  
**輸入**: 功能規格書來自 `/specs/001-enterprise-rag-platform/spec.md`

> ⚠️ **架構原則 V**：本文件全篇以繁體中文（zh-TW）撰寫。檔案路徑、識別碼、程式碼區塊及 Schema 鍵值可保留英文。

**注意**: 本範本由 `/speckit.plan` 命令填寫。執行工作流程請參見 `.specify/templates/plan-template.md`。

## 摘要

建立一個企業級多語系 RAG（Retrieval-Augmented Generation）與會議智慧分析平台，支援千人規模企業的知識檢索與會議紀錄自動化。系統透過 .NET Aspire 微服務編排架構，整合 OpenIddict OIDC 身分驗證、PostgreSQL + PGVector 向量儲存、RabbitMQ 非同步訊息匯流排及 LiteLLM API 閘道器，提供高度資安隔離、多語系（繁體中文、簡體中文、英文、泰文）查詢與複雜文件解析能力。

本期範圍聚焦於**文件匯入管線**與 **RabbitMQ 任務編排**，包含連線斷線恢復、重試機制與死信佇列（DLQ）處理。

## 技術脈絡

**語言/版本**: C# 12 / .NET 9.0  
**主要相依套件**: 
- ASP.NET Core 9.0（Web API）
- .NET Aspire（微服務編排）
- MassTransit 8.x（RabbitMQ 抽象層）
- Entity Framework Core 9.0（ORM）
- Npgsql.EntityFrameworkCore.PostgreSQL（PostgreSQL 驅動）
- Pgvector.EntityFrameworkCore（向量擴充）
- OpenIddict 5.x（OIDC IdP）
- Minio.AspNetCore（MinIO SDK）

**儲存方案**: 
- PostgreSQL 16+ 搭配 PGVector 擴充（關聯資料 + 向量索引）
- MinIO（物件儲存 / Data Lake，存放原始 PDF 與音檔）

**測試框架**: xUnit + Testcontainers（整合測試）+ Moq（單元測試）

**目標平台**: Linux 容器（Docker / Kubernetes），內網部署

**專案類型**: 分散式微服務後端（Web API + 背景任務處理器）

**效能目標**:
- 知識檢索 p50 ≤ 800ms, p95 ≤ 2,000ms
- 文件上傳確認回應 p95 ≤ 1,000ms
- 支援 500 並發使用者問答
- 備援切換 ≤ 500ms

**約束條件**:
- 資安隔離：所有查詢必須依角色過濾，零容忍越權洩漏
- 非同步處理：文件解析與會議轉錄必須背景執行，不阻塞前端
- 高可用性：RabbitMQ 必須處理連線斷線、自動重連與訊息重試（最多 3 次）
- 失敗隔離：失敗任務進入 DLQ，不影響其他任務

**規模/範疇**:
- 使用者規模：約 1,000 名企業員工
- 文件量：初期數千份 PDF，成長至數萬份
- 並發查詢：尖峰時段 500 並發
- 會議錄音：單檔最長 90 分鐘

## 架構原則檢查

*閘道：必須在 Phase 0 研究前通過。Phase 1 設計完成後重新檢查。*

| # | 原則 | 閘道問題 | 狀態 |
|---|------|---------|------|
| I | 程式碼品質 | 是否已配置 Linter/Formatter 並在 CI 強制執行？所有公開介面是否已文件化？ | ☑ |
| II | 測試優先標準 | 是否在實作前規劃測試？單元測試、整合測試與契約測試位置是否已識別？是否追蹤涵蓋率門檻（≥ 80%）？ | ☑ |
| III | 使用者體驗一致性 | 所有 API 回應是否遵循標準 `{ data, error, meta }` 結構？錯誤訊息是否可操作且面向使用者？領域術語是否與規格一致？ | ☑ |
| IV | 效能需求 | 是否為每個涉及檢索或生成的操作定義延遲目標？測試計畫是否包含效能基準？ | ☑ |
| V | 文件語言標準 | 本計畫文件是否以繁體中文（zh-TW）撰寫？所有任務描述與敘述是否為繁體中文？ | ☑ |

> 任何無法勾選的 ☐ 必須在下方**複雜度追蹤**中記錄並說明理由。

## 專案結構

### 文件（本功能）

```text
specs/001-enterprise-rag-platform/
├── spec.md              # 功能規格書（已存在）
├── plan.md              # 本文件（/speckit.plan 命令輸出）
├── research.md          # Phase 0 輸出（/speckit.plan 命令）
├── data-model.md        # Phase 1 輸出（/speckit.plan 命令）
├── quickstart.md        # Phase 1 輸出（/speckit.plan 命令）
├── contracts/           # Phase 1 輸出（/speckit.plan 命令）
└── tasks.md             # Phase 2 輸出（/speckit.tasks 命令 - 非 /speckit.plan 建立）
```

### 原始碼（儲存庫根目錄）

```text
src/
├── RagV1.AppHost/                     # .NET Aspire 編排主機
│   ├── Program.cs
│   └── appsettings.json
│
├── RagV1.ServiceDefaults/             # Aspire 共用服務預設值
│   └── Extensions.cs
│
├── RagV1.Identity/                    # OpenIddict IdP 服務
│   ├── Controllers/
│   │   ├── AuthorizationController.cs
│   │   └── TokenController.cs
│   ├── Models/
│   │   ├── User.cs
│   │   └── Role.cs
│   ├── Data/
│   │   └── IdentityDbContext.cs
│   └── Program.cs
│
├── RagV1.DocumentIngestion/          # 文件匯入服務（本期重點）
│   ├── Api/
│   │   ├── Controllers/
│   │   │   └── DocumentController.cs
│   │   └── Program.cs
│   ├── Consumers/                     # RabbitMQ Consumers
│   │   ├── DocumentUploadedConsumer.cs
│   │   ├── ParseDocumentConsumer.cs
│   │   ├── EmbedChunksConsumer.cs
│   │   └── DocumentFailureConsumer.cs (DLQ)
│   ├── Messages/                      # MassTransit 訊息契約
│   │   ├── DocumentUploaded.cs
│   │   ├── ParseDocument.cs
│   │   ├── EmbedChunks.cs
│   │   └── DocumentFailed.cs
│   ├── Services/
│   │   ├── IDocumentParser.cs
│   │   ├── UnstructuredIoParser.cs
│   │   ├── IEmbeddingService.cs
│   │   ├── BgeM3EmbeddingService.cs
│   │   └── IMinioStorageService.cs
│   ├── Data/
│   │   ├── DocumentDbContext.cs
│   │   ├── Entities/
│   │   │   ├── Document.cs
│   │   │   ├── Chunk.cs
│   │   │   └── DocumentRole.cs
│   │   └── Repositories/
│   │       ├── IDocumentRepository.cs
│   │       └── DocumentRepository.cs
│   └── Configuration/
│       └── MassTransitConfiguration.cs
│
├── RagV1.Query/                       # 查詢服務（後期）
│   ├── Api/
│   ├── Services/
│   └── Data/
│
├── RagV1.MeetingAnalysis/             # 會議分析服務（後期）
│   ├── Api/
│   ├── Consumers/
│   └── Services/
│
└── RagV1.Shared/                      # 共用基礎設施
    ├── Common/
    │   ├── ApiEnvelope.cs             # { data, error, meta } 標準回應
    │   └── ErrorDescriptor.cs
    ├── Auth/
    │   ├── IRoleProvider.cs
    │   └── CurrentUserContext.cs
    └── Observability/
        └── ActivityNames.cs

tests/
├── RagV1.DocumentIngestion.Tests/
│   ├── Unit/
│   │   ├── Services/
│   │   │   ├── UnstructuredIoParserTests.cs
│   │   │   └── BgeM3EmbeddingServiceTests.cs
│   │   └── Consumers/
│   │       ├── ParseDocumentConsumerTests.cs
│   │       └── EmbedChunksConsumerTests.cs
│   ├── Integration/
│   │   ├── DocumentUploadIntegrationTests.cs
│   │   ├── RabbitMqRetryIntegrationTests.cs
│   │   └── DlqIntegrationTests.cs
│   └── Contract/
│       └── DocumentApiContractTests.cs
│
└── RagV1.Identity.Tests/
    ├── Unit/
    └── Integration/
```

**結構決策**: 
採用 .NET Aspire 微服務編排架構，每個功能域（Identity、DocumentIngestion、Query、MeetingAnalysis）各自獨立為一個服務專案。本期範圍聚焦於 `RagV1.DocumentIngestion` 服務的完整實作，包含 RabbitMQ 消費者管線、重試機制與 DLQ 處理。

`RagV1.AppHost` 作為 Aspire 編排主機，負責啟動所有服務容器（PostgreSQL、MinIO、RabbitMQ、各微服務）並管理服務間相依性。

`RagV1.Shared` 提供跨服務共用的標準回應格式（ApiEnvelope）、身分脈絡與可觀測性工具。

測試結構遵循 TDD 原則，按 Unit / Integration / Contract 分層，整合測試使用 Testcontainers 啟動真實 PostgreSQL、RabbitMQ 與 MinIO 容器。

## 複雜度追蹤

> **僅在架構原則檢查有需要說明的違規時填寫**

| 違規項目 | 為何需要 | 被拒絕的更簡單替代方案與理由 |
|---------|---------|----------------------------|
| 無 | - | - |

---

## 工作計畫

### Phase 0：大綱與研究

- [x] 研究 .NET Aspire 最佳實務與服務間通訊模式
- [x] 研究 MassTransit + RabbitMQ 重試策略（Retry Policy、Circuit Breaker、DLQ）
- [x] 研究 Unstructured.io API 整合方式與 Markdown 輸出格式
- [x] 研究 BAAI/bge-m3 模型部署方式（本地推論 vs. API 呼叫）
- [x] 研究 PGVector 向量索引策略與效能調校
- [x] 研究 OpenIddict 多角色 JWT Claim 結構與驗證流程
- [x] 研究 MinIO .NET SDK 檔案上傳與權限管理
- [x] 整合所有研究結果至 `research.md`

**輸出**: ✅ `research.md`，所有「NEEDS CLARIFICATION」項目已解決

### Phase 1：設計與契約

**前置條件**: ✅ `research.md` 已完成

- [x] 從功能規格提取實體並撰寫 `data-model.md`：
  - Document（文件）
  - Chunk（段落）
  - User（使用者）
  - Role（角色）
  - DocumentRole（文件角色白名單）
  - QueryLog（查詢紀錄）
  - MeetingRecording（會議錄音）
  - Transcript（逐字稿）
  - MeetingSummary（會議摘要）
  - 定義實體關聯、驗證規則與狀態轉換
- [x] 定義介面契約並建立契約文件：
  - `contracts.md`（包含 Document API、RabbitMQ 訊息契約、API 回應格式）
  - 注：實作時應拆分至獨立的 OpenAPI/JSON Schema 檔案
- [x] 建立 `quickstart.md` 開發者快速啟動指南
- [x] 建立 GitHub Copilot 代理脈絡檔案（`.github/agents/copilot-instructions.md`）

**輸出**: ✅ `data-model.md`、✅ `contracts.md`、✅ `quickstart.md`、✅ 代理脈絡檔案已建立

### Phase 2：計畫總結

**本命令在此階段停止**。執行 `/speckit.tasks` 命令以產生可執行的工作分解（`tasks.md`）。

---

## 注意事項

- 本計畫遵循 RagV1 Constitution 1.1.0 版本的所有原則
- 所有文件與任務描述均以繁體中文撰寫（架構原則 V）
- 測試必須在實作前撰寫（TDD，架構原則 II）
- 效能基準必須納入測試套件（架構原則 IV）
- 文件匯入管線為本期核心範圍，查詢與會議分析功能於後期實作

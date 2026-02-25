---
description: "企業級多語系 RAG 平台的任務清單"
---

# Tasks: 企業級多語系 RAG 與會議智慧分析平台

**輸入**: 設計文件來自 `/specs/001-enterprise-rag-platform/`  
**前置條件**: plan.md（必要）、spec.md（必要，含使用者故事）、research.md、data-model.md、contracts/

> ⚠️ **架構原則 V**：任務標題、檢查點描述與所有敘述文字
> 均以繁體中文（zh-TW）撰寫。檔案路徑、識別碼與程式碼區塊可保留英文。

**測試**: 以下範例包含測試任務。測試是**強制性的**（架構原則 II：測試優先標準）。  
單元測試、整合測試與契約測試**必須**在實作程式碼前撰寫。下方的「OPTIONAL」標記僅指  
特定測試**類型**（契約 vs. 整合），而非測試的存在本身。

**組織方式**: 任務依使用者故事分組，以實現每個故事的獨立實作與測試。

## 格式: `[ID] [P?] [Story] Description`

- **[P]**: 可平行執行（不同檔案、無相依性）
- **[Story]**: 任務歸屬的使用者故事（例：US1、US2、US3）
- 描述中包含確切的檔案路徑

## 路徑慣例

本專案採用 .NET Aspire 微服務架構：
- **主機**: `src/RagV1.AppHost/`
- **服務**: `src/RagV1.{ServiceName}/`
- **共用**: `src/RagV1.Shared/`
- **測試**: `tests/RagV1.{ServiceName}.Tests/`

---

## Phase 1: Setup（共用基礎設施）

**目的**: 專案初始化與基本結構

- [ ] T001 建立 .NET Aspire 解決方案結構及 ServiceDefaults 專案（參照 plan.md 專案結構）
- [ ] T002 初始化 RagV1.AppHost 專案並配置服務編排設定於 src/RagV1.AppHost/Program.cs
- [ ] T003 [P] 初始化 RagV1.Shared 專案並建立標準 API 回應格式於 src/RagV1.Shared/Common/ApiEnvelope.cs
- [ ] T004 [P] 配置 .editorconfig 與 Linter 規則於儲存庫根目錄
- [ ] T005 [P] 建立 Docker Compose 開發環境配置檔於 docker-compose.dev.yml（PostgreSQL、MinIO、RabbitMQ）
- [ ] T006 配置 PostgreSQL 連線與 PGVector 擴充啟用於 RagV1.AppHost

---

## Phase 2: Foundational（阻塞性前置條件）

**目的**: 所有使用者故事開始前**必須**完成的核心基礎設施

**⚠️ 關鍵**: 在此階段完成前，任何使用者故事均無法開始

- [ ] T007 建立 RagV1.Shared 專案的基礎型別：ApiEnvelope<T>、ErrorDescriptor 於 src/RagV1.Shared/Common/
- [ ] T008 [P] 實作可觀測性基礎架構於 src/RagV1.Shared/Observability/ActivityNames.cs
- [ ] T009 [P] 建立 User、Role、UserRole 實體於 src/RagV1.Identity/Models/（參照 data-model.md）
- [ ] T010 建立 IdentityDbContext 與 EF Core 遷移於 src/RagV1.Identity/Data/
- [ ] T011 [P] 建立 Document、Chunk、DocumentRole 實體於 src/RagV1.DocumentIngestion/Data/Entities/（參照 data-model.md）
- [ ] T012 [P] 建立 QueryLog、ChunkCitation 實體於 src/RagV1.Query/Data/Entities/（參照 data-model.md）
- [ ] T013 建立 DocumentDbContext 與 EF Core 遷移於 src/RagV1.DocumentIngestion/Data/
- [ ] T014 配置 MassTransit 與 RabbitMQ 於 RagV1.AppHost（含 Retry Policy、DLQ 配置）
- [ ] T015 定義所有 RabbitMQ 訊息契約於 src/RagV1.DocumentIngestion/Messages/（DocumentUploaded.cs、ParseDocument.cs、EmbedChunks.cs、DocumentFailed.cs）

**檢查點**: 基礎就緒 - 使用者故事實作現在可以平行開始

---

## Phase 3: User Story 2 - SSO 登入與多角色權限管理 (Priority: P1) 🎯 MVP 元件 1

**目標**: 企業員工以 OIDC 單一登入，系統自動載入多重角色並建立工作階段

**獨立測試**: 完成 OIDC 登入流程，驗證工作階段包含正確角色陣列

### 測試（必須在實作前撰寫並確保失敗 - TDD 紅燈階段）

- [ ] T016 [P] [US2] 撰寫 AuthorizationController 契約測試於 tests/RagV1.Identity.Tests/Contract/AuthApiContractTests.cs
- [ ] T017 [P] [US2] 撰寫多角色 JWT Claims 整合測試於 tests/RagV1.Identity.Tests/Integration/OidcMultiRoleTests.cs

### 實作

- [ ] T018 [P] [US2] 配置 OpenIddict 於 RagV1.Identity 服務（Program.cs，參照 research.md OIDC 決策）
- [ ] T019 [P] [US2] 實作 AuthorizationController（OAuth 授權端點）於 src/RagV1.Identity/Controllers/AuthorizationController.cs
- [ ] T020 [P] [US2] 實作 TokenController（Token 端點與多角色 Claim 注入）於 src/RagV1.Identity/Controllers/TokenController.cs
- [ ] T021 [US2] 實作 IRoleProvider 介面與 UserRoleService 於 src/RagV1.Shared/Auth/（查詢使用者所屬角色）
- [ ] T022 [US2] 建立 JWT 驗證中介層於 src/RagV1.Shared/Auth/JwtAuthMiddleware.cs
- [ ] T023 [US2] 建立 CurrentUserContext（從 JWT Claims 提取當前使用者資訊）於 src/RagV1.Shared/Auth/CurrentUserContext.cs
- [ ] T024 [US2] 加入身分驗證日誌記錄（登入、角色變更）於 RagV1.Identity（滿足 FR-011）

**檢查點**: 此時 SSO 登入應完全可運作，可獨立測試 OIDC 流程與角色載入

---

## Phase 4: User Story 3 - 文件上傳與智慧解析 (Priority: P2) 🎯 MVP 元件 2

**目標**: 部門管理員上傳 PDF，系統自動解析並建立可檢索知識庫

**獨立測試**: 上傳含表格與化學式的 PDF，查詢資料庫確認段落已正確解析

### 測試（必須在實作前撰寫並確保失敗 - TDD 紅燈階段）

- [ ] T025 [P] [US3] 撰寫 Document API 契約測試於 tests/RagV1.DocumentIngestion.Tests/Contract/DocumentApiContractTests.cs
- [ ] T026 [P] [US3] 撰寫文件上傳整合測試（端到端：上傳 → 解析 → 向量化 → Ready）於 tests/RagV1.DocumentIngestion.Tests/Integration/DocumentUploadIntegrationTests.cs
- [ ] T027 [P] [US3] 撰寫 RabbitMQ 重試機制整合測試於 tests/RagV1.DocumentIngestion.Tests/Integration/RabbitMqRetryIntegrationTests.cs
- [ ] T028 [P] [US3] 撰寫 DLQ 失敗處理整合測試於 tests/RagV1.DocumentIngestion.Tests/Integration/DlqIntegrationTests.cs
- [ ] T029 [P] [US3] 撰寫 UnstructuredIoParser 單元測試於 tests/RagV1.DocumentIngestion.Tests/Unit/Services/UnstructuredIoParserTests.cs
- [ ] T030 [P] [US3] 撰寫 BgeM3EmbeddingService 單元測試於 tests/RagV1.DocumentIngestion.Tests/Unit/Services/BgeM3EmbeddingServiceTests.cs

### 實作

- [ ] T031 [P] [US3] 建立 IDocumentRepository 介面於 src/RagV1.DocumentIngestion/Data/Repositories/IDocumentRepository.cs
- [ ] T032 [P] [US3] 實作 DocumentRepository（包含狀態轉換與角色過濾邏輯）於 src/RagV1.DocumentIngestion/Data/Repositories/DocumentRepository.cs
- [ ] T033 [P] [US3] 建立 IMinioStorageService 介面於 src/RagV1.DocumentIngestion/Services/IMinioStorageService.cs
- [ ] T034 [P] [US3] 實作 MinioStorageService（檔案上傳與下載）於 src/RagV1.DocumentIngestion/Services/MinioStorageService.cs
- [ ] T035 [P] [US3] 建立 IDocumentParser 介面於 src/RagV1.DocumentIngestion/Services/IDocumentParser.cs
- [ ] T036 [P] [US3] 實作 UnstructuredIoParser（呼叫 Unstructured.io API）於 src/RagV1.DocumentIngestion/Services/UnstructuredIoParser.cs
- [ ] T037 [P] [US3] 建立 IEmbeddingService 介面於 src/RagV1.DocumentIngestion/Services/IEmbeddingService.cs
- [ ] T038 [P] [US3] 實作 BgeM3EmbeddingService（呼叫 bge-m3 模型）於 src/RagV1.DocumentIngestion/Services/BgeM3EmbeddingService.cs
- [ ] T039 [US3] 實作 DocumentController（POST /v1/documents 端點）於 src/RagV1.DocumentIngestion/Api/Controllers/DocumentController.cs
- [ ] T040 [US3] 實作 DocumentUploadedConsumer（處理上傳事件，觸發解析）於 src/RagV1.DocumentIngestion/Consumers/DocumentUploadedConsumer.cs
- [ ] T041 [US3] 實作 ParseDocumentConsumer（呼叫解析服務，產生 Chunks）於 src/RagV1.DocumentIngestion/Consumers/ParseDocumentConsumer.cs
- [ ] T042 [US3] 實作 EmbedChunksConsumer（批次向量化 Chunks）於 src/RagV1.DocumentIngestion/Consumers/EmbedChunksConsumer.cs
- [ ] T043 [US3] 實作 FinalizeDocumentConsumer（標記 Document.Status = Ready）於 src/RagV1.DocumentIngestion/Consumers/FinalizeDocumentConsumer.cs
- [ ] T044 [US3] 實作 DocumentFailureConsumer（DLQ 處理器，記錄失敗原因）於 src/RagV1.DocumentIngestion/Consumers/DocumentFailureConsumer.cs
- [ ] T045 [US3] 配置 MassTransit Retry Policy（最多 3 次，指數退避）於 src/RagV1.DocumentIngestion/Configuration/MassTransitConfiguration.cs
- [ ] T046 [US3] 實作 GET /v1/documents 端點（列表查詢，含角色過濾）於 DocumentController.cs
- [ ] T047 [US3] 實作 GET /v1/documents/{id} 端點（單一文件詳情）於 DocumentController.cs
- [ ] T048 [US3] 實作 DELETE /v1/documents/{id} 端點（軟刪除）於 DocumentController.cs
- [ ] T049 [US3] 實作 POST /v1/documents/{id}/retry 端點（手動重試失敗文件）於 DocumentController.cs
- [ ] T050 [US3] 加入文件上傳驗證（檔案大小 ≤ 50MB、僅允許 PDF）於 DocumentController.cs
- [ ] T051 [US3] 加入錯誤處理與使用者友善訊息於 DocumentController.cs（滿足 NFR-UX-002）
- [ ] T052 [US3] 加入文件 ETL 管線日誌記錄（上傳、解析、向量化各階段）

**檢查點**: 此時文件上傳與解析管線應完全可運作，可獨立測試上傳 PDF 並驗證段落已建立

---

## Phase 5: User Story 1 - 知識檢索與問答 (Priority: P1) 🎯 MVP 元件 3

**目標**: 研發工程師以自然語言查詢內部技術文件，數秒內獲得引用原文與整理答覆

**獨立測試**: 建立測試文件並上傳，以問答介面提問，驗證系統返回正確段落引用與答覆

### 測試（必須在實作前撰寫並確保失敗 - TDD 紅燈階段）

- [ ] T053 [P] [US1] 撰寫 Query API 契約測試於 tests/RagV1.Query.Tests/Contract/QueryApiContractTests.cs
- [ ] T054 [P] [US1] 撰寫角色過濾整合測試（確保越權查詢返回空結果）於 tests/RagV1.Query.Tests/Integration/RoleFilteringTests.cs
- [ ] T055 [P] [US1] 撰寫多語系查詢整合測試於 tests/RagV1.Query.Tests/Integration/MultilingualQueryTests.cs
- [ ] T056 [P] [US1] 撰寫 VectorSearchService 單元測試於 tests/RagV1.Query.Tests/Unit/Services/VectorSearchServiceTests.cs
- [ ] T057 [P] [US1] 撰寫 LlmService 單元測試（含備援切換邏輯）於 tests/RagV1.Query.Tests/Unit/Services/LlmServiceTests.cs

### 實作

- [ ] T058 [P] [US1] 初始化 RagV1.Query 服務專案結構於 src/RagV1.Query/
- [ ] T059 [P] [US1] 建立 IVectorSearchService 介面於 src/RagV1.Query/Services/IVectorSearchService.cs
- [ ] T060 [P] [US1] 實作 VectorSearchService（使用 PGVector 相似度搜尋，含角色過濾）於 src/RagV1.Query/Services/VectorSearchService.cs
- [ ] T061 [P] [US1] 建立 ILlmService 介面於 src/RagV1.Query/Services/ILlmService.cs
- [ ] T062 [P] [US1] 實作 LiteLlmService（透過 LiteLLM 閘道器呼叫 LLM，含備援切換）於 src/RagV1.Query/Services/LiteLlmService.cs
- [ ] T063 [US1] 實作 QueryController（POST /v1/query 端點）於 src/RagV1.Query/Api/Controllers/QueryController.cs
- [ ] T064 [US1] 實作 QueryOrchestrationService（整合向量搜尋 + LLM 生成 + QueryLog 記錄）於 src/RagV1.Query/Services/QueryOrchestrationService.cs
- [ ] T065 [US1] 實作 IQueryLogRepository 與 QueryLogRepository 於 src/RagV1.Query/Data/Repositories/
- [ ] T066 [US1] 加入查詢效能指標記錄（RetrievalLatencyMs、GenerationLatencyMs）於 QueryOrchestrationService.cs
- [ ] T067 [US1] 實作多語系查詢語言偵測於 src/RagV1.Query/Services/LanguageDetectionService.cs
- [ ] T068 [US1] 加入查詢驗證（最多 1000 字元、必須包含實際問題）於 QueryController.cs
- [ ] T069 [US1] 加入越權查詢稽核日誌（嘗試查詢無授權文件時記錄事件）於 QueryOrchestrationService.cs
- [ ] T070 [US1] 實作 GET /v1/query/history 端點（查詢使用者歷史紀錄）於 QueryController.cs

**檢查點**: 此時知識檢索與問答應完全可運作，研發工程師可提問並獲得引用答覆（MVP 核心功能完成）

---

## Phase 6: User Story 4 - 會議錄音自動摘要 (Priority: P2)

**目標**: 專案主管上傳會議錄音，系統自動產生逐字稿與結構化摘要

**獨立測試**: 上傳一段 30 分鐘語音，驗收逐字稿正確率與結構化輸出格式

### 測試（必須在實作前撰寫並確保失敗 - TDD 紅燈階段）

- [ ] T071 [P] [US4] 撰寫 MeetingRecording API 契約測試於 tests/RagV1.MeetingAnalysis.Tests/Contract/MeetingApiContractTests.cs
- [ ] T072 [P] [US4] 撰寫會議轉錄整合測試於 tests/RagV1.MeetingAnalysis.Tests/Integration/TranscriptionIntegrationTests.cs
- [ ] T073 [P] [US4] 撰寫會議摘要整合測試於 tests/RagV1.MeetingAnalysis.Tests/Integration/SummarizationIntegrationTests.cs
- [ ] T074 [P] [US4] 撰寫 WhisperService 單元測試於 tests/RagV1.MeetingAnalysis.Tests/Unit/Services/WhisperServiceTests.cs
- [ ] T075 [P] [US4] 撰寫 MeetingSummaryService 單元測試於 tests/RagV1.MeetingAnalysis.Tests/Unit/Services/MeetingSummaryServiceTests.cs

### 實作

- [ ] T076 [P] [US4] 初始化 RagV1.MeetingAnalysis 服務專案結構於 src/RagV1.MeetingAnalysis/
- [ ] T077 [P] [US4] 建立 MeetingRecording、Transcript、TranscriptSegment、MeetingSummary、ActionItem 實體於 src/RagV1.MeetingAnalysis/Data/Entities/（參照 data-model.md）
- [ ] T078 [US4] 建立 MeetingDbContext 與 EF Core 遷移於 src/RagV1.MeetingAnalysis/Data/
- [ ] T079 [P] [US4] 定義 RabbitMQ 訊息契約（MeetingUploaded、TranscribeMeeting、SummarizeMeeting）於 src/RagV1.MeetingAnalysis/Messages/
- [ ] T080 [P] [US4] 建立 IWhisperService 介面於 src/RagV1.MeetingAnalysis/Services/IWhisperService.cs
- [ ] T081 [P] [US4] 實作 WhisperService（呼叫 Whisper API 進行語音轉文字）於 src/RagV1.MeetingAnalysis/Services/WhisperService.cs
- [ ] T082 [P] [US4] 建立 IMeetingSummaryService 介面於 src/RagV1.MeetingAnalysis/Services/IMeetingSummaryService.cs
- [ ] T083 [P] [US4] 實作 MeetingSummaryService（呼叫 LLM 產生結構化摘要）於 src/RagV1.MeetingAnalysis/Services/MeetingSummaryService.cs
- [ ] T084 [US4] 實作 MeetingController（POST /v1/meetings 上傳錄音端點）於 src/RagV1.MeetingAnalysis/Api/Controllers/MeetingController.cs
- [ ] T085 [US4] 實作 MeetingUploadedConsumer（觸發轉錄流程）於 src/RagV1.MeetingAnalysis/Consumers/MeetingUploadedConsumer.cs
- [ ] T086 [US4] 實作 TranscribeMeetingConsumer（執行 Whisper 轉錄）於 src/RagV1.MeetingAnalysis/Consumers/TranscribeMeetingConsumer.cs
- [ ] T087 [US4] 實作 SummarizeMeetingConsumer（執行摘要生成）於 src/RagV1.MeetingAnalysis/Consumers/SummarizeMeetingConsumer.cs
- [ ] T088 [US4] 實作發言人標籤邏輯（pyannote.audio 或類似解決方案）於 WhisperService.cs
- [ ] T089 [US4] 實作 Action Items 解析（從摘要中提取待辦事項）於 MeetingSummaryService.cs
- [ ] T090 [US4] 實作 GET /v1/meetings 端點（列出會議記錄）於 MeetingController.cs
- [ ] T091 [US4] 實作 GET /v1/meetings/{id}/transcript 端點（取得逐字稿）於 MeetingController.cs
- [ ] T092 [US4] 實作 GET /v1/meetings/{id}/summary 端點（取得摘要）於 MeetingController.cs
- [ ] T093 [US4] 加入會議上傳驗證（檔案格式、最長 90 分鐘）於 MeetingController.cs
- [ ] T094 [US4] 配置 MassTransit 會議分析管線於 RagV1.MeetingAnalysis

**檢查點**: 所有使用者故事現已獨立可運作（US1 知識檢索、US2 SSO 登入、US3 文件上傳、US4 會議摘要）

---

## Phase 7: Polish & Cross-Cutting Concerns（跨領域關注點）

**目的**: 影響多個使用者故事的改善項目

- [ ] T095 [P] 撰寫整體系統文件於 docs/architecture.md
- [ ] T096 [P] 撰寫 API 使用指南於 docs/api-guide.md
- [ ] T097 [P] 撰寫部署指南於 docs/deployment.md
- [ ] T098 程式碼品質檢查與重構（移除重複程式碼、改善命名）
- [ ] T099 [P] 建立效能基準測試（所有延遲敏感路徑，滿足架構原則 IV）於 tests/RagV1.Performance.Tests/
- [ ] T100 驗證效能目標達到 NFR-PERF 門檻（p50/p95 延遲），若回退則阻塞發布
- [ ] T101 [P] 增補單元測試以達到 ≥ 80% 分支覆蓋率（架構原則 II）
- [ ] T102 資安加固：驗證所有端點強制執行角色過濾（零容忍洩漏）
- [ ] T103 驗證所有 API 回應符合 `{ data, error, meta }` 標準格式（架構原則 III）
- [ ] T104 執行 quickstart.md 驗證（從零啟動開發環境至第一次成功查詢）
- [ ] T105 [P] 配置 CI/CD 管線於 .github/workflows/ci.yml（Lint、Build、Test、Coverage）
- [ ] T106 [P] 建立 Docker 生產環境映像檔於 Dockerfile.{ServiceName}

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: 無相依性 - 可立即開始
- **Foundational (Phase 2)**: 依賴 Setup 完成 - **阻塞所有使用者故事**
- **User Stories (Phase 3-6)**: 全部依賴 Foundational 階段完成
  - 使用者故事可平行進行（若有多位開發者）
  - 或依優先順序循序執行（US2 → US3 → US1 → US4）
- **Polish (Phase 7)**: 依賴所有期望的使用者故事完成

### User Story Dependencies

- **User Story 2 (P1 - SSO 登入)**: Foundational 完成後可開始 - 無其他故事相依性
- **User Story 3 (P2 - 文件上傳)**: Foundational 完成後可開始 - 無其他故事相依性（**注意**：與 US1 整合但可獨立測試）
- **User Story 1 (P1 - 知識檢索)**: Foundational 完成後可開始 - 建議 US2（身分驗證）與 US3（文件來源）先行，但技術上可平行（**注意**：實務上 US3 先完成提供測試資料）
- **User Story 4 (P2 - 會議摘要)**: Foundational 完成後可開始 - 無其他故事相依性

### 建議實作順序（MVP First）

1. **Phase 1 & 2**: Setup + Foundational（必須先完成）
2. **Phase 3**: User Story 2（SSO 登入）- 身分驗證基礎
3. **Phase 4**: User Story 3（文件上傳）- 知識庫資料來源
4. **Phase 5**: User Story 1（知識檢索）- 核心價值交付 🎯 **完整 MVP**
5. **停止並驗證**: 測試 US1 + US2 + US3 整合運作，可作為首次部署
6. **Phase 6**: User Story 4（會議摘要）- 擴充功能
7. **Phase 7**: Polish - 跨領域改善

### Within Each User Story

- 測試**必須**先撰寫並失敗，才能開始實作（架構原則 II - 不可協商）
- Models 在 services 之前
- Services 在 endpoints 之前
- 核心實作在整合前
- 故事完成後才移至下個優先級

### Parallel Opportunities

- Setup 階段所有標記 [P] 的任務可平行執行
- Foundational 階段所有標記 [P] 的任務可平行執行（在 Phase 2 內）
- Foundational 完成後，所有使用者故事可同時開始（若團隊容量允許）
- 每個使用者故事內所有標記 [P] 的測試可平行執行
- 每個使用者故事內所有標記 [P] 的 Models 可平行執行
- 不同使用者故事可由不同團隊成員平行處理

---

## Parallel Example: User Story 3

```bash
# 同時啟動 User Story 3 的所有測試撰寫：
Task T025: "撰寫 Document API 契約測試於 tests/RagV1.DocumentIngestion.Tests/Contract/DocumentApiContractTests.cs"
Task T026: "撰寫文件上傳整合測試於 tests/RagV1.DocumentIngestion.Tests/Integration/DocumentUploadIntegrationTests.cs"
Task T027: "撰寫 RabbitMQ 重試機制整合測試於 tests/RagV1.DocumentIngestion.Tests/Integration/RabbitMqRetryIntegrationTests.cs"
Task T028: "撰寫 DLQ 失敗處理整合測試於 tests/RagV1.DocumentIngestion.Tests/Integration/DlqIntegrationTests.cs"
Task T029: "撰寫 UnstructuredIoParser 單元測試於 tests/RagV1.DocumentIngestion.Tests/Unit/Services/UnstructuredIoParserTests.cs"
Task T030: "撰寫 BgeM3EmbeddingService 單元測試於 tests/RagV1.DocumentIngestion.Tests/Unit/Services/BgeM3EmbeddingServiceTests.cs"

# 測試撰寫完成且失敗後，同時啟動所有介面與儲存庫實作：
Task T031: "建立 IDocumentRepository 介面於 src/RagV1.DocumentIngestion/Data/Repositories/IDocumentRepository.cs"
Task T033: "建立 IMinioStorageService 介面於 src/RagV1.DocumentIngestion/Services/IMinioStorageService.cs"
Task T035: "建立 IDocumentParser 介面於 src/RagV1.DocumentIngestion/Services/IDocumentParser.cs"
Task T037: "建立 IEmbeddingService 介面於 src/RagV1.DocumentIngestion/Services/IEmbeddingService.cs"

# 介面完成後，同時啟動所有具體實作：
Task T032: "實作 DocumentRepository 於 src/RagV1.DocumentIngestion/Data/Repositories/DocumentRepository.cs"
Task T034: "實作 MinioStorageService 於 src/RagV1.DocumentIngestion/Services/MinioStorageService.cs"
Task T036: "實作 UnstructuredIoParser 於 src/RagV1.DocumentIngestion/Services/UnstructuredIoParser.cs"
Task T038: "實作 BgeM3EmbeddingService 於 src/RagV1.DocumentIngestion/Services/BgeM3EmbeddingService.cs"
```

---

## Implementation Strategy

### MVP First（僅 User Story 2, 3, 1）

1. 完成 Phase 1: Setup
2. 完成 Phase 2: Foundational（**關鍵** - 阻塞所有故事）
3. 完成 Phase 3: User Story 2（SSO 登入）
4. 完成 Phase 4: User Story 3（文件上傳）
5. 完成 Phase 5: User Story 1（知識檢索）
6. **停止並驗證**: 獨立測試 US1、US2、US3 整合運作
7. 若就緒則部署/示範

### Incremental Delivery（漸進式交付）

1. 完成 Setup + Foundational → 基礎就緒
2. 加入 User Story 2 → 獨立測試 → 部署/示範（SSO 登入可運作）
3. 加入 User Story 3 → 獨立測試 → 部署/示範（SSO + 文件上傳可運作）
4. 加入 User Story 1 → 獨立測試 → 部署/示範（**完整 MVP！知識檢索上線**）
5. 加入 User Story 4 → 獨立測試 → 部署/示範（會議摘要上線）
6. 每個故事增加價值且不破壞先前故事

### Parallel Team Strategy（平行團隊策略）

若有多位開發者：

1. 團隊一起完成 Setup + Foundational
2. Foundational 完成後：
   - 開發者 A: User Story 2（SSO 登入）
   - 開發者 B: User Story 3（文件上傳）
   - 開發者 C: User Story 1（知識檢索）- 可與 A、B 同步開發，整合時需要 US2 與 US3 完成
   - 開發者 D: User Story 4（會議摘要）
3. 故事獨立完成並整合

---

## Notes

- [P] 任務 = 不同檔案、無相依性
- [Story] 標籤將任務映射至特定使用者故事以便追溯
- 每個使用者故事應可獨立完成與測試
- 實作前驗證測試失敗
- 每完成任務或邏輯群組後提交
- 在任何檢查點停止以獨立驗證故事
- 避免：模糊任務、同檔案衝突、破壞獨立性的跨故事相依
- **架構原則 II**: 測試覆蓋率必須 ≥ 80%，在 CI 中強制執行
- **架構原則 III**: 所有 API 回應必須符合標準 `{ data, error, meta }` 格式
- **架構原則 IV**: 所有延遲敏感路徑必須有效能基準測試
- **架構原則 V**: 所有敘述文字以繁體中文撰寫

---

## 總計

- **總任務數**: 106
- **Phase 1 (Setup)**: 6 任務
- **Phase 2 (Foundational)**: 9 任務
- **Phase 3 (US2 - SSO 登入)**: 9 任務
- **Phase 4 (US3 - 文件上傳)**: 28 任務
- **Phase 5 (US1 - 知識檢索)**: 18 任務
- **Phase 6 (US4 - 會議摘要)**: 24 任務
- **Phase 7 (Polish)**: 12 任務

**平行機會**: 63 任務標記為 [P]，可在各自階段內平行執行

**MVP 建議範圍**: Phase 1-5（US2 + US3 + US1 = SSO + 文件上傳 + 知識檢索）共 70 任務

**獨立測試標準**: 
- US2: OIDC 流程完成，角色陣列正確載入
- US3: PDF 上傳後段落可在資料庫查詢
- US1: 提問後返回引用段落與答覆
- US4: 錄音轉為逐字稿與結構化摘要

**格式驗證**: ✅ 所有任務遵循 `- [ ] [ID] [P?] [Story?] Description with file path` 格式

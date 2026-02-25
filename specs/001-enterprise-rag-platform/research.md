# 研究報告：企業級多語系 RAG 平台技術堆疊

**分支**: `001-enterprise-rag-platform` | **日期**: 2026-02-25  
**目的**: 解決實作計畫中所有 NEEDS CLARIFICATION 項目，確立技術選型與最佳實務

---

## 1. .NET Aspire 微服務編排最佳實務

### 決策
採用 .NET Aspire 作為微服務編排框架，使用 `AppHost` 專案統一管理服務生命週期、相依性注入與可觀測性。

### 理由
- **統一開發體驗**：開發者執行單一 `AppHost` 即可啟動所有相依服務（PostgreSQL、RabbitMQ、MinIO）與微服務
- **內建可觀測性**：自動整合 OpenTelemetry、Prometheus 與分散式追蹤
- **服務發現**：微服務間透過 Aspire 服務名稱自動發現，無需硬編碼端點
- **環境一致性**：開發、測試、生產環境使用相同的編排定義

### 考慮的替代方案
- **Docker Compose**：缺乏 .NET 原生整合，可觀測性需手動配置
- **Kubernetes**：過於複雜，對千人企業內網部署而言 overkill
- **手動啟動服務**：開發體驗差，容易因設定不一致導致問題

### 實作要點
```csharp
// RagV1.AppHost/Program.cs
var builder = DistributedApplication.CreateBuilder(args);

// 基礎設施
var postgres = builder.AddPostgres("postgres")
    .WithPgVector()
    .AddDatabase("ragv1db");

var rabbitmq = builder.AddRabbitMQ("messaging")
    .WithManagementPlugin();

var minio = builder.AddMinio("storage", port: 9000)
    .WithDataVolume();

// 微服務
var identity = builder.AddProject<Projects.RagV1_Identity>("identity")
    .WithReference(postgres);

var docIngestion = builder.AddProject<Projects.RagV1_DocumentIngestion>("document-ingestion")
    .WithReference(postgres)
    .WithReference(rabbitmq)
    .WithReference(minio)
    .WithReference(identity);

builder.Build().Run();
```

---

## 2. MassTransit + RabbitMQ 重試與 DLQ 策略

### 決策
使用 MassTransit 8.x 作為 RabbitMQ 抽象層，配置以下重試策略：
- **立即重試**（Immediate Retry）：失敗後立即重試 2 次，間隔 100ms
- **延遲重試**（Delayed Retry）：若立即重試失敗，延遲重試 3 次，間隔為 1min、5min、15min
- **死信佇列**（DLQ）：超過重試上限後進入 `_error` 佇列
- **斷線重連**：MassTransit 內建連線失敗自動重連（預設無限重試）

### 理由
- **容錯性**：暫時性故障（網路抖動、服務重啟）透過立即重試快速恢復
- **延遲重試**：處理外部 API（Unstructured.io）暫時不可用的情況
- **失敗隔離**：失敗訊息進入 DLQ 不阻塞後續任務，管理員可手動重新處理
- **可觀測性**：MassTransit 自動記錄重試次數與失敗原因

### 考慮的替代方案
- **手動實作重試**：容易出錯，缺乏統一可觀測性
- **無重試機制**：任何暫時性故障導致文件解析失敗
- **RabbitMQ 原生重試**：配置複雜，MassTransit 提供更友善的 API

### 實作要點
```csharp
// RagV1.DocumentIngestion/Configuration/MassTransitConfiguration.cs
services.AddMassTransit(x =>
{
    x.AddConsumer<ParseDocumentConsumer>();
    x.AddConsumer<EmbedChunksConsumer>();
    x.AddConsumer<DocumentFailureConsumer>();

    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host("rabbitmq", "/", h =>
        {
            h.Username("guest");
            h.Password("guest");
        });

        cfg.UseMessageRetry(r =>
        {
            r.Immediate(2); // 立即重試 2 次
            r.Intervals(TimeSpan.FromMinutes(1), 
                        TimeSpan.FromMinutes(5), 
                        TimeSpan.FromMinutes(15)); // 延遲重試 3 次
        });

        cfg.ConfigureEndpoints(context);
    });
});
```

**DLQ Consumer 實作**：
```csharp
public class DocumentFailureConsumer : IConsumer<Fault<ParseDocument>>
{
    public async Task Consume(ConsumeContext<Fault<ParseDocument>> context)
    {
        var documentId = context.Message.Message.DocumentId;
        var exceptions = context.Message.Exceptions;
        
        // 記錄至資料庫 DocumentFailure 表
        // 發送通知給管理員
        _logger.LogError("Document {DocumentId} failed after retries: {Reason}", 
                         documentId, exceptions.Last().Message);
    }
}
```

---

## 3. Unstructured.io API 整合與 Markdown 輸出

### 決策
透過 Unstructured.io SaaS API 解析 PDF，輸出格式為 Markdown，支援表格與化學結構圖轉文字描述。

### 理由
- **高品質解析**：專門處理科學文件，支援複雜表格與化學式
- **免維護**：SaaS 方案無需自建 OCR 與解析引擎
- **Markdown 格式**：易於切分段落（Chunking）與向量化
- **多語系支援**：原生支援中文、英文與泰文

### 考慮的替代方案
- **PyPDF2 / pdfplumber**：對複雜排版與化學式處理能力弱
- **Tesseract OCR**：需要掃描型 PDF，本案假設文字型 PDF 為主
- **自建 AI 模型**：維護成本高，Unstructured.io 已提供企業級方案

### 實作要點
```csharp
// RagV1.DocumentIngestion/Services/UnstructuredIoParser.cs
public class UnstructuredIoParser : IDocumentParser
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<UnstructuredIoParser> _logger;

    public async Task<ParsedDocument> ParseAsync(Stream pdfStream, CancellationToken ct)
    {
        using var content = new MultipartFormDataContent();
        content.Add(new StreamContent(pdfStream), "files", "document.pdf");
        content.Add(new StringContent("markdown"), "output_format");
        content.Add(new StringContent("true"), "include_page_breaks");

        var response = await _httpClient.PostAsync("/general/v0/general", content, ct);
        response.EnsureSuccessStatusCode();

        var markdown = await response.Content.ReadAsStringAsync(ct);
        
        // 切分段落（每 512 tokens 一個 Chunk）
        var chunks = ChunkMarkdown(markdown);
        
        return new ParsedDocument
        {
            Markdown = markdown,
            Chunks = chunks
        };
    }
}
```

**Chunking 策略**：
- 段落邊界優先（保留語意完整性）
- 每個 Chunk 512 tokens（BAAI/bge-m3 最佳實務）
- Overlap 50 tokens（避免語意截斷）

---

## 4. BAAI/bge-m3 模型部署與向量化

### 決策
部署本地推論服務（使用 HuggingFace Transformers + FastAPI），不依賴外部 API。

### 理由
- **資安合規**：企業機密文件不可傳送至外部 API
- **成本控制**：大量文件向量化成本可控
- **多語系原生支援**：bge-m3 原生支援中英泰文向量化
- **低延遲**：本地推論 p95 < 100ms（單 Chunk）

### 考慮的替代方案
- **OpenAI Embeddings API**：資安風險高，成本難以預測
- **BAAI/bge-large**：精度略高但速度慢 2 倍，不符合延遲需求
- **自訓練模型**：初期無領域特定訓練資料，使用預訓練模型即可

### 實作要點
**推論服務部署**（Python FastAPI）：
```python
# embedding-service/main.py
from fastapi import FastAPI
from sentence_transformers import SentenceTransformer

app = FastAPI()
model = SentenceTransformer('BAAI/bge-m3')

@app.post("/embed")
async def embed(texts: list[str]):
    embeddings = model.encode(texts, normalize_embeddings=True)
    return {"embeddings": embeddings.tolist()}
```

**C# 客戶端**：
```csharp
// RagV1.DocumentIngestion/Services/BgeM3EmbeddingService.cs
public class BgeM3EmbeddingService : IEmbeddingService
{
    private readonly HttpClient _httpClient;

    public async Task<float[]> EmbedAsync(string text, CancellationToken ct)
    {
        var response = await _httpClient.PostAsJsonAsync("/embed", 
            new { texts = new[] { text } }, ct);
        var result = await response.Content.ReadFromJsonAsync<EmbedResponse>(ct);
        return result.Embeddings[0];
    }
}
```

**效能調校**：
- 使用 GPU 推論（NVIDIA T4 或更高）
- 批次處理（每批 32 個 Chunks）
- 非同步管線（RabbitMQ Consumer 並發數 = 4）

---

## 5. PGVector 向量索引策略

### 決策
使用 **IVFFlat** 索引（初期）+ **HNSW** 索引（生產環境）。

### 理由
- **IVFFlat**：建立速度快，適合初期文件量 < 10 萬筆
- **HNSW**：查詢速度更快（p95 < 50ms），適合生產環境大規模資料
- **Cosine 相似度**：符合 bge-m3 模型訓練時使用的度量方式

### 考慮的替代方案
- **暴力搜尋（No Index）**：延遲高，不符合 p95 < 150ms 需求
- **僅使用 HNSW**：初期建立索引時間過長（數小時）

### 實作要點
```sql
-- Migration: 建立向量欄位與索引
ALTER TABLE chunks ADD COLUMN embedding vector(1024); -- bge-m3 維度

-- 初期使用 IVFFlat（文件量 < 10 萬）
CREATE INDEX idx_chunks_embedding_ivfflat 
ON chunks USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

-- 生產環境切換至 HNSW（文件量 > 10 萬）
DROP INDEX idx_chunks_embedding_ivfflat;
CREATE INDEX idx_chunks_embedding_hnsw 
ON chunks USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

**查詢範例**：
```csharp
// 向量相似度搜尋（Top-10）
var results = await _context.Chunks
    .Where(c => c.Roles.Any(r => userRoles.Contains(r.RoleName))) // 角色過濾
    .OrderBy(c => c.Embedding.CosineDistance(queryEmbedding))
    .Take(10)
    .ToListAsync();
```

---

## 6. OpenIddict 多角色 JWT 與驗證流程

### 決策
使用 OpenIddict 實作 OIDC Authorization Code Flow，JWT Claims 中包含 `roles` 陣列（如 `["RD_General", "Project_A_Secret"]`）。

### 理由
- **企業標準**：OIDC 為企業 SSO 標準協定
- **多角色支援**：JWT Claims 可攜帶多重角色，無需多次查詢資料庫
- **安全性**：Authorization Code Flow 避免 Token 暴露於瀏覽器歷史
- **可擴展**：未來可整合現有企業 IdP（Azure AD / Keycloak）

### 考慮的替代方案
- **自建 JWT**：安全性風險高，不符合 OIDC 標準
- **Session Cookie**：微服務架構下 Session 同步複雜
- **外部 IdP（Azure AD）**：本案需要企業內網獨立部署

### 實作要點
```csharp
// RagV1.Identity/Controllers/AuthorizationController.cs
[HttpPost("token")]
public async Task<IActionResult> Token()
{
    var request = HttpContext.GetOpenIddictServerRequest();
    
    // 驗證使用者身分
    var user = await _userManager.FindByNameAsync(request.Username);
    
    // 取得使用者所有角色
    var roles = await _userManager.GetRolesAsync(user);
    
    var identity = new ClaimsIdentity(TokenValidationParameters.DefaultAuthenticationType);
    identity.AddClaim(new Claim(ClaimTypes.NameIdentifier, user.Id));
    identity.AddClaim(new Claim(ClaimTypes.Name, user.UserName));
    identity.AddClaim(new Claim("roles", JsonSerializer.Serialize(roles))); // 多角色
    
    var principal = new ClaimsPrincipal(identity);
    return SignIn(principal, OpenIddictServerAspNetCoreDefaults.AuthenticationScheme);
}
```

**驗證中介軟體**：
```csharp
// RagV1.Shared/Auth/CurrentUserContext.cs
public class CurrentUserContext
{
    public string UserId { get; set; }
    public List<string> Roles { get; set; }
    
    public static CurrentUserContext FromClaims(ClaimsPrincipal principal)
    {
        var rolesJson = principal.FindFirst("roles")?.Value;
        return new CurrentUserContext
        {
            UserId = principal.FindFirst(ClaimTypes.NameIdentifier).Value,
            Roles = JsonSerializer.Deserialize<List<string>>(rolesJson)
        };
    }
}
```

---

## 7. MinIO .NET SDK 檔案管理

### 決策
使用 MinIO .NET SDK（Minio.AspNetCore）管理原始 PDF 與音檔，Bucket 結構為 `documents/` 與 `recordings/`。

### 理由
- **S3 相容**：未來可無縫切換至 AWS S3
- **企業私有雲**：符合資料不出內網的合規要求
- **版本控制**：支援檔案版本，可追溯歷史
- **權限管理**：支援 Bucket Policy 與 IAM

### 考慮的替代方案
- **本地檔案系統**：難以水平擴展，無版本控制
- **Azure Blob Storage**：需要外部雲端連線
- **PostgreSQL Large Object**：大檔案效能差

### 實作要點
```csharp
// RagV1.DocumentIngestion/Services/MinioStorageService.cs
public class MinioStorageService : IMinioStorageService
{
    private readonly IMinioClient _minioClient;

    public async Task<string> UploadDocumentAsync(Stream fileStream, string fileName, CancellationToken ct)
    {
        var objectName = $"documents/{Guid.NewGuid()}/{fileName}";
        
        await _minioClient.PutObjectAsync(new PutObjectArgs()
            .WithBucket("ragv1")
            .WithObject(objectName)
            .WithStreamData(fileStream)
            .WithObjectSize(fileStream.Length)
            .WithContentType("application/pdf"), ct);
        
        return objectName;
    }
}
```

---

## 總結與下一步

### 所有 NEEDS CLARIFICATION 項目已解決
- ✅ .NET Aspire 編排策略明確
- ✅ MassTransit 重試與 DLQ 配置確立
- ✅ Unstructured.io 整合方案確認
- ✅ bge-m3 本地部署架構決定
- ✅ PGVector 索引策略分階段執行
- ✅ OpenIddict 多角色 JWT 流程設計
- ✅ MinIO 檔案管理方案確立

### 技術風險與緩解措施
| 風險 | 機率 | 影響 | 緩解措施 |
|-----|------|------|---------|
| Unstructured.io API 限流 | 中 | 高 | 批次上傳限制 + 本地 fallback 方案（pdfplumber） |
| bge-m3 推論效能不足 | 低 | 中 | GPU 擴容 + 批次處理最佳化 |
| RabbitMQ 訊息堆積 | 中 | 中 | Consumer 水平擴展 + 監控告警 |
| PGVector 索引建立時間過長 | 低 | 低 | 分階段建立索引（IVFFlat → HNSW） |

### 進入 Phase 1 的前置條件
- ✅ 所有技術選型已確認
- ✅ 相依套件版本已確定
- ✅ 架構模式已建立共識
- ⏭️ 可開始撰寫 data-model.md 與 contracts/

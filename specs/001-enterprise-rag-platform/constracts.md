# API 契約定義

**分支**: `001-enterprise-rag-platform` | **日期**: 2026-02-25  
**目的**: 定義所有外部介面契約

> **注意**: 本檔案為臨時整合版本。實作時應將以下內容拆分至 `/contracts/` 子目錄：
> - `document-api.yaml` - OpenAPI 規格
> - `rabbitmq-messages.schema.json` - RabbitMQ 訊息契約
> - `api-envelope.schema.json` - 標準回應格式

---

## 1. Document API（OpenAPI 3.0）

### 端點概覽

| 方法 | 路徑 | 描述 | 授權 |
|------|------|------|------|
| POST | `/v1/documents` | 上傳文件 | ✓ |
| GET | `/v1/documents` | 列出文件 | ✓ |
| GET | `/v1/documents/{id}` | 取得文件詳情 | ✓ |
| DELETE | `/v1/documents/{id}` | 刪除文件 | ✓ |
| POST | `/v1/documents/{id}/retry` | 重新處理失敗文件 | ✓ |

### POST /v1/documents - 上傳文件

**請求**:
```http
POST /v1/documents HTTP/1.1
Host: api.ragv1.internal
Authorization: Bearer {JWT_TOKEN}
Content-Type: multipart/form-data

--boundary
Content-Disposition: form-data; name="file"; filename="report.pdf"
Content-Type: application/pdf

{PDF_BINARY_DATA}
--boundary
Content-Disposition: form-data; name="roles"

["RD_General", "Project_A_Secret"]
--boundary--
```

**回應 202 Accepted**:
```json
{
  "data": {
    "documentId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "fileName": "report.pdf",
    "status": "Uploaded",
    "uploadedAt": "2026-02-25T15:30:00Z"
  },
  "error": null,
  "meta": {
    "traceId": "8b7e4f2a-1c3d-4e5f-9a8b-7c6d5e4f3a2b",
    "timestamp": "2026-02-25T15:30:05Z"
  }
}
```

**錯誤回應**:
- `400 Bad Request` - 檔案格式錯誤或缺少必填欄位
- `401 Unauthorized` - Token 無效或過期
- `413 Payload Too Large` - 檔案超過 50MB
- `415 Unsupported Media Type` - 非 PDF 格式
- `500 Internal Server Error` - 伺服器錯誤

### GET /v1/documents - 列出文件

**請求**:
```http
GET /v1/documents?status=Ready&page=1&pageSize=20 HTTP/1.1
Authorization: Bearer {JWT_TOKEN}
```

**回應 200 OK**:
```json
{
  "data": {
    "items": [
      {
        "documentId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "fileName": "report.pdf",
        "status": "Ready",
        "uploadedAt": "2026-02-25T15:00:00Z",
        "parsedAt": "2026-02-25T15:05:00Z",
        "chunkCount": 42
      }
    ],
    "totalCount": 156,
    "page": 1,
    "pageSize": 20
  },
  "error": null,
  "meta": {
    "traceId": "8b7e4f2a-1c3d-4e5f-9a8b-7c6d5e4f3a2b",
    "timestamp": "2026-02-25T15:30:05Z"
  }
}
```

### GET /v1/documents/{id} - 取得文件詳情

**回應 200 OK**:
```json
{
  "data": {
    "documentId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "fileName": "report.pdf",
    "status": "Ready",
    "fileSizeBytes": 2457600,
    "contentType": "application/pdf",
    "uploadedAt": "2026-02-25T15:00:00Z",
    "parsedAt": "2026-02-25T15:05:00Z",
    "uploadedBy": {
      "userId": "7c8d9e0f-1a2b-3c4d-5e6f-7a8b9c0d1e2f",
      "displayName": "王小明"
    },
    "roles": ["RD_General", "Project_A_Secret"],
    "chunkCount": 42,
    "retryCount": 0,
    "failureReason": null
  },
  "error": null,
  "meta": {
    "traceId": "8b7e4f2a-1c3d-4e5f-9a8b-7c6d5e4f3a2b",
    "timestamp": "2026-02-25T15:30:05Z"
  }
}
```

**錯誤回應**:
- `401 Unauthorized` - 未授權
- `403 Forbidden` - 無權限查看此文件
- `404 Not Found` - 文件不存在

---

## 2. RabbitMQ 訊息契約（MassTransit）

### 訊息流程圖

```text
DocumentUploaded
  ↓
[ParseDocumentConsumer]
  ↓
DocumentParsed
  ↓
[EmbedChunksConsumer]
  ↓
ChunksEmbedded
  ↓
[FinalizeDocumentConsumer]
  → Document.Status = Ready

任何階段失敗 → DocumentFailed → DLQ
```

### DocumentUploaded（事件）

**用途**: 文件上傳完成，觸發解析流程

**消費者**: `ParseDocumentConsumer`

**Schema**:
```json
{
  "documentId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "fileName": "report.pdf",
  "storagePath": "documents/3fa85f64-5717-4562-b3fc-2c963f66afa6/report.pdf",
  "fileSizeBytes": 2457600,
  "uploadedBy": "7c8d9e0f-1a2b-3c4d-5e6f-7a8b9c0d1e2f",
  "roles": ["RD_General", "Project_A_Secret"],
  "timestamp": "2026-02-25T15:00:00Z"
}
```

### ParseDocument（命令）

**用途**: 呼叫 Unstructured.io API 解析 PDF

**消費者**: `ParseDocumentConsumer`

**Schema**:
```json
{
  "documentId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "storagePath": "documents/3fa85f64-5717-4562-b3fc-2c963f66afa6/report.pdf",
  "retryCount": 0
}
```

### DocumentParsed（事件）

**用途**: 文件解析完成，觸發向量化流程

**消費者**: `EmbedChunksConsumer`

**Schema**:
```json
{
  "documentId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "markdown": "# 研究報告\n\n## 摘要\n...",
  "chunks": [
    {
      "sequenceNumber": 0,
      "content": "研究報告摘要內容...",
      "tokenCount": 487
    },
    {
      "sequenceNumber": 1,
      "content": "第一章內容...",
      "tokenCount": 512
    }
  ],
  "timestamp": "2026-02-25T15:05:00Z"
}
```

### EmbedChunks（命令）

**用途**: 呼叫 bge-m3 模型生成向量

**消費者**: `EmbedChunksConsumer`

**Schema**:
```json
{
  "documentId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "chunks": [
    {
      "chunkId": "1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d",
      "content": "研究報告摘要內容..."
    },
    {
      "chunkId": "2b3c4d5e-6f7a-8b9c-0d1e-2f3a4b5c6d7e",
      "content": "第一章內容..."
    }
  ]
}
```

### ChunksEmbedded（事件）

**用途**: 向量化完成，標記文件為 Ready

**消費者**: `FinalizeDocumentConsumer`

**Schema**:
```json
{
  "documentId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "embeddedCount": 42,
  "timestamp": "2026-02-25T15:08:00Z"
}
```

### DocumentFailed（事件）

**用途**: 處理失敗，進入 DLQ

**消費者**: `DocumentFailureConsumer`

**Schema**:
```json
{
  "documentId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "stage": "Parsing",
  "reason": "Unstructured.io API 回應 500 Internal Server Error",
  "stackTrace": "System.Net.Http.HttpRequestException: ...",
  "retryCount": 3,
  "timestamp": "2026-02-25T15:15:00Z"
}
```

### 重試策略

**立即重試**:
- 次數：2 次
- 間隔：100ms
- 適用情況：暫時性網路抖動

**延遲重試**:
- 間隔：1 分鐘、5 分鐘、15 分鐘
- 適用情況：外部 API 暫時不可用

**DLQ（死信佇列）**:
- 隊列名稱：`ragv1-document-ingestion-dlq`
- 觸發條件：超過重試上限（總共 5 次：2 次立即 + 3 次延遲）
- 處理方式：記錄至 `DocumentFailure` 表，通知管理員

---

## 3. 標準 API 回應格式（ApiEnvelope）

### 結構定義

所有 API 回應**必須**遵循此格式（架構原則 III）：

```typescript
interface ApiEnvelope<T> {
  data: T | null;          // 成功時為資料，失敗時為 null
  error: ErrorDescriptor | null;  // 失敗時為錯誤，成功時為 null
  meta: {
    traceId: string;       // UUID，用於分散式追蹤
    timestamp: string;     // ISO 8601 格式（UTC）
    version?: string;      // API 版本（如 "1.0.0"）
  };
}

interface ErrorDescriptor {
  code: string;            // 全大寫底線分隔（如 "DOCUMENT_TOO_LARGE"）
  message: string;         // 繁體中文白話文錯誤訊息
  details?: string[];      // 額外細節（如驗證錯誤的欄位清單）
}
```

### C# 實作

```csharp
// RagV1.Shared/Common/ApiEnvelope.cs
public class ApiEnvelope<T>
{
    public T? Data { get; init; }
    public ErrorDescriptor? Error { get; init; }
    public required MetaData Meta { get; init; }

    public static ApiEnvelope<T> Success(T data, string traceId)
    {
        return new ApiEnvelope<T>
        {
            Data = data,
            Error = null,
            Meta = new MetaData 
            { 
                TraceId = traceId, 
                Timestamp = DateTimeOffset.UtcNow 
            }
        };
    }

    public static ApiEnvelope<T> Error(string code, string message, string traceId, List<string>? details = null)
    {
        return new ApiEnvelope<T>
        {
            Data = default,
            Error = new ErrorDescriptor 
            { 
                Code = code, 
                Message = message, 
                Details = details 
            },
            Meta = new MetaData 
            { 
                TraceId = traceId, 
                Timestamp = DateTimeOffset.UtcNow 
            }
        };
    }
}

public record ErrorDescriptor
{
    public required string Code { get; init; }
    public required string Message { get; init; }
    public List<string>? Details { get; init; }
}

public record MetaData
{
    public required string TraceId { get; init; }
    public required DateTimeOffset Timestamp { get; init; }
    public string Version { get; init; } = "1.0.0";
}
```

### 標準錯誤代碼

| 代碼 | HTTP 狀態 | 訊息範例 |
|------|----------|---------|
| `VALIDATION_ERROR` | 400 | 請求參數不符合規定 |
| `UNAUTHORIZED` | 401 | 請先登入企業帳號 |
| `FORBIDDEN` | 403 | 您無權限查看此文件 |
| `NOT_FOUND` | 404 | 找不到指定的文件 |
| `PAYLOAD_TOO_LARGE` | 413 | 檔案大小超過 50MB 限制 |
| `UNSUPPORTED_MEDIA_TYPE` | 415 | 僅支援 PDF 格式 |
| `INTERNAL_ERROR` | 500 | 系統暫時無法處理您的請求，請稍後再試 |
| `DOCUMENT_PARSING_FAILED` | 500 | 文件解析失敗，請確認檔案格式正確 |
| `EMBEDDING_SERVICE_UNAVAILABLE` | 503 | 向量化服務暫時無法使用 |

---

## 契約驗證

### OpenAPI 驗證

實作時應使用工具驗證 API 實作符合 OpenAPI 規格：

```bash
# 安裝 swagger-cli
npm install -g @apidevtools/swagger-cli

# 驗證規格
swagger-cli validate contracts/document-api.yaml

# 生成 C# 客戶端（測試用）
nswag openapi2csclient /input:contracts/document-api.yaml /output:tests/ApiClient.cs
```

### 訊息契約測試

MassTransit Consumer 必須包含契約測試：

```csharp
[Fact]
public async Task ParseDocumentConsumer_ShouldHandleValidMessage()
{
    var harness = new InMemoryTestHarness();
    var consumer = harness.Consumer<ParseDocumentConsumer>();

    await harness.Start();

    var message = new ParseDocument
    {
        DocumentId = Guid.NewGuid(),
        StoragePath = "documents/test/file.pdf",
        RetryCount = 0
    };

    await harness.InputQueueSendEndpoint.Send(message);

    Assert.True(await consumer.Consumed.Any<ParseDocument>());
    Assert.False(await harness.Published.Any<Fault<ParseDocument>>());

    await harness.Stop();
}
```

---

## 契約版本管理

### 向後相容原則

- **新增欄位**：允許，但必須為可選（nullable）
- **刪除欄位**：禁止，需發布新版本（如 `/v2/documents`）
- **修改欄位型別**：禁止，需發布新版本
- **修改必填性**：從必填改為可選允許，反之禁止

### 變更通知

所有契約變更必須：
1. 在 PR 中明確標註 `[BREAKING]` 或 `[NON-BREAKING]`
2. 更新契約檔案的版本號
3. 同步更新所有相依的消費者程式碼
4. 通過契約測試（Contract Tests）

# 資料模型：企業級多語系 RAG 平台

**分支**: `001-enterprise-rag-platform` | **日期**: 2026-02-25  
**目的**: 定義所有實體、欄位、關聯與驗證規則

---

## 實體概覽

```text
User (使用者)
  ├─ 1:N → UserRole (使用者角色關聯)
  └─ 1:N → QueryLog (查詢紀錄)

Role (角色)
  ├─ 1:N → UserRole (使用者角色關聯)
  └─ 1:N → DocumentRole (文件角色白名單)

Document (文件)
  ├─ 1:N → Chunk (段落)
  ├─ 1:N → DocumentRole (文件角色白名單)
  └─ 0:1 → DocumentFailure (失敗記錄)

Chunk (段落)
  ├─ N:1 → Document
  └─ N:M → QueryLog (透過 ChunkCitation)

QueryLog (查詢紀錄)
  ├─ N:1 → User
  └─ 1:N → ChunkCitation (引用段落)

MeetingRecording (會議錄音)
  ├─ N:1 → User (上傳者)
  ├─ 0:1 → Transcript (逐字稿)
  └─ 0:1 → MeetingSummary (會議摘要)

Transcript (逐字稿)
  ├─ 1:1 → MeetingRecording
  └─ 1:N → TranscriptSegment (分段)

MeetingSummary (會議摘要)
  ├─ 1:1 → MeetingRecording
  └─ 1:N → ActionItem (待辦事項)
```

---

## 1. User（使用者）

**描述**: 企業員工，透過 OIDC 驗證身分後存取系統。

### 欄位

| 欄位名 | 型別 | 必填 | 說明 | 驗證規則 |
|--------|------|------|------|---------|
| `UserId` | `Guid` | ✓ | 主鍵 | UUID v4 |
| `Email` | `string` | ✓ | 企業電子郵件 | RFC 5322 格式 |
| `DisplayName` | `string` | ✓ | 顯示名稱 | 1-100 字元 |
| `ExternalId` | `string` | ✓ | OIDC 提供者的唯一識別碼 | 不可重複 |
| `CreatedAt` | `DateTimeOffset` | ✓ | 建立時間 | UTC |
| `LastLoginAt` | `DateTimeOffset` | - | 最後登入時間 | UTC |
| `IsActive` | `bool` | ✓ | 帳號狀態 | 預設 `true` |

### 關聯
- **1:N** → `UserRole`：一位使用者可擁有多個角色
- **1:N** → `QueryLog`：一位使用者可產生多筆查詢紀錄

### 索引
- **PK**: `UserId`
- **Unique**: `Email`
- **Unique**: `ExternalId`

---

## 2. Role（角色）

**描述**: 存取控制單位，對應部門或專案（如 `RD_General`、`Project_A_Secret`）。

### 欄位

| 欄位名 | 型別 | 必填 | 說明 | 驗證規則 |
|--------|------|------|------|---------|
| `RoleId` | `Guid` | ✓ | 主鍵 | UUID v4 |
| `RoleName` | `string` | ✓ | 角色名稱 | 3-50 字元，僅允許 `[A-Za-z0-9_]` |
| `DisplayName` | `string` | ✓ | 顯示名稱（繁中） | 1-100 字元 |
| `Description` | `string` | - | 角色說明 | 最多 500 字元 |
| `CreatedAt` | `DateTimeOffset` | ✓ | 建立時間 | UTC |

### 關聯
- **1:N** → `UserRole`：一個角色可指派給多位使用者
- **1:N** → `DocumentRole`：一個角色可存取多份文件

### 索引
- **PK**: `RoleId`
- **Unique**: `RoleName`

---

## 3. UserRole（使用者角色關聯）

**描述**: 多對多關聯表，記錄使用者與角色的對應關係。

### 欄位

| 欄位名 | 型別 | 必填 | 說明 | 驗證規則 |
|--------|------|------|------|---------|
| `UserRoleId` | `Guid` | ✓ | 主鍵 | UUID v4 |
| `UserId` | `Guid` | ✓ | 外鍵 → User | - |
| `RoleId` | `Guid` | ✓ | 外鍵 → Role | - |
| `AssignedAt` | `DateTimeOffset` | ✓ | 指派時間 | UTC |

### 索引
- **PK**: `UserRoleId`
- **Unique**: (`UserId`, `RoleId`)
- **FK**: `UserId` → `User.UserId`
- **FK**: `RoleId` → `Role.RoleId`

---

## 4. Document（文件）

**描述**: 上傳的 PDF 文件，包含原始檔案位置與解析狀態。

### 欄位

| 欄位名 | 型別 | 必填 | 說明 | 驗證規則 |
|--------|------|------|------|---------|
| `DocumentId` | `Guid` | ✓ | 主鍵 | UUID v4 |
| `FileName` | `string` | ✓ | 原始檔名 | 1-255 字元 |
| `StoragePath` | `string` | ✓ | MinIO 物件路徑 | 格式：`documents/{GUID}/{filename}` |
| `FileSizeBytes` | `long` | ✓ | 檔案大小（位元組） | > 0 |
| `ContentType` | `string` | ✓ | MIME 類型 | `application/pdf` |
| `UploadedBy` | `Guid` | ✓ | 上傳者（外鍵 → User） | - |
| `UploadedAt` | `DateTimeOffset` | ✓ | 上傳時間 | UTC |
| `Status` | `enum` | ✓ | 處理狀態 | 見下方狀態轉換 |
| `ParsedAt` | `DateTimeOffset` | - | 解析完成時間 | UTC |
| `MarkdownContent` | `string` | - | 解析後的 Markdown | - |
| `RetryCount` | `int` | ✓ | 重試次數 | 預設 0，最大 3 |

### 狀態轉換（Status Enum）

```text
Uploaded (已上傳)
  ↓
Parsing (解析中)
  ↓
Embedding (向量化中)
  ↓
Ready (可查詢)

Parsing / Embedding → Failed (失敗，進入 DLQ)
```

### 關聯
- **N:1** → `User`：每份文件由一位使用者上傳
- **1:N** → `Chunk`：一份文件包含多個段落
- **1:N** → `DocumentRole`：一份文件可授權給多個角色

### 索引
- **PK**: `DocumentId`
- **FK**: `UploadedBy` → `User.UserId`
- **Index**: `Status`（查詢待處理文件）
- **Index**: `UploadedAt`（時間排序）

---

## 5. Chunk（段落）

**描述**: 文件的子片段，攜帶語意向量用於相似度搜尋。

### 欄位

| 欄位名 | 型別 | 必填 | 說明 | 驗證規則 |
|--------|------|------|------|---------|
| `ChunkId` | `Guid` | ✓ | 主鍵 | UUID v4 |
| `DocumentId` | `Guid` | ✓ | 外鍵 → Document | - |
| `SequenceNumber` | `int` | ✓ | 文件內序號 | ≥ 0 |
| `Content` | `string` | ✓ | 段落文字內容 | 最多 2000 字元 |
| `TokenCount` | `int` | ✓ | Token 數量 | 通常為 512 ± 50 |
| `Embedding` | `Vector(1024)` | ✓ | 語意向量（PGVector） | bge-m3 模型輸出 |
| `CreatedAt` | `DateTimeOffset` | ✓ | 建立時間 | UTC |

### 關聯
- **N:1** → `Document`：每個段落屬於一份文件
- **N:M** → `QueryLog`：透過 `ChunkCitation` 記錄被引用關係

### 索引
- **PK**: `ChunkId`
- **FK**: `DocumentId` → `Document.DocumentId`
- **Vector Index**: `Embedding` (HNSW / IVFFlat)
- **Composite**: (`DocumentId`, `SequenceNumber`)

---

## 6. DocumentRole（文件角色白名單）

**描述**: 多對多關聯表，定義哪些角色可存取特定文件。

### 欄位

| 欄位名 | 型別 | 必填 | 說明 | 驗證規則 |
|--------|------|------|------|---------|
| `DocumentRoleId` | `Guid` | ✓ | 主鍵 | UUID v4 |
| `DocumentId` | `Guid` | ✓ | 外鍵 → Document | - |
| `RoleId` | `Guid` | ✓ | 外鍵 → Role | - |
| `GrantedAt` | `DateTimeOffset` | ✓ | 授權時間 | UTC |

### 索引
- **PK**: `DocumentRoleId`
- **Unique**: (`DocumentId`, `RoleId`)
- **FK**: `DocumentId` → `Document.DocumentId`
- **FK**: `RoleId` → `Role.RoleId`

---

## 7. QueryLog（查詢紀錄）

**描述**: 使用者問答的完整紀錄，供稽核使用。

### 欄位

| 欄位名 | 型別 | 必填 | 說明 | 驗證規則 |
|--------|------|------|------|---------|
| `QueryLogId` | `Guid` | ✓ | 主鍵 | UUID v4 |
| `UserId` | `Guid` | ✓ | 外鍵 → User | - |
| `QueryText` | `string` | ✓ | 使用者問題 | 最多 1000 字元 |
| `QueryLanguage` | `enum` | ✓ | 查詢語言 | `zh-TW`, `zh-CN`, `en`, `th` |
| `ResponseText` | `string` | ✓ | 系統答覆 | - |
| `QueryEmbedding` | `Vector(1024)` | ✓ | 問題向量 | - |
| `RetrievalLatencyMs` | `int` | ✓ | 檢索延遲（毫秒） | - |
| `GenerationLatencyMs` | `int` | ✓ | 生成延遲（毫秒） | - |
| `CreatedAt` | `DateTimeOffset` | ✓ | 查詢時間 | UTC |

### 關聯
- **N:1** → `User`：每筆查詢由一位使用者發起
- **1:N** → `ChunkCitation`：一筆查詢可引用多個段落

### 索引
- **PK**: `QueryLogId`
- **FK**: `UserId` → `User.UserId`
- **Index**: `CreatedAt`（時間範圍查詢）

---

## 8. ChunkCitation（引用段落）

**描述**: 記錄查詢與引用段落的對應關係。

### 欄位

| 欄位名 | 型別 | 必填 | 說明 | 驗證規則 |
|--------|------|------|------|---------|
| `CitationId` | `Guid` | ✓ | 主鍵 | UUID v4 |
| `QueryLogId` | `Guid` | ✓ | 外鍵 → QueryLog | - |
| `ChunkId` | `Guid` | ✓ | 外鍵 → Chunk | - |
| `RelevanceScore` | `float` | ✓ | 相似度分數 | 0.0 ~ 1.0 |
| `Rank` | `int` | ✓ | 排序位置 | 1, 2, 3, ... |

### 索引
- **PK**: `CitationId`
- **FK**: `QueryLogId` → `QueryLog.QueryLogId`
- **FK**: `ChunkId` → `Chunk.ChunkId`

---

## 9. DocumentFailure（文件失敗記錄）

**描述**: 記錄進入 DLQ 的文件失敗資訊。

### 欄位

| 欄位名 | 型別 | 必填 | 說明 | 驗證規則 |
|--------|------|------|------|---------|
| `FailureId` | `Guid` | ✓ | 主鍵 | UUID v4 |
| `DocumentId` | `Guid` | ✓ | 外鍵 → Document | - |
| `FailureReason` | `string` | ✓ | 失敗原因 | 最多 2000 字元 |
| `StackTrace` | `string` | - | 例外堆疊（除錯用） | - |
| `FailedAt` | `DateTimeOffset` | ✓ | 失敗時間 | UTC |
| `RetryCount` | `int` | ✓ | 失敗前的重試次數 | - |

### 關聯
- **N:1** → `Document`：每筆失敗記錄對應一份文件

### 索引
- **PK**: `FailureId`
- **FK**: `DocumentId` → `Document.DocumentId`

---

## 10. MeetingRecording（會議錄音）

**描述**: 上傳的會議錄音或影片檔案。

### 欄位

| 欄位名 | 型別 | 必填 | 說明 | 驗證規則 |
|--------|------|------|------|---------|
| `RecordingId` | `Guid` | ✓ | 主鍵 | UUID v4 |
| `FileName` | `string` | ✓ | 原始檔名 | 1-255 字元 |
| `StoragePath` | `string` | ✓ | MinIO 物件路徑 | 格式：`recordings/{GUID}/{filename}` |
| `DurationSeconds` | `int` | - | 會議時長（秒） | ≥ 0 |
| `UploadedBy` | `Guid` | ✓ | 上傳者（外鍵 → User） | - |
| `UploadedAt` | `DateTimeOffset` | ✓ | 上傳時間 | UTC |
| `Status` | `enum` | ✓ | 處理狀態 | Uploaded / Transcribing / Summarizing / Ready / Failed |
| `Language` | `enum` | ✓ | 主要語言 | `zh-TW`, `en` |

### 關聯
- **N:1** → `User`：每筆錄音由一位使用者上傳
- **0:1** → `Transcript`：一筆錄音產生一份逐字稿
- **0:1** → `MeetingSummary`：一筆錄音產生一份摘要

### 索引
- **PK**: `RecordingId`
- **FK**: `UploadedBy` → `User.UserId`

---

## 11. Transcript（逐字稿）

**描述**: 由錄音生成的文字稿，包含發言人標籤。

### 欄位

| 欄位名 | 型別 | 必填 | 說明 | 驗證規則 |
|--------|------|------|------|---------|
| `TranscriptId` | `Guid` | ✓ | 主鍵 | UUID v4 |
| `RecordingId` | `Guid` | ✓ | 外鍵 → MeetingRecording | - |
| `FullText` | `string` | ✓ | 完整逐字稿 | - |
| `CreatedAt` | `DateTimeOffset` | ✓ | 生成時間 | UTC |

### 關聯
- **1:1** → `MeetingRecording`：一份逐字稿對應一筆錄音
- **1:N** → `TranscriptSegment`：一份逐字稿包含多個分段

### 索引
- **PK**: `TranscriptId`
- **Unique FK**: `RecordingId` → `MeetingRecording.RecordingId`

---

## 12. TranscriptSegment（逐字稿分段）

**描述**: 逐字稿的單一發言片段，包含發言人與時間戳記。

### 欄位

| 欄位名 | 型別 | 必填 | 說明 | 驗證規則 |
|--------|------|------|------|---------|
| `SegmentId` | `Guid` | ✓ | 主鍵 | UUID v4 |
| `TranscriptId` | `Guid` | ✓ | 外鍵 → Transcript | - |
| `SequenceNumber` | `int` | ✓ | 分段序號 | ≥ 0 |
| `SpeakerLabel` | `string` | ✓ | 發言人標籤 | 例：`Speaker_1` |
| `StartTimeSeconds` | `int` | ✓ | 開始時間（秒） | ≥ 0 |
| `EndTimeSeconds` | `int` | ✓ | 結束時間（秒） | > StartTimeSeconds |
| `Text` | `string` | ✓ | 發言內容 | - |

### 索引
- **PK**: `SegmentId`
- **FK**: `TranscriptId` → `Transcript.TranscriptId`
- **Composite**: (`TranscriptId`, `SequenceNumber`)

---

## 13. MeetingSummary（會議摘要）

**描述**: 結構化的會議摘要輸出。

### 欄位

| 欄位名 | 型別 | 必填 | 說明 | 驗證規則 |
|--------|------|------|------|---------|
| `SummaryId` | `Guid` | ✓ | 主鍵 | UUID v4 |
| `RecordingId` | `Guid` | ✓ | 外鍵 → MeetingRecording | - |
| `Conclusion` | `string` | ✓ | 會議結論 | - |
| `Decisions` | `string` | ✓ | 決議事項（JSON 陣列） | - |
| `CreatedAt` | `DateTimeOffset` | ✓ | 生成時間 | UTC |

### 關聯
- **1:1** → `MeetingRecording`：一份摘要對應一筆錄音
- **1:N** → `ActionItem`：一份摘要包含多個待辦事項

### 索引
- **PK**: `SummaryId`
- **Unique FK**: `RecordingId` → `MeetingRecording.RecordingId`

---

## 14. ActionItem（待辦事項）

**描述**: 會議摘要中提取的待辦事項。

### 欄位

| 欄位名 | 型別 | 必填 | 說明 | 驗證規則 |
|--------|------|------|------|---------|
| `ActionItemId` | `Guid` | ✓ | 主鍵 | UUID v4 |
| `SummaryId` | `Guid` | ✓ | 外鍵 → MeetingSummary | - |
| `Description` | `string` | ✓ | 待辦事項描述 | 1-500 字元 |
| `AssignedTo` | `string` | - | 負責人 | - |
| `DueDate` | `DateTimeOffset` | - | 截止日期 | - |
| `Status` | `enum` | ✓ | 狀態 | Pending / InProgress / Completed |

### 索引
- **PK**: `ActionItemId`
- **FK**: `SummaryId` → `MeetingSummary.SummaryId`

---

## Entity Framework Core Fluent API 範例

```csharp
// RagV1.DocumentIngestion/Data/DocumentDbContext.cs
public class DocumentDbContext : DbContext
{
    public DbSet<Document> Documents { get; set; }
    public DbSet<Chunk> Chunks { get; set; }
    public DbSet<DocumentRole> DocumentRoles { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Document 實體
        modelBuilder.Entity<Document>(entity =>
        {
            entity.HasKey(e => e.DocumentId);
            entity.Property(e => e.FileName).IsRequired().HasMaxLength(255);
            entity.Property(e => e.Status).IsRequired();
            entity.HasIndex(e => e.Status);
            entity.HasIndex(e => e.UploadedAt);
        });

        // Chunk 實體（向量欄位）
        modelBuilder.Entity<Chunk>(entity =>
        {
            entity.HasKey(e => e.ChunkId);
            entity.Property(e => e.Content).IsRequired().HasMaxLength(2000);
            entity.Property(e => e.Embedding).HasColumnType("vector(1024)");
            
            // 關聯
            entity.HasOne<Document>()
                  .WithMany()
                  .HasForeignKey(e => e.DocumentId);

            // 複合索引
            entity.HasIndex(e => new { e.DocumentId, e.SequenceNumber });
        });

        // DocumentRole 多對多
        modelBuilder.Entity<DocumentRole>(entity =>
        {
            entity.HasKey(e => e.DocumentRoleId);
            entity.HasIndex(e => new { e.DocumentId, e.RoleId }).IsUnique();
        });
    }
}
```

---

## 驗證規則總結

### 資料完整性
- 所有 GUID 主鍵使用 UUID v4 格式
- 所有時間戳記使用 `DateTimeOffset`（UTC）
- 所有外鍵必須啟用 CASCADE DELETE（除 QueryLog 與 ChunkCitation）

### 業務規則
- 文件重試次數上限：3 次
- Chunk Token 數量範圍：462-562（512 ± 50）
- 向量相似度分數範圍：0.0-1.0
- 角色名稱僅允許英數字與底線

### 效能約束
- Chunk.Content 最多 2000 字元（避免單筆過大）
- QueryLog.QueryText 最多 1000 字元
- Vector Index 必須在 Chunk.Embedding 欄位建立

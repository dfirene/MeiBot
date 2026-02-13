# 資料治理管理平台 (Data Governance Platform) - 系統開發規格書

**版本**: 1.0  
**日期**: 2026-02-13  
**專案代號**: DGP (Data Governance Platform)

---

## 1. 專案概述

### 1.1 系統目標
建立一個統一的 Web-based 管理平台，提供資料清理、ETL 管道、多資料源整合的全方位可視化管理與監控能力。

### 1.2 核心價值
- **統一視圖**: 一站式查看所有資料管道的運行狀態
- **即時監控**: 即時追蹤資料品質、清理進度、異常告警
- **多源整合**: 支援 PostgreSQL、MySQL、API 等多種資料源
- **可追溯性**: 完整的資料血緣與變更歷史記錄
- **自動化運維**: 自動化的資料品質檢查與異常處理

### 1.3 系統範圍
- ✅ Medallion 架構管理（Source → Clean → Mart）
- ✅ 元數據管理與血緣追蹤
- ✅ ETL 作業監控與排程管理
- ✅ 資料品質監控與告警
- ✅ 多資料源連接管理
- ✅ 資料清理規則配置
- ✅ 報表與儀表板
- ❌ 實際 ETL 執行引擎（使用現有工具如 Airflow/自建腳本）

---

## 2. 功能需求

### 2.1 核心功能模組

#### 模組 1: 儀表板 (Dashboard)
**目的**: 提供系統整體健康狀態的一目了然視圖

**功能清單**:
- **系統概覽卡片**
  - 資料源總數（active/total）
  - ETL 作業總數（running/success/failed）
  - 資料品質評分（整體平均）
  - 今日處理資料量
- **即時監控圖表**
  - 最近 24 小時 ETL 作業執行趨勢
  - 資料品質趨勢圖（7 天）
  - Top 5 失敗作業
  - 資料量增長趨勢
- **告警中心**
  - 未處理告警列表（按嚴重性排序）
  - 告警統計（CRITICAL/WARNING/INFO）
  - 快速處理入口

**UI 設計重點**:
- Grid layout，可自訂卡片位置
- 顏色編碼（綠/黃/紅）表示狀態
- 可點擊卡片深入查看詳情

---

#### 模組 2: 資料源管理 (Data Source Management)
**目的**: 統一管理所有資料源連接與配置

**功能清單**:
- **資料源註冊**
  - 支援類型: PostgreSQL, MySQL, SQL Server, Oracle, REST API, GraphQL
  - 連接參數配置（host, port, database, credentials）
  - 連線測試功能
  - SSL/TLS 配置
- **資料源監控**
  - 連線狀態（在線/離線）
  - 最後連接時間
  - 連線錯誤日誌
  - 效能指標（查詢延遲、連線池使用率）
- **資料源分組**
  - 按環境分組（生產/測試/開發）
  - 按系統分組（ERP/WMS/MES/CRM）
  - 自定義標籤

**資料模型**:
```sql
CREATE TABLE datasources (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL UNIQUE,
    type VARCHAR(50) NOT NULL, -- postgresql, mysql, rest_api, etc.
    environment VARCHAR(20), -- production, staging, development
    system_group VARCHAR(50), -- ERP, WMS, MES, etc.
    connection_config JSONB NOT NULL, -- 連接參數（加密）
    status VARCHAR(20) DEFAULT 'inactive', -- active, inactive, error
    last_connected_at TIMESTAMPTZ,
    health_check_interval INTEGER DEFAULT 300, -- 秒
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    created_by VARCHAR(100),
    tags TEXT[]
);

CREATE TABLE datasource_health_logs (
    id BIGSERIAL PRIMARY KEY,
    datasource_id UUID REFERENCES datasources(id),
    check_time TIMESTAMPTZ DEFAULT NOW(),
    status VARCHAR(20), -- ok, error, timeout
    response_time_ms INTEGER,
    error_message TEXT,
    details JSONB
);
```

---

#### 模組 3: ETL 作業管理 (ETL Job Management)
**目的**: 管理與監控所有資料管道作業

**功能清單**:
- **作業定義**
  - 作業名稱、描述、類型（Extract/Transform/Load）
  - 來源與目標資料源
  - 轉換邏輯配置（SQL/Python script）
  - 排程設定（Cron 表達式）
  - 依賴關係配置（DAG）
- **作業執行監控**
  - 執行狀態（queued/running/success/failed）
  - 執行歷史記錄
  - 執行時間統計（平均/最快/最慢）
  - 資料量統計（讀取/寫入/轉換行數）
- **作業控制**
  - 手動執行
  - 暫停/恢復排程
  - 強制終止
  - 重試失敗作業
- **批次管理**
  - Batch ID 追蹤（BATCH-YYYYMMDD-NNN）
  - 批次級別的成功/失敗統計
  - 批次回滾功能

**資料模型**:
```sql
CREATE TABLE etl_jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    job_type VARCHAR(50), -- extract, transform, load, full_pipeline
    source_datasource_id UUID REFERENCES datasources(id),
    target_datasource_id UUID REFERENCES datasources(id),
    schedule_cron VARCHAR(100), -- Cron 表達式
    is_enabled BOOLEAN DEFAULT true,
    config JSONB, -- 作業配置（SQL、Python script 等）
    dependencies UUID[], -- 依賴的其他 job IDs
    timeout_seconds INTEGER DEFAULT 3600,
    retry_count INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    tags TEXT[]
);

CREATE TABLE etl_job_runs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id UUID REFERENCES etl_jobs(id),
    batch_id VARCHAR(50), -- BATCH-YYYYMMDD-NNN
    status VARCHAR(20), -- queued, running, success, failed, timeout
    start_time TIMESTAMPTZ,
    end_time TIMESTAMPTZ,
    duration_seconds INTEGER,
    rows_read BIGINT,
    rows_written BIGINT,
    rows_error BIGINT,
    error_message TEXT,
    log_file_path TEXT,
    triggered_by VARCHAR(50), -- scheduler, manual, api
    execution_details JSONB
);

CREATE INDEX idx_job_runs_job_id ON etl_job_runs(job_id);
CREATE INDEX idx_job_runs_batch_id ON etl_job_runs(batch_id);
CREATE INDEX idx_job_runs_status ON etl_job_runs(status);
CREATE INDEX idx_job_runs_start_time ON etl_job_runs(start_time DESC);
```

---

#### 模組 4: 元數據管理 (Metadata Management)
**目的**: 管理資料字典、血緣關係、欄位對應

**功能清單**:
- **資料表註冊**
  - 自動掃描資料源發現資料表
  - 手動註冊資料表
  - 資料表描述（中英文名稱）
  - 業務分類標籤
- **欄位管理**
  - 自動同步欄位定義
  - 欄位中文名稱維護
  - 欄位描述與範例值
  - 資料類型與約束
- **血緣追蹤**
  - 資料表級別血緣（source → clean → mart）
  - 欄位級別血緣（欄位轉換對應）
  - 血緣圖可視化（DAG 圖）
- **欄位對應管理**
  - Source-to-Clean 對應規則
  - Clean-to-Mart 對應規則
  - 轉換邏輯說明
- **資料字典導出**
  - Excel/CSV 格式導出
  - HTML 格式報表
  - API 接口查詢

**UI 設計重點**:
- 樹狀結構顯示資料表層級
- 搜尋與篩選功能
- 血緣圖使用 D3.js 或 vis.js 繪製
- Inline editing 快速維護

**整合現有元數據**:
- 讀取 `clean._metadata_table`
- 讀取 `clean._metadata_column`
- 讀取 `clean._metadata_column_mapping`

---

#### 模組 5: 資料品質管理 (Data Quality Management)
**目的**: 監控與管理資料品質指標

**功能清單**:
- **品質規則配置**
  - 規則類型: NOT_NULL, UNIQUE, RANGE, PATTERN, CUSTOM_SQL
  - 規則嚴重性: CRITICAL, WARNING, INFO
  - 規則啟用/停用
  - 規則排程執行
- **品質檢查執行**
  - 手動執行檢查
  - 自動排程檢查
  - 批次檢查（多個規則）
- **品質報告**
  - 資料表品質評分（PERFECT/EXCELLENT/GOOD/POOR）
  - 欄位級別品質統計
  - 品質趨勢圖
  - 問題明細列表
- **異常處理**
  - 異常問題追蹤（OPEN/FIXED/IGNORED）
  - 問題優先級排序
  - 問題指派與處理
  - 修復記錄

**整合現有系統**:
- 讀取 `clean._metadata_quality_checks`
- 讀取 `clean._metadata_quality_issues`
- 讀取 `clean._metadata_validation_rules`
- 使用現有的 `check_metadata_quality()` 函數

**UI 設計重點**:
- 品質儀表板（總覽）
- 品質趨勢圖表（時間序列）
- 問題列表（可篩選、排序）
- 問題詳情頁（含修復建議）

---

#### 模組 6: 資料清理管理 (Data Cleansing Management)
**目的**: 管理與監控資料清理規則與執行結果

**功能清單**:
- **清理規則配置**
  - 4-level 清理層級（Format/Normalize/Infer/Custom）
  - 規則定義（正則表達式、轉換函數、自定義 SQL）
  - 規則優先級
  - 規則測試工具（預覽清理結果）
- **清理執行監控**
  - 清理作業執行歷史
  - 清理統計（總數/成功/失敗/跳過）
  - 清理前後對比
- **清理日誌查詢**
  - 按資料表/欄位查詢
  - 按批次查詢
  - 按清理類型查詢
  - 原始值與清理值對比
- **清理規則分析**
  - 最常使用的規則
  - 清理成功率統計
  - 規則效能分析

**整合現有系統**:
- 讀取 `clean._cleansing_log`
- 讀取 `clean._metadata_cleansing_rules`
- 使用現有的清理邏輯

---

#### 模組 7: 告警管理 (Alert Management)
**目的**: 即時告警與通知

**功能清單**:
- **告警規則配置**
  - 觸發條件（作業失敗、品質下降、連線中斷等）
  - 告警等級（CRITICAL/WARNING/INFO）
  - 通知方式（Email/Telegram/Slack/Webhook）
  - 通知對象（用戶/群組）
  - 靜默期設定（避免告警疲勞）
- **告警記錄**
  - 告警歷史
  - 告警狀態（NEW/ACK/RESOLVED）
  - 處理記錄
- **通知整合**
  - Telegram Bot 通知
  - Email 通知
  - Webhook 通知

**資料模型**:
```sql
CREATE TABLE alert_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    description TEXT,
    rule_type VARCHAR(50), -- job_failed, quality_degraded, connection_lost, etc.
    condition JSONB, -- 觸發條件
    severity VARCHAR(20), -- CRITICAL, WARNING, INFO
    notification_channels TEXT[], -- [telegram, email, slack]
    notification_targets TEXT[], -- 用戶 IDs 或 channel IDs
    is_enabled BOOLEAN DEFAULT true,
    silence_minutes INTEGER DEFAULT 0, -- 靜默期
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE alerts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rule_id UUID REFERENCES alert_rules(id),
    severity VARCHAR(20),
    title VARCHAR(200),
    message TEXT,
    status VARCHAR(20) DEFAULT 'NEW', -- NEW, ACK, RESOLVED
    triggered_at TIMESTAMPTZ DEFAULT NOW(),
    acknowledged_at TIMESTAMPTZ,
    resolved_at TIMESTAMPTZ,
    acknowledged_by VARCHAR(100),
    resolved_by VARCHAR(100),
    related_entity_type VARCHAR(50), -- job, datasource, table, etc.
    related_entity_id UUID,
    metadata JSONB
);
```

---

#### 模組 8: 報表中心 (Report Center)
**目的**: 生成與管理各類報表

**功能清單**:
- **預設報表範本**
  - 每日 ETL 執行報表
  - 每週資料品質報表
  - 每月資料量統計報表
  - 元數據完整性報表
- **自定義報表**
  - 報表建構器（拖拉式）
  - SQL 查詢編輯器
  - 參數化報表
- **報表排程**
  - 自動生成報表
  - 郵件發送
  - 存檔管理
- **報表導出**
  - PDF 格式
  - Excel 格式
  - HTML 格式
  - Markdown 格式（整合 GitHub）

---

#### 模組 9: 系統設定 (System Settings)
**目的**: 系統配置與權限管理

**功能清單**:
- **使用者管理**
  - 使用者帳號管理
  - 角色定義（Admin/Manager/Viewer）
  - 權限控制（RBAC）
- **系統配置**
  - SMTP 設定（郵件通知）
  - Telegram Bot 設定
  - 資料保留政策
  - 日誌級別設定
- **API 管理**
  - API Token 生成與管理
  - API 使用量監控
  - API 文件（Swagger/OpenAPI）
- **稽核日誌**
  - 使用者操作記錄
  - 系統變更記錄
  - 登入日誌

---

## 3. 技術架構

### 3.1 系統架構圖

```
┌─────────────────────────────────────────────────────────────┐
│                        使用者介面層                            │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐             │
│  │ Web 前端   │  │ Mobile 響應 │  │ API 客戶端  │             │
│  │ (Vue.js 3) │  │    設計     │  │            │             │
│  └────────────┘  └────────────┘  └────────────┘             │
└───────────────────────────┬─────────────────────────────────┘
                            │ HTTPS/WebSocket
┌───────────────────────────┴─────────────────────────────────┐
│                      應用服務層 (API Layer)                   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │          Node.js / Express.js REST API               │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │ 認證/授權  │  作業調度   │  資料驗證  │  告警服務      │   │
│  │ (JWT)     │  (Bull/Cron)│           │  (Telegram)   │   │
│  └──────────────────────────────────────────────────────┘   │
└───────────────────────────┬─────────────────────────────────┘
                            │ Database Connection Pool
┌───────────────────────────┴─────────────────────────────────┐
│                       資料持久層                              │
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────┐  │
│  │  PostgreSQL    │  │  Redis Cache   │  │ Message Queue│  │
│  │  (AWS RDS)     │  │  (Session/Job) │  │  (Bull/RabbitMQ)││
│  └────────────────┘  └────────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            │
┌───────────────────────────┴─────────────────────────────────┐
│                      資料源連接層                             │
│  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐           │
│  │ ERP DB │  │ WMS DB │  │ MES DB │  │ REST   │           │
│  │(PG/源)  │  │        │  │        │  │ APIs   │           │
│  └────────┘  └────────┘  └────────┘  └────────┘           │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 技術棧選型

#### 前端技術棧
- **框架**: Vue.js 3 (Composition API)
- **UI 組件庫**: Ant Design Vue / Element Plus
- **狀態管理**: Pinia
- **路由**: Vue Router 4
- **圖表庫**: 
  - ECharts (資料圖表)
  - D3.js (血緣圖、DAG 圖)
- **HTTP 客戶端**: Axios
- **表單驗證**: VeeValidate / Yup
- **即時通訊**: Socket.io-client (WebSocket)
- **程式碼編輯器**: Monaco Editor (SQL 編輯)

#### 後端技術棧
- **運行環境**: Node.js 18+ LTS
- **Web 框架**: Express.js 4
- **ORM**: Prisma (推薦) 或 Sequelize
- **驗證授權**: 
  - JWT (JSON Web Token)
  - bcrypt (密碼加密)
- **排程工具**: 
  - node-cron (簡單排程)
  - Bull (複雜作業佇列，基於 Redis)
- **日誌**: Winston
- **API 文件**: Swagger / OpenAPI 3.0
- **WebSocket**: Socket.io
- **郵件**: Nodemailer
- **檔案上傳**: Multer

#### 資料庫
- **主資料庫**: PostgreSQL 14+ (AWS RDS)
  - 應用層資料庫（新建 `dgp` database）
  - 監控現有的 `source` database
- **快取**: Redis 6+
  - Session 儲存
  - 作業佇列
  - 即時狀態快取
- **訊息佇列**: Bull (基於 Redis) 或 RabbitMQ

#### DevOps & 部署
- **容器化**: Docker + Docker Compose
- **反向代理**: Nginx
- **CI/CD**: GitHub Actions
- **監控**: 
  - Prometheus + Grafana (系統監控)
  - PM2 (Node.js 程序管理)
- **日誌收集**: ELK Stack (可選)

---

## 4. 資料庫設計

### 4.1 資料庫架構

**新建資料庫**: `dgp` (Data Governance Platform)

**Schema 設計**:
- `public`: 應用層資料表（使用者、權限、配置等）
- 與現有 `source` 資料庫的 `clean` schema 整合（讀取元數據）

### 4.2 核心資料表（已在各模組中定義）

參考各模組的資料模型設計：
- `datasources` - 資料源註冊
- `etl_jobs` & `etl_job_runs` - ETL 作業管理
- `alert_rules` & `alerts` - 告警管理
- 其他支援表...

### 4.3 與現有系統整合

**讀取現有元數據**（`source` database, `clean` schema）:
- `clean._metadata_table` → 資料表註冊資訊
- `clean._metadata_column` → 欄位定義
- `clean._metadata_column_mapping` → 血緣對應
- `clean._cleansing_log` → 清理日誌
- `clean._metadata_quality_checks` → 品質檢查記錄
- `clean._metadata_quality_issues` → 品質問題

**資料同步策略**:
- 使用 Foreign Data Wrapper (FDW) 或 dblink 跨資料庫查詢
- 或透過 API 層統一查詢介面

---

## 5. API 設計

### 5.1 RESTful API 風格

**基礎 URL**: `https://api.dgp.example.com/v1`

**通用回應格式**:
```json
{
  "success": true,
  "data": { ... },
  "message": "Operation successful",
  "timestamp": "2026-02-13T06:00:00Z"
}
```

**錯誤回應格式**:
```json
{
  "success": false,
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "Job not found",
    "details": { "jobId": "xxx" }
  },
  "timestamp": "2026-02-13T06:00:00Z"
}
```

### 5.2 主要 API 端點

#### 認證相關
```
POST   /auth/login           # 使用者登入
POST   /auth/logout          # 使用者登出
POST   /auth/refresh-token   # 刷新 JWT Token
GET    /auth/me              # 取得當前使用者資訊
```

#### 資料源管理
```
GET    /datasources                    # 列出所有資料源
POST   /datasources                    # 新增資料源
GET    /datasources/:id                # 取得資料源詳情
PUT    /datasources/:id                # 更新資料源
DELETE /datasources/:id                # 刪除資料源
POST   /datasources/:id/test           # 測試連線
GET    /datasources/:id/health         # 健康檢查
GET    /datasources/:id/health/history # 健康歷史
```

#### ETL 作業管理
```
GET    /jobs                      # 列出所有作業
POST   /jobs                      # 新增作業
GET    /jobs/:id                  # 取得作業詳情
PUT    /jobs/:id                  # 更新作業
DELETE /jobs/:id                  # 刪除作業
POST   /jobs/:id/run              # 手動執行作業
POST   /jobs/:id/stop             # 停止作業
GET    /jobs/:id/runs             # 作業執行歷史
GET    /jobs/:id/runs/:runId      # 作業執行詳情
POST   /jobs/:id/runs/:runId/retry # 重試作業
GET    /jobs/:id/dependencies     # 作業依賴關係
```

#### 元數據管理
```
GET    /metadata/tables              # 列出所有資料表
GET    /metadata/tables/:tableId     # 資料表詳情
PUT    /metadata/tables/:tableId     # 更新資料表元數據
GET    /metadata/tables/:tableId/columns # 資料表欄位列表
PUT    /metadata/columns/:columnId   # 更新欄位元數據
GET    /metadata/lineage/:tableId    # 資料血緣
POST   /metadata/scan                # 掃描資料源並同步元數據
GET    /metadata/dictionary          # 資料字典導出
```

#### 資料品質管理
```
GET    /quality/checks                    # 品質檢查列表
POST   /quality/checks                    # 新增檢查規則
GET    /quality/checks/:id                # 檢查詳情
POST   /quality/checks/:id/run            # 執行品質檢查
GET    /quality/checks/:id/history        # 檢查歷史
GET    /quality/issues                    # 品質問題列表
PUT    /quality/issues/:id                # 更新問題狀態
GET    /quality/dashboard                 # 品質儀表板資料
GET    /quality/trend                     # 品質趨勢
```

#### 資料清理管理
```
GET    /cleansing/rules                  # 清理規則列表
POST   /cleansing/rules                  # 新增清理規則
GET    /cleansing/rules/:id              # 規則詳情
PUT    /cleansing/rules/:id              # 更新規則
DELETE /cleansing/rules/:id              # 刪除規則
POST   /cleansing/rules/:id/test         # 測試規則
GET    /cleansing/logs                   # 清理日誌查詢
GET    /cleansing/statistics             # 清理統計
```

#### 告警管理
```
GET    /alerts                       # 告警列表
POST   /alerts                       # 新增告警規則
GET    /alerts/:id                   # 告警詳情
PUT    /alerts/:id/acknowledge       # 確認告警
PUT    /alerts/:id/resolve           # 解決告警
GET    /alert-rules                  # 告警規則列表
POST   /alert-rules                  # 新增告警規則
PUT    /alert-rules/:id              # 更新告警規則
DELETE /alert-rules/:id              # 刪除告警規則
```

#### 報表中心
```
GET    /reports                      # 報表列表
POST   /reports                      # 新增報表定義
GET    /reports/:id                  # 報表詳情
POST   /reports/:id/generate         # 生成報表
GET    /reports/:id/download         # 下載報表
GET    /reports/templates            # 報表範本
```

#### 儀表板
```
GET    /dashboard/overview           # 系統概覽
GET    /dashboard/etl-trend          # ETL 趨勢
GET    /dashboard/quality-trend      # 品質趨勢
GET    /dashboard/alerts             # 最新告警
GET    /dashboard/statistics         # 統計資料
```

### 5.3 WebSocket 事件

**即時更新事件**:
```javascript
// 客戶端訂閱
socket.on('job:status', (data) => {
  // { jobId, status, progress }
});

socket.on('alert:new', (data) => {
  // { alertId, severity, message }
});

socket.on('quality:check:complete', (data) => {
  // { checkId, result }
});

// 伺服器推送
io.emit('job:status', { jobId: 'xxx', status: 'running', progress: 50 });
```

---

## 6. UI/UX 設計規範

### 6.1 設計原則
- **清晰**: 資訊層級分明，避免過載
- **一致**: 統一的色彩、圖示、交互模式
- **即時**: 關鍵指標即時更新
- **可操作**: 從查看到操作的快速路徑

### 6.2 色彩編碼

**狀態顏色**:
- 🟢 **成功/正常**: `#52c41a` (綠色)
- 🟡 **警告**: `#faad14` (橙色)
- 🔴 **錯誤/嚴重**: `#f5222d` (紅色)
- 🔵 **執行中**: `#1890ff` (藍色)
- ⚪ **未啟用/停止**: `#d9d9d9` (灰色)

**品質評級顏色**:
- PERFECT (100%): 深綠 `#237804`
- EXCELLENT (≥95%): 綠色 `#52c41a`
- GOOD (≥80%): 橙色 `#faad14`
- POOR (<80%): 紅色 `#f5222d`

### 6.3 主要頁面佈局

#### 儀表板 (Dashboard)
```
┌─────────────────────────────────────────────────────────┐
│  📊 系統概覽                                              │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐      │
│  │資料源    │ │ETL 作業  │ │品質評分  │ │今日資料量│      │
│  │ 12/15   │ │ 45/50   │ │  95.2%  │ │ 1.2M    │      │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘      │
├─────────────────────────────────────────────────────────┤
│  📈 即時監控                                              │
│  ┌───────────────────────┐ ┌──────────────────────┐    │
│  │ ETL 執行趨勢 (24h)     │ │ 品質趨勢 (7天)        │    │
│  │   [折線圖]             │ │   [折線圖]            │    │
│  └───────────────────────┘ └──────────────────────┘    │
├─────────────────────────────────────────────────────────┤
│  🚨 告警中心                  📋 最近執行                │
│  ┌────────────────────┐    ┌──────────────────────┐  │
│  │ [告警列表]          │    │ [作業執行列表]        │  │
│  └────────────────────┘    └──────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

#### ETL 作業管理
```
┌─────────────────────────────────────────────────────────┐
│  ⚙️ ETL 作業管理                          [+ 新增作業]    │
├─────────────────────────────────────────────────────────┤
│  🔍 [搜尋] [篩選: 全部▼] [狀態: 全部▼] [排序: 最近更新▼] │
├─────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────┐  │
│  │ 📦 material_master_sync              🟢 啟用      │  │
│  │ ERP 物料主檔同步 | 每日 02:00                     │  │
│  │ 最後執行: 2小時前 ✅ 成功 | 1,348 rows           │  │
│  │ [執行] [編輯] [查看記錄] [⋮]                      │  │
│  └──────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────┐  │
│  │ 📦 vendor_master_sync                🔴 失敗      │  │
│  │ ERP 廠商主檔同步 | 每日 02:30                     │  │
│  │ 最後執行: 30分鐘前 ❌ 失敗 | Timeout              │  │
│  │ [重試] [編輯] [查看記錄] [⋮]                      │  │
│  └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

#### 資料品質管理
```
┌─────────────────────────────────────────────────────────┐
│  ✅ 資料品質管理                                          │
├─────────────────────────────────────────────────────────┤
│  📊 品質總覽                                              │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐         │
│  │ 整體評分    │ │ 未解決問題  │ │ 檢查次數    │         │
│  │   95.2%    │ │    12      │ │   150      │         │
│  │  EXCELLENT │ │  3🔴 9🟡   │ │  (本週)    │         │
│  └────────────┘ └────────────┘ └────────────┘         │
├─────────────────────────────────────────────────────────┤
│  📈 品質趨勢 (7天)                                        │
│  [折線圖: 各資料表品質分數變化]                           │
├─────────────────────────────────────────────────────────┤
│  🔍 品質問題                                              │
│  [嚴重性: 全部▼] [狀態: 未解決▼] [資料表: 全部▼]         │
│  ┌──────────────────────────────────────────────────┐  │
│  │ 🔴 CRITICAL | vendor_master.tax_number           │  │
│  │ 空值比例過高 (15.2%)                               │  │
│  │ 影響: 9,972 rows | 發現於: 2小時前                │  │
│  │ [查看詳情] [標記為已處理]                          │  │
│  └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### 6.4 響應式設計
- **桌面端** (≥1200px): 完整功能，多欄佈局
- **平板** (768-1199px): 調整佈局，保留核心功能
- **手機** (≤767px): 精簡版，優先顯示監控與告警

---

## 7. 安全性設計

### 7.1 認證與授權
- **JWT Token**: 有效期 24 小時，支援 Refresh Token
- **角色權限**:
  - **Admin**: 完整權限
  - **Manager**: 可管理作業、規則，無系統設定權限
  - **Viewer**: 唯讀權限
- **API Token**: 用於外部系統整合，可設定到期時間與權限範圍

### 7.2 資料安全
- **敏感資料加密**: 資料庫連線密碼使用 AES-256 加密
- **HTTPS 強制**: 所有 API 通訊必須使用 HTTPS
- **SQL 注入防護**: 使用 ORM/參數化查詢
- **XSS 防護**: 前端輸入驗證與轉義
- **CSRF 防護**: CSRF Token 驗證

### 7.3 稽核日誌
- 記錄所有關鍵操作（新增/修改/刪除作業、修改規則等）
- 日誌包含: 使用者、時間、操作類型、IP、變更前後對比

---

## 8. 效能優化

### 8.1 資料庫優化
- **索引**: 為常用查詢欄位建立索引
- **分區**: 大表（如 `etl_job_runs`）使用時間分區
- **連線池**: 使用連線池管理資料庫連線
- **查詢優化**: 避免 N+1 查詢，使用 JOIN 或批次查詢

### 8.2 快取策略
- **Redis 快取**: 
  - 資料源狀態（TTL: 5 分鐘）
  - 元數據（TTL: 1 小時）
  - 儀表板統計（TTL: 5 分鐘）
- **HTTP 快取**: 靜態資源使用瀏覽器快取

### 8.3 非同步處理
- **作業執行**: 使用 Bull 佇列非同步執行
- **大量資料匯出**: 非同步生成，完成後通知使用者

---

## 9. 開發計畫

### 9.1 開發階段

#### Phase 1: 基礎架構 (2 週)
**目標**: 建立專案骨架與核心功能

**任務**:
- [x] 專案初始化（前後端）
- [x] 資料庫設計與建立
- [x] 認證授權系統
- [x] 基礎 UI 框架（佈局、導航、主題）
- [x] 資料源管理（CRUD + 連線測試）

**交付物**:
- 可運行的專案骨架
- 使用者可登入並管理資料源

---

#### Phase 2: ETL 作業管理 (2 週)
**目標**: 實現 ETL 作業的管理與監控

**任務**:
- [x] ETL 作業 CRUD
- [x] 作業排程整合（node-cron）
- [x] 作業執行引擎（Bull Queue）
- [x] 執行歷史與日誌
- [x] 作業監控頁面
- [x] 儀表板基礎版（ETL 執行統計）

**交付物**:
- ETL 作業可配置與執行
- 執行狀態可視化

---

#### Phase 3: 元數據 & 品質管理 (2 週)
**目標**: 整合現有元數據系統與品質檢查

**任務**:
- [x] 元數據掃描與同步
- [x] 資料表/欄位管理頁面
- [x] 血緣追蹤可視化（D3.js DAG）
- [x] 品質檢查規則管理
- [x] 品質檢查執行與結果展示
- [x] 品質儀表板

**交付物**:
- 元數據可視化瀏覽與維護
- 品質問題可追蹤

---

#### Phase 4: 資料清理 & 告警 (1.5 週)
**目標**: 資料清理管理與告警系統

**任務**:
- [x] 清理規則管理頁面
- [x] 清理日誌查詢與分析
- [x] 告警規則配置
- [x] 告警記錄與處理
- [x] Telegram 通知整合

**交付物**:
- 清理規則可配置與監控
- 告警可即時推送

---

#### Phase 5: 報表 & 進階功能 (1.5 週)
**目標**: 報表中心與系統完善

**任務**:
- [x] 報表範本與生成
- [x] 報表排程與匯出
- [x] API 文件（Swagger）
- [x] 系統設定頁面
- [x] 使用者權限管理
- [x] 稽核日誌

**交付物**:
- 完整的報表功能
- 系統管理功能完善

---

#### Phase 6: 測試 & 部署 (1 週)
**目標**: 測試、優化與正式部署

**任務**:
- [x] 單元測試
- [x] 整合測試
- [x] 效能測試與優化
- [x] Docker 容器化
- [x] CI/CD 流程建立
- [x] 部署文件撰寫
- [x] 使用者手冊

**交付物**:
- 生產環境就緒的系統
- 完整文件

---

### 9.2 時程規劃

**總開發時間**: 10 週（約 2.5 個月）

```
Week 1-2:  Phase 1 - 基礎架構
Week 3-4:  Phase 2 - ETL 作業管理
Week 5-6:  Phase 3 - 元數據 & 品質管理
Week 7-8:  Phase 4 - 資料清理 & 告警
Week 9:    Phase 5 - 報表 & 進階功能
Week 10:   Phase 6 - 測試 & 部署
```

---

## 10. 風險評估

### 10.1 技術風險

| 風險 | 嚴重性 | 可能性 | 緩解措施 |
|------|--------|--------|----------|
| 跨資料庫查詢效能問題 | 高 | 中 | 使用 FDW 或快取，優化查詢 |
| 大量資料處理效能瓶頸 | 中 | 高 | 非同步處理、分批處理 |
| WebSocket 連線穩定性 | 中 | 中 | 心跳機制、自動重連 |
| 資料源連線失敗處理 | 中 | 高 | 重試機制、告警通知 |

### 10.2 專案風險

| 風險 | 嚴重性 | 可能性 | 緩解措施 |
|------|--------|--------|----------|
| 需求變更頻繁 | 中 | 高 | 敏捷開發、模組化設計 |
| 時程延誤 | 中 | 中 | 優先核心功能、分階段交付 |
| 資源不足 | 高 | 低 | 使用成熟框架、減少自行開發 |

---

## 11. 成功指標

### 11.1 功能性指標
- ✅ 支援至少 5 種資料源類型
- ✅ 同時管理 100+ ETL 作業
- ✅ 元數據覆蓋率 100%
- ✅ 品質檢查可自動化執行

### 11.2 效能指標
- ⚡ 頁面載入時間 < 2 秒
- ⚡ API 回應時間 < 500ms (95th percentile)
- ⚡ 支援 1000+ 並發連線
- ⚡ 資料同步延遲 < 5 分鐘

### 11.3 可用性指標
- 🔒 系統可用性 ≥ 99.5%
- 🔒 告警回應時間 < 1 分鐘
- 🔒 資料備份每日執行

---

## 12. 維護計畫

### 12.1 日常維護
- **資料庫備份**: 每日自動備份
- **日誌清理**: 保留 90 天
- **效能監控**: Prometheus + Grafana
- **安全更新**: 每月檢查依賴套件更新

### 12.2 未來擴展
- **機器學習整合**: 異常檢測、品質預測
- **更多資料源**: Snowflake, BigQuery, MongoDB 等
- **資料目錄**: 類似 DataHub 的企業級資料目錄
- **自助式 BI**: 整合 Superset 或 Metabase

---

## 13. 附錄

### 13.1 詞彙表

| 術語 | 說明 |
|------|------|
| Medallion Architecture | 分層資料架構（Bronze/Silver/Gold 或 Source/Clean/Mart） |
| Data Lineage | 資料血緣，追蹤資料的來源與流向 |
| ETL | Extract, Transform, Load（資料抽取、轉換、載入） |
| Data Quality | 資料品質，衡量資料的準確性、完整性、一致性等 |
| Data Governance | 資料治理，管理資料資產的政策、流程與標準 |

### 13.2 參考資料
- [Airflow 文件](https://airflow.apache.org/docs/)
- [Great Expectations](https://greatexpectations.io/) - 資料品質檢查工具
- [DataHub](https://datahubproject.io/) - 開源資料目錄
- [dbt (data build tool)](https://www.getdbt.com/) - 資料轉換工具

---

## 14. 結論

本規格書定義了一個功能完整、架構清晰的**資料治理管理平台**，涵蓋：

✅ **核心功能**: 資料源管理、ETL 作業、元數據、品質監控、資料清理、告警、報表  
✅ **技術架構**: Vue.js 3 + Node.js/Express + PostgreSQL + Redis  
✅ **開發計畫**: 6 個階段，10 週完成  
✅ **擴展性**: 模組化設計，易於擴展新功能  

**下一步行動**:
1. 審核規格書並確認需求
2. 準備開發環境（資料庫、伺服器等）
3. 啟動 Phase 1 開發

---

**文件版本管理**:
- v1.0 (2026-02-13): 初版發布

**聯絡人**:
- PM: [待補]
- Tech Lead: [待補]

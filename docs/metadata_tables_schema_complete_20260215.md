# Metadata Tables Schema - 完整表結構說明

**日期**: 2026-02-15  
**資料庫**: source  
**涵蓋範圍**: clean schema 7 張表 + public schema 2 張表

---

## 目錄

### Clean Schema (7 張表)
1. [_metadata_table](#1-_metadata_table) - 資料表註冊
2. [_metadata_column](#2-_metadata_column) - 欄位定義
3. [_metadata_column_mapping](#3-_metadata_column_mapping) - 欄位對應關係
4. [_cleansing_log](#4-_cleansing_log) - 資料清理日誌
5. [_metadata_cleansing_rules](#5-_metadata_cleansing_rules) - 清理規則
6. [_metadata_validation_rules](#6-_metadata_validation_rules) - 驗證規則
7. [_metadata_quality_checks](#7-_metadata_quality_checks) - 品質檢查記錄

### Public Schema (2 張表)
8. [md_table](#8-md_table) - 資料表元數據（來源系統）
9. [md_column](#9-md_column) - 欄位元數據（來源系統）

---

## Clean Schema

### 1. _metadata_table

**用途**: 資料表註冊與基本資訊管理

**表結構**:
```sql
CREATE TABLE clean._metadata_table (
    table_id         INTEGER PRIMARY KEY DEFAULT nextval('clean.md_table_table_id_seq'),
    table_name       TEXT NOT NULL UNIQUE,
    table_zh_name    TEXT,
    table_desc       TEXT,
    source_tables    TEXT[],
    source_systems   TEXT[],
    owner_team       TEXT,
    update_frequency TEXT,
    is_active        BOOLEAN DEFAULT TRUE,
    row_count        BIGINT,
    created_at       TIMESTAMP DEFAULT NOW(),
    updated_at       TIMESTAMP DEFAULT NOW()
);
```

**欄位說明**:

| 欄位名稱 | 資料型別 | 必填 | 預設值 | 說明 |
|---------|---------|------|--------|------|
| table_id | INTEGER | ✅ | 自動序號 | 資料表唯一識別碼（主鍵） |
| table_name | TEXT | ✅ | - | 資料表名稱（唯一約束） |
| table_zh_name | TEXT | ❌ | - | 資料表中文名稱 |
| table_desc | TEXT | ❌ | - | 資料表描述說明 |
| source_tables | TEXT[] | ❌ | - | 來源資料表陣列 |
| source_systems | TEXT[] | ❌ | - | 來源系統陣列 |
| owner_team | TEXT | ❌ | - | 負責團隊 |
| update_frequency | TEXT | ❌ | - | 更新頻率（如：每日、每週） |
| is_active | BOOLEAN | ❌ | TRUE | 是否啟用 |
| row_count | BIGINT | ❌ | - | 資料筆數 |
| created_at | TIMESTAMP | ❌ | NOW() | 建立時間 |
| updated_at | TIMESTAMP | ❌ | NOW() | 最後更新時間 |

**索引**:
- PRIMARY KEY: `table_id`
- UNIQUE: `table_name`

**使用範例**:
```sql
-- 查詢所有啟用的資料表
SELECT table_name, table_zh_name, row_count
FROM clean._metadata_table
WHERE is_active = TRUE
ORDER BY table_name;

-- 查詢特定資料表資訊
SELECT *
FROM clean._metadata_table
WHERE table_name = 'vendor_master';
```

---

### 2. _metadata_column

**用途**: 欄位定義與屬性管理

**表結構**:
```sql
CREATE TABLE clean._metadata_column (
    column_id         INTEGER PRIMARY KEY DEFAULT nextval('clean.md_column_column_id_seq'),
    table_name        TEXT NOT NULL,
    column_name       TEXT NOT NULL,
    column_zh_name    TEXT,
    column_desc       TEXT,
    data_type         TEXT,
    is_pk             BOOLEAN DEFAULT FALSE,
    is_nullable       BOOLEAN DEFAULT TRUE,
    default_value     TEXT,
    ordinal_position  INTEGER,
    is_active         BOOLEAN DEFAULT TRUE,
    created_at        TIMESTAMP DEFAULT NOW(),
    updated_at        TIMESTAMP DEFAULT NOW(),
    is_foreign_key    BOOLEAN DEFAULT FALSE,
    referenced_table  VARCHAR(100),
    referenced_column VARCHAR(100),
    fk_relationship   TEXT,
    UNIQUE (table_name, column_name)
);
```

**欄位說明**:

| 欄位名稱 | 資料型別 | 必填 | 預設值 | 說明 |
|---------|---------|------|--------|------|
| column_id | INTEGER | ✅ | 自動序號 | 欄位唯一識別碼（主鍵） |
| table_name | TEXT | ✅ | - | 所屬資料表名稱 |
| column_name | TEXT | ✅ | - | 欄位名稱 |
| column_zh_name | TEXT | ❌ | - | 欄位中文名稱 |
| column_desc | TEXT | ❌ | - | 欄位描述說明 |
| data_type | TEXT | ❌ | - | 資料型別（如：VARCHAR, INTEGER） |
| is_pk | BOOLEAN | ❌ | FALSE | 是否為主鍵 |
| is_nullable | BOOLEAN | ❌ | TRUE | 是否可為 NULL |
| default_value | TEXT | ❌ | - | 預設值 |
| ordinal_position | INTEGER | ❌ | - | 欄位順序位置 |
| is_active | BOOLEAN | ❌ | TRUE | 是否啟用 |
| created_at | TIMESTAMP | ❌ | NOW() | 建立時間 |
| updated_at | TIMESTAMP | ❌ | NOW() | 最後更新時間 |
| is_foreign_key | BOOLEAN | ❌ | FALSE | 是否為外鍵 |
| referenced_table | VARCHAR(100) | ❌ | - | 參考的資料表名稱 |
| referenced_column | VARCHAR(100) | ❌ | - | 參考的欄位名稱 |
| fk_relationship | TEXT | ❌ | - | 外鍵關係說明 |

**索引**:
- PRIMARY KEY: `column_id`
- UNIQUE: `(table_name, column_name)`

**使用範例**:
```sql
-- 查詢特定資料表的所有欄位
SELECT 
    column_name,
    column_zh_name,
    data_type,
    is_pk,
    is_nullable
FROM clean._metadata_column
WHERE table_name = 'vendor_master'
ORDER BY ordinal_position;

-- 查詢所有主鍵欄位
SELECT table_name, column_name, column_zh_name
FROM clean._metadata_column
WHERE is_pk = TRUE;

-- 查詢所有外鍵關係
SELECT 
    table_name,
    column_name,
    referenced_table,
    referenced_column,
    fk_relationship
FROM clean._metadata_column
WHERE is_foreign_key = TRUE;
```

---

### 3. _metadata_column_mapping

**用途**: 欄位對應關係與轉換規則

**表結構**:
```sql
CREATE TABLE clean._metadata_column_mapping (
    mapping_id     INTEGER PRIMARY KEY DEFAULT nextval('clean.md_column_mapping_mapping_id_seq'),
    target_table   TEXT NOT NULL,
    target_column  TEXT NOT NULL,
    source_system  TEXT,
    source_table   TEXT,
    source_column  TEXT,
    transform_rule TEXT,
    priority       INTEGER DEFAULT 1,
    is_active      BOOLEAN DEFAULT TRUE,
    created_at     TIMESTAMP DEFAULT NOW()
);
```

**欄位說明**:

| 欄位名稱 | 資料型別 | 必填 | 預設值 | 說明 |
|---------|---------|------|--------|------|
| mapping_id | INTEGER | ✅ | 自動序號 | 對應關係唯一識別碼（主鍵） |
| target_table | TEXT | ✅ | - | 目標資料表名稱（clean layer） |
| target_column | TEXT | ✅ | - | 目標欄位名稱 |
| source_system | TEXT | ❌ | - | 來源系統名稱（如：ERP, WMS） |
| source_table | TEXT | ❌ | - | 來源資料表名稱 |
| source_column | TEXT | ❌ | - | 來源欄位名稱 |
| transform_rule | TEXT | ❌ | - | 轉換規則（SQL 表達式或說明） |
| priority | INTEGER | ❌ | 1 | 優先級（數字越小優先級越高） |
| is_active | BOOLEAN | ❌ | TRUE | 是否啟用 |
| created_at | TIMESTAMP | ❌ | NOW() | 建立時間 |

**索引**:
- PRIMARY KEY: `mapping_id`

**使用範例**:
```sql
-- 查詢 vendor_master 的所有欄位對應
SELECT 
    target_column,
    source_table,
    source_column,
    transform_rule
FROM clean._metadata_column_mapping
WHERE target_table = 'vendor_master'
ORDER BY priority;

-- 查詢特定欄位的血緣關係（上游）
SELECT 
    source_system,
    source_table,
    source_column,
    transform_rule
FROM clean._metadata_column_mapping
WHERE target_table = 'vendor_master'
  AND target_column = 'tax_number';

-- 查詢特定來源表的下游對應
SELECT 
    target_table,
    target_column,
    transform_rule
FROM clean._metadata_column_mapping
WHERE source_table = 'erp_m2201_pmc_file'
ORDER BY target_table, target_column;
```

---

### 4. _cleansing_log

**用途**: 資料清理過程的日誌記錄

**表結構**:
```sql
CREATE TABLE clean._cleansing_log (
    id             BIGINT PRIMARY KEY DEFAULT nextval('clean._cleansing_log_id_seq'),
    table_name     VARCHAR(100) NOT NULL,
    record_id      VARCHAR(200) NOT NULL,
    column_name    VARCHAR(100) NOT NULL,
    source_system  VARCHAR(50) NOT NULL,
    original_value TEXT,
    cleaned_value  TEXT,
    data_type      VARCHAR(50),
    fix_level      SMALLINT NOT NULL,
    fix_type       VARCHAR(50) NOT NULL,
    fix_rule       VARCHAR(500),
    is_significant BOOLEAN DEFAULT FALSE,
    batch_id       VARCHAR(50) NOT NULL,
    created_at     TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_cleansing_log_batch ON clean._cleansing_log(batch_id);
CREATE INDEX idx_cleansing_log_significant ON clean._cleansing_log(is_significant, created_at);
CREATE INDEX idx_cleansing_log_table_record ON clean._cleansing_log(table_name, record_id);
```

**欄位說明**:

| 欄位名稱 | 資料型別 | 必填 | 預設值 | 說明 |
|---------|---------|------|--------|------|
| id | BIGINT | ✅ | 自動序號 | 日誌唯一識別碼（主鍵） |
| table_name | VARCHAR(100) | ✅ | - | 資料表名稱 |
| record_id | VARCHAR(200) | ✅ | - | 記錄識別碼（主鍵值） |
| column_name | VARCHAR(100) | ✅ | - | 欄位名稱 |
| source_system | VARCHAR(50) | ✅ | - | 來源系統 |
| original_value | TEXT | ❌ | - | 原始值（清理前） |
| cleaned_value | TEXT | ❌ | - | 清理後的值 |
| data_type | VARCHAR(50) | ❌ | - | 資料型別 |
| fix_level | SMALLINT | ✅ | - | 清理層級（1-4）<br>1: Format, 2: Normalize, 3: Infer, 4: Custom |
| fix_type | VARCHAR(50) | ✅ | - | 清理類型（如：TRIM, UPPER, FILL_NULL） |
| fix_rule | VARCHAR(500) | ❌ | - | 清理規則說明 |
| is_significant | BOOLEAN | ❌ | FALSE | 是否為重大變更 |
| batch_id | VARCHAR(50) | ✅ | - | 批次識別碼（格式：BATCH-YYYYMMDD-NNN） |
| created_at | TIMESTAMP | ❌ | CURRENT_TIMESTAMP | 建立時間 |

**索引**:
- PRIMARY KEY: `id`
- INDEX: `idx_cleansing_log_batch` (batch_id)
- INDEX: `idx_cleansing_log_significant` (is_significant, created_at)
- INDEX: `idx_cleansing_log_table_record` (table_name, record_id)

**使用範例**:
```sql
-- 查詢特定批次的清理日誌
SELECT 
    table_name,
    column_name,
    fix_type,
    COUNT(*) as fix_count
FROM clean._cleansing_log
WHERE batch_id = 'BATCH-20260213-001'
GROUP BY table_name, column_name, fix_type
ORDER BY fix_count DESC;

-- 查詢重大變更記錄
SELECT 
    table_name,
    record_id,
    column_name,
    original_value,
    cleaned_value,
    fix_type,
    created_at
FROM clean._cleansing_log
WHERE is_significant = TRUE
ORDER BY created_at DESC
LIMIT 100;

-- 統計清理模式（AI 學習用）
SELECT 
    table_name,
    column_name,
    fix_level,
    fix_type,
    COUNT(*) as frequency
FROM clean._cleansing_log
WHERE created_at >= NOW() - INTERVAL '90 days'
GROUP BY table_name, column_name, fix_level, fix_type
HAVING COUNT(*) > 10
ORDER BY frequency DESC;
```

---

### 5. _metadata_cleansing_rules

**用途**: 資料清理規則定義

**表結構**:
```sql
CREATE TABLE clean._metadata_cleansing_rules (
    rule_id         INTEGER PRIMARY KEY DEFAULT nextval('clean._metadata_cleansing_rules_rule_id_seq'),
    table_name      VARCHAR(100) NOT NULL,
    column_name     VARCHAR(100) NOT NULL,
    rule_name       VARCHAR(100) NOT NULL,
    rule_desc       TEXT,
    rule_type       VARCHAR(50) NOT NULL,
    rule_expression TEXT NOT NULL,
    rule_priority   INTEGER DEFAULT 1,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (table_name, column_name, rule_name)
);
```

**欄位說明**:

| 欄位名稱 | 資料型別 | 必填 | 預設值 | 說明 |
|---------|---------|------|--------|------|
| rule_id | INTEGER | ✅ | 自動序號 | 規則唯一識別碼（主鍵） |
| table_name | VARCHAR(100) | ✅ | - | 資料表名稱 |
| column_name | VARCHAR(100) | ✅ | - | 欄位名稱 |
| rule_name | VARCHAR(100) | ✅ | - | 規則名稱 |
| rule_desc | TEXT | ❌ | - | 規則描述 |
| rule_type | VARCHAR(50) | ✅ | - | 規則類型（如：REGEX, SQL, FUNCTION） |
| rule_expression | TEXT | ✅ | - | 規則表達式（正則表達式或 SQL） |
| rule_priority | INTEGER | ❌ | 1 | 規則優先級（數字越小優先級越高） |
| is_active | BOOLEAN | ❌ | TRUE | 是否啟用 |
| created_at | TIMESTAMP | ❌ | CURRENT_TIMESTAMP | 建立時間 |
| updated_at | TIMESTAMP | ❌ | CURRENT_TIMESTAMP | 最後更新時間 |

**索引**:
- PRIMARY KEY: `rule_id`
- UNIQUE: `(table_name, column_name, rule_name)`

**規則類型說明**:
- `REGEX`: 正則表達式替換
- `SQL`: SQL 表達式轉換
- `FUNCTION`: 自定義函數
- `LOOKUP`: 查表替換
- `DEFAULT`: 預設值填充

**使用範例**:
```sql
-- 查詢 vendor_master 的所有清理規則
SELECT 
    column_name,
    rule_name,
    rule_type,
    rule_expression,
    rule_priority
FROM clean._metadata_cleansing_rules
WHERE table_name = 'vendor_master'
  AND is_active = TRUE
ORDER BY column_name, rule_priority;

-- 新增清理規則
INSERT INTO clean._metadata_cleansing_rules (
    table_name, column_name, rule_name, 
    rule_desc, rule_type, rule_expression
) VALUES (
    'vendor_master', 'tax_number', 'format_tax_number',
    '統一編號格式化：移除非數字字元',
    'REGEX', 'REGEXP_REPLACE(tax_number, ''[^0-9]'', '''', ''g'')'
);

-- 停用特定規則
UPDATE clean._metadata_cleansing_rules
SET is_active = FALSE, updated_at = NOW()
WHERE rule_id = 123;
```

---

### 6. _metadata_validation_rules

**用途**: 資料驗證規則定義

**表結構**:
```sql
CREATE TABLE clean._metadata_validation_rules (
    rule_id         INTEGER PRIMARY KEY DEFAULT nextval('clean._metadata_validation_rules_rule_id_seq'),
    table_name      VARCHAR(100) NOT NULL,
    column_name     VARCHAR(100),
    rule_name       VARCHAR(100) NOT NULL,
    rule_desc       TEXT,
    rule_type       VARCHAR(50) NOT NULL,
    rule_expression TEXT NOT NULL,
    error_level     VARCHAR(20) DEFAULT 'ERROR',
    error_message   TEXT,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (table_name, column_name, rule_name)
);
```

**欄位說明**:

| 欄位名稱 | 資料型別 | 必填 | 預設值 | 說明 |
|---------|---------|------|--------|------|
| rule_id | INTEGER | ✅ | 自動序號 | 規則唯一識別碼（主鍵） |
| table_name | VARCHAR(100) | ✅ | - | 資料表名稱 |
| column_name | VARCHAR(100) | ❌ | - | 欄位名稱（NULL 表示表級規則） |
| rule_name | VARCHAR(100) | ✅ | - | 規則名稱 |
| rule_desc | TEXT | ❌ | - | 規則描述 |
| rule_type | VARCHAR(50) | ✅ | - | 規則類型（NOT_NULL, UNIQUE, RANGE, PATTERN, CUSTOM_SQL） |
| rule_expression | TEXT | ✅ | - | 規則表達式 |
| error_level | VARCHAR(20) | ❌ | 'ERROR' | 錯誤等級（ERROR, WARNING, INFO） |
| error_message | TEXT | ❌ | - | 錯誤訊息範本 |
| is_active | BOOLEAN | ❌ | TRUE | 是否啟用 |
| created_at | TIMESTAMP | ❌ | CURRENT_TIMESTAMP | 建立時間 |
| updated_at | TIMESTAMP | ❌ | CURRENT_TIMESTAMP | 最後更新時間 |

**索引**:
- PRIMARY KEY: `rule_id`
- UNIQUE: `(table_name, column_name, rule_name)`

**規則類型說明**:
- `NOT_NULL`: 非空檢查
- `UNIQUE`: 唯一性檢查
- `RANGE`: 數值範圍檢查
- `PATTERN`: 格式檢查（正則表達式）
- `ENUM`: 枚舉值檢查
- `CUSTOM_SQL`: 自定義 SQL 檢查
- `FK_INTEGRITY`: 外鍵完整性檢查

**使用範例**:
```sql
-- 查詢 vendor_master 的所有驗證規則
SELECT 
    column_name,
    rule_name,
    rule_type,
    error_level,
    rule_expression
FROM clean._metadata_validation_rules
WHERE table_name = 'vendor_master'
  AND is_active = TRUE
ORDER BY column_name, error_level;

-- 新增驗證規則
INSERT INTO clean._metadata_validation_rules (
    table_name, column_name, rule_name, 
    rule_desc, rule_type, rule_expression, 
    error_level, error_message
) VALUES (
    'vendor_master', 'tax_number', 'tax_number_format',
    '統一編號必須為 8 碼數字',
    'PATTERN', '^[0-9]{8}$',
    'ERROR', '統一編號格式錯誤：{value}'
);

-- 執行驗證並記錄問題
WITH validation_check AS (
    SELECT 
        vendor_id,
        tax_number,
        CASE 
            WHEN tax_number !~ '^[0-9]{8}$' THEN 'FAIL'
            ELSE 'PASS'
        END as check_result
    FROM clean.vendor_master
)
SELECT vendor_id, tax_number
FROM validation_check
WHERE check_result = 'FAIL';
```

---

### 7. _metadata_quality_checks

**用途**: 資料品質檢查記錄

**表結構**:
```sql
CREATE TABLE clean._metadata_quality_checks (
    check_id           INTEGER PRIMARY KEY DEFAULT nextval('clean._metadata_quality_checks_check_id_seq'),
    check_date         TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    check_type         VARCHAR(50),
    total_tables       INTEGER,
    total_columns      INTEGER,
    registered_columns INTEGER,
    missing_columns    INTEGER,
    orphan_metadata    INTEGER,
    registration_rate  NUMERIC(5,2),
    quality_grade      VARCHAR(10),
    critical_issues    INTEGER DEFAULT 0,
    warning_issues     INTEGER DEFAULT 0,
    check_status       VARCHAR(20) DEFAULT 'COMPLETED',
    error_message      TEXT,
    execution_time_ms  INTEGER,
    checked_by         VARCHAR(50) DEFAULT 'SYSTEM'
);

CREATE INDEX idx_quality_checks_date ON clean._metadata_quality_checks(check_date);
CREATE INDEX idx_quality_checks_grade ON clean._metadata_quality_checks(quality_grade);
```

**欄位說明**:

| 欄位名稱 | 資料型別 | 必填 | 預設值 | 說明 |
|---------|---------|------|--------|------|
| check_id | INTEGER | ✅ | 自動序號 | 檢查唯一識別碼（主鍵） |
| check_date | TIMESTAMP | ❌ | CURRENT_TIMESTAMP | 檢查時間 |
| check_type | VARCHAR(50) | ❌ | - | 檢查類型（METADATA, DATA_QUALITY, COMPLETENESS） |
| total_tables | INTEGER | ❌ | - | 檢查的資料表總數 |
| total_columns | INTEGER | ❌ | - | 檢查的欄位總數 |
| registered_columns | INTEGER | ❌ | - | 已註冊的欄位數 |
| missing_columns | INTEGER | ❌ | - | 未註冊的欄位數 |
| orphan_metadata | INTEGER | ❌ | - | 孤立的元數據數量 |
| registration_rate | NUMERIC(5,2) | ❌ | - | 註冊率（%） |
| quality_grade | VARCHAR(10) | ❌ | - | 品質評級（PERFECT, EXCELLENT, GOOD, POOR） |
| critical_issues | INTEGER | ❌ | 0 | 嚴重問題數量 |
| warning_issues | INTEGER | ❌ | 0 | 警告問題數量 |
| check_status | VARCHAR(20) | ❌ | 'COMPLETED' | 檢查狀態（RUNNING, COMPLETED, FAILED） |
| error_message | TEXT | ❌ | - | 錯誤訊息（如果失敗） |
| execution_time_ms | INTEGER | ❌ | - | 執行時間（毫秒） |
| checked_by | VARCHAR(50) | ❌ | 'SYSTEM' | 執行者 |

**索引**:
- PRIMARY KEY: `check_id`
- INDEX: `idx_quality_checks_date` (check_date)
- INDEX: `idx_quality_checks_grade` (quality_grade)

**品質評級標準**:
- `PERFECT`: 100%
- `EXCELLENT`: ≥ 95%
- `GOOD`: ≥ 80%
- `POOR`: < 80%

**使用範例**:
```sql
-- 查詢最近 10 次品質檢查記錄
SELECT 
    check_id,
    check_date,
    total_tables,
    registration_rate,
    quality_grade,
    critical_issues,
    warning_issues
FROM clean._metadata_quality_checks
ORDER BY check_date DESC
LIMIT 10;

-- 查詢品質趨勢（最近 30 天）
SELECT 
    DATE(check_date) as check_date,
    AVG(registration_rate) as avg_registration_rate,
    MAX(quality_grade) as best_grade,
    SUM(critical_issues) as total_critical
FROM clean._metadata_quality_checks
WHERE check_date >= NOW() - INTERVAL '30 days'
GROUP BY DATE(check_date)
ORDER BY check_date;

-- 執行品質檢查（插入新記錄）
INSERT INTO clean._metadata_quality_checks (
    check_type,
    total_tables,
    total_columns,
    registered_columns,
    registration_rate,
    quality_grade,
    critical_issues,
    warning_issues,
    execution_time_ms
) VALUES (
    'METADATA',
    7,
    167,
    167,
    100.00,
    'PERFECT',
    0,
    0,
    1250
);
```

---

## Public Schema

### 8. md_table

**用途**: 來源系統的資料表元數據（ETL 原始定義）

**表結構**:
```sql
CREATE TABLE public.md_table (
    src_system        VARCHAR(50) NOT NULL,
    src_table_name    VARCHAR(128) NOT NULL,
    target_table_name VARCHAR(128),
    table_zh_name     VARCHAR(200),
    table_desc        VARCHAR(500),
    owner_team        VARCHAR(100),
    effective_from    DATE DEFAULT CURRENT_DATE,
    effective_to      DATE DEFAULT '9999-12-31',
    is_active         SMALLINT DEFAULT 1,
    created_at        TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at        TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (src_system, src_table_name)
);
```

**欄位說明**:

| 欄位名稱 | 資料型別 | 必填 | 預設值 | 說明 |
|---------|---------|------|--------|------|
| src_system | VARCHAR(50) | ✅ | - | 來源系統名稱（如：ERP, WMS, MES）<br>**主鍵之一** |
| src_table_name | VARCHAR(128) | ✅ | - | 來源資料表名稱<br>**主鍵之一** |
| target_table_name | VARCHAR(128) | ❌ | - | 目標資料表名稱（clean layer） |
| table_zh_name | VARCHAR(200) | ❌ | - | 資料表中文名稱 |
| table_desc | VARCHAR(500) | ❌ | - | 資料表描述 |
| owner_team | VARCHAR(100) | ❌ | - | 負責團隊 |
| effective_from | DATE | ❌ | CURRENT_DATE | 生效開始日期 |
| effective_to | DATE | ❌ | '9999-12-31' | 生效結束日期（9999-12-31 表示永久有效） |
| is_active | SMALLINT | ❌ | 1 | 是否啟用（1=是, 0=否） |
| created_at | TIMESTAMP | ❌ | CURRENT_TIMESTAMP | 建立時間 |
| updated_at | TIMESTAMP | ❌ | CURRENT_TIMESTAMP | 最後更新時間 |

**索引**:
- PRIMARY KEY: `(src_system, src_table_name)`

**使用範例**:
```sql
-- 查詢所有 ERP 系統的資料表
SELECT 
    src_table_name,
    target_table_name,
    table_zh_name,
    table_desc
FROM public.md_table
WHERE src_system = 'ERP'
  AND is_active = 1
ORDER BY src_table_name;

-- 查詢特定來源表的對應目標表
SELECT 
    src_system,
    src_table_name,
    target_table_name,
    table_zh_name
FROM public.md_table
WHERE src_table_name = 'erp_m2201_pmc_file';

-- 查詢有效的資料表對應
SELECT *
FROM public.md_table
WHERE CURRENT_DATE BETWEEN effective_from AND effective_to
  AND is_active = 1;
```

---

### 9. md_column

**用途**: 來源系統的欄位元數據（ETL 原始定義）

**表結構**:
```sql
CREATE TABLE public.md_column (
    src_system          VARCHAR(50) NOT NULL,
    src_table_name      VARCHAR(128) NOT NULL,
    column_name         VARCHAR(128) NOT NULL,
    column_zh_name      VARCHAR(200),
    business_definition VARCHAR(1000),
    data_type           VARCHAR(50),
    nullable_flag       SMALLINT DEFAULT 1,
    is_pk               SMALLINT DEFAULT 0,
    ordinal_position    INTEGER,
    domain_name         VARCHAR(100),
    example_values      TEXT,
    notes               VARCHAR(1000),
    version             VARCHAR(30) DEFAULT 'v1',
    effective_from      DATE DEFAULT CURRENT_DATE,
    effective_to        DATE DEFAULT '9999-12-31',
    is_active           SMALLINT DEFAULT 1,
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (src_system, src_table_name, column_name)
);
```

**欄位說明**:

| 欄位名稱 | 資料型別 | 必填 | 預設值 | 說明 |
|---------|---------|------|--------|------|
| src_system | VARCHAR(50) | ✅ | - | 來源系統名稱<br>**主鍵之一** |
| src_table_name | VARCHAR(128) | ✅ | - | 來源資料表名稱<br>**主鍵之一** |
| column_name | VARCHAR(128) | ✅ | - | 欄位名稱<br>**主鍵之一** |
| column_zh_name | VARCHAR(200) | ❌ | - | 欄位中文名稱 |
| business_definition | VARCHAR(1000) | ❌ | - | 業務定義說明 |
| data_type | VARCHAR(50) | ❌ | - | 資料型別 |
| nullable_flag | SMALLINT | ❌ | 1 | 是否可為空（1=可, 0=不可） |
| is_pk | SMALLINT | ❌ | 0 | 是否為主鍵（1=是, 0=否） |
| ordinal_position | INTEGER | ❌ | - | 欄位順序位置 |
| domain_name | VARCHAR(100) | ❌ | - | 領域名稱（業務分類） |
| example_values | TEXT | ❌ | - | 範例值 |
| notes | VARCHAR(1000) | ❌ | - | 備註說明 |
| version | VARCHAR(30) | ❌ | 'v1' | 版本號 |
| effective_from | DATE | ❌ | CURRENT_DATE | 生效開始日期 |
| effective_to | DATE | ❌ | '9999-12-31' | 生效結束日期 |
| is_active | SMALLINT | ❌ | 1 | 是否啟用（1=是, 0=否） |
| created_at | TIMESTAMP | ❌ | CURRENT_TIMESTAMP | 建立時間 |
| updated_at | TIMESTAMP | ❌ | CURRENT_TIMESTAMP | 最後更新時間 |

**索引**:
- PRIMARY KEY: `(src_system, src_table_name, column_name)`

**使用範例**:
```sql
-- 查詢特定來源表的所有欄位
SELECT 
    column_name,
    column_zh_name,
    data_type,
    nullable_flag,
    is_pk,
    business_definition
FROM public.md_column
WHERE src_system = 'ERP'
  AND src_table_name = 'erp_m2201_pmc_file'
  AND is_active = 1
ORDER BY ordinal_position;

-- 查詢所有主鍵欄位
SELECT 
    src_system,
    src_table_name,
    column_name,
    column_zh_name
FROM public.md_column
WHERE is_pk = 1
  AND is_active = 1
ORDER BY src_system, src_table_name;

-- 查詢特定領域的欄位
SELECT 
    src_table_name,
    column_name,
    column_zh_name,
    business_definition
FROM public.md_column
WHERE src_system = 'ERP'
  AND domain_name = '客戶管理'
  AND is_active = 1;

-- 查詢有效的欄位定義（含版本控制）
SELECT *
FROM public.md_column
WHERE src_system = 'ERP'
  AND src_table_name = 'erp_m2201_pmc_file'
  AND CURRENT_DATE BETWEEN effective_from AND effective_to
  AND is_active = 1
ORDER BY ordinal_position;
```

---

## 表關係圖

```
public.md_table (來源表定義)
       ↓
public.md_column (來源欄位定義)
       ↓
clean._metadata_column_mapping (欄位對應關係)
       ↓
clean._metadata_table (目標表註冊)
       ↓
clean._metadata_column (目標欄位定義)
       ↓
clean._metadata_cleansing_rules (清理規則)
       ↓
clean._cleansing_log (清理日誌)
       ↓
clean._metadata_validation_rules (驗證規則)
       ↓
clean._metadata_quality_checks (品質檢查)
```

---

## 使用建議

### 1. 元數據管理流程

```sql
-- Step 1: 從 public 查詢來源定義
SELECT * FROM public.md_table WHERE src_system = 'ERP';

-- Step 2: 註冊到 clean._metadata_table
INSERT INTO clean._metadata_table (table_name, table_zh_name, ...)
SELECT target_table_name, table_zh_name, ...
FROM public.md_table WHERE ...;

-- Step 3: 註冊欄位到 clean._metadata_column
INSERT INTO clean._metadata_column (table_name, column_name, ...)
SELECT ..., FROM public.md_column WHERE ...;

-- Step 4: 建立欄位對應關係
INSERT INTO clean._metadata_column_mapping (...)
VALUES (...);

-- Step 5: 定義清理規則
INSERT INTO clean._metadata_cleansing_rules (...)
VALUES (...);

-- Step 6: 定義驗證規則
INSERT INTO clean._metadata_validation_rules (...)
VALUES (...);
```

### 2. 品質檢查流程

```sql
-- Step 1: 執行品質檢查（自定義函數或腳本）
SELECT check_metadata_quality();

-- Step 2: 查看檢查結果
SELECT * FROM clean._metadata_quality_checks
ORDER BY check_date DESC LIMIT 1;

-- Step 3: 查看問題詳情
SELECT * FROM clean._metadata_quality_issues
WHERE check_id = (
    SELECT check_id FROM clean._metadata_quality_checks
    ORDER BY check_date DESC LIMIT 1
);
```

### 3. 資料血緣追蹤

```sql
-- 查詢完整血緣（上游到下游）
WITH RECURSIVE lineage AS (
    -- 起點：clean layer 某個欄位
    SELECT 
        1 as level,
        target_table,
        target_column,
        source_table,
        source_column,
        transform_rule
    FROM clean._metadata_column_mapping
    WHERE target_table = 'vendor_master'
      AND target_column = 'tax_number'
    
    UNION ALL
    
    -- 遞迴：找上游
    SELECT 
        l.level + 1,
        m.target_table,
        m.target_column,
        m.source_table,
        m.source_column,
        m.transform_rule
    FROM clean._metadata_column_mapping m
    JOIN lineage l ON m.target_table = l.source_table
    WHERE l.level < 5
)
SELECT * FROM lineage
ORDER BY level;
```

---

## 附錄

### A. 資料型別對應

| PostgreSQL | 說明 | 範例 |
|-----------|------|------|
| TEXT | 不限長度文字 | 'Hello' |
| VARCHAR(n) | 限定長度文字 | 'Test' |
| INTEGER | 整數 | 123 |
| BIGINT | 大整數 | 9223372036854775807 |
| SMALLINT | 小整數 | 32767 |
| NUMERIC(p,s) | 精確數值 | 123.45 |
| BOOLEAN | 布林值 | TRUE/FALSE |
| TIMESTAMP | 時間戳記 | '2026-02-15 12:00:00' |
| DATE | 日期 | '2026-02-15' |
| TEXT[] | 文字陣列 | ARRAY['a','b','c'] |

### B. 命名規範

**表命名**:
- 元數據表: `_metadata_*`（底線開頭）
- 業務表: 小寫英文，底線分隔

**欄位命名**:
- 英文: 小寫，底線分隔
- 主鍵: `*_id`
- 時間戳記: `*_at`
- 布林: `is_*` 或 `has_*`

**約束命名**:
- 主鍵: `{table_name}_pkey`
- 外鍵: `{table_name}_{column_name}_fkey`
- 唯一: `{table_name}_{column_name}_key`
- 索引: `idx_{table_name}_{column_name}`

---

**文件版本**: 1.0  
**最後更新**: 2026-02-15  
**作者**: meimei

**END OF DOCUMENT**

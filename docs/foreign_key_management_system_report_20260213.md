# 外鍵關係管理系統建置報告
**建立日期:** 2026-02-13  
**系統狀態:** ✅ 已部署並運行  
**首次檢查結果:** 🏆 所有關係 100% 完美

---

## 📋 系統概述

### 功能
自動識別、追蹤和檢查資料表之間的外鍵關係（包含實際約束和邏輯關聯），確保參照完整性。

### 核心組件
1. **_metadata_foreign_keys** - 外鍵關係記錄表
2. **check_fk_integrity()** - 參照完整性檢查函數
3. **3 個便捷視圖** - 快速查詢關係資訊
4. **_metadata_column 擴充** - 欄位標記外鍵屬性

---

## 🗄️ 資料表結構

### _metadata_foreign_keys (外鍵關係表)

**主要欄位：**
```sql
fk_id                -- 關係ID (主鍵)
source_table         -- 來源表格
source_column        -- 來源欄位
referenced_table     -- 參照表格
referenced_column    -- 參照欄位
constraint_type      -- 'ACTUAL' 或 'LOGICAL'
relationship_name    -- 關係名稱
orphan_count         -- 孤兒記錄數量
integrity_rate       -- 完整性比率 (%)
last_check_date      -- 最後檢查時間
```

**關係類型說明：**
- **ACTUAL (實際約束)**: 資料庫層級的 FOREIGN KEY 約束
- **LOGICAL (邏輯關聯)**: 業務上的參照關係，但沒有建立資料庫約束

---

## 📊 當前外鍵關係

### 關係總覽
| 來源欄位 | 參照表格 | 類型 | 完整性 | 狀態 |
|---------|---------|------|--------|------|
| material_cost.material_id | material_master | 🔒 實際約束 | 100% | ✅ 完美 |
| _metadata_quality_issues.check_id | _metadata_quality_checks | 🔒 實際約束 | 100% | ✅ 完美 |
| vendor_master.default_tax_code | tax_code_master | 📋 邏輯關聯 | 100% | ✅ 完美 |

### 表格關聯統計
| 來源表格 | 參照表格數 | 參照哪些表格 | 外鍵數量 | 實際約束 | 邏輯關聯 |
|---------|-----------|------------|---------|---------|---------|
| material_cost | 1 | material_master | 1 | 1 | 0 |
| _metadata_quality_issues | 1 | _metadata_quality_checks | 1 | 1 | 0 |
| vendor_master | 1 | tax_code_master | 1 | 0 | 1 |

---

## 🔍 外鍵關係詳解

### 1. material_cost → material_master
**關係描述：** 料件成本必須對應存在的料件

```
material_cost.material_id (VARCHAR(50)) 
    ↓ 🔒 FOREIGN KEY 約束
material_master.material_id (VARCHAR(50))
```

**約束詳情：**
- **約束名稱：** `fk_material_cost_material`
- **約束類型：** ACTUAL (實際資料庫約束)
- **完整性：** 100% (0 個孤兒記錄 / 1,050 筆)
- **業務意義：** 確保所有成本記錄都對應到有效的料件

---

### 2. vendor_master → tax_code_master
**關係描述：** 供應商的慣用稅別必須是有效的稅別代碼

```
vendor_master.default_tax_code (VARCHAR(20))
    ↓ 📋 邏輯關聯
tax_code_master.tax_code (VARCHAR(20))
```

**約束詳情：**
- **約束類型：** LOGICAL (邏輯參照，無資料庫約束)
- **完整性：** 100% (0 個孤兒記錄 / 9,972 筆)
- **業務意義：** 供應商的預設稅別必須在稅別主檔中存在

**為何不建立實際約束？**
- 允許業務彈性（稅別可能需要歷史保留）
- 避免刪除稅別時連鎖影響
- 透過檢查函數定期監控即可

---

### 3. _metadata_quality_issues → _metadata_quality_checks
**關係描述：** 品質問題必須關聯到品質檢查記錄

```
_metadata_quality_issues.check_id (INTEGER)
    ↓ 🔒 FOREIGN KEY 約束
_metadata_quality_checks.check_id (INTEGER)
```

**約束詳情：**
- **約束名稱：** `_metadata_quality_issues_check_id_fkey`
- **約束類型：** ACTUAL (實際資料庫約束)
- **完整性：** 100% (0 個孤兒記錄 / 8 筆)
- **業務意義：** 確保每個品質問題都有對應的檢查記錄

---

## 🔧 核心功能

### 1. check_fk_integrity() - 檢查參照完整性

**功能：** 檢查所有外鍵關係的參照完整性，識別孤兒記錄。

**使用方法：**
```sql
SELECT * FROM clean.check_fk_integrity();
```

**輸出範例：**
```
來源欄位                          | 參照表格        | 孤兒數 | 總記錄數 | 完整性  | 狀態
---------------------------------|----------------|--------|----------|---------|--------
material_cost.material_id         | material_master|      0 |     1050 | 100.00% | ✅ 完美
vendor_master.default_tax_code    | tax_code_master|      0 |     9972 | 100.00% | ✅ 完美
```

**狀態說明：**
- **✅ 完美** (100%): 所有記錄都有有效參照
- **🟢 良好** (95-99%): 少量孤兒記錄
- **🟡 待改善** (80-94%): 明顯的參照問題
- **🔴 嚴重** (<80%): 大量孤兒記錄，需立即處理

---

### 2. _metadata_column 外鍵標記

外鍵欄位在 `_metadata_column` 中會被標記：

**新增欄位：**
- `is_foreign_key` - 是否為外鍵 (BOOLEAN)
- `referenced_table` - 參照的表格
- `referenced_column` - 參照的欄位
- `fk_relationship` - 關係說明

**查詢外鍵欄位：**
```sql
SELECT 
    table_name,
    column_name,
    referenced_table,
    fk_relationship
FROM clean._metadata_column
WHERE is_foreign_key = true;
```

---

## 📊 便捷視圖

### 1. v_foreign_key_relationships - 外鍵關係總覽

**用途：** 查看所有外鍵關係及其完整性狀態。

```sql
SELECT * FROM clean.v_foreign_key_relationships;
```

**輸出欄位：**
- 來源 (table.column)
- 參照 (table.column)
- 類型 (ACTUAL/LOGICAL)
- 關係名稱
- 約束狀態 (🔒實際約束 / 📋邏輯關聯)
- 完整性 (%)
- 品質狀態 (✅/🟡/🔴)
- 孤兒記錄數
- 最後檢查時間

---

### 2. v_table_relationships - 表格關聯統計

**用途：** 查看每個表格參照了哪些其他表格。

```sql
SELECT * FROM clean.v_table_relationships;
```

**輸出欄位：**
- 來源表格
- 參照表格數
- 參照哪些表格 (逗號分隔)
- 外鍵數量
- 實際約束數量
- 邏輯關聯數量

---

### 3. v_foreign_key_columns - 外鍵欄位列表

**用途：** 查看所有被標記為外鍵的欄位。

```sql
SELECT * FROM clean.v_foreign_key_columns;
```

**輸出欄位：**
- 表格
- 欄位
- 中文名稱
- 參照 (table.column)
- 關係說明
- 資料型態

---

## 💼 使用場景

### 場景 1：每日參照完整性檢查

```sql
-- 執行檢查
SELECT * FROM clean.check_fk_integrity();

-- 如果發現問題，查看詳情
SELECT * FROM clean.v_foreign_key_relationships 
WHERE 品質 != '✅';
```

---

### 場景 2：新增外鍵關係

當發現新的業務參照關係時：

```sql
-- 新增邏輯外鍵
INSERT INTO clean._metadata_foreign_keys (
    source_table, source_column,
    referenced_table, referenced_column,
    constraint_type, relationship_name, relationship_desc
) VALUES (
    'your_table', 'your_column',
    'ref_table', 'ref_column',
    'LOGICAL', '關係名稱', '關係說明'
);

-- 更新 metadata 標記
UPDATE clean._metadata_column mc
SET 
    is_foreign_key = true,
    referenced_table = 'ref_table',
    referenced_column = 'ref_column',
    fk_relationship = '關係名稱'
WHERE mc.table_name = 'your_table'
  AND mc.column_name = 'your_column';

-- 立即檢查完整性
SELECT * FROM clean.check_fk_integrity();
```

---

### 場景 3：處理孤兒記錄

發現孤兒記錄時的處理方式：

```sql
-- 1. 識別孤兒記錄
SELECT s.* 
FROM clean.vendor_master s
WHERE s.default_tax_code IS NOT NULL
  AND NOT EXISTS (
    SELECT 1 FROM clean.tax_code_master r 
    WHERE r.tax_code = s.default_tax_code
  );

-- 2. 決定處理方式：

-- 方案 A: 新增缺少的參照資料
INSERT INTO clean.tax_code_master (...) VALUES (...);

-- 方案 B: 設為 NULL
UPDATE clean.vendor_master 
SET default_tax_code = NULL
WHERE default_tax_code NOT IN (
    SELECT tax_code FROM clean.tax_code_master
);

-- 方案 C: 設為預設值
UPDATE clean.vendor_master 
SET default_tax_code = 'DEFAULT_CODE'
WHERE default_tax_code NOT IN (
    SELECT tax_code FROM clean.tax_code_master
);

-- 3. 重新檢查
SELECT * FROM clean.check_fk_integrity();
```

---

### 場景 4：建立實際約束

將邏輯關聯升級為實際約束：

```sql
-- 1. 先確保沒有孤兒記錄
SELECT * FROM clean.check_fk_integrity() 
WHERE 來源欄位 = 'vendor_master.default_tax_code';

-- 2. 如果完整性是 100%，建立實際約束
ALTER TABLE clean.vendor_master
ADD CONSTRAINT fk_vendor_tax_code
FOREIGN KEY (default_tax_code) 
REFERENCES clean.tax_code_master(tax_code);

-- 3. 更新關係表
UPDATE clean._metadata_foreign_keys
SET constraint_type = 'ACTUAL',
    constraint_name = 'fk_vendor_tax_code',
    enforce_integrity = true
WHERE source_table = 'vendor_master'
  AND source_column = 'default_tax_code';
```

---

## 📈 品質指標說明

### 參照完整性比率
```
完整性 = (有效參照記錄數 / 總非空記錄數) × 100%
```

**目標：**
- **目標值：** 100% (無孤兒記錄)
- **警戒線：** 95%
- **危險線：** 80%

### 孤兒記錄 (Orphan Records)
指來源欄位有值，但在參照表格中找不到對應記錄的資料。

**影響：**
- 資料一致性問題
- 查詢 JOIN 時會遺失資料
- 業務邏輯錯誤

---

## 🎯 首次檢查結果

**檢查時間：** 2026-02-13 04:24:46  
**檢查關係數：** 3 個

### 檢查結果
| 來源欄位 | 參照表格 | 總記錄數 | 孤兒數 | 完整性 | 狀態 |
|---------|---------|----------|--------|--------|------|
| material_cost.material_id | material_master | 1,050 | 0 | 100% | ✅ |
| vendor_master.default_tax_code | tax_code_master | 9,972 | 0 | 100% | ✅ |
| _metadata_quality_issues.check_id | _metadata_quality_checks | 8 | 0 | 100% | ✅ |

**總結：** 🏆 **所有外鍵關係 100% 完美！**

---

## 🔄 與其他系統整合

### 1. 整合到品質檢查系統

可以將外鍵檢查納入每日品質檢查：

```sql
-- 在 run_quality_check() 後執行
SELECT clean.run_quality_check();
SELECT * FROM clean.check_fk_integrity();
```

### 2. 自動化執行

**使用 cron 定期檢查：**
```sql
-- PostgreSQL pg_cron
SELECT cron.schedule('fk-integrity-check',
    '0 9 * * *',  -- 每天早上9點
    'SELECT clean.check_fk_integrity();'
);
```

---

## ⚠️ 常見問題處理

### Q1: 發現孤兒記錄怎麼辦？

**步驟：**
1. 識別孤兒記錄數量和具體資料
2. 判斷原因（資料錯誤？參照資料被刪除？）
3. 選擇處理方式（修正/刪除/設NULL/新增參照資料）
4. 執行修正
5. 重新檢查

---

### Q2: 如何新增更多外鍵關係？

**步驟：**
1. 識別業務上的參照關係
2. 插入到 `_metadata_foreign_keys`
3. 更新 `_metadata_column` 標記
4. 執行 `check_fk_integrity()` 驗證

**範例：** 如果發現 material_master 有供應商欄位參照 vendor_master

```sql
INSERT INTO clean._metadata_foreign_keys (
    source_table, source_column,
    referenced_table, referenced_column,
    constraint_type, relationship_name
) VALUES (
    'material_master', 'supplier_code',
    'vendor_master', 'vendor_id',
    'LOGICAL', '料件的供應商'
);
```

---

### Q3: 實際約束 vs 邏輯關聯，何時使用？

**使用實際約束 (ACTUAL) 當：**
- ✅ 參照完整性必須嚴格保證
- ✅ 資料不會有歷史遺留問題
- ✅ 刪除參照資料時可接受限制或連鎖刪除

**使用邏輯關聯 (LOGICAL) 當：**
- ✅ 需要業務彈性
- ✅ 歷史資料可能參照已刪除的記錄
- ✅ 透過定期檢查維護即可

---

## 📋 維護建議

### 每日維護
- ✅ 執行 `check_fk_integrity()` 檢查
- ✅ 確認所有關係維持高完整性 (95%+)
- ✅ 處理新發現的孤兒記錄

### 每週維護
- ✅ 檢視是否有新的表格需要建立關聯
- ✅ 評估邏輯關聯是否應升級為實際約束
- ✅ 檢查 _metadata_column 的外鍵標記是否完整

### 每月維護
- ✅ 產生外鍵關係圖文檔
- ✅ 檢視孤兒記錄趨勢
- ✅ 優化檢查流程

---

## 🎉 總結

**系統狀態：** ✅ 已成功部署  
**當前關係數：** 3 個  
**參照完整性：** 🏆 100% 完美  

**系統優勢：**
- 🔍 自動識別實際約束和邏輯關聯
- 📊 定期檢查參照完整性
- 🏷️ 在 metadata 中清楚標記外鍵
- 📈 視圖便捷，關係一目了然
- ⚡ 孤兒記錄立即發現

**業務價值：**
- 確保資料一致性
- 避免 JOIN 遺失資料
- 清楚展示表格關聯
- 支援資料治理

**下一步：**
1. ⏳ 識別更多業務參照關係
2. ⏳ 整合到自動化檢查流程
3. ⏳ 建立表格關聯圖視覺化

---

**文檔版本：** 1.0  
**最後更新：** 2026-02-13  
**維護者：** meimei 🤖

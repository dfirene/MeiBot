# Metadata 自動品質檢查系統使用手冊
**建立日期:** 2026-02-13  
**系統狀態:** ✅ 已部署並運行  
**首次檢查結果:** 🏆 100% 完美

---

## 📋 系統概述

### 功能
自動化檢查 metadata 註冊完整性，確保所有業務表格的欄位都正確註冊到 `_metadata_column`。

### 核心組件
1. **品質檢查表** - 儲存歷史檢查記錄
2. **問題明細表** - 追蹤發現的品質問題
3. **檢查函數** - 自動執行品質檢查
4. **問題偵測函數** - 識別具體問題
5. **便捷視圖** - 快速查詢當前狀態

---

## 🗄️ 資料表結構

### 1. _metadata_quality_checks (品質檢查記錄表)
儲存每次品質檢查的結果。

**主要欄位：**
```sql
check_id              -- 檢查ID (主鍵)
check_date            -- 檢查時間
check_type            -- 檢查類型
total_tables          -- 總表格數
total_columns         -- 總欄位數
registered_columns    -- 已註冊欄位數
missing_columns       -- 未註冊欄位數
registration_rate     -- 註冊率 (%)
quality_grade         -- 品質等級 (PERFECT/EXCELLENT/GOOD/POOR)
critical_issues       -- 嚴重問題數
execution_time_ms     -- 執行時間 (毫秒)
```

### 2. _metadata_quality_issues (品質問題明細表)
儲存檢測到的具體問題。

**主要欄位：**
```sql
issue_id              -- 問題ID (主鍵)
check_id              -- 關聯的檢查ID
issue_type            -- 問題類型 (MISSING_COLUMN/ORPHAN_METADATA)
severity              -- 嚴重程度 (CRITICAL/WARNING/INFO)
table_name            -- 問題表格
column_name           -- 問題欄位
issue_description     -- 問題描述
status                -- 處理狀態 (OPEN/FIXED/IGNORED)
fixed_date            -- 修復時間
```

---

## 🔧 核心函數

### 1. run_quality_check() - 執行品質檢查

**功能：** 統計所有業務表格的 metadata 註冊情況並記錄結果。

**使用方法：**
```sql
SELECT * FROM clean.run_quality_check();
```

**返回結果：**
```
 check_id | tables | columns | registered | missing | rate  | grade
----------+--------+---------+------------+---------+-------+---------
        3 |      7 |     181 |        181 |       0 | 100.00| PERFECT
```

**欄位說明：**
- `check_id`: 本次檢查的ID
- `tables`: 檢查的表格數量
- `columns`: 總欄位數
- `registered`: 已註冊欄位數
- `missing`: 未註冊欄位數
- `rate`: 註冊完整率 (%)
- `grade`: 品質等級

**品質等級標準：**
- **PERFECT** (100%): 🏆 所有欄位已註冊
- **EXCELLENT** (95-99%): 🟢 優秀
- **GOOD** (80-94%): 🟡 良好
- **POOR** (<80%): 🔴 待改善

---

### 2. detect_metadata_issues() - 偵測問題

**功能：** 識別具體的metadata問題並記錄到問題表。

**使用方法：**
```sql
-- 偵測最新檢查的問題
SELECT clean.detect_metadata_issues() as issues_count;

-- 偵測指定檢查的問題
SELECT clean.detect_metadata_issues(3) as issues_count;
```

**偵測的問題類型：**
1. **MISSING_COLUMN** (嚴重): 表格有欄位但metadata未註冊
2. **ORPHAN_METADATA** (警告): metadata有註冊但表格中不存在

---

## 📊 便捷視圖

### 1. v_latest_quality_check - 最新檢查結果

**用途：** 快速查看最新的品質檢查狀態。

```sql
SELECT * FROM clean.v_latest_quality_check;
```

**輸出範例：**
```
check_id | check_date          | total_tables | registered | rate   | grade   | status_emoji
---------|---------------------|--------------|------------|--------|---------|-------------
      3  | 2026-02-13 02:50:17 |            7 |    181/181 | 100.00 | PERFECT | 🏆 完美
```

---

### 2. v_open_quality_issues - 未解決問題

**用途：** 查看所有未解決的品質問題。

```sql
SELECT * FROM clean.v_open_quality_issues;
```

**輸出範例：**
```
issue_id | severity | table_name     | column_name  | issue_description
---------|----------|----------------|--------------|-------------------
     123  | CRITICAL | vendor_master  | new_field    | 欄位未註冊到metadata
```

---

### 3. v_quality_trend - 品質趨勢

**用途：** 查看品質變化趨勢。

```sql
SELECT * FROM clean.v_quality_trend;
```

**輸出範例：**
```
check_date | avg_rate | min_rate | max_rate | total_critical | check_count
-----------|----------|----------|----------|----------------|------------
2026-02-13 |   100.00 |   100.00 |   100.00 |              0 |           3
```

---

## 💼 使用場景

### 場景 1：每日品質檢查

**建議頻率：** 每天早上

```sql
-- 1. 執行檢查
SELECT * FROM clean.run_quality_check();

-- 2. 偵測問題
SELECT clean.detect_metadata_issues();

-- 3. 查看狀態
SELECT * FROM clean.v_latest_quality_check;

-- 4. 如果有問題，查看詳情
SELECT * FROM clean.v_open_quality_issues;
```

---

### 場景 2：新增表格後驗證

當新增表格或欄位後，立即驗證 metadata 是否完整註冊。

```sql
-- 執行檢查
SELECT 
    tables,
    registered || '/' || columns as 欄位註冊情況,
    rate || '%' as 完整度,
    grade as 等級
FROM clean.run_quality_check();

-- 如果不是100%，查看遺漏的欄位
SELECT 
    table_name,
    column_name,
    issue_description
FROM clean.v_open_quality_issues
WHERE severity = 'CRITICAL'
  AND issue_type = 'MISSING_COLUMN';
```

---

### 場景 3：修復問題後確認

完成 metadata 註冊修復後，驗證問題是否解決。

```sql
-- 1. 執行檢查
SELECT * FROM clean.run_quality_check();

-- 2. 確認品質等級
SELECT 
    CASE 
        WHEN rate = 100 THEN '✅ 所有問題已解決'
        ELSE '⚠️ 還有 ' || missing || ' 個欄位未註冊'
    END as 狀態
FROM clean.run_quality_check();

-- 3. 標記問題為已修復（如果需要）
UPDATE clean._metadata_quality_issues
SET status = 'FIXED',
    fixed_date = CURRENT_TIMESTAMP,
    fixed_by = 'USER_NAME'
WHERE status = 'OPEN' 
  AND table_name = 'your_table_name';
```

---

### 場景 4：定期報告

產生品質報告供管理層查看。

```sql
-- 最近7天的品質趨勢
SELECT 
    check_date as 日期,
    avg_rate as 平均完整度,
    total_critical as 嚴重問題數,
    check_count as 檢查次數
FROM clean.v_quality_trend
WHERE check_date >= CURRENT_DATE - INTERVAL '7 days'
ORDER BY check_date DESC;

-- 當前開放問題統計
SELECT 
    severity as 嚴重程度,
    COUNT(*) as 問題數量,
    STRING_AGG(table_name, ', ') as 涉及表格
FROM clean.v_open_quality_issues
GROUP BY severity
ORDER BY 
    CASE severity 
        WHEN 'CRITICAL' THEN 1 
        WHEN 'WARNING' THEN 2 
        ELSE 3 
    END;
```

---

## ⏰ 自動化執行（推薦）

### 方案 A：使用 PostgreSQL pg_cron

```sql
-- 安裝 pg_cron 擴展（需要超級用戶權限）
CREATE EXTENSION IF NOT EXISTS pg_cron;

-- 每天早上8點執行檢查
SELECT cron.schedule('metadata-quality-check',
    '0 8 * * *',
    $$SELECT clean.run_quality_check(); SELECT clean.detect_metadata_issues();$$
);

-- 查看已排程的任務
SELECT * FROM cron.job;
```

---

### 方案 B：使用 OpenClaw Cron

建立 OpenClaw cron job，每天自動執行並通知到 Telegram。

```javascript
// 在 OpenClaw 中建立 cron job
{
  "name": "Metadata Quality Check",
  "schedule": {
    "kind": "cron",
    "expr": "0 8 * * *",  // 每天早上8點
    "tz": "Asia/Taipei"
  },
  "payload": {
    "kind": "systemEvent",
    "text": "執行 metadata 品質檢查：psql ... -c 'SELECT * FROM clean.run_quality_check()'"
  },
  "sessionTarget": "main"
}
```

---

### 方案 C：使用系統 crontab

```bash
# 編輯 crontab
crontab -e

# 新增每天早上8點執行
0 8 * * * psql "postgresql://myuser:mypassword@...rds.amazonaws.com:5432/source" -c "SELECT clean.run_quality_check(); SELECT clean.detect_metadata_issues();" > /var/log/metadata_check.log 2>&1
```

---

## 📈 品質指標說明

### 註冊完整率
```
註冊率 = (已註冊欄位數 / 總欄位數) × 100%
```

**目標：**
- **目標值：** 100%
- **警戒線：** 95%
- **危險線：** 80%

### 嚴重問題數
- **0 個：** 🏆 完美
- **1-5 個：** 🟡 需關注
- **6+ 個：** 🔴 需立即處理

---

## 🎯 首次檢查結果

**檢查時間：** 2026-02-13 02:50:17  
**檢查ID：** 3

### 檢查結果
| 指標 | 數值 | 狀態 |
|-----|------|------|
| 業務表格數 | 7 | ✅ |
| 總欄位數 | 181 | ✅ |
| 已註冊欄位 | 181 | ✅ |
| 未註冊欄位 | 0 | 🏆 |
| **註冊完整率** | **100.00%** | 🏆 **完美** |
| 品質等級 | PERFECT | 🏆 |

### 檢查的表格
1. department_master (13 欄位)
2. employee_master (13 欄位)
3. material_cost (36 欄位)
4. material_master (38 欄位)
5. tax_code_master (25 欄位)
6. unit_master (13 欄位)
7. vendor_master (43 欄位)

---

## ⚠️ 常見問題處理

### Q1: 檢測到 MISSING_COLUMN 問題
**原因：** 表格有新增欄位但未註冊到 metadata

**解決方案：**
```sql
-- 1. 查看遺漏的欄位
SELECT table_name, column_name
FROM clean.v_open_quality_issues
WHERE issue_type = 'MISSING_COLUMN';

-- 2. 手動補齊註冊（參考前面的 Phase 1/2 修復範例）

-- 3. 重新檢查
SELECT * FROM clean.run_quality_check();
```

---

### Q2: 檢測到 ORPHAN_METADATA 問題
**原因：** metadata 中有註冊但表格中已刪除該欄位

**解決方案：**
```sql
-- 1. 查看過時的註冊
SELECT table_name, column_name
FROM clean.v_open_quality_issues
WHERE issue_type = 'ORPHAN_METADATA';

-- 2. 刪除過時的 metadata
DELETE FROM clean._metadata_column
WHERE table_name = 'your_table'
  AND column_name = 'deleted_column';

-- 3. 標記問題為已修復
UPDATE clean._metadata_quality_issues
SET status = 'FIXED', fixed_date = CURRENT_TIMESTAMP
WHERE issue_type = 'ORPHAN_METADATA'
  AND table_name = 'your_table';
```

---

### Q3: 想要忽略某些問題
**場景：** 某些問題暫時無法修復或不需要修復

**解決方案：**
```sql
UPDATE clean._metadata_quality_issues
SET status = 'IGNORED',
    fix_note = '暫時保留此欄位，待後續整合時處理'
WHERE issue_id = 123;
```

---

## 📝 維護建議

### 每日維護
- ✅ 執行一次品質檢查
- ✅ 查看是否有新問題
- ✅ 確認註冊率維持 100%

### 每週維護
- ✅ 檢視品質趨勢
- ✅ 處理累積的開放問題
- ✅ 清理已修復的舊記錄

### 每月維護
- ✅ 產生月度品質報告
- ✅ 分析問題模式
- ✅ 優化檢查流程

---

## 🎉 總結

**系統狀態：** ✅ 已成功部署  
**當前品質：** 🏆 100% 完美  
**檢查表格：** 7 張業務表格  
**總欄位數：** 181 個（全部已註冊）

**系統優勢：**
- 🤖 自動化檢查，無需人工干預
- 📊 歷史記錄完整，可追蹤趨勢
- 🔍 問題識別精準，定位快速
- ⏰ 支援定期執行，持續監控
- 📈 視圖便捷，一目了然

**下一步：**
1. ⏳ 設定自動化執行（cron job）
2. ⏳ 建立告警機制（品質下降時通知）
3. ⏳ 整合到監控儀表板

---

**文檔版本：** 1.0  
**最後更新：** 2026-02-13  
**維護者：** meimei 🤖

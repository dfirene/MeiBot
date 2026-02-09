# ERP 資料整合完成報告
**日期:** 2026-02-09  
**批次編號:** BATCH-20260209-001

## 📊 整合表格總覽

| Clean 表格 | 中文名稱 | 來源表格 | 資料筆數 | 說明 |
|-----------|---------|----------|---------|------|
| `unit_master` | 單位主檔 | `erp_m2201_gfe_file` | 5 | 單位代碼與名稱 (KG, MT, PCS, YD, L) |
| `vendor_master` | 供應商主檔 | `erp_m2201_pmc_file` | 9,972 | 供應商資訊，保留 WMS/MES 擴展欄位 |
| `material_cost` | 料件成本主檔 | `erp_m2201_imb_file` | 1,050 | 350 料件 × 3 成本類型 (STANDARD/CURRENT/ACTUAL) |

---

## 1️⃣ Unit Master (單位主檔)

### 📋 表格結構
- **主鍵:** `unit_code`
- **總欄位:** 11 個
- **核心欄位:** 6 個

### 🔑 核心欄位
| 欄位名稱 | 中文名稱 | 資料型態 | 說明 |
|---------|---------|---------|------|
| `unit_code` | 單位代碼 | VARCHAR(20) | PK - KG, MT, PCS, YD, L |
| `unit_name` | 單位名稱 | VARCHAR(100) | 公斤, 公噸, 件, 碼, 公升 |
| `decimal_places` | 小數位數 | SMALLINT | 數值精度 |
| `is_active` | 是否有效 | BOOLEAN | 資料有效性 |

### 🔄 資料轉換
- `gfeacti` (Y/N) → `is_active` (BOOLEAN)

### 📊 資料品質
- ✅ 100% 資料完整性
- ✅ 5 筆全部有效

---

## 2️⃣ Vendor Master (供應商主檔)

### 📋 表格結構
- **主鍵:** `vendor_id`
- **總欄位:** 24 個
- **核心欄位:** 18 個
- **保留欄位:** `wms_vendor_id`, `mes_vendor_id`, `source_systems[]`

### 🔑 核心欄位
| 欄位名稱 | 中文名稱 | 資料型態 | 完整度 |
|---------|---------|---------|--------|
| `vendor_id` | 供應商編號 | VARCHAR(50) | 100% |
| `vendor_category` | 廠商分類 | VARCHAR(20) | 98.55% |
| `vendor_short_name` | 供應商簡稱 | VARCHAR(200) | 100% |
| `vendor_full_name` | 供應商全名 | VARCHAR(500) | 99.99% |
| `phone` | 電話號碼 | VARCHAR(100) | 96.02% |
| `fax` | 傳真號碼 | VARCHAR(100) | 94.60% |
| `payment_terms` | 付款方式 | VARCHAR(50) | - |
| `vendor_rating` | 廠商評鑑等級 | VARCHAR(10) | - |

### 🔄 資料轉換
1. **Boolean 轉換:** `pmcacti` (Y/N) → `is_active` (BOOLEAN)
2. **欄位合併:** `pmc081` + `pmc082` → `vendor_full_name` (TRIM)
3. **系統標記:** `source_systems = ARRAY['ERP']`

### 📊 資料品質
- ✅ 9,972 筆供應商資料
- ✅ 核心欄位完整度 94-100%
- ✅ 保留欄位供 WMS/MES 未來整合

---

## 3️⃣ Material Cost (料件成本主檔)

### 📋 表格結構
- **複合主鍵:** `(material_id, cost_type)`
- **總欄位:** 30+ 個
- **成本類型:** STANDARD (標準), CURRENT (現時), ACTUAL (實際)

### 🔑 核心欄位
| 欄位名稱 | 中文名稱 | 資料型態 | 說明 |
|---------|---------|---------|------|
| `material_id` | 料件編號 | VARCHAR(50) | PK - 關聯到 material_master |
| `cost_type` | 成本類型 | VARCHAR(20) | PK - STANDARD/CURRENT/ACTUAL |
| `material_cost` | 本階材料成本 | DECIMAL(18,6) | 直接材料成本 |
| `labor_cost` | 本階人工成本 | DECIMAL(18,6) | 直接人工成本 |
| `outsourcing_cost` | 本階廠外加工成本 | DECIMAL(18,6) | 委外加工成本 |
| `current_level_total` | 本階成本總計 | DECIMAL(18,6) | **自動計算** |
| `lower_level_total` | 下階成本總計 | DECIMAL(18,6) | **自動計算** |
| `total_cost` | 總成本 | DECIMAL(18,6) | **自動計算** |

### 🧮 自動計算欄位
```sql
current_level_total = material_cost + material_overhead_cost + labor_cost + 
                     labor_overhead_cost + fixed_overhead_cost + 
                     variable_overhead_cost + outsourcing_cost + 
                     machine_cost + additional_cost

lower_level_total = lower_material_cost + lower_material_overhead_cost + 
                   lower_labor_cost + lower_labor_overhead_cost + 
                   lower_fixed_overhead_cost + lower_variable_overhead_cost + 
                   lower_outsourcing_cost + lower_machine_cost + 
                   lower_additional_cost

total_cost = current_level_total + lower_level_total
```

### 🔄 資料轉換
1. **成本拆分:** 1 筆 imb_file → 3 筆 material_cost (STANDARD/CURRENT/ACTUAL)
2. **NULL 處理:** 所有數值欄位 `COALESCE(value, 0)`
3. **外鍵約束:** 關聯到 `material_master.material_id`

### 📊 資料品質
- ✅ 350 料件 × 3 成本類型 = 1,050 筆
- ✅ 100% 資料完整性
- ✅ 自動計算總成本，避免手動錯誤

---

## 🗂️ Metadata 管理

### 已註冊的 Metadata

#### _metadata_table (3 筆)
- `unit_master` - 5 rows
- `vendor_master` - 9,972 rows
- `material_cost` - 1,050 rows

#### _metadata_column (22 筆)
- `unit_master`: 6 欄位
- `vendor_master`: 9 欄位
- `material_cost`: 7 欄位

#### _metadata_column_mapping (10 筆)
- 記錄所有來源欄位對應關係
- 包含轉換規則 (DIRECT, BOOLEAN, CONCAT, CALCULATED)

#### _cleansing_log (5 筆)
| 表格 | 欄位 | 清洗層級 | 類型 | 說明 |
|-----|------|---------|------|------|
| unit_master | is_active | 1-FORMAT | FORMAT | Y/N → boolean |
| vendor_master | vendor_full_name | 2-NORMALIZE | NORMALIZE | 合併兩欄 |
| vendor_master | is_active | 1-FORMAT | FORMAT | Y/N → boolean |
| material_cost | cost_type | 2-NORMALIZE | NORMALIZE | 拆分成本類型 ⚠️ |
| material_cost | total_cost | 3-INFER | INFER | 自動計算 |

#### _metadata_statistics (3 筆)
| 表格 | 筆數 | 成功率 | ETL耗時 |
|-----|------|--------|---------|
| unit_master | 5 | 100% | 1 秒 |
| vendor_master | 9,972 | 100% | 5 秒 |
| material_cost | 1,050 | 100% | 3 秒 |

---

## 🎯 資料整合特色

### 1. **Medallion Architecture 分層**
- **Source Layer:** public schema (原始資料)
- **Clean Layer:** clean schema (清洗 + 整合)
- **Mart Layer:** 待建立 (應用 + 報表)

### 2. **多系統整合設計**
- 保留欄位: `wms_vendor_id`, `mes_vendor_id`
- 來源追蹤: `source_systems[]`, `source_system`, `source_id`
- 未來可整合 WMS、MES 資料到同一張表

### 3. **審計追蹤完整**
- ETL 批次追蹤: `etl_batch_id`, `etl_loaded_at`
- 資料來源記錄: `source_system`, `source_table`, `source_id`
- 清洗過程記錄: `_cleansing_log` 保留原始值

### 4. **自動化計算**
- 使用 PostgreSQL `GENERATED ALWAYS AS ... STORED`
- 成本自動加總，避免人工錯誤
- 資料一致性保證

---

## ✅ 完成檢查清單

- [x] 建立 3 個 clean layer 表格
- [x] 導入 11,027 筆資料 (5 + 9,972 + 1,050)
- [x] 註冊 metadata (table/column/mapping)
- [x] 記錄 cleansing log
- [x] 更新統計資料
- [x] 建立外鍵約束 (material_cost → material_master)
- [x] 建立索引 (提升查詢效能)

---

## 📈 下一步建議

### 立即可做
1. ✅ 驗證資料正確性 (抽樣檢查)
2. ✅ 測試查詢效能
3. ✅ 建立 Mart Layer 表格 (報表用)

### 後續規劃
1. ⏳ 整合其他 ERP 主檔 (客戶、倉庫、BOM...)
2. ⏳ 建立 ETL 自動化程序 (定期同步)
3. ⏳ 整合 WMS、MES 資料
4. ⏳ 建立監控儀表板 (Grafana + _metadata_statistics)

---

## 🔍 快速查詢範例

### 查看所有單位
```sql
SELECT * FROM clean.unit_master ORDER BY unit_code;
```

### 查看前10名供應商
```sql
SELECT vendor_id, vendor_short_name, phone, is_active
FROM clean.vendor_master
WHERE is_active = true
ORDER BY vendor_id
LIMIT 10;
```

### 查看料件成本 (標準成本)
```sql
SELECT 
    material_id,
    current_level_total,
    lower_level_total,
    total_cost
FROM clean.material_cost
WHERE cost_type = 'STANDARD'
ORDER BY total_cost DESC
LIMIT 10;
```

### 比較三種成本
```sql
SELECT 
    material_id,
    MAX(CASE WHEN cost_type = 'STANDARD' THEN total_cost END) as standard_cost,
    MAX(CASE WHEN cost_type = 'CURRENT' THEN total_cost END) as current_cost,
    MAX(CASE WHEN cost_type = 'ACTUAL' THEN total_cost END) as actual_cost
FROM clean.material_cost
GROUP BY material_id
ORDER BY material_id
LIMIT 10;
```

---

**整合完成！** 🎉

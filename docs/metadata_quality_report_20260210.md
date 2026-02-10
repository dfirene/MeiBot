# Metadata 品質檢查報告
**日期:** 2026-02-10  
**檢查範圍:** clean schema 所有業務表格  
**檢查項目:** 表格欄位與 _metadata_column 註冊一致性

---

## 📊 執行摘要

### 整體統計
| 項目 | 數量 |
|-----|------|
| 業務表格總數 | 7 張 |
| 總欄位數 | 151 個 |
| 已註冊欄位 | 116 個 |
| **未註冊欄位** | **35 個** ❌ |
| **整體註冊率** | **76.82%** |

### 品質等級分布
| 等級 | 註冊率 | 表格數 | 表格名稱 |
|-----|--------|--------|----------|
| 🟢 優秀 | 100% | 3 | material_master, tax_code_master, vendor_master |
| 🟡 良好 | 80-99% | 2 | department_master (84.62%), employee_master (84.62%) |
| 🔴 待改善 | <80% | 2 | unit_master (46.15%), **material_cost (19.44%)** |

---

## 🚨 品質問題詳情

### 問題 1: material_cost - 嚴重欠缺 🔴
**註冊率:** 19.44% (7/36)  
**遺漏:** 29 個欄位未註冊

#### 已註冊 (7 個)
- material_id, cost_type, material_cost, labor_cost
- current_level_total, lower_level_total, total_cost

#### ❌ 未註冊 (29 個關鍵欄位)
```
material_overhead_cost, labor_overhead_cost, fixed_overhead_cost, 
variable_overhead_cost, outsourcing_material_cost, outsourcing_cost,
outsourcing_fixed_cost, outsourcing_variable_cost, machine_cost,
additional_cost, purchase_cost, lower_material_cost,
lower_material_overhead_cost, lower_labor_cost, lower_labor_overhead_cost,
lower_fixed_overhead_cost, lower_variable_overhead_cost, lower_outsourcing_cost,
lower_machine_cost, lower_additional_cost, created_by, created_at,
modified_by, modified_at, source_system, source_table, source_id,
etl_batch_id, etl_loaded_at
```

**影響:** 成本明細欄位、審計欄位、來源追蹤欄位全部遺漏！

---

### 問題 2: unit_master - 註冊不完整 🟡
**註冊率:** 46.15% (6/13)  
**遺漏:** 7 個欄位未註冊

#### 已註冊 (6 個)
- unit_code, unit_name, decimal_places, is_active, source_system, etl_batch_id

#### ❌ 未註冊 (7 個)
```
created_by, created_at, modified_by, modified_at,
source_table, source_id, etl_loaded_at
```

**影響:** 審計欄位、來源追蹤欄位遺漏

---

### 問題 3: department_master - 輕微遺漏 🟢
**註冊率:** 84.62% (11/13)  
**遺漏:** 2 個欄位未註冊

#### ❌ 未註冊
- etl_batch_id
- etl_loaded_at

---

### 問題 4: employee_master - 輕微遺漏 🟢
**註冊率:** 84.62% (11/13)  
**遺漏:** 2 個欄位未註冊

#### ❌ 未註冊
- etl_batch_id
- etl_loaded_at

---

## ✅ 完整註冊的表格

### material_master
- **註冊率:** 100% (38/38)
- **狀態:** ✅ 完美

### tax_code_master
- **註冊率:** 100% (25/25)
- **狀態:** ✅ 完美

### vendor_master
- **註冊率:** 100% (29/29)
- **狀態:** ✅ 完美

---

## 🔧 修復建議

### 🔴 高優先級 - 立即修復

#### 1. 補齊 material_cost 的 29 個欄位
```sql
-- 需要註冊完整的成本欄位結構
-- 包含：本階成本、下階成本、審計欄位、來源追蹤
```

**影響範圍:** 成本分析報表、成本追蹤、資料溯源

---

### 🟡 中優先級 - 近期修復

#### 2. 補齊 unit_master 的 7 個欄位
```sql
-- 審計欄位：created_by, created_at, modified_by, modified_at
-- 來源追蹤：source_table, source_id, etl_loaded_at
```

**影響範圍:** 資料溯源、審計追蹤

---

### 🟢 低優先級 - 可延後

#### 3. 補齊 department_master 和 employee_master 的 ETL 欄位
```sql
-- ETL 欄位：etl_batch_id, etl_loaded_at
```

**影響範圍:** ETL 批次追蹤

---

## 📋 修復執行計畫

### Phase 1: 立即執行（今天）
1. ✅ 補齊 **material_cost** 的 29 個欄位註冊
2. ✅ 補齊 **unit_master** 的 7 個欄位註冊

### Phase 2: 本週執行
3. ⏳ 補齊 **department_master** 和 **employee_master** 的 2 個欄位

### Phase 3: 建立自動檢查
4. ⏳ 建立 metadata 品質檢查定期任務
5. ⏳ 將品質指標納入 `_metadata_statistics`

---

## 🎯 品質改善目標

| 階段 | 目標註冊率 | 預計完成 |
|-----|-----------|----------|
| **Phase 1** | **95%+** | 2026-02-10 |
| Phase 2 | 98%+ | 2026-02-12 |
| Phase 3 | 100% | 2026-02-15 |

---

## 🔍 根本原因分析

### 為什麼會有遺漏？
1. **建表時只註冊核心欄位**，忽略審計和 ETL 欄位
2. **material_cost 註冊時只記錄了摘要欄位**，遺漏大量成本明細
3. **缺乏自動化檢查機制**，無法及時發現遺漏

### 改善措施
1. ✅ 統一 metadata 表格名稱格式（去除 schema 前綴）
2. ✅ 建立 metadata 品質檢查腳本
3. ⏳ 建立欄位自動註冊程序
4. ⏳ 將品質檢查納入 CI/CD 流程

---

## 📊 附錄：完整表格清單

| 表格名稱 | 實際欄位 | 已註冊 | 未註冊 | 註冊率 | 狀態 |
|---------|---------|--------|--------|--------|------|
| material_cost | 36 | 7 | 29 | 19.44% | 🔴 |
| unit_master | 13 | 6 | 7 | 46.15% | 🟡 |
| department_master | 13 | 11 | 2 | 84.62% | 🟢 |
| employee_master | 13 | 11 | 2 | 84.62% | 🟢 |
| material_master | 38 | 38 | 0 | 100% | ✅ |
| tax_code_master | 25 | 25 | 0 | 100% | ✅ |
| vendor_master | 29 | 29 | 0 | 100% | ✅ |

---

**報告完成時間:** 2026-02-10 06:35 UTC  
**下一次檢查:** Phase 1 修復完成後

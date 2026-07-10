---
name: odoo-perf-tune
description: Tối ưu hiệu năng module Odoo — phát hiện N+1 query, thiếu index, batch operation, memory issue và đề xuất fix cụ thể. Kích hoạt khi user nói "chậm", "slow", "timeout", "tối ưu", "performance", "N+1", "query nhiều", "memory", "tối ưu ORM", "batch", "cron chậm".
---

# Odoo Performance Tuning

## Goal
Phân tích code Odoo → phát hiện bottleneck → đề xuất fix cụ thể với code example sẵn sàng áp dụng.

**Input**: Code cần review hoặc mô tả vấn đề  
**Output**: Danh sách issues (CRITICAL/HIGH/MEDIUM) + fix code + checklist

## When to use this skill
- "slow page load", "timeout", "server lag"
- "tối ưu query", "N+1 problem", "too many SQL queries"
- "memory error", "cron job chậm"
- Review code trước khi deploy production

## Instructions

### Bước 1 — Phân loại vấn đề

| Triệu chứng | Nguyên nhân | Ưu tiên |
|---|---|---|
| Page load chậm | N+1 queries, thiếu prefetch | CRITICAL |
| Timeout khi search | Thiếu index, domain phức tạp | HIGH |
| Cron job treo | Không batch, không clear cache | HIGH |
| Memory error | Load quá nhiều records | HIGH |
| Computed field chậm | Query trong vòng lặp | MEDIUM |

### Bước 2 — Fix patterns theo loại
Xem `references/GUIDE.md` cho code examples đầy đủ:

- **N+1 Query**: `references/GUIDE.md#n-plus-1` — dùng `mapped()` prefetch hoặc `search_read()`
- **Batch Ops**: `references/GUIDE.md#batch` — `create(list)`, `records.write()`, `records.unlink()`
- **Efficient Search**: `references/GUIDE.md#search` — `search_count()`, domain filter thay Python filter
- **Indexing**: `references/GUIDE.md#index` — `index=True`, `index='trigram'`, `index='btree_not_null'`
- **Computed Fields**: `references/GUIDE.md#computed` — stored vs non-stored, `_read_group` batch
- **SQL bulk**: `references/GUIDE.md#sql` — `SQL()` builder + `invalidate_recordset()`
- **Cron batch**: `references/GUIDE.md#cron` — `limit/offset` loop + `cr.commit()` + `invalidate_all()`
- **Memory**: `references/GUIDE.md#memory` — generator pattern, `invalidate_all()` định kỳ

### Bước 3 — Monitoring
Xem `references/GUIDE.md#monitoring`:
- `log_level = debug_sql` để đếm queries thực tế
- `QueryCounter` trong tests
- `time.perf_counter()` cho timing
- `@profile` decorator của Odoo

### Bước 4 — Xuất báo cáo

```
⚡ PERFORMANCE ANALYSIS — [module name]
Issues: [N] vấn đề phát hiện

── CRITICAL ──────────────────────────────────────────
[PERF-001] file.py:line — N+1 query → fix: mapped() prefetch
── HIGH ──────────────────────────────────────────────
[PERF-002] file.py:line — Thiếu index state → fix: index=True
── CHECKLIST ─────────────────────────────────────────
□ Index fields trong search domain
□ Stored computed fields cho values thường đọc
□ Batch create/write/unlink
□ search_count() thay len(search())
□ Cron: batch 1000/lần + commit + invalidate
```

## Constraints
- **KHÔNG** tự sửa code khi chưa được user xác nhận
- Raw SQL chỉ khi ORM không đáp ứng — **bắt buộc** dùng `SQL()` builder (Odoo 19)

## Best practices
- Enable `debug_sql` để đếm queries thực tế trước khi tối ưu
- Thứ tự ưu tiên: N+1 → index → batch → SQL
- Test với production-size data, không phải demo data

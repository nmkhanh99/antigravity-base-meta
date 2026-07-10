---
name: odoo-performance
description: Hướng dẫn tối ưu hóa performance trong Odoo 19 - database optimization, caching, profiling, batch processing, và best practices. Use when the user asks to optimize performance, fix slow queries, add caching, or profile Odoo 19 code.
---

# Odoo 19 Performance Optimization

## Goal
Giúp agent tối ưu hóa performance Odoo 19: ORM efficiency, SQL optimization, caching, prefetching, và profiling. Xác định và sửa N+1 queries, thêm index, dùng đúng API Odoo 19.

## When to use this skill
- "tối ưu performance", "optimize", "slow", "chậm"
- "caching", "prefetch", "batch"
- "profiling", "query optimization", "explain analyze"
- "N+1 query", "memory leak", "memory"
- "search_fetch", "ormcache", "index"

## Instructions

### Bước 1 — Xác định bottleneck
Trước khi optimize, hỏi user:
- Vấn đề cụ thể (slow list view / slow report / heavy cron)?
- Có log SQL / profiler output chưa?
- Số lượng records ước tính?

### Bước 2 — ORM Performance Rules
Áp dụng theo thứ tự ưu tiên:

1. **Dùng `search_fetch()` (Odoo 19, thay `search_read`)**
   - Trả về recordset với fields pre-fetched, ít SQL hơn.
2. **Batch write thay loop write**
   - `records.write({...})` = 1 query; `for r in records: r.write(...)` = N queries.
3. **Prefetch trước khi loop qua related fields**
   - Gọi `records.mapped('partner_id')` trước khi loop để tránh N+1.
4. **Dùng `_read_group()` (không phải `read_group()` deprecated)**
   - API mới Odoo 19: `Model._read_group(domain, groupby, aggregates)`.
5. **Store computed fields** nếu field được dùng trong `search` / `order`.

### Bước 3 — Database Indexes
- Dùng `models.Index('field1', 'field2')` (Declarative API Odoo 19).
- Dùng `models.UniqueIndex('field')` cho uniqueness constraint.
- Thêm `index=True` trên field cho single-column index.

### Bước 4 — Caching
- `@tools.ormcache(...)` cho method-level cache (invalidated tự động khi env.cache.invalidate()).
- Xóa cache thủ công: `self.method_name.clear_cache(self)`.
- KHÔNG dùng `@lru_cache` cho Odoo model methods (không thread-safe trong multi-worker).

### Bước 5 — SQL an toàn
- Dùng `SQL()` wrapper (Odoo 19) thay f-string / % interpolation trực tiếp.
- Dùng `self.env.execute_query(SQL(...))` để chạy.
- Chỉ dùng raw SQL khi ORM không thể handle (aggregation phức tạp, window functions).

### Bước 6 — Profiling
- Enable `log_level = debug_sql` trong `odoo.conf` để log mọi query.
- Dùng Odoo built-in profiler: `odoo/tools/profiler.py` hoặc `/web/dataset/call_kw` với `?debug=1`.
- Xem GUIDE.md section "Profiling & Monitoring" cho code mẫu chi tiết.

## Constraints
- KHÔNG dùng `name_get()`, `read_group()` (deprecated trước Odoo 17).
- KHÔNG dùng `self.env.cr.execute()` với string interpolation — luôn dùng `SQL()` wrapper.
- KHÔNG create/write records bên trong `@api.depends` computed field methods.
- KHÔNG dùng `browse()` trong vòng lặp — dùng `browse(ids)` một lần bên ngoài.
- KHÔNG dùng `@lru_cache` trên model methods (dùng `@tools.ormcache` thay thế).
- KHÔNG commit `env.cr.commit()` giữa chừng trừ batch cron jobs có kiểm soát.

## References
- [Odoo 19 Performance Guidelines](https://www.odoo.com/documentation/19.0/developer/reference/backend/performance.html)
- [Odoo 19 ORM — Cache](https://www.odoo.com/documentation/19.0/developer/reference/backend/orm.html#performance)
- [Odoo 19 SQL() wrapper](https://www.odoo.com/documentation/19.0/developer/reference/backend/orm.html#odoo.tools.SQL)
- [PostgreSQL Performance Tips](https://wiki.postgresql.org/wiki/Performance_Optimization)
- Xem chi tiết code examples: `references/GUIDE.md`

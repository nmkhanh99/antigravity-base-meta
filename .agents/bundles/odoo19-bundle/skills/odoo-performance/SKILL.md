---
name: odoo-performance
description: Hướng dẫn tối ưu hóa performance trong Odoo 19 - database optimization, caching, profiling, batch processing, và best practices. Use when the user asks to optimize performance, fix slow queries, add caching, or profile Odoo 19 code.
---

# Odoo 19 Performance Optimization

## Goal
Giúp agent tối ưu hóa performance Odoo 19: ORM efficiency, SQL optimization, caching, prefetching, và profiling.

## When to use this skill
- "tối ưu performance", "optimize", "slow"
- "caching", "prefetch", "batch"
- "profiling", "query optimization"
- "N+1 query", "memory leak"

## Instructions

### 1. ORM Performance Rules
```python
# ✅ Batch operations (1 query)
partners.write({'active': False})

# ❌ Loop operations (N queries)
for p in partners:
    p.write({'active': False})

# ✅ search_fetch (Odoo 19 — optimized)
records = self.env['sale.order'].search_fetch(
    [('state', '=', 'sale')],
    ['name', 'partner_id', 'amount_total'],
    limit=100,
)

# ✅ Prefetch related fields
records.fetch(['partner_id.name', 'partner_id.email'])

# ✅ Use mapped/filtered (avoid loops)
emails = partners.mapped('email')
active = partners.filtered('active')
```

### 2. SQL Optimization
```python
# Add database indexes
partner_date_idx = models.Index('partner_id', 'date')

# Stored computed fields (avoid recomputation)
total = fields.Float(compute='_compute_total', store=True)

# Use _read_group for aggregation (NOT loops)
result = self.env['sale.order']._read_group(
    [('state', '=', 'sale')],
    groupby=['partner_id'],
    aggregates=['amount_total:sum'],
)
```

### 3. Avoid N+1 Queries
```python
# ❌ N+1 problem
for order in orders:
    print(order.partner_id.name)  # Separate query per record

# ✅ Prefetch first
orders.mapped('partner_id')  # Prefetch all partners
for order in orders:
    print(order.partner_id.name)  # No extra queries
```

### 4. Caching
```python
from odoo import tools

@tools.ormcache('self.env.uid', 'key')
def _get_config(self, key):
    return self.env['ir.config_parameter'].get_param(key)

# Clear cache when needed
self._get_config.clear_cache(self)
```

### 5. Profiling
```python
# Enable profiling in odoo.conf
# log_level = debug_sql

# Or programmatic profiling
import logging
_logger = logging.getLogger(__name__)
_logger.info("Query count: %s", self.env.cr.sql_log_count)
```

## Constraints
- KHÔNG dùng `self.env.cr.execute()` cho operations ORM có thể handle.
- KHÔNG create/write records trong computed field methods.
- Avoid `browse()` in loops — dùng `browse(ids)` một lần.

## Best practices
- Dùng `search_fetch()` thay `search_read()`.
- Thêm `store=True` cho computed fields dùng trong search/filter.
- Dùng `models.Index()` cho fields thường xuyên search/filter.
- Đọc `resources/reference.md` cho memory management, SQL profiling tools.

# Odoo Performance Tuning — Full Code Reference

## n-plus-1

```python
# ❌ SAI — N queries
for order in orders:
    print(order.partner_id.name)

# ✅ ĐÚNG — prefetch trước
orders.mapped('partner_id')  # 1 query
for order in orders:
    print(order.partner_id.name)  # cached

# ✅ TỐT NHẤT — search_read khi chỉ cần 1 số field
data = self.env['sale.order'].search_read(
    [('state', '=', 'sale')], ['name', 'partner_id', 'amount_total'], limit=100,
)

# Computed field batch với _read_group
@api.depends('partner_id')
def _compute_order_count(self):
    if not self: return
    order_data = self.env['sale.order']._read_group(
        [('partner_id', 'in', self.mapped('partner_id').ids)],
        groupby=['partner_id'], aggregates=['__count'],
    )
    counts = {partner.id: count for partner, count in order_data}
    for record in self:
        record.order_count = counts.get(record.partner_id.id, 0)
```

## batch

```python
# ❌ SAI — N queries
for data in data_list:
    self.env['my.model'].create(data)

# ✅ ĐÚNG — 1 query
self.env['my.model'].create(data_list)

# ❌ SAI
for record in records:
    record.write({'state': 'done'})

# ✅ ĐÚNG
records.write({'state': 'done'})
records.unlink()
```

## search

```python
# search_count thay len(search())
count = self.env['my.model'].search_count([('state', '=', 'draft')])

# domain filter thay Python filter
draft_records = self.env['my.model'].search([('state', '=', 'draft')])

# join domain thay 2 queries
orders = self.env['sale.order'].search([('partner_id.customer_rank', '>', 0)])
```

## index

```python
# B-tree index
state = fields.Selection([...], index=True)
company_id = fields.Many2one('res.company', index=True)
date = fields.Date(index=True)

# Trigram index cho ILIKE search (v16+)
name = fields.Char(index='trigram')

# Index non-NULL (v16+)
code = fields.Char(index='btree_not_null')
```

Khi nào index: Selection thường dùng filter, Many2one trong search/record rules, Date trong range queries, Char với `=`, Char pattern dùng `index='trigram'`.

## computed

```python
# STORED — đọc thường xuyên, ít thay đổi
total = fields.Float(compute='_compute_total', store=True)

@api.depends('line_ids.amount')
def _compute_total(self):
    for record in self:
        record.total = sum(record.line_ids.mapped('amount'))

# NON-STORED — thay đổi liên tục
days_left = fields.Integer(compute='_compute_days_left')  # store=False default
```

## sql

```python
from odoo.tools import SQL

# Bulk update với SQL
self.env.cr.execute(SQL(
    "UPDATE my_model SET state = %s, write_date = NOW() WHERE state = %s AND company_id = %s",
    'done', 'confirmed', self.env.company.id
))
# Invalidate cache sau SQL
self.browse(updated_ids).invalidate_recordset()

# Aggregation
self.env.cr.execute(SQL(
    "SELECT partner_id, COUNT(*) as cnt, SUM(amount_total) as total FROM sale_order WHERE state = %s GROUP BY partner_id",
    'sale'
))
results = self.env.cr.dictfetchall()
```

## cron

```python
@api.model
def _cron_process_large_dataset(self) -> None:
    batch_size = 1000
    offset = 0
    while True:
        records = self.search([('state', '=', 'pending')], limit=batch_size, offset=offset)
        if not records:
            break
        for record in records:
            try:
                record._process_single()
            except Exception as e:
                _logger.error("Lỗi xử lý %s: %s", record.id, e)
        self.env.cr.commit()
        self.env.invalidate_all()
        offset += batch_size
```

## memory

```python
# Generator pattern
def _iter_records(self, domain, batch_size=1000):
    offset = 0
    while True:
        records = self.search(domain, limit=batch_size, offset=offset)
        if not records: break
        yield from records
        offset += batch_size

# Clear cache định kỳ
def process_many_records(self):
    count = 0
    for record in self:
        record._do_processing()
        count += 1
        if count % 500 == 0:
            self.env.invalidate_all()
```

## monitoring

```ini
# odoo.conf
log_level = debug_sql
```

```python
# QueryCounter trong tests
from odoo.tests.common import QueryCounter
with QueryCounter(self.env.cr) as qc:
    self.env['sale.order'].search([('state', '=', 'sale')])
print(f"Số queries: {qc.count}")

# Timing
import time, logging
_logger = logging.getLogger(__name__)

def method(self):
    start = time.perf_counter()
    # ...
    elapsed = time.perf_counter() - start
    if elapsed > 5.0:
        _logger.warning("Slow: %.2f giây", elapsed)

# Odoo Profiler
from odoo.tools.profiler import profile

@profile
def slow_method(self):
    pass
```

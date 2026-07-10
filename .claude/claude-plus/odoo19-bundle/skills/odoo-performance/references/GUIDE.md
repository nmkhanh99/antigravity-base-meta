---
name: Odoo Performance Optimization
description: Hướng dẫn toàn diện về tối ưu hóa performance trong Odoo 19 - database optimization, caching, profiling, và best practices
---

# Odoo 19 Performance Optimization Guide

## Mục Lục

1. [ORM Performance](#orm-performance)
2. [Database Indexes](#database-indexes)
3. [Caching Strategies](#caching-strategies)
4. [Query Optimization](#query-optimization)
5. [Frontend Performance](#frontend-performance)
6. [Profiling & Monitoring](#profiling--monitoring)
7. [Batch Processing](#batch-processing)
8. [Best Practices Checklist](#best-practices-checklist)

---

## ORM Performance

### Tránh N+1 Queries

```python
# ❌ BAD: N+1 — mỗi iteration tạo 1 query riêng
for order in orders:
    print(order.partner_id.name)

# ✅ GOOD: Prefetch trước, sau đó loop
orders.mapped('partner_id')       # 1 query fetch tất cả partners
for order in orders:
    print(order.partner_id.name)  # Không query thêm

# ✅ BEST (Odoo 19): search_fetch — ít SQL hơn search() + read()
records = self.env['sale.order'].search_fetch(
    [('state', '=', 'sale')],
    ['name', 'partner_id', 'amount_total'],
    limit=100,
)
# records là recordset với fields đã pre-fetched
```

### Batch Write thay Loop Write

```python
# ❌ BAD: N queries
for p in partners:
    p.write({'active': False})

# ✅ GOOD: 1 query
partners.write({'active': False})
```

### Fetch Related Fields

```python
# Prefetch nhiều related fields cùng lúc
records.fetch(['partner_id.name', 'partner_id.email'])

# Dùng mapped/filtered thay vì vòng lặp thủ công
emails = partners.mapped('email')
active = partners.filtered('active')
active_confirmed = orders.filtered(lambda o: o.state == 'sale')
```

### _read_group thay vì loop aggregation (Odoo 19 API)

```python
# ✅ Dùng _read_group (API Odoo 17+, không phải read_group deprecated)
result = self.env['sale.order']._read_group(
    domain=[('state', '=', 'sale')],
    groupby=['partner_id'],
    aggregates=['amount_total:sum', 'id:count'],
)
# result: list of tuples (partner_id record, sum_amount, count)
for partner, total, count in result:
    print(partner.name, total, count)
```

### Store Computed Fields

```python
class SaleOrder(models.Model):
    _inherit = 'sale.order'

    # ✅ store=True: tính 1 lần, lưu DB, search/order được
    total_weight = fields.Float(
        compute='_compute_total_weight',
        store=True,
    )

    @api.depends('order_line.product_id.weight', 'order_line.product_uom_qty')
    def _compute_total_weight(self):
        for order in self:
            order.total_weight = sum(
                line.product_id.weight * line.product_uom_qty
                for line in order.order_line
            )
```

---

## Database Indexes

### Declarative Index API (Odoo 19)

```python
from odoo import models, fields

class StockMove(models.Model):
    _name = 'stock.move'

    # Single-column index (field level)
    reference = fields.Char(index=True)

    # Composite index (Odoo 19 Declarative API)
    partner_date_idx = models.Index('partner_id', 'date')

    # Unique constraint as index
    barcode_unique = models.UniqueIndex('barcode')

    def init(self):
        """Custom partial index (dùng khi cần WHERE clause)"""
        super().init()
        self.env.cr.execute("""
            CREATE INDEX IF NOT EXISTS stock_move_active_state_idx
            ON stock_move (state)
            WHERE state NOT IN ('done', 'cancel')
        """)
```

### Khi nào thêm index

| Trường hợp | Giải pháp |
|------------|-----------|
| Field thường xuyên trong `search` domain | `index=True` hoặc `models.Index` |
| Hai fields kết hợp trong query | `models.Index('field1', 'field2')` |
| Field cần unique | `models.UniqueIndex('field')` |
| Partial index (chỉ rows thỏa điều kiện) | `CREATE INDEX ... WHERE ...` trong `init()` |
| Computed field dùng trong `order=` | Thêm `store=True` và `index=True` |

---

## Caching Strategies

### @tools.ormcache

```python
from odoo import models, tools

class IrConfigParameter(models.Model):
    _inherit = 'ir.config_parameter'

    @tools.ormcache('self.env.uid', 'key')
    def get_param_cached(self, key, default=False):
        """Cache config parameter per user per key"""
        return self.get_param(key, default)

    def set_param(self, key, value):
        result = super().set_param(key, value)
        # Xóa cache khi giá trị thay đổi
        self.get_param_cached.clear_cache(self)
        return result
```

### Invalidate Cache

```python
# Xóa toàn bộ ORM cache của environment (dùng sau batch updates lớn)
self.env.cache.invalidate()

# Xóa cache của 1 method cụ thể
self._get_product_price.clear_cache(self)
```

### Lưu ý quan trọng

- KHÔNG dùng `@functools.lru_cache` trên model methods — không thread-safe trong Odoo multi-worker.
- `@tools.ormcache` tự động invalidate khi `env.cache.invalidate()` được gọi.
- Cache bị clear khi server restart.

---

## Query Optimization

### SQL() Wrapper (Odoo 19)

```python
from odoo.tools import SQL

def get_sales_stats(self, date_from, date_to):
    """Dùng SQL() wrapper — tránh SQL injection"""
    query = SQL("""
        SELECT
            pt.id,
            pt.name,
            SUM(sol.product_uom_qty) AS total_qty,
            SUM(sol.price_subtotal) AS total_amount,
            COUNT(DISTINCT so.id) AS order_count
        FROM sale_order_line sol
        JOIN sale_order so ON sol.order_id = so.id
        JOIN product_product pp ON sol.product_id = pp.id
        JOIN product_template pt ON pp.product_tmpl_id = pt.id
        WHERE so.date_order >= %(date_from)s
          AND so.date_order <= %(date_to)s
          AND so.state IN ('sale', 'done')
        GROUP BY pt.id, pt.name
        ORDER BY total_amount DESC
        LIMIT 100
    """, date_from=date_from, date_to=date_to)

    return self.env.execute_query(query)
```

### search_fetch vs search_read (Odoo 19)

```python
# ❌ search_read (legacy, vẫn hoạt động nhưng có overhead)
data = self.env['res.partner'].search_read(
    domain=[('customer_rank', '>', 0)],
    fields=['name', 'email', 'phone'],
    limit=100,
)

# ✅ search_fetch (Odoo 19, preferred — trả về recordset)
partners = self.env['res.partner'].search_fetch(
    [('customer_rank', '>', 0)],
    ['name', 'email', 'phone'],
    limit=100,
)
# partners là recordset, truy cập field không query thêm
for p in partners:
    print(p.name, p.email)
```

### Đếm records: search_count

```python
# ✅ Đếm không load records
count = self.env['sale.order'].search_count([('state', '=', 'draft')])

# ❌ Load tất cả rồi đếm (lãng phí)
count = len(self.env['sale.order'].search([('state', '=', 'draft')]))
```

### Exists thay bool

```python
order = self.env['sale.order'].browse(123)

# ✅ Dùng exists()
if order.exists():
    order.action_confirm()

# ❌ Dùng bool (load toàn bộ record)
if order:
    order.action_confirm()
```

---

## Frontend Performance

### Lazy Loading Images

```xml
<template id="product_list_lazy">
    <t t-foreach="products" t-as="product">
        <div class="product-card">
            <img t-att-src="'/web/image/product.product/%s/image_128' % product.id"
                 loading="lazy"
                 alt="Product Image"/>
        </div>
    </t>
</template>
```

### Tối ưu List View

```xml
<record id="view_sale_order_list_optimized" model="ir.ui.view">
    <field name="name">sale.order.list.optimized</field>
    <field name="model">sale.order</field>
    <field name="arch" type="xml">
        <!-- limit=80 để giảm lượng data load ban đầu -->
        <list limit="80" default_order="date_order desc">
            <!-- Chỉ hiển thị fields cần thiết -->
            <field name="name"/>
            <field name="partner_id"/>
            <field name="date_order"/>
            <field name="amount_total"/>
            <field name="state"/>
        </list>
    </field>
</record>
```

### Reduce RPC Calls (OWL/JS)

```javascript
// ❌ BAD: Nhiều RPC calls riêng lẻ
async loadDataBad() {
    const orders = await this.orm.searchRead('sale.order', [], ['name']);
    for (let order of orders) {
        const partner = await this.orm.read('res.partner', [order.partner_id[0]], ['email']);
    }
}

// ✅ GOOD: Một RPC call với đủ fields
async loadDataGood() {
    return await this.orm.searchRead(
        'sale.order',
        [],
        ['name', 'partner_id', 'amount_total'],
        { limit: 100 }
    );
}
```

---

## Profiling & Monitoring

### Enable SQL Logging

```ini
# odoo.conf
[options]
log_level = debug_sql
```

### Context Manager Timer

```python
import time
import logging
from contextlib import contextmanager

_logger = logging.getLogger(__name__)

@contextmanager
def timer(name):
    start = time.time()
    try:
        yield
    finally:
        duration = time.time() - start
        _logger.info('%s took %.2f seconds', name, duration)

# Dùng:
def process_orders(self):
    with timer('Confirm Orders'):
        orders = self.env['sale.order'].search([('state', '=', 'draft')])
        orders.action_confirm()
```

### Python cProfile

```python
import cProfile
import pstats
from io import StringIO

def profile_operation(self):
    profiler = cProfile.Profile()
    profiler.enable()

    # Operation cần profile
    self._heavy_computation()

    profiler.disable()
    s = StringIO()
    ps = pstats.Stats(profiler, stream=s).sort_stats('cumulative')
    ps.print_stats(20)
    _logger.info(s.getvalue())
```

### Odoo Built-in Profiler

```
# Kích hoạt qua URL (debug mode)
/web?debug=1

# Hoặc qua odoo.conf
[options]
dev_mode = all
```

---

## Batch Processing

### Batch với commit định kỳ (Cron Jobs)

```python
def _cron_process_pending(self):
    """Cron job xử lý records pending — batch + commit"""
    Record = self.env['my.record']
    pending_ids = Record.search([('state', '=', 'pending')]).ids

    batch_size = 500
    for i in range(0, len(pending_ids), batch_size):
        batch = Record.browse(pending_ids[i:i + batch_size])
        batch._process()
        # Commit mỗi batch để tránh transaction lock dài
        self.env.cr.commit()
        _logger.info('Processed %d/%d records', i + len(batch), len(pending_ids))
```

### _commit_progress (Odoo 19 Cron)

```python
def _cron_large_import(self):
    """Dùng _commit_progress cho Odoo 19 cron — tự quản lý commit và cron state"""
    records = self.search([('state', '=', 'pending')])
    for record in records:
        record._process_single()
        # Odoo 19: tự commit định kỳ và cập nhật nextcall của cron
        self._commit_progress()
```

### Savepoints cho Error Recovery

```python
def batch_update_with_recovery(self):
    orders = self.env['sale.order'].search([('state', '=', 'draft')])
    errors = []

    for order in orders:
        try:
            self.env.cr.execute('SAVEPOINT batch_op')
            order.action_confirm()
            self.env.cr.execute('RELEASE SAVEPOINT batch_op')
        except Exception as e:
            self.env.cr.execute('ROLLBACK TO SAVEPOINT batch_op')
            errors.append((order.id, str(e)))
            _logger.warning('Failed order %s: %s', order.name, e)

    if errors:
        _logger.error('Failed %d orders: %s', len(errors), errors)
```

---

## Best Practices Checklist

Khi optimize performance, kiểm tra theo thứ tự:

### Database
- [ ] Index fields thường xuyên trong `search` domain và `order`
- [ ] Composite index cho queries kết hợp nhiều fields
- [ ] Partial index (`WHERE ...`) cho filtered queries
- [ ] `ANALYZE` tables sau data migration lớn

### ORM
- [ ] Không N+1: dùng `mapped()` prefetch trước loop
- [ ] Dùng `search_fetch()` thay `search()` + `read()` (Odoo 19)
- [ ] Dùng `_read_group()` (không phải `read_group()` deprecated)
- [ ] `store=True` cho computed fields dùng trong search/sort
- [ ] Batch `write()` thay loop `write()`
- [ ] `search_count()` thay `len(search(...))`
- [ ] `exists()` thay `bool(record)`

### Caching
- [ ] `@tools.ormcache` cho expensive methods
- [ ] Clear cache khi data thay đổi
- [ ] KHÔNG dùng `@lru_cache` trên model methods

### SQL
- [ ] Dùng `SQL()` wrapper, không string interpolation
- [ ] `self.env.execute_query()` để chạy
- [ ] Chỉ raw SQL cho aggregation ORM không handle được

### Batch/Cron
- [ ] Commit định kỳ mỗi batch (không commit trong loop nhỏ)
- [ ] Dùng `_commit_progress()` cho Odoo 19 cron
- [ ] Savepoints cho error recovery trong batch

### Frontend
- [ ] `loading="lazy"` cho images
- [ ] `limit` trong list views
- [ ] Gom nhiều fields vào 1 RPC call

---

## Tài Liệu Tham Khảo

- [Odoo 19 Performance Guidelines](https://www.odoo.com/documentation/19.0/developer/reference/backend/performance.html)
- [Odoo 19 ORM Cache](https://www.odoo.com/documentation/19.0/developer/reference/backend/orm.html#performance)
- [Odoo 19 SQL() wrapper](https://www.odoo.com/documentation/19.0/developer/reference/backend/orm.html#odoo.tools.SQL)
- [Odoo 19 search_fetch](https://www.odoo.com/documentation/19.0/developer/reference/backend/orm.html#odoo.models.Model.search_fetch)
- [PostgreSQL Performance Tips](https://wiki.postgresql.org/wiki/Performance_Optimization)
- [Python cProfile](https://docs.python.org/3/library/profile.html)

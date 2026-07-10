---
name: Odoo Performance Optimization
description: Hướng dẫn toàn diện về tối ưu hóa performance trong Odoo - database optimization, caching, profiling, và best practices
---

# Odoo Performance Optimization Skill

## 📋 Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [Database Optimization](#database-optimization)
3. [ORM Performance](#orm-performance)
4. [Caching Strategies](#caching-strategies)
5. [Query Optimization](#query-optimization)
6. [Frontend Performance](#frontend-performance)
7. [Profiling & Monitoring](#profiling--monitoring)
8. [Best Practices](#best-practices)
9. [Ví Dụ Thực Tế](#ví-dụ-thực-tế)

---

## Tổng Quan

### Performance Bottlenecks Phổ Biến

- ❌ N+1 query problems
- ❌ Missing database indexes
- ❌ Inefficient ORM usage
- ❌ Lack of caching
- ❌ Heavy computed fields
- ❌ Unoptimized searches
- ❌ Large recordsets in memory

### Performance Optimization Principles

- ✅ Minimize database queries
- ✅ Use proper indexing
- ✅ Leverage caching
- ✅ Optimize ORM operations
- ✅ Profile before optimizing
- ✅ Monitor production metrics

---

## Database Optimization

### 1. Database Indexes

```python
# models/product.py
from odoo import models, fields

class Product(models.Model):
    _name = 'product.product'
    _description = 'Product'
    
    # Add index for frequently searched fields
    name = fields.Char(string='Name', index=True)
    barcode = fields.Char(string='Barcode', index=True)
    default_code = fields.Char(string='Internal Reference', index=True)
    
    # Odoo 19: Declarative Index API (preferred)
    barcode_unique = models.UniqueIndex('barcode')
    
    # Composite index with declarative API
    partner_date_idx = models.Index('partner_id', 'date')
    
    def init(self):
        """Create custom indexes"""
        # Composite index for common queries
        self.env.cr.execute("""
            CREATE INDEX IF NOT EXISTS product_active_type_idx 
            ON product_product (active, type)
            WHERE active = true
        """)
        
        # Partial index for specific conditions
        self.env.cr.execute("""
            CREATE INDEX IF NOT EXISTS product_sale_ok_idx 
            ON product_template (sale_ok)
            WHERE sale_ok = true
        """)
```

### 2. Database Constraints

```python
class SaleOrder(models.Model):
    _inherit = 'sale.order'
    
    def init(self):
        """Add database constraints for data integrity"""
        super().init()
        
        # Add check constraint
        self.env.cr.execute("""
            ALTER TABLE sale_order 
            DROP CONSTRAINT IF EXISTS sale_order_amount_check
        """)
        self.env.cr.execute("""
            ALTER TABLE sale_order 
            ADD CONSTRAINT sale_order_amount_check 
            CHECK (amount_total >= 0)
        """)
```

### 3. Vacuum và Analyze

```python
def _vacuum_analyze_tables(self):
    """Optimize database tables"""
    tables = ['sale_order', 'sale_order_line', 'product_product']
    
    for table in tables:
        # Analyze table statistics
        self.env.cr.execute(f"ANALYZE {table}")
        
        # Vacuum (requires superuser in production)
        # self.env.cr.execute(f"VACUUM ANALYZE {table}")
```

---

## ORM Performance

### 1. Avoid N+1 Queries

```python
# ❌ BAD: N+1 queries
def get_order_info_bad(self):
    orders = self.env['sale.order'].search([])
    for order in orders:
        # Each iteration triggers a query
        partner_name = order.partner_id.name
        line_count = len(order.order_line)

# ✅ GOOD: Prefetch related data
def get_order_info_good(self):
    orders = self.env['sale.order'].search([])
    
    # Prefetch related fields
    orders.mapped('partner_id.name')
    orders.mapped('order_line')
    
    for order in orders:
        partner_name = order.partner_id.name
        line_count = len(order.order_line)

# ✅ BETTER: Use read() for specific fields
def get_order_info_better(self):
    orders = self.env['sale.order'].search([])
    
    # Read only needed fields
    data = orders.read(['name', 'partner_id', 'amount_total'])
    
    return data
```

### 2. Batch Operations

```python
class ProductUpdate(models.TransientModel):
    _name = 'product.update.wizard'
    
    def update_products_batch(self):
        """Update products in batches"""
        Product = self.env['product.product']
        
        # Get all products
        product_ids = Product.search([]).ids
        
        # Process in batches of 1000
        batch_size = 1000
        for i in range(0, len(product_ids), batch_size):
            batch_ids = product_ids[i:i + batch_size]
            products = Product.browse(batch_ids)
            
            # Batch update
            products.write({'active': True})
            
            # Commit every batch to free memory
            self.env.cr.commit()
            
            _logger.info(f'Processed {i + len(batch_ids)}/{len(product_ids)} products')
```

### 3. Use SQL for Complex Queries

```python
def get_sales_statistics(self, date_from, date_to):
    """Get sales statistics using SQL wrapper (Odoo 19)"""
    from odoo.tools import SQL
    
    query = SQL("""
        SELECT 
            p.id,
            p.name,
            SUM(sol.product_uom_qty) as total_qty,
            SUM(sol.price_subtotal) as total_amount,
            COUNT(DISTINCT so.id) as order_count
        FROM sale_order_line sol
        JOIN sale_order so ON sol.order_id = so.id
        JOIN product_product pp ON sol.product_id = pp.id
        JOIN product_template p ON pp.product_tmpl_id = p.id
        WHERE so.date_order >= %s 
          AND so.date_order <= %s
          AND so.state IN ('sale', 'done')
        GROUP BY p.id, p.name
        ORDER BY total_amount DESC
        LIMIT 100
    """, date_from, date_to)
    
    return self.env.execute_query(query)
```

### 4. Optimize Computed Fields

```python
class SaleOrder(models.Model):
    _inherit = 'sale.order'
    
    # ❌ BAD: Recomputes on every access
    @api.depends('order_line.price_subtotal')
    def _compute_total_bad(self):
        for order in self:
            total = 0
            for line in order.order_line:
                total += line.price_subtotal
            order.total_amount = total
    
    # ✅ GOOD: Use efficient aggregation
    @api.depends('order_line.price_subtotal')
    def _compute_total_good(self):
        for order in self:
            order.total_amount = sum(order.order_line.mapped('price_subtotal'))
    
    # ✅ BETTER: Store computed field if accessed frequently
    total_amount = fields.Float(
        string='Total Amount',
        compute='_compute_total_good',
        store=True,  # Store in database
        readonly=True
    )
```

---

## Caching Strategies

### 1. Method Caching với @tools.ormcache

```python
from odoo import models, tools

class Product(models.Model):
    _inherit = 'product.product'
    
    @tools.ormcache('self.id')
    def _get_product_price(self):
        """Cache product price calculation"""
        # Expensive calculation
        price = self._calculate_complex_price()
        return price
    
    @tools.ormcache('category_id')
    def get_category_products(self, category_id):
        """Cache category products"""
        return self.search([('categ_id', '=', category_id)]).ids
    
    def write(self, vals):
        """Clear cache on write"""
        # Clear specific cache
        if 'list_price' in vals:
            self._get_product_price.clear_cache(self)
        
        return super().write(vals)
```

### 2. Redis Caching (External)

```python
import redis
import json
from odoo import models

class ProductCached(models.Model):
    _inherit = 'product.product'
    
    def _get_redis_client(self):
        """Get Redis client"""
        return redis.Redis(
            host='localhost',
            port=6379,
            db=0,
            decode_responses=True
        )
    
    def get_product_data_cached(self):
        """Get product data with Redis cache"""
        cache_key = f'product:{self.id}'
        redis_client = self._get_redis_client()
        
        # Try to get from cache
        cached_data = redis_client.get(cache_key)
        if cached_data:
            return json.loads(cached_data)
        
        # Calculate and cache
        data = {
            'id': self.id,
            'name': self.name,
            'price': self.list_price,
        }
        
        # Cache for 1 hour
        redis_client.setex(cache_key, 3600, json.dumps(data))
        
        return data
```

### 3. Memoization

```python
from functools import lru_cache

class ReportHelper(models.AbstractModel):
    _name = 'report.helper'
    
    @lru_cache(maxsize=128)
    def _get_tax_rate(self, tax_id):
        """Cache tax rate lookup"""
        tax = self.env['account.tax'].browse(tax_id)
        return tax.amount
    
    def calculate_with_tax(self, amount, tax_id):
        """Calculate amount with cached tax rate"""
        tax_rate = self._get_tax_rate(tax_id)
        return amount * (1 + tax_rate / 100)
```

---

## Query Optimization

### 1. Optimize Search Domains

```python
# ❌ BAD: Multiple searches
def get_active_products_bad(self):
    products = self.env['product.product'].search([])
    active_products = products.filtered(lambda p: p.active)
    return active_products

# ✅ GOOD: Single optimized search
def get_active_products_good(self):
    return self.env['product.product'].search([
        ('active', '=', True)
    ])

# ✅ BETTER: Use database indexes
def get_active_sale_products(self):
    return self.env['product.product'].search([
        ('active', '=', True),
        ('sale_ok', '=', True),
    ], order='name')
```

### 2. Limit Results

```python
def get_recent_orders(self, limit=100):
    """Get recent orders with limit"""
    return self.env['sale.order'].search(
        [('state', '=', 'sale')],
        limit=limit,
        order='date_order desc'
    )

def get_paginated_products(self, page=1, page_size=20):
    """Paginated product listing"""
    offset = (page - 1) * page_size
    
    products = self.env['product.product'].search(
        [],
        limit=page_size,
        offset=offset,
        order='name'
    )
    
    total = self.env['product.product'].search_count([])
    
    return {
        'products': products,
        'total': total,
        'page': page,
        'pages': (total + page_size - 1) // page_size,
    }
```

### 3. Use search_fetch() Instead of search() + read() (Odoo 19)

```python
# ❌ BAD: Two operations
def get_partner_data_bad(self):
    partners = self.env['res.partner'].search([('customer_rank', '>', 0)])
    return partners.read(['name', 'email', 'phone'])

# ✅ GOOD: search_read (legacy, still works)
def get_partner_data_good(self):
    return self.env['res.partner'].search_read(
        domain=[('customer_rank', '>', 0)],
        fields=['name', 'email', 'phone'],
        limit=100
    )

# ✅ BEST (Odoo 19): search_fetch (fewer SQL queries)
def get_partner_data_best(self):
    partners = self.env['res.partner'].search_fetch(
        [('customer_rank', '>', 0)],
        ['name', 'email', 'phone'],
        limit=100,
    )
    # Returns recordset with pre-fetched fields
    return partners
```

---

## Frontend Performance

### 1. Lazy Loading

```xml
<!-- Lazy load images -->
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

### 2. Optimize List Views

```xml
<!-- Optimize tree view -->
<record id="view_product_tree_optimized" model="ir.ui.view">
    <field name="name">product.product.tree.optimized</field>
    <field name="model">product.product</field>
    <field name="arch" type="xml">
        <list 
            limit="80"
            default_order="name"
            decoration-muted="not active">
            
            <!-- Only essential fields -->
            <field name="name"/>
            <field name="default_code"/>
            <field name="list_price"/>
            <field name="active" invisible="1"/>
        </list>
    </field>
</record>
```

### 3. Reduce RPC Calls

```javascript
// ❌ BAD: Multiple RPC calls
async loadDataBad() {
    const products = await this.rpc('/web/dataset/search_read', {
        model: 'product.product',
        fields: ['name'],
    });
    
    for (let product of products) {
        const details = await this.rpc('/web/dataset/call_kw', {
            model: 'product.product',
            method: 'read',
            args: [[product.id], ['description']],
        });
    }
}

// ✅ GOOD: Single RPC call
async loadDataGood() {
    const products = await this.rpc('/web/dataset/search_read', {
        model: 'product.product',
        fields: ['name', 'description'],
    });
}
```

---

## Profiling & Monitoring

### 1. Python Profiling

```python
import cProfile
import pstats
from io import StringIO

def profile_method(self):
    """Profile a method"""
    profiler = cProfile.Profile()
    profiler.enable()
    
    # Your code here
    self._expensive_operation()
    
    profiler.disable()
    
    # Print stats
    s = StringIO()
    ps = pstats.Stats(profiler, stream=s).sort_stats('cumulative')
    ps.print_stats(20)
    
    _logger.info(s.getvalue())
```

### 2. SQL Query Logging

```python
# In odoo.conf
[options]
log_level = debug
log_db = True
log_db_level = debug

# Or programmatically
import logging
_logger = logging.getLogger(__name__)

def debug_queries(self):
    """Log SQL queries"""
    # Enable query logging
    self.env.cr._obj.notices = []
    
    # Your code
    orders = self.env['sale.order'].search([])
    
    # Log queries
    for notice in self.env.cr._obj.notices:
        _logger.debug(notice)
```

### 3. Performance Monitoring

```python
import time
from contextlib import contextmanager

@contextmanager
def timer(name):
    """Context manager for timing operations"""
    start = time.time()
    try:
        yield
    finally:
        duration = time.time() - start
        _logger.info(f'{name} took {duration:.2f} seconds')

def process_orders(self):
    """Process orders with timing"""
    with timer('Order Processing'):
        orders = self.env['sale.order'].search([])
        
        with timer('Order Validation'):
            orders.action_confirm()
        
        with timer('Invoice Creation'):
            orders._create_invoices()
```

### 4. Memory Profiling

```python
from memory_profiler import profile

class MemoryIntensiveOperation(models.Model):
    _name = 'memory.operation'
    
    @profile
    def process_large_dataset(self):
        """Profile memory usage"""
        # Load large dataset
        products = self.env['product.product'].search([])
        
        # Process in chunks to reduce memory
        chunk_size = 1000
        for i in range(0, len(products), chunk_size):
            chunk = products[i:i + chunk_size]
            self._process_chunk(chunk)
            
            # Clear cache
            self.env.cache.invalidate()
```

---

## Best Practices

### 1. Efficient Record Processing

```python
class EfficientProcessing(models.Model):
    _name = 'efficient.processing'
    
    def process_records_efficiently(self):
        """Best practices for record processing"""
        # 1. Use search_count for counting
        count = self.env['sale.order'].search_count([('state', '=', 'draft')])
        
        # 2. Use exists() instead of bool(record)
        order = self.env['sale.order'].browse(123)
        if order.exists():
            # Process order
            pass
        
        # 3. Use mapped() for field extraction
        partner_ids = orders.mapped('partner_id.id')
        
        # 4. Use filtered() efficiently
        confirmed_orders = orders.filtered(lambda o: o.state == 'sale')
        
        # 5. Use sorted() for in-memory sorting
        sorted_orders = orders.sorted(key=lambda o: o.date_order, reverse=True)
```

### 2. Database Transaction Management

```python
def batch_update_with_savepoints(self):
    """Use savepoints for batch operations"""
    orders = self.env['sale.order'].search([('state', '=', 'draft')])
    
    for order in orders:
        try:
            # Create savepoint
            self.env.cr.execute('SAVEPOINT batch_update')
            
            # Process order
            order.action_confirm()
            
            # Release savepoint
            self.env.cr.execute('RELEASE SAVEPOINT batch_update')
            
        except Exception as e:
            # Rollback to savepoint
            self.env.cr.execute('ROLLBACK TO SAVEPOINT batch_update')
            _logger.error(f'Failed to process order {order.name}: {e}')
```

### 3. Optimize Onchange Methods

```python
class OptimizedOnchange(models.Model):
    _name = 'optimized.onchange'
    
    # ❌ BAD: Heavy computation on every change
    @api.onchange('product_id')
    def _onchange_product_bad(self):
        if self.product_id:
            # Expensive calculation
            self.price = self._calculate_complex_price()
    
    # ✅ GOOD: Cache and optimize
    @api.onchange('product_id')
    def _onchange_product_good(self):
        if self.product_id:
            # Use cached value if available
            self.price = self.product_id.list_price
```

---

## Ví Dụ Thực Tế

### 1. Optimize Product Search

```python
class ProductSearch(models.Model):
    _inherit = 'product.product'
    
    @api.model
    def search_products_optimized(self, query, limit=20):
        """Optimized product search"""
        # Use PostgreSQL full-text search
        self.env.cr.execute("""
            SELECT id, name, default_code, list_price
            FROM product_product pp
            JOIN product_template pt ON pp.product_tmpl_id = pt.id
            WHERE 
                pt.active = true
                AND pp.active = true
                AND (
                    pt.name ILIKE %s
                    OR pp.default_code ILIKE %s
                    OR pt.barcode ILIKE %s
                )
            ORDER BY 
                CASE 
                    WHEN pt.name ILIKE %s THEN 1
                    WHEN pp.default_code ILIKE %s THEN 2
                    ELSE 3
                END,
                pt.name
            LIMIT %s
        """, (
            f'%{query}%', f'%{query}%', f'%{query}%',
            f'{query}%', f'{query}%',
            limit
        ))
        
        return self.env.cr.dictfetchall()
```

### 2. Optimize Report Generation

```python
class SalesReport(models.AbstractModel):
    _name = 'report.my_module.sales_report'
    
    @api.model
    def _get_report_values(self, docids, data=None):
        """Optimized report data preparation"""
        # Use SQL for aggregation
        self.env.cr.execute("""
            SELECT 
                p.id,
                p.name as product_name,
                SUM(sol.product_uom_qty) as total_qty,
                SUM(sol.price_subtotal) as total_amount,
                COUNT(DISTINCT so.partner_id) as customer_count
            FROM sale_order_line sol
            JOIN sale_order so ON sol.order_id = so.id
            JOIN product_product pp ON sol.product_id = pp.id
            JOIN product_template p ON pp.product_tmpl_id = p.id
            WHERE so.id IN %s
            GROUP BY p.id, p.name
            ORDER BY total_amount DESC
        """, (tuple(docids),))
        
        product_stats = self.env.cr.dictfetchall()
        
        return {
            'doc_ids': docids,
            'docs': self.env['sale.order'].browse(docids),
            'product_stats': product_stats,
        }
```

### 3. Background Job Processing

```python
from odoo import models, api
from odoo.addons.queue_job.job import job

class BackgroundProcessing(models.Model):
    _name = 'background.processing'
    
    # Odoo 19: Use _commit_progress in cron methods for batch commits
    def _cron_process_large_import(self):
        """Process large import with _commit_progress (Odoo 19)"""
        records = self.search([('state', '=', 'pending')])
        
        for record in records:
            record._process_single()
            # Commits progress periodically & updates cron state
            self._commit_progress()
    
    @job
    def process_large_import(self, file_data):
        """Process large import in background (requires queue_job)"""
        chunk_size = 1000
        lines = file_data.split('\n')
        
        for i in range(0, len(lines), chunk_size):
            chunk = lines[i:i + chunk_size]
            self._process_chunk(chunk)
            self.env.cr.commit()
    
    def trigger_background_job(self, file_data):
        """Trigger background processing"""
        self.with_delay().process_large_import(file_data)
```

---

## 🎯 Performance Checklist

Khi optimize performance, kiểm tra:

- [ ] Database indexes cho frequently searched fields
- [ ] Avoid N+1 queries (use prefetch, mapped)
- [ ] Use read() hoặc search_fetch() thay vì browse() (Odoo 19)
- [ ] Implement caching cho expensive operations
- [ ] Use SQL cho complex aggregations
- [ ] Store computed fields nếu accessed frequently
- [ ] Batch operations và commit periodically
- [ ] Limit recordsets và use pagination
- [ ] Profile code để identify bottlenecks
- [ ] Monitor query performance in production
- [ ] Optimize frontend (lazy loading, reduce RPC)
- [ ] Use background jobs cho heavy tasks

---

## 📚 Tài Liệu Tham Khảo

- [Odoo Performance Guidelines](https://www.odoo.com/documentation/19.0/developer/reference/backend/performance.html)
- [PostgreSQL Performance Tips](https://wiki.postgresql.org/wiki/Performance_Optimization)
- [Python Profiling](https://docs.python.org/3/library/profile.html)
- [ORM Cache](https://www.odoo.com/documentation/19.0/developer/reference/backend/orm.html#performance)

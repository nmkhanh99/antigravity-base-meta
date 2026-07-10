---
name: Odoo 19 Debugging Guide
description: Hướng dẫn toàn diện về debugging trong Odoo 19 - debug mode, logging, pdb, profiling, và troubleshooting techniques
---

# Odoo 19 Debugging - Reference Guide

## Mục Lục

1. [Debug Mode](#debug-mode)
2. [Python Debugging](#python-debugging)
3. [Logging](#logging)
4. [Database Debugging](#database-debugging)
5. [Frontend Debugging](#frontend-debugging)
6. [Common Issues](#common-issues)
7. [Performance Debugging](#performance-debugging)
8. [Debugging Checklist](#debugging-checklist)

---

## Debug Mode

### Enable Debug Mode

```
# Method 1: URL parameter (recommended)
http://localhost:8069/web?debug=1

# Method 2: Assets debug (JS/CSS source maps)
http://localhost:8069/web?debug=assets

# Method 3: Tests mode
http://localhost:8069/web?debug=tests

# Method 4: Settings menu
Settings → Developer Tools → Activate Developer Mode
```

### Debug Mode Features

Khi bật debug mode:
- View metadata (Edit View, Edit Action)
- Field technical info
- Technical menu (Settings → Technical)
- Database query counter
- Asset source maps

### Check Debug Mode in Code

```python
from odoo import http

class MyController(http.Controller):
    @http.route('/my/route', auth='user')
    def my_route(self, **kwargs):
        if http.request.session.debug:
            # Debug mode is active
            pass
```

---

## Python Debugging

### 1. PDB - Python Debugger

```python
from odoo import models

class ProductDebug(models.Model):
    _inherit = 'product.product'

    def compute_price(self):
        breakpoint()  # Python 3.7+ — preferred over pdb.set_trace()
        # Hoặc: import pdb; pdb.set_trace()

        price = self.list_price
        discount = self._get_discount()
        return price * (1 - discount)

# PDB Commands:
# n (next)        - Execute next line
# s (step)        - Step into function
# c (continue)    - Continue execution
# p variable      - Print variable value
# pp variable     - Pretty print variable
# l               - List source code
# w               - Show stack trace
# q               - Quit debugger
# self.env['res.partner'].search([])  - Execute ORM in context
```

### 2. Conditional Breakpoints

```python
def process_orders(self):
    orders = self.env['sale.order'].search([])

    for order in orders:
        # Chỉ break cho điều kiện cụ thể
        if order.amount_total > 10000:
            breakpoint()

        order.action_confirm()
```

### 3. Remote Debugging với VS Code

```json
// .vscode/launch.json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Odoo: Attach",
            "type": "python",
            "request": "attach",
            "port": 5678,
            "host": "localhost",
            "pathMappings": [
                {
                    "localRoot": "${workspaceFolder}",
                    "remoteRoot": "/opt/odoo"
                }
            ]
        }
    ]
}
```

```python
# Trong Odoo code — chỉ dùng khi dev, KHÔNG deploy production
import debugpy

debugpy.listen(("0.0.0.0", 5678))
print("Waiting for debugger attach...")
debugpy.wait_for_client()
```

---

## Logging

### 1. Basic Logging

```python
import logging
from odoo import models

_logger = logging.getLogger(__name__)

class ProductLogging(models.Model):
    _inherit = 'product.product'

    def update_price(self, new_price):
        old_price = self.list_price

        # Dùng %s, KHÔNG dùng f-string (lazy evaluation, tránh overhead khi log level cao)
        _logger.debug('Updating price for %s', self.name)
        _logger.info('Price changed: %s to %s', old_price, new_price)

        if new_price < old_price * 0.5:
            _logger.warning('Large price decrease for product %s', self.name)

        try:
            self.list_price = new_price
        except Exception as e:
            _logger.error('Failed to update price for %s: %s', self.name, e)
            raise

        if new_price <= 0:
            _logger.critical('Invalid price %s for product %s', new_price, self.name)
```

### 2. Exception Logging (tự động include traceback)

```python
def risky_operation(self):
    try:
        result = self._do_something()
    except Exception:
        # _logger.exception tự động ghi traceback đầy đủ
        _logger.exception('Operation failed for record %s', self.id)
        raise
```

### 3. Log Configuration (odoo.conf)

```ini
[options]
# Log level: debug, info, warning, error, critical
log_level = info

# Log to file
logfile = /var/log/odoo/odoo.log

# Per-module log level (comma-separated)
log_handler = :INFO,werkzeug:WARNING,odoo.addons.my_module:DEBUG

# CLI equivalent
# ./odoo-bin --log-level=debug --log-handler=odoo.addons.my_module:DEBUG
```

---

## Database Debugging

### 1. SQL Query Logging

```python
# Enable SQL logging programmatically
import logging
logging.getLogger('odoo.sql_db').setLevel(logging.DEBUG)
```

```ini
# Hoặc trong odoo.conf
log_handler = odoo.sql_db:DEBUG
```

### 2. Query Count

```python
def check_query_count(self):
    cr = self.env.cr
    start = cr.sql_log_count

    # ... operations ...
    products = self.env['product.product'].search([])
    for product in products:
        _ = product.name  # N+1 query nếu không prefetch

    _logger.info('Queries executed: %d', cr.sql_log_count - start)
```

### 3. EXPLAIN ANALYZE cho slow queries

```python
def debug_slow_query(self):
    query = """
        SELECT p.id, p.name, COUNT(sol.id) as order_count
        FROM product_product p
        LEFT JOIN sale_order_line sol ON sol.product_id = p.id
        GROUP BY p.id, p.name
        ORDER BY order_count DESC
    """

    self.env.cr.execute('EXPLAIN ANALYZE ' + query)
    plan = self.env.cr.fetchall()

    for line in plan:
        _logger.debug(line[0])
```

### 4. Database Inspection

```python
def inspect_table_sizes(self):
    self.env.cr.execute("""
        SELECT
            tablename,
            pg_size_pretty(pg_total_relation_size('public.'||tablename)) as size
        FROM pg_tables
        WHERE schemaname = 'public'
        ORDER BY pg_total_relation_size('public.'||tablename) DESC
        LIMIT 10
    """)
    rows = self.env.cr.fetchall()
    for table, size in rows:
        _logger.info('Table %s: %s', table, size)

def inspect_indexes(self, table_name):
    self.env.cr.execute("""
        SELECT indexname, indexdef
        FROM pg_indexes
        WHERE schemaname = 'public'
        AND tablename = %s
    """, (table_name,))
    for indexname, indexdef in self.env.cr.fetchall():
        _logger.info('Index %s: %s', indexname, indexdef)
```

### 5. Odoo Shell

```bash
# Start interactive shell
./odoo-bin shell -d mydb

# Docker
docker-compose exec web odoo shell -d mydb
```

```python
# Trong Odoo shell
>>> env['product.product'].search_count([])
>>> product = env['product.product'].browse(1)
>>> product.name
>>> product.write({'list_price': 100})
>>> env.cr.commit()  # Commit transaction
```

---

## Frontend Debugging

### 1. OWL Component Debugging (Odoo 19)

```javascript
/** @odoo-module **/

import { Component, onMounted, onWillStart, useState } from "@odoo/owl";

export class MyComponent extends Component {
    static template = "my_module.MyComponent";

    setup() {
        this.state = useState({ loading: false });

        onWillStart(async () => {
            console.log("Component will start, props:", this.props);
        });

        onMounted(() => {
            console.log("Component mounted");
        });
    }

    async loadData() {
        try {
            this.state.loading = true;
            const result = await this.env.services.rpc('/my/endpoint', {
                params: { id: 123 }
            });
            console.log("Data loaded:", result);
        } catch (error) {
            console.error("Failed to load data:", error);
        } finally {
            this.state.loading = false;
        }
    }
}
```

### 2. QWeb Template Debugging

```xml
<!-- t-debug attribute — chỉ dùng trong dev -->
<template id="debug_template">
    <t t-if="debug_mode">
        <pre style="background:#f5f5f5;padding:1rem;">
            <t t-esc="json.dumps(data, indent=2)"/>
        </pre>
    </t>
</template>
```

### 3. Browser Console Tips

```javascript
// Odoo debug shortcuts trong browser console
odoo.debug           // Check if debug mode is on
odoo.__DEBUG__       // Internal debug object

// Network inspection
// Chrome DevTools → Network → XHR/Fetch
// Lọc theo /web/dataset/call_kw để xem RPC calls
```

---

## Common Issues

### 1. Module Not Loading

```python
def debug_module_loading(self):
    module_name = 'my_module'

    module = self.env['ir.module.module'].search([
        ('name', '=', module_name)
    ])

    if not module:
        _logger.error('Module %s not found', module_name)
        return

    _logger.info('Module state: %s, version: %s', module.state, module.latest_version)

    for dep in module.dependencies_id:
        _logger.info('Dependency: %s (state: %s)', dep.name, dep.state)
```

**Checklist khi module không load:**
- Kiểm tra `__manifest__.py` syntax
- Kiểm tra `depends` list có đầy đủ không
- Xem server logs khi restart Odoo
- Chạy `./odoo-bin -u my_module` để update

### 2. View Not Rendering

```python
def debug_view_issues(self):
    model = 'product.product'
    view_type = 'form'

    views = self.env['ir.ui.view'].search([
        ('model', '=', model),
        ('type', '=', view_type),
        ('active', '=', True),
    ])

    for view in views:
        _logger.info('View: %s (priority: %s)', view.name, view.priority)
        for child in view.inherit_children_ids:
            _logger.info('  Inherited by: %s (%s)', child.name, child.module)
```

**Checklist khi view không render:**
- Kiểm tra XML syntax với `xmllint`
- Xem `ir.ui.view` trong debug mode (Settings → Technical → Views)
- Kiểm tra `inherit_id` có đúng view ref không
- Tìm conflict với view khác cùng priority

### 3. Access Rights Issues

```python
from odoo.exceptions import AccessError

def debug_access_rights(self):
    model = 'sale.order'
    user = self.env.user

    _logger.info('Checking access for user: %s (groups: %s)',
                 user.name,
                 user.groups_id.mapped('full_name'))

    for operation in ['read', 'write', 'create', 'unlink']:
        try:
            self.env[model].check_access_rights(operation)
            _logger.info('%s: %s has %s access', model, user.name, operation)
        except AccessError:
            _logger.warning('%s: %s DENIED %s access', model, user.name, operation)

    # Check number of accessible records
    record_count = self.env[model].search_count([])
    _logger.info('User can see %d records of %s', record_count, model)
```

**Checklist access errors:**
- Kiểm tra `ir.model.access.csv` có entry cho model không
- Kiểm tra `ir.rule` (record rules) không filter out record
- Xem `Security → Access Rights` trong debug mode
- Dùng superuser context để verify data tồn tại: `self.env[model].sudo().search_count([])`

---

## Performance Debugging

### 1. Timer Context Manager

```python
import time
from contextlib import contextmanager

@contextmanager
def timer(name):
    start = time.time()
    try:
        yield
    finally:
        duration = time.time() - start
        _logger.info('%s took %.3fs', name, duration)

def debug_performance(self):
    with timer('Total execution'):
        with timer('Search products'):
            products = self.env['product.product'].search([])

        with timer('Process products'):
            # Tránh N+1: prefetch bằng cách access field trên recordset
            names = products.mapped('name')

        _logger.info('Processed %d products', len(products))
```

### 2. N+1 Query Detection

```python
def detect_n_plus_1(self):
    # BAD: N+1 queries
    cr = self.env.cr
    start = cr.sql_log_count

    orders = self.env['sale.order'].search([], limit=100)
    for order in orders:
        _ = order.partner_id.name  # 1 query per order

    bad_count = cr.sql_log_count - start
    _logger.warning('N+1 pattern: %d queries for %d orders', bad_count, len(orders))

    # GOOD: Prefetch with mapped
    start = cr.sql_log_count
    orders = self.env['sale.order'].search([], limit=100)
    names = orders.mapped('partner_id.name')  # 2 queries total

    good_count = cr.sql_log_count - start
    _logger.info('Prefetch pattern: %d queries for %d orders', good_count, len(orders))
```

### 3. Profiling với cProfile

```python
import cProfile
import pstats
import io

def profile_method(self):
    pr = cProfile.Profile()
    pr.enable()

    # Code to profile
    products = self.env['product.product'].search([])
    for p in products:
        p._compute_display_name()

    pr.disable()

    s = io.StringIO()
    ps = pstats.Stats(pr, stream=s).sort_stats('cumulative')
    ps.print_stats(20)
    _logger.info('Profile results:\n%s', s.getvalue())
```

---

## Debugging Checklist

Khi gặp issue, follow checklist này theo thứ tự:

- [ ] Bật debug mode (`?debug=1`)
- [ ] Kiểm tra server logs (`tail -f /var/log/odoo/odoo.log`)
- [ ] Verify module installed và không có lỗi upgrade
- [ ] Kiểm tra XML syntax của views
- [ ] Verify access rights (ir.model.access.csv, ir.rule)
- [ ] Test trong Odoo shell với superuser
- [ ] Add `breakpoint()` tại điểm nghi ngờ
- [ ] Log SQL queries nếu nghi performance issue
- [ ] Kiểm tra browser console cho JS errors
- [ ] Test với user khác (admin vs regular user)
- [ ] Kiểm tra data integrity trong database

---

## Debug Helpers Pattern

```python
class DebugMixin(models.AbstractModel):
    _name = 'debug.mixin'

    def debug_record(self):
        """In toàn bộ field values của record"""
        _logger.info('%s(id=%s):', self._name, self.id)
        for fname, field in self._fields.items():
            try:
                value = self[fname]
                _logger.info('  %s = %r', fname, value)
            except Exception:
                _logger.info('  %s = <error reading field>', fname)

    def debug_context(self):
        """In context hiện tại"""
        import json
        _logger.info('Context: %s', json.dumps(
            dict(self.env.context), indent=2, default=str
        ))
```

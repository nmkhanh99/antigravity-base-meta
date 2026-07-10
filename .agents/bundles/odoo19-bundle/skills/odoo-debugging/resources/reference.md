---
name: Odoo Debugging
description: Hướng dẫn toàn diện về debugging trong Odoo - debug mode, logging, pdb, profiling, và troubleshooting techniques
---

# Odoo Debugging Skill

## 📋 Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [Debug Mode](#debug-mode)
3. [Python Debugging](#python-debugging)
4. [Logging](#logging)
5. [Database Debugging](#database-debugging)
6. [Frontend Debugging](#frontend-debugging)
7. [Common Issues](#common-issues)
8. [Debugging Tools](#debugging-tools)
9. [Best Practices](#best-practices)

---

## Tổng Quan

### Debug Techniques

1. **Debug Mode** - Odoo's built-in debug features
2. **Logging** - Python logging framework
3. **PDB** - Python debugger
4. **SQL Logging** - Database query inspection
5. **Browser DevTools** - Frontend debugging
6. **Profiling** - Performance analysis

### Common Debugging Scenarios

- ❌ Module not loading
- ❌ View not rendering
- ❌ Computed field not updating
- ❌ Security access errors
- ❌ Performance issues
- ❌ JavaScript errors

---

## Debug Mode

### 1. Enable Debug Mode

```python
# Method 1: URL parameter
https://yourdomain.com/web?debug=1

# Method 2: Developer menu
Settings → Activate Developer Mode

# Method 3: Assets debug
https://yourdomain.com/web?debug=assets

# Method 4: Tests mode
https://yourdomain.com/web?debug=tests
```

### 2. Debug Mode Features

```python
# In debug mode, you get:
# - View metadata (Edit View, Edit Action, etc.)
# - Field technical info
# - Access to technical menu
# - View structure inspector
# - Database query counter
# - Asset debugging

# Check if debug mode is active
from odoo import http

if http.request.session.debug:
    # Debug mode is active
    pass
```

### 3. Debug Mode in Code

```python
from odoo import models, api

class DebugHelper(models.Model):
    _name = 'debug.helper'
    
    def debug_info(self):
        """Get debug information"""
        return {
            'debug_mode': self.env.context.get('debug'),
            'user': self.env.user.name,
            'company': self.env.company.name,
            'lang': self.env.lang,
            'context': self.env.context,
        }
    
    def view_metadata(self):
        """View technical metadata"""
        # Available in debug mode
        return {
            'model': self._name,
            'fields': self.fields_get(),
            'views': self.env['ir.ui.view'].search([
                ('model', '=', self._name)
            ]).read(['name', 'type', 'arch']),
        }
```

---

## Python Debugging

### 1. Using PDB (Python Debugger)

```python
from odoo import models
import pdb

class ProductDebug(models.Model):
    _inherit = 'product.product'
    
    def compute_price(self):
        """Debug price computation"""
        # Set breakpoint
        pdb.set_trace()
        
        # Code execution will pause here
        price = self.list_price
        discount = self._get_discount()
        final_price = price * (1 - discount)
        
        return final_price

# PDB Commands:
# n (next) - Execute next line
# s (step) - Step into function
# c (continue) - Continue execution
# p variable - Print variable value
# pp variable - Pretty print variable
# l - List source code
# w - Show stack trace
# q - Quit debugger
```

### 2. Using ipdb (Enhanced PDB)

```python
# Install ipdb
# pip install ipdb

from odoo import models
import ipdb

class OrderDebug(models.Model):
    _inherit = 'sale.order'
    
    def action_confirm(self):
        """Debug order confirmation"""
        # Enhanced breakpoint with syntax highlighting
        ipdb.set_trace()
        
        result = super().action_confirm()
        return result

# ipdb features:
# - Syntax highlighting
# - Tab completion
# - Better interface
```

### 3. Conditional Breakpoints

```python
def process_orders(self):
    """Debug specific orders"""
    orders = self.env['sale.order'].search([])
    
    for order in orders:
        # Break only for specific condition
        if order.amount_total > 10000:
            import pdb; pdb.set_trace()
        
        order.action_confirm()
```

### 4. Remote Debugging với VS Code

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
# In your Odoo code
import debugpy

# Start debug server
debugpy.listen(("0.0.0.0", 5678))
print("Waiting for debugger attach...")
debugpy.wait_for_client()

# Your code here
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
        """Update price with logging"""
        old_price = self.list_price
        
        # Debug level
        _logger.debug(f'Updating price for {self.name}')
        
        # Info level
        _logger.info(f'Price changed: {old_price} → {new_price}')
        
        # Warning level
        if new_price < old_price * 0.5:
            _logger.warning(f'Large price decrease for {self.name}')
        
        # Error level
        try:
            self.list_price = new_price
        except Exception as e:
            _logger.error(f'Failed to update price: {e}')
            raise
        
        # Critical level
        if new_price <= 0:
            _logger.critical(f'Invalid price for {self.name}: {new_price}')
```

### 2. Structured Logging

```python
class StructuredLogging(models.Model):
    _name = 'structured.logging'
    
    def process_with_logging(self):
        """Structured logging example"""
        _logger.info('Starting process', extra={
            'user_id': self.env.user.id,
            'company_id': self.env.company.id,
            'context': self.env.context,
        })
        
        try:
            result = self._do_processing()
            
            _logger.info('Process completed', extra={
                'result': result,
                'duration': 1.23,
            })
            
        except Exception as e:
            _logger.exception('Process failed', extra={
                'error_type': type(e).__name__,
                'error_message': str(e),
            })
            raise
```

### 3. Log Configuration

```ini
# odoo.conf
[options]
# Log level: debug, info, warning, error, critical
log_level = info

# Log to file
logfile = /var/log/odoo/odoo.log

# Log format
log_handler = :INFO,werkzeug:WARNING,odoo.addons:DEBUG

# Specific module logging
log_handler = :INFO,odoo.addons.my_module:DEBUG

# Disable specific loggers
log_handler = :INFO,werkzeug:CRITICAL
```

### 4. Custom Log Handler

```python
import logging

class OdooLogHandler(logging.Handler):
    """Custom log handler to store logs in database"""
    
    def emit(self, record):
        """Store log record in database"""
        try:
            env = odoo.api.Environment(
                odoo.registry(record.dbname),
                odoo.SUPERUSER_ID,
                {}
            )
            
            env['system.log'].create({
                'level': record.levelname,
                'message': record.getMessage(),
                'module': record.name,
                'function': record.funcName,
                'line': record.lineno,
            })
            
        except Exception:
            # Don't break on logging errors
            pass

# Register handler
handler = OdooLogHandler()
logging.getLogger('odoo.addons').addHandler(handler)
```

---

## Database Debugging

### 1. SQL Query Logging

```python
# Enable SQL logging in odoo.conf
[options]
log_db = True
log_db_level = debug

# Or programmatically
import logging
logging.getLogger('odoo.sql_db').setLevel(logging.DEBUG)
```

### 2. Query Analysis

```python
class QueryDebug(models.Model):
    _name = 'query.debug'
    
    def analyze_queries(self):
        """Analyze SQL queries"""
        # Enable query counting
        self.env.cr._obj.queries = []
        
        # Your code
        products = self.env['product.product'].search([])
        for product in products:
            _ = product.name
        
        # Print queries
        for query in self.env.cr._obj.queries:
            _logger.debug(f'Query: {query}')
        
        _logger.info(f'Total queries: {len(self.env.cr._obj.queries)}')
```

### 3. EXPLAIN ANALYZE

```python
def debug_slow_query(self):
    """Debug slow queries with EXPLAIN"""
    query = """
        SELECT p.id, p.name, COUNT(sol.id) as order_count
        FROM product_product p
        LEFT JOIN sale_order_line sol ON sol.product_id = p.id
        GROUP BY p.id, p.name
        ORDER BY order_count DESC
    """
    
    # Get query plan
    self.env.cr.execute(f'EXPLAIN ANALYZE {query}')
    plan = self.env.cr.fetchall()
    
    for line in plan:
        _logger.debug(line[0])
```

### 4. Database Inspection

```python
def inspect_database(self):
    """Inspect database structure"""
    # List all tables
    self.env.cr.execute("""
        SELECT table_name 
        FROM information_schema.tables 
        WHERE table_schema = 'public'
    """)
    tables = [row[0] for row in self.env.cr.fetchall()]
    
    # Get table size
    self.env.cr.execute("""
        SELECT 
            schemaname,
            tablename,
            pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename))
        FROM pg_tables
        WHERE schemaname = 'public'
        ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
        LIMIT 10
    """)
    
    # Check indexes
    self.env.cr.execute("""
        SELECT 
            tablename,
            indexname,
            indexdef
        FROM pg_indexes
        WHERE schemaname = 'public'
        AND tablename = 'product_product'
    """)
```

---

## Frontend Debugging

### 1. Browser DevTools

```javascript
// Console debugging
console.log('Debug message');
console.warn('Warning message');
console.error('Error message');
console.table(data);

// Debugger statement
function myFunction() {
    debugger; // Execution will pause here
    // Your code
}

// Network inspection
// Chrome DevTools → Network tab
// - View XHR requests
// - Check request/response
// - Monitor timing
```

### 2. Odoo JavaScript Debugging

```javascript
/** @odoo-module **/

import { Component } from "@odoo/owl";

export class MyComponent extends Component {
    setup() {
        console.log('Component setup', this.props);
        
        // Debug lifecycle
        onWillStart(() => {
            console.log('Component will start');
        });
        
        onMounted(() => {
            console.log('Component mounted');
        });
    }
    
    async loadData() {
        try {
            console.log('Loading data...');
            
            const result = await this.rpc('/my/endpoint', {
                params: { id: 123 }
            });
            
            console.log('Data loaded:', result);
            
        } catch (error) {
            console.error('Failed to load data:', error);
        }
    }
}
```

### 3. QWeb Template Debugging

```xml
<!-- Add debug info to templates -->
<template id="debug_template">
    <t t-debug="">
        <div class="debug-info">
            <p>Context: <t t-esc="context"/></p>
            <p>User: <t t-esc="user_id"/></p>
        </div>
    </t>
    
    <!-- Conditional debug output -->
    <t t-if="debug">
        <pre><t t-esc="json.dumps(data, indent=2)"/></pre>
    </t>
</template>
```

---

## Common Issues

### 1. Module Not Loading

```python
# Check module installation
def debug_module_loading(self):
    """Debug module loading issues"""
    module_name = 'my_module'
    
    # Check if module exists
    module = self.env['ir.module.module'].search([
        ('name', '=', module_name)
    ])
    
    if not module:
        _logger.error(f'Module {module_name} not found')
        return
    
    _logger.info(f'Module state: {module.state}')
    _logger.info(f'Module path: {module.latest_version}')
    
    # Check dependencies
    for dep in module.dependencies_id:
        _logger.info(f'Dependency: {dep.name} ({dep.state})')
    
    # Check for errors
    if module.state == 'uninstalled':
        _logger.warning('Module is not installed')
    elif module.state == 'to upgrade':
        _logger.warning('Module needs upgrade')
```

### 2. View Not Rendering

```python
def debug_view_issues(self):
    """Debug view rendering issues"""
    model = 'product.product'
    view_type = 'form'
    
    # Get view
    view = self.env['ir.ui.view'].search([
        ('model', '=', model),
        ('type', '=', view_type),
    ], limit=1)
    
    if not view:
        _logger.error(f'No {view_type} view found for {model}')
        return
    
    # Validate view
    try:
        self.env['ir.ui.view']._validate_view(view.arch)
        _logger.info('View is valid')
    except Exception as e:
        _logger.error(f'View validation failed: {e}')
    
    # Check inheritance
    for inherit in view.inherit_children_ids:
        _logger.info(f'Inherited by: {inherit.name}')
```

### 3. Access Rights Issues

```python
def debug_access_rights(self):
    """Debug access rights issues"""
    model = 'sale.order'
    user = self.env.user
    
    # Check model access
    try:
        self.env[model].check_access_rights('read')
        _logger.info(f'{user.name} has read access to {model}')
    except AccessError as e:
        _logger.error(f'Access denied: {e}')
    
    # Check record rules
    orders = self.env[model].search([])
    _logger.info(f'User can access {len(orders)} orders')
    
    # Check field access
    try:
        self.env[model].check_field_access_rights('write', ['amount_total'])
        _logger.info('User can write amount_total')
    except:
        _logger.error('User cannot write amount_total')
```

### 4. Performance Issues

```python
import time
from contextlib import contextmanager

@contextmanager
def timer(name):
    """Time code execution"""
    start = time.time()
    try:
        yield
    finally:
        duration = time.time() - start
        _logger.info(f'{name} took {duration:.2f}s')

def debug_performance(self):
    """Debug performance issues"""
    with timer('Total execution'):
        
        with timer('Database query'):
            products = self.env['product.product'].search([])
        
        with timer('Processing'):
            for product in products:
                product.list_price = product.list_price * 1.1
        
        with timer('Write operation'):
            products.flush()
```

---

## Debugging Tools

### 1. Odoo Shell

```bash
# Start Odoo shell
odoo-bin shell -d production

# Or with Docker
docker-compose exec web odoo shell -d production
```

```python
# In Odoo shell
>>> env['product.product'].search([])
>>> product = env['product.product'].browse(1)
>>> product.name
>>> product.write({'list_price': 100})
>>> env.cr.commit()
```

### 2. Scaffold Tool

```bash
# Create module structure
odoo-bin scaffold my_module /path/to/addons

# This creates:
# - __init__.py
# - __manifest__.py
# - models/
# - views/
# - security/
```

### 3. Database Manager

```python
# Access database manager
https://yourdomain.com/web/database/manager

# Features:
# - Create database
# - Duplicate database
# - Backup database
# - Restore database
# - Delete database
```

---

## Best Practices

### 1. Defensive Programming

```python
class DefensiveCoding(models.Model):
    _name = 'defensive.coding'
    
    def safe_operation(self):
        """Defensive programming example"""
        # Check preconditions
        if not self.exists():
            _logger.warning('Record does not exist')
            return False
        
        # Validate input
        if not self.name:
            raise ValidationError('Name is required')
        
        # Use try-except
        try:
            result = self._risky_operation()
        except Exception as e:
            _logger.exception('Operation failed')
            return False
        
        # Verify postconditions
        if not result:
            _logger.error('Operation returned invalid result')
            return False
        
        return True
```

### 2. Debug Helpers

```python
class DebugHelpers(models.AbstractModel):
    _name = 'debug.helpers'
    
    def print_record(self, record):
        """Pretty print record"""
        _logger.info(f'\n{record._name} ({record.id}):')
        for field in record._fields:
            value = record[field]
            _logger.info(f'  {field}: {value}')
    
    def print_context(self):
        """Print current context"""
        import json
        _logger.info(f'Context: {json.dumps(self.env.context, indent=2)}')
    
    def print_sql(self, query):
        """Print formatted SQL"""
        import sqlparse
        formatted = sqlparse.format(
            query,
            reindent=True,
            keyword_case='upper'
        )
        _logger.info(f'\n{formatted}')
```

---

## 🎯 Debugging Checklist

When debugging issues:

- [ ] Enable debug mode
- [ ] Check server logs
- [ ] Verify module is installed
- [ ] Check view XML syntax
- [ ] Verify access rights
- [ ] Test in Odoo shell
- [ ] Use PDB breakpoints
- [ ] Log SQL queries
- [ ] Check browser console
- [ ] Verify data integrity
- [ ] Test with different users
- [ ] Check for JavaScript errors

---

## 📚 Tài Liệu Tham Khảo

- [Odoo Debug Mode](https://www.odoo.com/documentation/19.0/developer/reference/frontend/framework_overview.html#debug-mode)
- [Python Debugging](https://docs.python.org/3/library/pdb.html)
- [Logging in Python](https://docs.python.org/3/library/logging.html)
- [Chrome DevTools](https://developer.chrome.com/docs/devtools/)

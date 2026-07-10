---
name: odoo-debugging
description: Hướng dẫn debugging trong Odoo 19 - debug mode, logging, pdb, profiling, và troubleshooting. Use when the user asks to debug, add logging, troubleshoot errors, or profile Odoo 19 code.
---

# Odoo 19 Debugging

## Goal
Giúp agent debug và troubleshoot Odoo 19 issues hiệu quả: debug mode, logging, pdb, SQL tracing.

## When to use this skill
- "debug", "gỡ lỗi", "troubleshoot"
- "logging", "log", "error"
- "pdb", "breakpoint", "profiling"

## Instructions

### 1. Debug Mode
```
# URL: append ?debug=1 or ?debug=assets
http://localhost:8069/web?debug=1

# Enable via Settings → Developer Tools → Activate Developer Mode
```

### 2. Logging
```python
import logging
_logger = logging.getLogger(__name__)

# Log levels
_logger.debug("Debug message: %s", variable)
_logger.info("Info: record %s created", record.id)
_logger.warning("Warning: field %s is empty", field_name)
_logger.error("Error: %s", str(e))
_logger.exception("Exception occurred")  # includes traceback
```

### 3. Odoo Config Logging
```ini
# odoo.conf
log_level = debug
log_handler = :DEBUG
log_handler = odoo.addons.my_module:DEBUG
log_handler = werkzeug:WARNING
```

```bash
# CLI
./odoo-bin --log-level=debug --log-handler=odoo.addons.my_module:DEBUG
```

### 4. PDB Debugging
```python
def my_method(self):
    import pdb; pdb.set_trace()  # Classic
    # or
    breakpoint()  # Python 3.7+

    # PDB commands:
    # n (next), s (step), c (continue), p var (print), l (list)
    # pp self.env.context  (pretty print)
    # self.env['res.partner'].search([])  (execute ORM)
```

### 5. SQL Tracing
```python
# Temporary SQL logging
import logging
logging.getLogger('odoo.sql_db').setLevel(logging.DEBUG)

# Count queries
cr = self.env.cr
start = cr.sql_log_count
# ... operations ...
_logger.info("Queries executed: %d", cr.sql_log_count - start)
```

### 6. Shell Access
```bash
./odoo-bin shell -d mydb
>>> env['res.partner'].search_count([])
>>> env['res.partner'].browse(1).name
```

### 7. Common Error Patterns
| Error | Cause | Fix |
|-------|-------|-----|
| `AccessError` | Missing ACL/record rules | Check ir.model.access.csv |
| `ValidationError` | Python constraint failed | Check @api.constrains |
| `MissingError` | Record deleted | Check existence first |
| `RecursionError` | Infinite compute loop | Check @api.depends cycle |
| `KeyError 'xxx'` | Missing field in vals | Add default or check |

## Constraints
- KHÔNG dùng `print()` — luôn dùng `_logger`.
- Remove `pdb.set_trace()` trước khi deploy production.
- KHÔNG set `log_level=debug` trong production.

## Best practices
- Dùng `_logger.exception()` trong except blocks (auto-includes traceback).
- Dùng `%s` formatting (NOT f-strings) trong logger calls cho lazy evaluation.
- Đọc `resources/reference.md` cho advanced profiling, memory debugging.

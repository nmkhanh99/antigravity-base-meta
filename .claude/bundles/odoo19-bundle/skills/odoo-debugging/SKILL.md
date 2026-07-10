---
name: odoo-debugging
description: Hướng dẫn debugging trong Odoo 19 - debug mode, logging, pdb, profiling, và troubleshooting. Use when the user asks to debug, add logging, troubleshoot errors, or profile Odoo 19 code.
---

# Odoo 19 Debugging

## Goal
Giúp agent debug và troubleshoot Odoo 19 issues hiệu quả: debug mode, logging, pdb, SQL tracing, frontend debugging và performance profiling.

## When to use this skill
- "debug", "gỡ lỗi", "troubleshoot"
- "logging", "log", "error", "lỗi"
- "pdb", "breakpoint", "profiling"
- "SQL chậm", "query slow", "performance"
- "view không render", "module không load"
- "AccessError", "ValidationError", "MissingError"

## Instructions

### 1. Debug Mode

Kích hoạt debug mode qua URL hoặc Settings:

```
http://localhost:8069/web?debug=1        # Standard debug
http://localhost:8069/web?debug=assets   # Debug with asset source maps
```

Xem chi tiết: `references/GUIDE.md` → **Debug Mode**

### 2. Logging (bắt buộc dùng _logger, không dùng print)

```python
import logging
_logger = logging.getLogger(__name__)

_logger.debug("Debug: %s", variable)          # Lazy evaluation với %s
_logger.info("Record %s created", record.id)
_logger.warning("Field %s is empty", field_name)
_logger.error("Error: %s", str(e))
_logger.exception("Exception occurred")       # Auto-includes traceback
```

Config trong `odoo.conf`:
```ini
log_level = debug
log_handler = odoo.addons.my_module:DEBUG
log_handler = werkzeug:WARNING
```

### 3. PDB Debugging

```python
def my_method(self):
    breakpoint()  # Python 3.7+ — dùng thay cho pdb.set_trace()
    # Hoặc: import pdb; pdb.set_trace()
```

PDB commands quan trọng: `n` (next), `s` (step), `c` (continue), `p var` (print), `q` (quit).

Xem thêm conditional breakpoints và VS Code remote debug: `references/GUIDE.md` → **Python Debugging**

### 4. SQL Tracing

```python
import logging
logging.getLogger('odoo.sql_db').setLevel(logging.DEBUG)

# Đếm số queries
cr = self.env.cr
start = cr.sql_log_count
# ... operations ...
_logger.info("Queries executed: %d", cr.sql_log_count - start)
```

Xem EXPLAIN ANALYZE và database inspection: `references/GUIDE.md` → **Database Debugging**

### 5. Odoo Shell

```bash
./odoo-bin shell -d mydb
```

```python
>>> env['res.partner'].search_count([])
>>> env['res.partner'].browse(1).name
>>> env.cr.commit()
```

### 6. Common Error Patterns

| Error | Cause | Fix |
|-------|-------|-----|
| `AccessError` | Missing ACL/record rules | Check `ir.model.access.csv` |
| `ValidationError` | Python constraint failed | Check `@api.constrains` |
| `MissingError` | Record deleted | Check existence trước |
| `RecursionError` | Infinite compute loop | Check `@api.depends` cycle |
| `KeyError 'xxx'` | Missing field in vals | Add default hoặc check |

Xem thêm: module not loading, view not rendering, performance issues → `references/GUIDE.md` → **Common Issues**

### 7. Frontend Debugging

```javascript
/** @odoo-module **/
import { Component, onMounted } from "@odoo/owl";

export class MyComponent extends Component {
    setup() {
        onMounted(() => console.log("Mounted:", this.props));
    }
}
```

Xem QWeb template debugging: `references/GUIDE.md` → **Frontend Debugging**

## Constraints
- KHÔNG dùng `print()` — luôn dùng `_logger`.
- KHÔNG dùng f-strings trong logger calls — dùng `%s` lazy formatting.
- Remove `pdb.set_trace()` / `breakpoint()` trước khi deploy production.
- KHÔNG set `log_level=debug` trên production server.
- KHÔNG commit raw SQL nếu không dùng `SQL()` wrapper theo chuẩn Odoo 19.

## References
- [Odoo 19 Developer Documentation](https://www.odoo.com/documentation/19.0/developer/)
- [Python PDB Documentation](https://docs.python.org/3/library/pdb.html)
- [Python Logging Documentation](https://docs.python.org/3/library/logging.html)
- [Chrome DevTools](https://developer.chrome.com/docs/devtools/)

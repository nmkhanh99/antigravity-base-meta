---
name: odoo-controllers
description: Hướng dẫn tạo Controllers trong Odoo 19 - HTTP routing, JSON-RPC endpoints, request/response handling, và frontend integration. Use when the user asks to create controller, add route, handle HTTP request, build API endpoint, or integrate frontend in Odoo 19.
---

# Odoo 19 Controllers

## Goal
Giúp agent tạo HTTP controllers cho web routes, JSON-RPC endpoints, file downloads, và portal pages đúng chuẩn Odoo 19.

## When to use this skill
- "tạo controller", "create controller", "add route"
- "HTTP endpoint", "JSON-RPC", "API endpoint"
- "portal page", "website controller"
- "file download", "upload handler"
- "webhook", "AJAX endpoint"

## Instructions

### 1. Tạo cấu trúc thư mục
```
my_module/
├── controllers/
│   ├── __init__.py      # from . import main, portal, api
│   ├── main.py
│   ├── portal.py
│   └── api.py
```

### 2. Basic HTTP Controller
```python
from odoo import http
from odoo.http import request

class MyController(http.Controller):

    @http.route('/my/page', type='http', auth='user', website=True)
    def my_page(self, **kw):
        records = request.env['my.model'].search([])
        return request.render('my_module.my_template', {'records': records})

    @http.route('/my/api/data', type='json', auth='user')
    def get_data(self, model_id, **kw):
        record = request.env['my.model'].browse(model_id)
        return {'name': record.name, 'amount': record.amount}
```

> **Lưu ý Odoo 19**: Dùng `type='json'` (không phải `type='jsonrpc'`) cho JSON endpoints trong Odoo 19.

### 3. Route Parameters
| Param | Values | Description |
|-------|--------|-------------|
| `type` | `'http'`, `'json'` | http trả HTML/binary, json trả dict |
| `auth` | `'user'`, `'public'`, `'none'` | Mức xác thực |
| `website` | `True/False` | Thêm website layout |
| `csrf` | `True/False` | Bảo vệ CSRF |
| `methods` | `['GET','POST']` | HTTP methods cho phép |
| `cors` | `'*'` hoặc domain | CORS policy |

### 4. File Download
```python
@http.route('/my/download/<int:record_id>', type='http', auth='user')
def download_file(self, record_id):
    record = request.env['my.model'].browse(record_id)
    if not record.exists():
        return request.not_found()
    return request.make_response(
        record.file_data,
        headers=[
            ('Content-Type', 'application/pdf'),
            ('Content-Disposition', f'attachment; filename="{record.name}.pdf"'),
        ]
    )
```

### 5. Portal Controller Pattern
```python
from odoo.addons.portal.controllers.portal import CustomerPortal

class MyPortal(CustomerPortal):
    @http.route('/my/records', type='http', auth='user', website=True)
    def portal_my_records(self, **kw):
        records = request.env['my.model'].search([
            ('partner_id', '=', request.env.user.partner_id.id)
        ])
        return request.render('my_module.portal_records', {'records': records})
```

### 6. Xem GUIDE.md cho patterns nâng cao
- JSON-RPC / RESTful API patterns → `references/GUIDE.md#json-rpc-controllers`
- File upload handler → `references/GUIDE.md#file-uploaddownload`
- Webhook handler → `references/GUIDE.md#ví-dụ-thực-tế`
- Error handling, input validation → `references/GUIDE.md#best-practices`

## Constraints
- JSON endpoints (`type='json'`) nhận/trả JSON, KHÔNG trả HTML.
- Portal routes PHẢI kiểm tra ownership (chỉ show records của đúng user).
- KHÔNG disable CSRF (`csrf=False`) trừ khi endpoint dành cho external API/webhook.
- Dùng `auth='user'` cho internal routes, `auth='public'` cho portal/website công khai.
- Luôn validate input trước khi write vào database.
- Dùng `.sudo()` cẩn thận - chỉ khi thực sự cần bypass access rules.
- KHÔNG dùng `type='jsonrpc'` - trong Odoo 19 JSON endpoints dùng `type='json'`.

## References
- https://www.odoo.com/documentation/19.0/developer/reference/backend/http.html
- https://www.odoo.com/documentation/19.0/developer/reference/backend/http.html#routing
- https://www.odoo.com/documentation/19.0/developer/reference/backend/http.html#request

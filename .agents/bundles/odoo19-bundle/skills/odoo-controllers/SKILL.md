---
name: odoo-controllers
description: Hướng dẫn tạo Controllers trong Odoo 19 - HTTP routing, JSON-RPC endpoints, request/response handling, và frontend integration. Use when the user asks to create controller, add route, handle HTTP request, build API endpoint, or integrate frontend in Odoo 19.
---

# Odoo 19 Controllers

## Goal
Giúp agent tạo HTTP controllers cho web routes, JSON-RPC endpoints, file downloads, và portal pages.

## When to use this skill
- "tạo controller", "create controller", "add route"
- "HTTP endpoint", "JSON-RPC", "API endpoint"
- "portal page", "website controller"
- "file download", "upload handler"

## Instructions

### 1. Basic Controller
```python
from odoo import http
from odoo.http import request

class MyController(http.Controller):

    @http.route('/my/page', type='http', auth='user', website=True)
    def my_page(self, **kw):
        records = request.env['my.model'].search([])
        return request.render('my_module.my_template', {
            'records': records,
        })

    @http.route('/my/api/data', type='json', auth='user')
    def get_data(self, model_id, **kw):
        record = request.env['my.model'].browse(model_id)
        return {'name': record.name, 'amount': record.amount}
```

### 2. Route Parameters
```python
@http.route('/my/<int:record_id>', type='http', auth='public',
            methods=['GET', 'POST'], csrf=False, cors='*')
def my_route(self, record_id, **kw):
    pass
```

| Param | Values | Description |
|-------|--------|-------------|
| `type` | `'http'`, `'json'` | HTTP returns HTML/binary, JSON returns dict |
| `auth` | `'user'`, `'public'`, `'none'` | Authentication level |
| `website` | `True/False` | Adds website layout |
| `csrf` | `True/False` | CSRF protection |
| `methods` | `['GET','POST']` | Allowed HTTP methods |

### 3. File Download
```python
@http.route('/my/download/<int:record_id>', type='http', auth='user')
def download_file(self, record_id):
    record = request.env['my.model'].browse(record_id)
    return request.make_response(
        record.file_data,
        headers=[
            ('Content-Type', 'application/pdf'),
            ('Content-Disposition', f'attachment; filename="{record.name}.pdf"'),
        ]
    )
```

### 4. Portal Controller Pattern
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

## Constraints
- JSON endpoints (`type='json'`) nhận/trả JSON, KHÔNG trả HTML.
- Portal routes PHẢI kiểm tra ownership (chỉ show records của user).
- KHÔNG disable CSRF (`csrf=False`) trừ khi endpoint cho external API.

## Best practices
- Dùng `auth='user'` cho internal routes, `auth='public'` cho portal/website.
- Luôn validate input trước khi dùng.
- Đọc `resources/reference.md` cho WebSocket, file upload, error handling patterns.

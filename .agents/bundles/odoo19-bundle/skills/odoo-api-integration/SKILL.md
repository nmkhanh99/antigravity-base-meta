---
name: odoo-api-integration
description: Hướng dẫn toàn diện về XML-RPC, JSON-RPC, REST API, và external API integration trong Odoo 19. Use when the user asks to integrate external API, use XML-RPC, JSON-RPC, build REST endpoint, or connect Odoo to external systems.
---

# Odoo 19 API Integration

## Goal
Giúp agent tích hợp Odoo với hệ thống bên ngoài qua XML-RPC, JSON-RPC, REST API và webhooks.

## When to use this skill
- "tích hợp API", "API integration"
- "XML-RPC", "JSON-RPC"
- "REST API", "webhook"
- "external system", "gọi API ngoài"

## Instructions

### 1. XML-RPC Client (Python)
```python
import xmlrpc.client

url, db, user, pwd = 'http://localhost:8069', 'mydb', 'admin', 'admin'
common = xmlrpc.client.ServerProxy(f'{url}/xmlrpc/2/common')
uid = common.authenticate(db, user, pwd, {})
models = xmlrpc.client.ServerProxy(f'{url}/xmlrpc/2/object')

# Search + Read
partners = models.execute_kw(db, uid, pwd, 'res.partner', 'search_read',
    [[('is_company', '=', True)]], {'fields': ['name', 'email'], 'limit': 5})

# Create
new_id = models.execute_kw(db, uid, pwd, 'res.partner', 'create',
    [{'name': 'New Partner'}])

# Write
models.execute_kw(db, uid, pwd, 'res.partner', 'write',
    [[new_id], {'phone': '+123'}])
```

### 2. JSON-RPC Call
```python
import requests, json

def jsonrpc(url, method, params):
    return requests.post(url, json={
        "jsonrpc": "2.0", "method": method, "params": params, "id": 1
    }).json()

# Authenticate
result = jsonrpc(f"{url}/web/session/authenticate", "call", {
    "db": db, "login": user, "password": pwd
})
session_id = result['result']['session_id']

# Call model method
result = jsonrpc(f"{url}/web/dataset/call_kw", "call", {
    "model": "res.partner",
    "method": "search_read",
    "args": [[('is_company', '=', True)]],
    "kwargs": {"fields": ["name"], "limit": 5},
})
```

### 3. External API Call from Odoo
```python
import requests

class MyModel(models.Model):
    _name = 'my.model'

    def sync_external_data(self):
        response = requests.get('https://api.example.com/data',
            headers={'Authorization': 'Bearer TOKEN'}, timeout=30)
        response.raise_for_status()
        for item in response.json():
            self.env['my.model'].create({'name': item['name']})
```

### 4. Webhook Receiver
```python
class WebhookController(http.Controller):
    @http.route('/webhook/receive', type='json', auth='none', csrf=False)
    def receive_webhook(self, **kw):
        data = request.jsonrequest
        request.env['my.model'].sudo().create({'name': data['event']})
        return {'status': 'ok'}
```

## Constraints
- KHÔNG hardcode credentials — dùng `ir.config_parameter` hoặc env variables.
- Luôn set `timeout` cho external API calls.
- Webhook endpoints PHẢI validate source (secret/signature).

## Best practices
- Dùng `requests` library cho external HTTP calls.
- Log API calls để debug.
- Đọc `resources/reference.md` cho OAuth, batch API, error handling patterns.

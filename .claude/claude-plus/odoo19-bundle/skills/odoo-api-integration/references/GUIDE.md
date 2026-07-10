---
name: Odoo API Integration
description: Hướng dẫn toàn diện về XML-RPC, JSON-RPC, REST API, và external API integration trong Odoo 19
---

# Odoo API Integration — Reference Guide

## Mục Lục

1. [XML-RPC API](#xml-rpc-api)
2. [JSON-RPC API](#json-rpc-api)
3. [REST API](#rest-api)
4. [External API Integration](#external-api-integration)
5. [Webhook Receiver](#webhook-receiver)
6. [Authentication](#authentication)
7. [Best Practices](#best-practices)

---

## XML-RPC API

### Python Client (Odoo 19)

```python
import xmlrpc.client

url = 'http://localhost:8069'
db = 'mydb'
username = 'admin'
password = 'admin'

# Authenticate
common = xmlrpc.client.ServerProxy(f'{url}/xmlrpc/2/common')
uid = common.authenticate(db, username, password, {})

# Models proxy
models = xmlrpc.client.ServerProxy(f'{url}/xmlrpc/2/object')

# Search
partner_ids = models.execute_kw(
    db, uid, password,
    'res.partner', 'search',
    [[['is_company', '=', True]]],
    {'limit': 5}
)

# Read
partners = models.execute_kw(
    db, uid, password,
    'res.partner', 'read',
    [partner_ids],
    {'fields': ['name', 'email', 'phone']}
)

# search_read (combine search + read in one call)
partners = models.execute_kw(
    db, uid, password,
    'res.partner', 'search_read',
    [[['is_company', '=', True]]],
    {'fields': ['name', 'email'], 'limit': 5}
)

# Create
partner_id = models.execute_kw(
    db, uid, password,
    'res.partner', 'create',
    [{'name': 'New Partner', 'email': 'partner@example.com'}]
)

# Write
models.execute_kw(
    db, uid, password,
    'res.partner', 'write',
    [[partner_id], {'phone': '+1234567890'}]
)

# Delete
models.execute_kw(
    db, uid, password,
    'res.partner', 'unlink',
    [[partner_id]]
)
```

---

## JSON-RPC API

### Python Client

```python
import requests

url = 'http://localhost:8069'
db = 'mydb'
user = 'admin'
pwd = 'admin'

def jsonrpc(endpoint, params):
    response = requests.post(
        f'{url}{endpoint}',
        json={"jsonrpc": "2.0", "method": "call", "params": params, "id": 1},
        timeout=30
    )
    response.raise_for_status()
    result = response.json()
    if 'error' in result:
        raise Exception(result['error'])
    return result['result']

# Authenticate
session = jsonrpc('/web/session/authenticate', {
    'db': db, 'login': user, 'password': pwd
})

# Call model method
partners = jsonrpc('/web/dataset/call_kw', {
    'model': 'res.partner',
    'method': 'search_read',
    'args': [[['is_company', '=', True]]],
    'kwargs': {'fields': ['name', 'email'], 'limit': 5},
})
```

### JavaScript Client

```javascript
// Authenticate
async function authenticate() {
    const response = await fetch('http://localhost:8069/web/session/authenticate', {
        method: 'POST',
        headers: {'Content-Type': 'application/json'},
        body: JSON.stringify({
            jsonrpc: '2.0',
            method: 'call',
            params: {db: 'mydb', login: 'admin', password: 'admin'},
        }),
    });
    const data = await response.json();
    return data.result;
}

// Call model method
async function callMethod(model, method, args = [], kwargs = {}) {
    const response = await fetch('http://localhost:8069/web/dataset/call_kw', {
        method: 'POST',
        headers: {'Content-Type': 'application/json'},
        credentials: 'include',  // gửi session cookie
        body: JSON.stringify({
            jsonrpc: '2.0',
            method: 'call',
            params: {model, method, args, kwargs},
        }),
    });
    const data = await response.json();
    if (data.error) throw new Error(data.error.message);
    return data.result;
}

// Search and read
const partners = await callMethod(
    'res.partner', 'search_read', [],
    {domain: [['is_company', '=', true]], fields: ['name', 'email'], limit: 10}
);
```

---

## REST API

### Custom REST Controller (Odoo 19)

```python
# controllers/api.py
from odoo import http
from odoo.http import request
import json
import logging

_logger = logging.getLogger(__name__)


class RestAPI(http.Controller):

    @http.route('/api/v1/partners', type='http', auth='user',
                methods=['GET'], csrf=False)
    def get_partners(self, **kwargs):
        """Get partners list"""
        try:
            limit = int(kwargs.get('limit', 10))
            offset = int(kwargs.get('offset', 0))

            partners = request.env['res.partner'].search(
                [], limit=limit, offset=offset
            )

            data = [{
                'id': p.id,
                'name': p.name,
                'email': p.email or '',
                'phone': p.phone or '',
            } for p in partners]

            return request.make_response(
                json.dumps({
                    'status': 'success',
                    'data': data,
                    'total': request.env['res.partner'].search_count([]),
                }),
                headers=[('Content-Type', 'application/json')]
            )

        except Exception as e:
            _logger.exception('Error in GET /api/v1/partners')
            return request.make_response(
                json.dumps({'status': 'error', 'message': str(e)}),
                status=500,
                headers=[('Content-Type', 'application/json')]
            )

    @http.route('/api/v1/partners/<int:partner_id>', type='http',
                auth='user', methods=['GET'], csrf=False)
    def get_partner(self, partner_id, **kwargs):
        """Get single partner"""
        partner = request.env['res.partner'].browse(partner_id)

        if not partner.exists():
            return request.make_response(
                json.dumps({'status': 'error', 'message': 'Not found'}),
                status=404,
                headers=[('Content-Type', 'application/json')]
            )

        return request.make_response(
            json.dumps({
                'status': 'success',
                'data': {
                    'id': partner.id,
                    'name': partner.name,
                    'email': partner.email or '',
                    'phone': partner.phone or '',
                },
            }),
            headers=[('Content-Type', 'application/json')]
        )

    @http.route('/api/v1/partners', type='json', auth='user',
                methods=['POST'], csrf=False)
    def create_partner(self, **kwargs):
        """Create partner — type='json' tự parse body và trả dict"""
        try:
            data = request.jsonrequest
            partner = request.env['res.partner'].create({
                'name': data.get('name'),
                'email': data.get('email'),
                'phone': data.get('phone'),
            })
            return {'status': 'success', 'id': partner.id}

        except Exception as e:
            _logger.exception('Error in POST /api/v1/partners')
            return {'status': 'error', 'message': str(e)}
```

> **Lưu ý Odoo 19**: `type='json'` → Odoo tự parse JSON body, return dict được serialize tự động. `type='http'` → dùng `request.make_response()` với JSON manual.

---

## External API Integration

### Gọi API bên ngoài từ Odoo

```python
# models/external_api.py
import requests
import logging
from odoo import models, api
from odoo.exceptions import UserError

_logger = logging.getLogger(__name__)


class ExternalAPIIntegration(models.Model):
    _name = 'external.api'
    _description = 'External API Integration'

    @api.model
    def _get_api_config(self):
        """Lấy config từ ir.config_parameter — không hardcode"""
        params = self.env['ir.config_parameter'].sudo()
        return {
            'base_url': params.get_param('external.api.base_url', ''),
            'api_key': params.get_param('external.api.key', ''),
            'timeout': int(params.get_param('external.api.timeout', '30')),
        }

    @api.model
    def call_external_api(self, endpoint, method='GET', data=None, params=None):
        """Generic external API caller với error handling"""
        config = self._get_api_config()

        if not config['base_url']:
            raise UserError('External API base URL chưa được cấu hình.')

        headers = {
            'Authorization': f'Bearer {config["api_key"]}',
            'Content-Type': 'application/json',
        }
        url = f'{config["base_url"].rstrip("/")}/{endpoint.lstrip("/")}'

        try:
            response = requests.request(
                method=method.upper(),
                url=url,
                headers=headers,
                json=data,
                params=params,
                timeout=config['timeout']
            )
            response.raise_for_status()
            return response.json()

        except requests.exceptions.Timeout:
            _logger.error('External API timeout: %s', url)
            raise UserError(f'API timeout sau {config["timeout"]}s: {url}')
        except requests.exceptions.HTTPError as e:
            _logger.error('External API HTTP error: %s — %s', url, e)
            raise UserError(f'API lỗi HTTP {response.status_code}: {response.text}')
        except requests.exceptions.RequestException as e:
            _logger.error('External API error: %s — %s', url, e)
            raise UserError(f'Lỗi kết nối API: {e}')

    def sync_products_from_external(self):
        """Ví dụ: sync products từ hệ thống ngoài"""
        products_data = self.call_external_api('products')

        for product_data in products_data:
            product = self.env['product.product'].search([
                ('default_code', '=', product_data.get('sku'))
            ], limit=1)

            vals = {
                'name': product_data['name'],
                'default_code': product_data.get('sku', ''),
                'list_price': product_data.get('price', 0.0),
                'description': product_data.get('description', ''),
            }

            if product:
                product.write(vals)
            else:
                self.env['product.product'].create(vals)

        return True
```

---

## Webhook Receiver

```python
# controllers/webhook.py
import hmac
import hashlib
import json
import logging
from odoo import http
from odoo.http import request

_logger = logging.getLogger(__name__)


class WebhookController(http.Controller):

    def _validate_webhook_signature(self, payload_bytes, signature_header):
        """Validate HMAC-SHA256 signature từ sender"""
        secret = request.env['ir.config_parameter'].sudo().get_param(
            'webhook.secret', ''
        )
        if not secret:
            _logger.warning('Webhook secret chưa được cấu hình!')
            return False

        expected = hmac.new(
            secret.encode(), payload_bytes, hashlib.sha256
        ).hexdigest()
        return hmac.compare_digest(f'sha256={expected}', signature_header or '')

    @http.route('/webhook/receive', type='json', auth='none', csrf=False,
                methods=['POST'])
    def receive_webhook(self, **kw):
        """Webhook receiver với signature validation"""
        # Validate source
        signature = request.httprequest.headers.get('X-Hub-Signature-256', '')
        payload_bytes = request.httprequest.get_data()

        if not self._validate_webhook_signature(payload_bytes, signature):
            _logger.warning('Webhook signature không hợp lệ')
            return {'status': 'error', 'message': 'Invalid signature'}

        try:
            data = request.jsonrequest
            event_type = data.get('event', 'unknown')
            _logger.info('Webhook received: %s', event_type)

            # Xử lý async để không block
            request.env['my.model'].sudo().create({
                'name': event_type,
                'payload': json.dumps(data),
            })
            return {'status': 'ok'}

        except Exception as e:
            _logger.exception('Webhook processing error')
            return {'status': 'error', 'message': str(e)}
```

---

## Authentication

### API Key Decorator

```python
# controllers/api_auth.py
import functools
import json
from odoo import http
from odoo.http import request


def api_key_required(func):
    """Decorator validate API key từ header X-API-Key"""
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        api_key = request.httprequest.headers.get('X-API-Key', '')

        if not api_key:
            return request.make_response(
                json.dumps({'error': 'API key required'}),
                status=401,
                headers=[('Content-Type', 'application/json')]
            )

        valid_key = request.env['ir.config_parameter'].sudo().get_param('api.key')

        if not valid_key or api_key != valid_key:
            return request.make_response(
                json.dumps({'error': 'Invalid API key'}),
                status=403,
                headers=[('Content-Type', 'application/json')]
            )

        return func(*args, **kwargs)
    return wrapper


class SecureAPI(http.Controller):

    @http.route('/api/secure/data', type='http', auth='none',
                methods=['GET'], csrf=False)
    @api_key_required
    def get_secure_data(self, **kwargs):
        return request.make_response(
            json.dumps({'data': 'secure data'}),
            headers=[('Content-Type', 'application/json')]
        )
```

### Odoo 19 — `@api.private`

```python
from odoo import api, models


class SecureModel(models.Model):
    _name = 'secure.model'
    _description = 'Secure Model'

    @api.private
    def _internal_logic(self):
        """
        @api.private: method này KHÔNG thể gọi qua XML-RPC hoặc JSON-RPC.
        Chỉ dùng cho internal business logic nhạy cảm.
        Có từ Odoo 17+, dùng trong Odoo 19.
        """
        return self.env['sensitive.data'].search([])
```

---

## Best Practices

### 1. Rate Limiting

```python
import json
from datetime import datetime, timedelta
from functools import wraps
from odoo.http import request


def rate_limit(max_calls=100, period_seconds=3600):
    """Rate limiting theo IP — in-memory, reset khi restart server"""
    calls = {}

    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            now = datetime.now()
            key = request.httprequest.remote_addr
            cutoff = now - timedelta(seconds=period_seconds)

            calls[key] = [t for t in calls.get(key, []) if t > cutoff]

            if len(calls[key]) >= max_calls:
                return request.make_response(
                    json.dumps({'error': 'Rate limit exceeded'}),
                    status=429,
                    headers=[('Content-Type', 'application/json')]
                )

            calls[key].append(now)
            return func(*args, **kwargs)
        return wrapper
    return decorator
```

### 2. Chuẩn hóa Response

```python
import json
from datetime import datetime
from odoo.http import request


def api_response(data=None, error=None, status=200):
    """Standardized API response"""
    body = {
        'success': error is None,
        'timestamp': datetime.now().isoformat(),
    }
    if data is not None:
        body['data'] = data
    if error is not None:
        body['error'] = error

    return request.make_response(
        json.dumps(body),
        status=status,
        headers=[('Content-Type', 'application/json')]
    )
```

### 3. Error Handling chuẩn

```python
from odoo import http
from odoo.exceptions import ValidationError, AccessError
from odoo.http import request
import logging

_logger = logging.getLogger(__name__)


class SafeAPI(http.Controller):

    @http.route('/api/v1/safe-endpoint', type='http', auth='user',
                methods=['GET'], csrf=False)
    def safe_endpoint(self, **kwargs):
        try:
            result = self._process_data(kwargs)
            return api_response(data=result)

        except ValidationError as e:
            return api_response(error=str(e), status=400)

        except AccessError:
            return api_response(error='Access denied', status=403)

        except Exception:
            _logger.exception('Unexpected error in /api/v1/safe-endpoint')
            return api_response(error='Internal server error', status=500)
```

### 4. Lưu config trong ir.config_parameter

```python
# Ghi config (thường qua Settings UI hoặc data XML)
self.env['ir.config_parameter'].sudo().set_param('external.api.key', 'secret')

# Đọc config
api_key = self.env['ir.config_parameter'].sudo().get_param('external.api.key', '')
```

---

## Checklist

- [ ] XML-RPC client implemented và tested
- [ ] JSON-RPC endpoints tested
- [ ] REST API có versioning (`/api/v1/`)
- [ ] Authentication cấu hình đúng (`api_key_required` hoặc `auth='user'`)
- [ ] Rate limiting enabled cho public endpoints
- [ ] Error handling đầy đủ với logging
- [ ] Webhook validate signature
- [ ] Không hardcode credentials
- [ ] `timeout` set cho mọi external call
- [ ] Method nội bộ dùng `@api.private`
- [ ] Security / ACL review hoàn tất

---

## Tài Liệu Tham Khảo

- [Odoo 19 External API](https://www.odoo.com/documentation/19.0/developer/reference/external_api.html)
- [XML-RPC Library](https://www.odoo.com/documentation/19.0/developer/reference/external_api.html#xml-rpc-library)
- [JSON-RPC Library](https://www.odoo.com/documentation/19.0/developer/reference/external_api.html#json-rpc-library)
- [Web Services Tutorial](https://www.odoo.com/documentation/19.0/developer/tutorials/web_services.html)

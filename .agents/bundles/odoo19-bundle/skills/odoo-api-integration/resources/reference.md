---
name: Odoo API Integration
description: Hướng dẫn toàn diện về XML-RPC, JSON-RPC, REST API, và external API integration trong Odoo
---

# Odoo API Integration Skill

## 📋 Mục Lục

1. [XML-RPC API](#xml-rpc-api)
2. [JSON-RPC API](#json-rpc-api)
3. [REST API](#rest-api)
4. [External API Integration](#external-api-integration)
5. [Authentication](#authentication)
6. [Best Practices](#best-practices)

---

## XML-RPC API

### Python Client

```python
import xmlrpc.client

# Connection
url = 'http://localhost:8069'
db = 'mydb'
username = 'admin'
password = 'admin'

# Authenticate
common = xmlrpc.client.ServerProxy(f'{url}/xmlrpc/2/common')
uid = common.authenticate(db, username, password, {})

# Models proxy
models = xmlrpc.client.ServerProxy(f'{url}/xmlrpc/2/object')

# Search and read
partner_ids = models.execute_kw(
    db, uid, password,
    'res.partner', 'search',
    [[['is_company', '=', True]]],
    {'limit': 5}
)

partners = models.execute_kw(
    db, uid, password,
    'res.partner', 'read',
    [partner_ids],
    {'fields': ['name', 'email', 'phone']}
)

# Create
partner_id = models.execute_kw(
    db, uid, password,
    'res.partner', 'create',
    [{
        'name': 'New Partner',
        'email': 'partner@example.com',
    }]
)

# Update
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

### JavaScript Client

```javascript
// Authenticate
async function authenticate() {
    const response = await fetch('http://localhost:8069/web/session/authenticate', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
        },
        body: JSON.stringify({
            jsonrpc: '2.0',
            params: {
                db: 'mydb',
                login: 'admin',
                password: 'admin',
            },
        }),
    });
    
    const data = await response.json();
    return data.result;
}

// Call method
async function callMethod(model, method, args, kwargs = {}) {
    const response = await fetch('http://localhost:8069/web/dataset/call_kw', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
        },
        credentials: 'include',
        body: JSON.stringify({
            jsonrpc: '2.0',
            method: 'call',
            params: {
                model: model,
                method: method,
                args: args,
                kwargs: kwargs,
            },
        }),
    });
    
    const data = await response.json();
    return data.result;
}

// Search and read
const partners = await callMethod(
    'res.partner',
    'search_read',
    [],
    {
        domain: [['is_company', '=', true]],
        fields: ['name', 'email'],
        limit: 10,
    }
);
```

---

## REST API

### Custom REST Controller

```python
# controllers/api.py
from odoo import http
from odoo.http import request
import json

class RestAPI(http.Controller):
    
    @http.route('/api/partners', type='http', auth='user', 
                methods=['GET'], csrf=False)
    def get_partners(self, **kwargs):
        """Get partners list"""
        try:
            limit = int(kwargs.get('limit', 10))
            offset = int(kwargs.get('offset', 0))
            
            partners = request.env['res.partner'].search(
                [],
                limit=limit,
                offset=offset
            )
            
            data = [{
                'id': p.id,
                'name': p.name,
                'email': p.email,
                'phone': p.phone,
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
            return request.make_response(
                json.dumps({
                    'status': 'error',
                    'message': str(e),
                }),
                status=500,
                headers=[('Content-Type', 'application/json')]
            )
    
    @http.route('/api/partners/<int:partner_id>', type='http', 
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
                    'email': partner.email,
                    'phone': partner.phone,
                },
            }),
            headers=[('Content-Type', 'application/json')]
        )
    
    @http.route('/api/partners', type='jsonrpc', auth='user', 
                methods=['POST'], csrf=False)
    def create_partner(self, **kwargs):
        """Create partner"""
        try:
            data = request.jsonrequest
            
            partner = request.env['res.partner'].create({
                'name': data.get('name'),
                'email': data.get('email'),
                'phone': data.get('phone'),
            })
            
            return {
                'status': 'success',
                'id': partner.id,
            }
            
        except Exception as e:
            return {
                'status': 'error',
                'message': str(e),
            }
```

---

## External API Integration

### Calling External APIs

```python
# models/external_api.py
import requests
from odoo import models, api
import logging

_logger = logging.getLogger(__name__)

class ExternalAPIIntegration(models.Model):
    _name = 'external.api'
    
    @api.model
    def call_external_api(self, endpoint, method='GET', data=None):
        """Generic external API caller"""
        base_url = self.env['ir.config_parameter'].sudo().get_param(
            'external.api.base_url'
        )
        api_key = self.env['ir.config_parameter'].sudo().get_param(
            'external.api.key'
        )
        
        headers = {
            'Authorization': f'Bearer {api_key}',
            'Content-Type': 'application/json',
        }
        
        url = f'{base_url}/{endpoint}'
        
        try:
            if method == 'GET':
                response = requests.get(url, headers=headers, timeout=30)
            elif method == 'POST':
                response = requests.post(
                    url, 
                    headers=headers, 
                    json=data,
                    timeout=30
                )
            elif method == 'PUT':
                response = requests.put(
                    url,
                    headers=headers,
                    json=data,
                    timeout=30
                )
            elif method == 'DELETE':
                response = requests.delete(url, headers=headers, timeout=30)
            
            response.raise_for_status()
            return response.json()
            
        except requests.exceptions.RequestException as e:
            _logger.error(f'External API error: {e}')
            raise
    
    def sync_products_from_external(self):
        """Sync products from external system"""
        products_data = self.call_external_api('products')
        
        for product_data in products_data:
            # Find or create product
            product = self.env['product.product'].search([
                ('default_code', '=', product_data['sku'])
            ], limit=1)
            
            vals = {
                'name': product_data['name'],
                'default_code': product_data['sku'],
                'list_price': product_data['price'],
                'description': product_data['description'],
            }
            
            if product:
                product.write(vals)
            else:
                self.env['product.product'].create(vals)
```

---

## Authentication

### API Key Authentication

```python
# controllers/api_auth.py
from odoo import http
from odoo.http import request
import functools

def api_key_required(func):
    """Decorator for API key authentication"""
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        api_key = request.httprequest.headers.get('X-API-Key')
        
        if not api_key:
            return request.make_response(
                json.dumps({'error': 'API key required'}),
                status=401,
                headers=[('Content-Type', 'application/json')]
            )
        
        # Validate API key
        valid_key = request.env['ir.config_parameter'].sudo().get_param(
            'api.key'
        )
        
        if api_key != valid_key:
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
        """Secured endpoint"""
        return request.make_response(
            json.dumps({'data': 'secure data'}),
            headers=[('Content-Type', 'application/json')]
        )
```

### OAuth2 Authentication

```python
# OAuth2 implementation
from oauthlib.oauth2 import BackendApplicationServer
from odoo import http

class OAuth2Controller(http.Controller):
    
    @http.route('/oauth/token', type='http', auth='none', 
                methods=['POST'], csrf=False)
    def get_token(self, **kwargs):
        """OAuth2 token endpoint"""
        # Implement OAuth2 token generation
        pass
```

### Odoo 19: @api.private

```python
# Odoo 19: Use @api.private to prevent methods from being called via RPC
from odoo import api

class SecureModel(models.Model):
    _name = 'secure.model'
    
    @api.private
    def _internal_logic(self):
        """This method cannot be called via XML-RPC or JSON-RPC"""
        return self.env['sensitive.data'].search([])
```

---

## Best Practices

### 1. Rate Limiting

```python
from functools import wraps
from datetime import datetime, timedelta

def rate_limit(max_calls=100, period=3600):
    """Rate limiting decorator"""
    calls = {}
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            now = datetime.now()
            key = request.httprequest.remote_addr
            
            if key in calls:
                calls[key] = [
                    call for call in calls[key]
                    if call > now - timedelta(seconds=period)
                ]
            else:
                calls[key] = []
            
            if len(calls[key]) >= max_calls:
                return request.make_response(
                    json.dumps({'error': 'Rate limit exceeded'}),
                    status=429
                )
            
            calls[key].append(now)
            return func(*args, **kwargs)
        
        return wrapper
    return decorator
```

### 2. Response Formatting

```python
def api_response(data=None, error=None, status=200):
    """Standardized API response"""
    response = {
        'success': error is None,
        'timestamp': datetime.now().isoformat(),
    }
    
    if data is not None:
        response['data'] = data
    
    if error is not None:
        response['error'] = error
    
    return request.make_response(
        json.dumps(response),
        status=status,
        headers=[('Content-Type', 'application/json')]
    )
```

### 3. Error Handling

```python
@http.route('/api/safe-endpoint', type='http', auth='user')
def safe_endpoint(self, **kwargs):
    """Endpoint with proper error handling"""
    try:
        # Your logic here
        result = self.process_data(kwargs)
        return api_response(data=result)
        
    except ValidationError as e:
        return api_response(error=str(e), status=400)
        
    except AccessError as e:
        return api_response(error='Access denied', status=403)
        
    except Exception as e:
        _logger.exception('Unexpected error in API')
        return api_response(error='Internal server error', status=500)
```

---

## 🎯 Checklist

- [ ] XML-RPC client implemented
- [ ] JSON-RPC endpoints tested
- [ ] REST API documented
- [ ] Authentication configured
- [ ] Rate limiting enabled
- [ ] Error handling comprehensive
- [ ] API versioning planned
- [ ] Security reviewed

---

## 📚 Tài Liệu Tham Khảo

- [Odoo External API](https://www.odoo.com/documentation/19.0/developer/reference/external_api.html)
- [XML-RPC](https://www.odoo.com/documentation/19.0/developer/reference/external_api.html#xml-rpc-library)
- [JSON-RPC](https://www.odoo.com/documentation/19.0/developer/reference/external_api.html#json-rpc-library)

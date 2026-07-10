---
name: Odoo Controllers
description: Hướng dẫn toàn diện về Controllers trong Odoo - xử lý HTTP requests, routing, JSON-RPC, và tích hợp frontend
---

# Odoo Controllers Skill

## 📋 Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [Cấu Trúc Controller](#cấu-trúc-controller)
3. [HTTP Routing](#http-routing)
4. [Request & Response](#request--response)
5. [Authentication & Security](#authentication--security)
6. [JSON-RPC Controllers](#json-rpc-controllers)
7. [File Upload/Download](#file-uploaddownload)
8. [Best Practices](#best-practices)
9. [Ví Dụ Thực Tế](#ví-dụ-thực-tế)

---

## Tổng Quan

### Controllers Là Gì?

Controllers trong Odoo xử lý các HTTP requests từ web browser hoặc API clients. Chúng là cầu nối giữa frontend và backend.

### Khi Nào Sử Dụng Controllers?

- ✅ Tạo custom web pages/portals
- ✅ Xây dựng REST APIs
- ✅ Xử lý file uploads/downloads
- ✅ Tích hợp với external systems
- ✅ Custom authentication flows
- ✅ Webhooks và callbacks

### Cấu Trúc Thư Mục

```
my_module/
├── __manifest__.py
├── controllers/
│   ├── __init__.py          # Import controllers
│   ├── main.py              # Main controllers
│   ├── portal.py            # Portal controllers
│   └── api.py               # API controllers
```

---

## Cấu Trúc Controller

### 1. Basic Controller

```python
# controllers/__init__.py
from . import main
from . import portal
from . import api
```

```python
# controllers/main.py
from odoo import http
from odoo.http import request

class MyController(http.Controller):
    
    @http.route('/my_module/hello', type='http', auth='public', website=True)
    def hello_world(self, **kwargs):
        """Simple HTTP endpoint"""
        return "Hello, World!"
    
    @http.route('/my_module/page', type='http', auth='user', website=True)
    def my_page(self, **kwargs):
        """Render a template"""
        return request.render('my_module.my_template', {
            'title': 'My Page',
            'data': 'Some data'
        })
```

### 2. Controller với Multiple Routes

```python
class ProductController(http.Controller):
    
    @http.route([
        '/shop/products',
        '/shop/products/page/<int:page>',
    ], type='http', auth='public', website=True)
    def product_list(self, page=1, **kwargs):
        """Product listing with pagination"""
        Product = request.env['product.template'].sudo()
        
        # Pagination
        limit = 20
        offset = (page - 1) * limit
        
        products = Product.search([], limit=limit, offset=offset)
        total = Product.search_count([])
        
        return request.render('my_module.product_list', {
            'products': products,
            'page': page,
            'total_pages': (total + limit - 1) // limit,
        })
    
    @http.route('/shop/product/<int:product_id>', type='http', auth='public', website=True)
    def product_detail(self, product_id, **kwargs):
        """Product detail page"""
        product = request.env['product.template'].sudo().browse(product_id)
        
        if not product.exists():
            return request.not_found()
        
        return request.render('my_module.product_detail', {
            'product': product,
        })
```

---

## HTTP Routing

### 1. Route Parameters

```python
@http.route(
    route='/path/to/endpoint',      # URL path (string hoặc list)
    type='http',                     # 'http' hoặc 'jsonrpc' (Odoo 19)
    auth='public',                   # 'public', 'user', 'none'
    methods=['GET', 'POST'],         # HTTP methods
    website=True,                    # Website-specific route
    sitemap=True,                    # Include in sitemap
    csrf=True,                       # CSRF protection
    cors='*',                        # CORS policy
)
```

### 2. Authentication Types

```python
class AuthController(http.Controller):
    
    @http.route('/public/endpoint', type='http', auth='public')
    def public_endpoint(self):
        """Accessible without login"""
        return "Public content"
    
    @http.route('/user/endpoint', type='http', auth='user')
    def user_endpoint(self):
        """Requires login"""
        user = request.env.user
        return f"Hello, {user.name}!"
    
    @http.route('/api/endpoint', type='jsonrpc', auth='none')
    def api_endpoint(self):
        """No authentication (use with caution!)"""
        return {'status': 'ok'}
```

### 3. URL Parameters

```python
class ParamsController(http.Controller):
    
    @http.route('/search', type='http', auth='public')
    def search(self, query='', category=None, **kwargs):
        """
        URL: /search?query=laptop&category=electronics
        """
        results = request.env['product.template'].sudo().search([
            ('name', 'ilike', query),
            ('categ_id.name', '=', category) if category else (1, '=', 1),
        ])
        
        return request.render('my_module.search_results', {
            'query': query,
            'category': category,
            'results': results,
        })
    
    @http.route('/product/<model("product.template"):product>', 
                type='http', auth='public')
    def product_by_model(self, product, **kwargs):
        """
        URL: /product/123
        Automatically converts ID to record
        """
        return request.render('my_module.product_detail', {
            'product': product,
        })
```

---

## Request & Response

### 1. Request Object

```python
class RequestController(http.Controller):
    
    @http.route('/request/info', type='http', auth='user')
    def request_info(self, **kwargs):
        """Access request information"""
        
        # Current user
        user = request.env.user
        
        # HTTP method
        method = request.httprequest.method
        
        # Headers
        headers = request.httprequest.headers
        
        # Query parameters
        params = request.params
        
        # Form data
        form_data = request.httprequest.form
        
        # Files
        files = request.httprequest.files
        
        # Session
        session = request.session
        
        # Environment
        env = request.env
        
        # Website
        website = request.website
        
        return request.render('my_module.request_info', {
            'user': user,
            'method': method,
            'params': params,
        })
```

### 2. Response Types

```python
class ResponseController(http.Controller):
    
    @http.route('/response/html', type='http', auth='public')
    def html_response(self):
        """HTML response"""
        return request.render('my_module.template', {})
    
    @http.route('/response/redirect', type='http', auth='public')
    def redirect_response(self):
        """Redirect response"""
        return request.redirect('/shop')
    
    @http.route('/response/json', type='jsonrpc', auth='public')
    def json_response(self):
        """JSON response"""
        return {
            'status': 'success',
            'data': {'key': 'value'}
        }
    
    @http.route('/response/file', type='http', auth='user')
    def file_response(self):
        """File download response"""
        content = b'File content here'
        
        return request.make_response(
            content,
            headers=[
                ('Content-Type', 'application/pdf'),
                ('Content-Disposition', 'attachment; filename="report.pdf"'),
            ]
        )
    
    @http.route('/response/error', type='http', auth='public')
    def error_response(self):
        """Error responses"""
        # 404 Not Found
        return request.not_found()
        
        # Custom error
        return request.render('http_routing.404', {
            'message': 'Custom error message'
        }, status=404)
```

---

## Authentication & Security

### 1. User Authentication

```python
class SecureController(http.Controller):
    
    @http.route('/secure/data', type='jsonrpc', auth='user')
    def secure_data(self):
        """Only authenticated users"""
        user = request.env.user
        
        # Check specific permission
        if not user.has_group('base.group_user'):
            return {'error': 'Unauthorized'}
        
        return {
            'user': user.name,
            'data': 'Sensitive information'
        }
    
    @http.route('/admin/panel', type='http', auth='user')
    def admin_panel(self):
        """Admin only"""
        if not request.env.user.has_group('base.group_system'):
            return request.redirect('/web/login')
        
        return request.render('my_module.admin_panel', {})
```

### 2. CSRF Protection

```python
class CSRFController(http.Controller):
    
    @http.route('/form/submit', type='http', auth='user', 
                methods=['POST'], csrf=True)
    def form_submit(self, **post):
        """CSRF protected form submission"""
        # Process form data
        name = post.get('name')
        email = post.get('email')
        
        # Create record
        request.env['res.partner'].create({
            'name': name,
            'email': email,
        })
        
        return request.redirect('/thank-you')
```

### 3. API Key Authentication

```python
class APIController(http.Controller):
    
    def _check_api_key(self, api_key):
        """Validate API key"""
        ApiKey = request.env['my_module.api_key'].sudo()
        key = ApiKey.search([('key', '=', api_key)], limit=1)
        return key.exists()
    
    @http.route('/api/v1/data', type='jsonrpc', auth='none', csrf=False)
    def api_data(self, api_key=None, **kwargs):
        """API endpoint with key authentication"""
        if not api_key or not self._check_api_key(api_key):
            return {
                'error': 'Invalid API key',
                'code': 401
            }
        
        # Process request
        return {
            'status': 'success',
            'data': []
        }
```

---

## JSON-RPC Controllers

### 1. Basic JSON Controller

```python
class JSONController(http.Controller):
    
    @http.route('/api/products', type='jsonrpc', auth='user')
    def get_products(self, limit=10, offset=0):
        """JSON-RPC endpoint"""
        products = request.env['product.template'].search(
            [], limit=limit, offset=offset
        )
        
        return {
            'count': len(products),
            'products': [{
                'id': p.id,
                'name': p.name,
                'price': p.list_price,
            } for p in products]
        }
    
    @http.route('/api/product/create', type='jsonrpc', auth='user', methods=['POST'])
    def create_product(self, name, price, **kwargs):
        """Create product via JSON-RPC"""
        try:
            product = request.env['product.template'].create({
                'name': name,
                'list_price': price,
            })
            
            return {
                'status': 'success',
                'id': product.id,
                'name': product.name,
            }
        except Exception as e:
            return {
                'status': 'error',
                'message': str(e)
            }
```

### 2. RESTful API

```python
class RESTController(http.Controller):
    
    @http.route('/api/v1/partners', type='jsonrpc', auth='user', methods=['GET'])
    def list_partners(self, limit=20, offset=0, search=None):
        """GET /api/v1/partners"""
        domain = []
        if search:
            domain = [('name', 'ilike', search)]
        
        partners = request.env['res.partner'].search(
            domain, limit=limit, offset=offset
        )
        
        return {
            'count': len(partners),
            'data': [{
                'id': p.id,
                'name': p.name,
                'email': p.email,
            } for p in partners]
        }
    
    @http.route('/api/v1/partners/<int:partner_id>', 
                type='jsonrpc', auth='user', methods=['GET'])
    def get_partner(self, partner_id):
        """GET /api/v1/partners/:id"""
        partner = request.env['res.partner'].browse(partner_id)
        
        if not partner.exists():
            return {'error': 'Partner not found', 'code': 404}
        
        return {
            'id': partner.id,
            'name': partner.name,
            'email': partner.email,
            'phone': partner.phone,
        }
    
    @http.route('/api/v1/partners', type='jsonrpc', auth='user', methods=['POST'])
    def create_partner(self, name, email, **kwargs):
        """POST /api/v1/partners"""
        try:
            partner = request.env['res.partner'].create({
                'name': name,
                'email': email,
            })
            return {'status': 'created', 'id': partner.id}
        except Exception as e:
            return {'error': str(e), 'code': 400}
    
    @http.route('/api/v1/partners/<int:partner_id>', 
                type='jsonrpc', auth='user', methods=['PUT'])
    def update_partner(self, partner_id, **kwargs):
        """PUT /api/v1/partners/:id"""
        partner = request.env['res.partner'].browse(partner_id)
        
        if not partner.exists():
            return {'error': 'Partner not found', 'code': 404}
        
        partner.write(kwargs)
        return {'status': 'updated', 'id': partner.id}
    
    @http.route('/api/v1/partners/<int:partner_id>', 
                type='jsonrpc', auth='user', methods=['DELETE'])
    def delete_partner(self, partner_id):
        """DELETE /api/v1/partners/:id"""
        partner = request.env['res.partner'].browse(partner_id)
        
        if not partner.exists():
            return {'error': 'Partner not found', 'code': 404}
        
        partner.unlink()
        return {'status': 'deleted'}
```

---

## File Upload/Download

### 1. File Upload

```python
class FileController(http.Controller):
    
    @http.route('/upload/file', type='http', auth='user', 
                methods=['POST'], csrf=True)
    def upload_file(self, file=None, **kwargs):
        """Handle file upload"""
        if not file:
            return request.redirect('/upload?error=no_file')
        
        # Read file content
        file_content = file.read()
        file_name = file.filename
        
        # Create attachment
        attachment = request.env['ir.attachment'].create({
            'name': file_name,
            'datas': base64.b64encode(file_content),
            'res_model': 'res.partner',
            'res_id': request.env.user.partner_id.id,
        })
        
        return request.redirect(f'/upload/success?id={attachment.id}')
    
    @http.route('/upload/multiple', type='http', auth='user', 
                methods=['POST'], csrf=True)
    def upload_multiple(self, files=None, **kwargs):
        """Handle multiple file uploads"""
        if not files:
            return request.redirect('/upload?error=no_files')
        
        attachment_ids = []
        for file in request.httprequest.files.getlist('files'):
            attachment = request.env['ir.attachment'].create({
                'name': file.filename,
                'datas': base64.b64encode(file.read()),
            })
            attachment_ids.append(attachment.id)
        
        return request.render('my_module.upload_success', {
            'attachments': request.env['ir.attachment'].browse(attachment_ids)
        })
```

### 2. File Download

```python
class DownloadController(http.Controller):
    
    @http.route('/download/<int:attachment_id>', type='http', auth='user')
    def download_file(self, attachment_id):
        """Download attachment"""
        attachment = request.env['ir.attachment'].browse(attachment_id)
        
        if not attachment.exists():
            return request.not_found()
        
        # Decode base64 content
        file_content = base64.b64decode(attachment.datas)
        
        return request.make_response(
            file_content,
            headers=[
                ('Content-Type', attachment.mimetype or 'application/octet-stream'),
                ('Content-Disposition', f'attachment; filename="{attachment.name}"'),
                ('Content-Length', len(file_content)),
            ]
        )
    
    @http.route('/download/report/<int:order_id>', type='http', auth='user')
    def download_report(self, order_id):
        """Generate and download PDF report"""
        order = request.env['sale.order'].browse(order_id)
        
        if not order.exists():
            return request.not_found()
        
        # Generate PDF
        pdf_content, _ = request.env['ir.actions.report']._render_qweb_pdf(
            'sale.action_report_saleorder',
            [order_id]
        )
        
        return request.make_response(
            pdf_content,
            headers=[
                ('Content-Type', 'application/pdf'),
                ('Content-Disposition', f'attachment; filename="SO{order.name}.pdf"'),
            ]
        )
```

---

## Best Practices

### 1. Error Handling

```python
class BestPracticeController(http.Controller):
    
    @http.route('/api/safe', type='jsonrpc', auth='user')
    def safe_endpoint(self, **kwargs):
        """Proper error handling"""
        try:
            # Business logic
            result = self._process_data(kwargs)
            
            return {
                'status': 'success',
                'data': result
            }
            
        except ValidationError as e:
            return {
                'status': 'error',
                'message': str(e),
                'code': 400
            }
        except AccessError as e:
            return {
                'status': 'error',
                'message': 'Access denied',
                'code': 403
            }
        except Exception as e:
            _logger.error('Unexpected error: %s', e)
            return {
                'status': 'error',
                'message': 'Internal server error',
                'code': 500
            }
```

### 2. Input Validation

```python
from odoo.exceptions import ValidationError

class ValidationController(http.Controller):
    
    def _validate_email(self, email):
        """Validate email format"""
        import re
        pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        return re.match(pattern, email) is not None
    
    @http.route('/api/contact', type='jsonrpc', auth='public')
    def create_contact(self, name, email, phone=None):
        """Create contact with validation"""
        # Validate required fields
        if not name or not email:
            raise ValidationError('Name and email are required')
        
        # Validate email format
        if not self._validate_email(email):
            raise ValidationError('Invalid email format')
        
        # Create contact
        contact = request.env['res.partner'].sudo().create({
            'name': name,
            'email': email,
            'phone': phone,
        })
        
        return {'status': 'success', 'id': contact.id}
```

### 3. Performance Optimization

```python
class PerformanceController(http.Controller):
    
    @http.route('/api/products/optimized', type='jsonrpc', auth='public')
    def optimized_products(self, limit=100):
        """Optimized product listing"""
        # Use sudo() for public access
        Product = request.env['product.template'].sudo()
        
        # Limit results
        products = Product.search([], limit=limit)
        
        # Use read() instead of browsing
        product_data = products.read(['id', 'name', 'list_price'])
        
        return {
            'count': len(product_data),
            'products': product_data
        }
```

---

## Ví Dụ Thực Tế

### 1. Portal Customer Dashboard

```python
class PortalController(http.Controller):
    
    @http.route('/my/dashboard', type='http', auth='user', website=True)
    def customer_dashboard(self, **kwargs):
        """Customer portal dashboard"""
        partner = request.env.user.partner_id
        
        # Get customer data
        orders = request.env['sale.order'].search([
            ('partner_id', '=', partner.id)
        ], limit=10, order='date_order desc')
        
        invoices = request.env['account.move'].search([
            ('partner_id', '=', partner.id),
            ('move_type', '=', 'out_invoice')
        ], limit=10, order='invoice_date desc')
        
        return request.render('my_module.customer_dashboard', {
            'partner': partner,
            'orders': orders,
            'invoices': invoices,
        })
```

### 2. Webhook Handler

```python
class WebhookController(http.Controller):
    
    @http.route('/webhook/payment', type='jsonrpc', auth='none', csrf=False)
    def payment_webhook(self, **kwargs):
        """Handle payment gateway webhook"""
        # Validate webhook signature
        signature = request.httprequest.headers.get('X-Signature')
        if not self._validate_signature(signature, kwargs):
            return {'error': 'Invalid signature'}
        
        # Process payment
        payment_id = kwargs.get('payment_id')
        status = kwargs.get('status')
        
        payment = request.env['account.payment'].sudo().search([
            ('id', '=', payment_id)
        ], limit=1)
        
        if payment:
            payment.write({'state': status})
        
        return {'status': 'received'}
    
    def _validate_signature(self, signature, data):
        """Validate webhook signature"""
        import hmac
        import hashlib
        
        secret = request.env['ir.config_parameter'].sudo().get_param(
            'payment.webhook.secret'
        )
        
        expected = hmac.new(
            secret.encode(),
            str(data).encode(),
            hashlib.sha256
        ).hexdigest()
        
        return signature == expected
```

### 3. AJAX Form Handler

```python
class AjaxController(http.Controller):
    
    @http.route('/ajax/search', type='jsonrpc', auth='public')
    def ajax_search(self, query='', **kwargs):
        """AJAX search endpoint"""
        if len(query) < 3:
            return {'results': []}
        
        products = request.env['product.template'].sudo().search([
            ('name', 'ilike', query)
        ], limit=10)
        
        return {
            'results': [{
                'id': p.id,
                'name': p.name,
                'price': p.list_price,
                'image_url': f'/web/image/product.template/{p.id}/image_128',
            } for p in products]
        }
```

---

## 🎯 Checklist

Khi tạo Controllers, đảm bảo:

- [ ] Route paths rõ ràng và có ý nghĩa
- [ ] Authentication type phù hợp (public/user/none)
- [ ] CSRF protection cho POST requests
- [ ] Error handling đầy đủ
- [ ] Input validation
- [ ] Proper HTTP status codes
- [ ] Security checks (permissions, access rights)
- [ ] Performance optimization (limit queries, use read())
- [ ] Logging cho debugging
- [ ] Documentation cho API endpoints

---

## 📚 Tài Liệu Tham Khảo

- [Odoo Controllers Documentation](https://www.odoo.com/documentation/19.0/developer/reference/backend/http.html)
- [HTTP Routing](https://www.odoo.com/documentation/19.0/developer/reference/backend/http.html#routing)
- [Request Object](https://www.odoo.com/documentation/19.0/developer/reference/backend/http.html#request)

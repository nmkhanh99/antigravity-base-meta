---
name: Odoo Controllers
description: Hướng dẫn toàn diện về Controllers trong Odoo 19 - xử lý HTTP requests, routing, JSON-RPC, và tích hợp frontend
---

# Odoo 19 Controllers - Reference Guide

## Mục Lục

1. [Cấu Trúc Controller](#cấu-trúc-controller)
2. [HTTP Routing](#http-routing)
3. [Request & Response](#request--response)
4. [Authentication & Security](#authentication--security)
5. [JSON-RPC Controllers](#json-rpc-controllers)
6. [File Upload/Download](#file-uploaddownload)
7. [Best Practices](#best-practices)
8. [Ví Dụ Thực Tế](#ví-dụ-thực-tế)

---

## Cấu Trúc Controller

### Controller với Multiple Routes

```python
class ProductController(http.Controller):

    @http.route([
        '/shop/products',
        '/shop/products/page/<int:page>',
    ], type='http', auth='public', website=True)
    def product_list(self, page=1, **kwargs):
        Product = request.env['product.template'].sudo()
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
        product = request.env['product.template'].sudo().browse(product_id)
        if not product.exists():
            return request.not_found()
        return request.render('my_module.product_detail', {'product': product})
```

---

## HTTP Routing

### Authentication Types

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

    @http.route('/api/endpoint', type='json', auth='none')
    def api_endpoint(self):
        """No authentication - dùng với caution"""
        return {'status': 'ok'}
```

### URL Parameters

```python
class ParamsController(http.Controller):

    @http.route('/search', type='http', auth='public')
    def search(self, query='', category=None, **kwargs):
        """
        URL: /search?query=laptop&category=electronics
        """
        domain = [('name', 'ilike', query)]
        if category:
            domain.append(('categ_id.name', '=', category))
        results = request.env['product.template'].sudo().search(domain)
        return request.render('my_module.search_results', {
            'query': query,
            'category': category,
            'results': results,
        })

    @http.route('/product/<model("product.template"):product>',
                type='http', auth='public')
    def product_by_model(self, product, **kwargs):
        """
        URL: /product/123 - tự động convert ID thành record
        """
        return request.render('my_module.product_detail', {'product': product})
```

---

## Request & Response

### Request Object

```python
class RequestController(http.Controller):

    @http.route('/request/info', type='http', auth='user')
    def request_info(self, **kwargs):
        user = request.env.user                          # Current user
        method = request.httprequest.method              # HTTP method
        headers = request.httprequest.headers            # Headers
        params = request.params                          # Query params
        form_data = request.httprequest.form             # Form data (POST)
        files = request.httprequest.files                # Uploaded files
        session = request.session                        # Session
        env = request.env                                # Environment
        website = request.website                        # Website object

        return request.render('my_module.request_info', {
            'user': user,
            'method': method,
            'params': params,
        })
```

### Response Types

```python
class ResponseController(http.Controller):

    @http.route('/response/html', type='http', auth='public')
    def html_response(self):
        return request.render('my_module.template', {})

    @http.route('/response/redirect', type='http', auth='public')
    def redirect_response(self):
        return request.redirect('/shop')

    @http.route('/response/json', type='json', auth='public')
    def json_response(self):
        return {'status': 'success', 'data': {'key': 'value'}}

    @http.route('/response/file', type='http', auth='user')
    def file_response(self):
        content = b'File content here'
        return request.make_response(
            content,
            headers=[
                ('Content-Type', 'application/pdf'),
                ('Content-Disposition', 'attachment; filename="report.pdf"'),
            ]
        )

    @http.route('/response/not-found', type='http', auth='public')
    def not_found_response(self):
        return request.not_found()
```

---

## Authentication & Security

### CSRF Protection

```python
class CSRFController(http.Controller):

    @http.route('/form/submit', type='http', auth='user',
                methods=['POST'], csrf=True)
    def form_submit(self, **post):
        name = post.get('name')
        email = post.get('email')
        request.env['res.partner'].create({'name': name, 'email': email})
        return request.redirect('/thank-you')
```

### Permission Check

```python
class SecureController(http.Controller):

    @http.route('/secure/data', type='json', auth='user')
    def secure_data(self):
        user = request.env.user
        if not user.has_group('base.group_user'):
            return {'error': 'Unauthorized'}
        return {'user': user.name, 'data': 'Sensitive information'}

    @http.route('/admin/panel', type='http', auth='user')
    def admin_panel(self):
        if not request.env.user.has_group('base.group_system'):
            return request.redirect('/web/login')
        return request.render('my_module.admin_panel', {})
```

### API Key Authentication

```python
class APIController(http.Controller):

    def _check_api_key(self, api_key):
        ApiKey = request.env['my_module.api_key'].sudo()
        return ApiKey.search([('key', '=', api_key)], limit=1).exists()

    @http.route('/api/v1/data', type='json', auth='none', csrf=False)
    def api_data(self, api_key=None, **kwargs):
        if not api_key or not self._check_api_key(api_key):
            return {'error': 'Invalid API key', 'code': 401}
        return {'status': 'success', 'data': []}
```

---

## JSON-RPC Controllers

> **Odoo 19**: Dùng `type='json'` (không phải `type='jsonrpc'`).

### Basic JSON Endpoint

```python
class JSONController(http.Controller):

    @http.route('/api/products', type='json', auth='user')
    def get_products(self, limit=10, offset=0):
        products = request.env['product.template'].search(
            [], limit=limit, offset=offset
        )
        return {
            'count': len(products),
            'products': [{'id': p.id, 'name': p.name, 'price': p.list_price}
                         for p in products]
        }

    @http.route('/api/product/create', type='json', auth='user', methods=['POST'])
    def create_product(self, name, price, **kwargs):
        try:
            product = request.env['product.template'].create({
                'name': name,
                'list_price': price,
            })
            return {'status': 'success', 'id': product.id, 'name': product.name}
        except Exception as e:
            return {'status': 'error', 'message': str(e)}
```

### RESTful API Pattern

```python
class RESTController(http.Controller):

    @http.route('/api/v1/partners', type='json', auth='user', methods=['GET'])
    def list_partners(self, limit=20, offset=0, search=None):
        domain = [('name', 'ilike', search)] if search else []
        partners = request.env['res.partner'].search(domain, limit=limit, offset=offset)
        return {
            'count': len(partners),
            'data': [{'id': p.id, 'name': p.name, 'email': p.email}
                     for p in partners]
        }

    @http.route('/api/v1/partners/<int:partner_id>',
                type='json', auth='user', methods=['GET'])
    def get_partner(self, partner_id):
        partner = request.env['res.partner'].browse(partner_id)
        if not partner.exists():
            return {'error': 'Partner not found', 'code': 404}
        return {'id': partner.id, 'name': partner.name, 'email': partner.email,
                'phone': partner.phone}

    @http.route('/api/v1/partners', type='json', auth='user', methods=['POST'])
    def create_partner(self, name, email, **kwargs):
        try:
            partner = request.env['res.partner'].create({'name': name, 'email': email})
            return {'status': 'created', 'id': partner.id}
        except Exception as e:
            return {'error': str(e), 'code': 400}

    @http.route('/api/v1/partners/<int:partner_id>',
                type='json', auth='user', methods=['PUT'])
    def update_partner(self, partner_id, **kwargs):
        partner = request.env['res.partner'].browse(partner_id)
        if not partner.exists():
            return {'error': 'Partner not found', 'code': 404}
        partner.write(kwargs)
        return {'status': 'updated', 'id': partner.id}

    @http.route('/api/v1/partners/<int:partner_id>',
                type='json', auth='user', methods=['DELETE'])
    def delete_partner(self, partner_id):
        partner = request.env['res.partner'].browse(partner_id)
        if not partner.exists():
            return {'error': 'Partner not found', 'code': 404}
        partner.unlink()
        return {'status': 'deleted'}
```

---

## File Upload/Download

### File Upload

```python
import base64

class FileUploadController(http.Controller):

    @http.route('/upload/file', type='http', auth='user',
                methods=['POST'], csrf=True)
    def upload_file(self, file=None, **kwargs):
        if not file:
            return request.redirect('/upload?error=no_file')
        attachment = request.env['ir.attachment'].create({
            'name': file.filename,
            'datas': base64.b64encode(file.read()),
            'res_model': 'res.partner',
            'res_id': request.env.user.partner_id.id,
        })
        return request.redirect(f'/upload/success?id={attachment.id}')

    @http.route('/upload/multiple', type='http', auth='user',
                methods=['POST'], csrf=True)
    def upload_multiple(self, **kwargs):
        attachment_ids = []
        for file in request.httprequest.files.getlist('files'):
            att = request.env['ir.attachment'].create({
                'name': file.filename,
                'datas': base64.b64encode(file.read()),
            })
            attachment_ids.append(att.id)
        return request.render('my_module.upload_success', {
            'attachments': request.env['ir.attachment'].browse(attachment_ids)
        })
```

### File Download

```python
class FileDownloadController(http.Controller):

    @http.route('/download/<int:attachment_id>', type='http', auth='user')
    def download_file(self, attachment_id):
        attachment = request.env['ir.attachment'].browse(attachment_id)
        if not attachment.exists():
            return request.not_found()
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
        order = request.env['sale.order'].browse(order_id)
        if not order.exists():
            return request.not_found()
        pdf_content, _ = request.env['ir.actions.report']._render_qweb_pdf(
            'sale.action_report_saleorder', [order_id]
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

### Error Handling

```python
import logging
from odoo.exceptions import ValidationError, AccessError

_logger = logging.getLogger(__name__)

class BestPracticeController(http.Controller):

    @http.route('/api/safe', type='json', auth='user')
    def safe_endpoint(self, **kwargs):
        try:
            result = self._process_data(kwargs)
            return {'status': 'success', 'data': result}
        except ValidationError as e:
            return {'status': 'error', 'message': str(e), 'code': 400}
        except AccessError as e:
            return {'status': 'error', 'message': 'Access denied', 'code': 403}
        except Exception as e:
            _logger.error('Unexpected error in safe_endpoint: %s', e)
            return {'status': 'error', 'message': 'Internal server error', 'code': 500}
```

### Input Validation

```python
import re
from odoo.exceptions import ValidationError

class ValidationController(http.Controller):

    def _validate_email(self, email):
        pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        return re.match(pattern, email) is not None

    @http.route('/api/contact', type='json', auth='public')
    def create_contact(self, name, email, phone=None):
        if not name or not email:
            raise ValidationError('Name and email are required')
        if not self._validate_email(email):
            raise ValidationError('Invalid email format')
        contact = request.env['res.partner'].sudo().create({
            'name': name,
            'email': email,
            'phone': phone,
        })
        return {'status': 'success', 'id': contact.id}
```

### Performance Optimization

```python
class PerformanceController(http.Controller):

    @http.route('/api/products/optimized', type='json', auth='public')
    def optimized_products(self, limit=100):
        Product = request.env['product.template'].sudo()
        products = Product.search([], limit=limit)
        # Dùng read() thay vì truy cập từng field trên browse record
        product_data = products.read(['id', 'name', 'list_price'])
        return {'count': len(product_data), 'products': product_data}
```

---

## Ví Dụ Thực Tế

### Portal Customer Dashboard

```python
class PortalController(http.Controller):

    @http.route('/my/dashboard', type='http', auth='user', website=True)
    def customer_dashboard(self, **kwargs):
        partner = request.env.user.partner_id
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

### Webhook Handler

```python
import hmac
import hashlib

class WebhookController(http.Controller):

    @http.route('/webhook/payment', type='json', auth='none', csrf=False)
    def payment_webhook(self, **kwargs):
        signature = request.httprequest.headers.get('X-Signature')
        if not self._validate_signature(signature, kwargs):
            return {'error': 'Invalid signature'}
        payment_id = kwargs.get('payment_id')
        status = kwargs.get('status')
        payment = request.env['account.payment'].sudo().search(
            [('id', '=', payment_id)], limit=1
        )
        if payment:
            payment.write({'state': status})
        return {'status': 'received'}

    def _validate_signature(self, signature, data):
        secret = request.env['ir.config_parameter'].sudo().get_param(
            'payment.webhook.secret'
        )
        expected = hmac.new(
            secret.encode(), str(data).encode(), hashlib.sha256
        ).hexdigest()
        return signature == expected
```

### AJAX Search Endpoint

```python
class AjaxController(http.Controller):

    @http.route('/ajax/search', type='json', auth='public')
    def ajax_search(self, query='', **kwargs):
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

## Checklist

Khi tạo Controllers, đảm bảo:

- [ ] Route paths rõ ràng và có ý nghĩa
- [ ] Authentication type phù hợp (public/user/none)
- [ ] CSRF protection cho POST requests (form submit)
- [ ] Error handling đầy đủ với logging
- [ ] Input validation trước khi write DB
- [ ] Security checks (permissions, ownership)
- [ ] Dùng `read()` thay vì iterate browse records khi cần performance
- [ ] KHÔNG dùng `type='jsonrpc'` - Odoo 19 dùng `type='json'`
- [ ] Portal routes kiểm tra ownership
- [ ] `csrf=False` chỉ cho external webhook/API endpoints

---

## Tài Liệu Tham Khảo

- [Odoo 19 HTTP Controllers](https://www.odoo.com/documentation/19.0/developer/reference/backend/http.html)
- [Routing](https://www.odoo.com/documentation/19.0/developer/reference/backend/http.html#routing)
- [Request Object](https://www.odoo.com/documentation/19.0/developer/reference/backend/http.html#request)

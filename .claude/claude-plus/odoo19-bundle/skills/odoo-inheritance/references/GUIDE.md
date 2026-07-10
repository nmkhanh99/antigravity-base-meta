# Inheritance Skill - Odoo 19

## 1. Inheritance Types Overview

Odoo có 3 loại inheritance chính:

| Type | Syntax | Purpose | Creates New Model? |
|------|--------|---------|-------------------|
| **Classical** | `_inherit` + `_name` | Create specialized model | Yes |
| **Extension** | `_inherit` only | Extend existing model | No |
| **Delegation** | `_inherits` | Composition (has-a) | Yes |

---

## 2. Extension Inheritance (Most Common)

### 2.1 Basic Extension

**Use case**: Add fields/methods to existing model

```python
from odoo import models, fields, api

class ResPartner(models.Model):
    _inherit = 'res.partner'

    # Add new fields
    tax_id = fields.Char(string='Tax ID')
    customer_code = fields.Char(string='Customer Code')
    credit_limit = fields.Float(string='Credit Limit')

    # Add new method
    def check_credit_limit(self):
        for partner in self:
            if partner.credit_limit > 0:
                pass

    # Odoo 19: @api.model_create_multi (không dùng @api.model + create đơn)
    @api.model_create_multi
    def create(self, vals_list):
        for vals in vals_list:
            if not vals.get('customer_code'):
                vals['customer_code'] = self.env['ir.sequence'].next_by_code('res.partner') or '/'
        return super().create(vals_list)
```

### 2.2 Extend Multiple Models

```python
# File: models/res_partner.py
class ResPartner(models.Model):
    _inherit = 'res.partner'
    custom_field = fields.Char()

# File: models/sale_order.py
class SaleOrder(models.Model):
    _inherit = 'sale.order'
    custom_field = fields.Char()
```

### 2.3 Override Computed Fields

```python
class SaleOrder(models.Model):
    _inherit = 'sale.order'

    custom_discount = fields.Float(string='Custom Discount')

    @api.depends('order_line.price_total', 'custom_discount')
    def _compute_amount_total(self):
        super()._compute_amount_total()
        for order in self:
            order.amount_total -= order.custom_discount
```

---

## 3. Classical Inheritance

### 3.1 Basic Classical Inheritance

**Use case**: Create new model based on existing one

```python
# Base model (Odoo built-in example)
class ProductTemplate(models.Model):
    _name = 'product.template'
    _description = 'Product Template'

    name = fields.Char(required=True)

# Classical inheritance - creates new model with own table
class ProductProduct(models.Model):
    _name = 'product.product'
    _inherit = 'product.template'
    _description = 'Product Variant'

    # Inherits all fields from product.template
    barcode = fields.Char(string='Barcode')
    default_code = fields.Char(string='Internal Reference')
```

### 3.2 Real-world Example

```python
class BaseDocument(models.Model):
    _name = 'base.document'
    _description = 'Base Document'

    name = fields.Char(required=True)
    date = fields.Date(default=fields.Date.today)
    state = fields.Selection([
        ('draft', 'Draft'),
        ('confirmed', 'Confirmed'),
    ], default='draft')

    def action_confirm(self):
        self.write({'state': 'confirmed'})

class Invoice(models.Model):
    _name = 'account.invoice.custom'
    _inherit = 'base.document'

    partner_id = fields.Many2one('res.partner')
    amount_total = fields.Float()

class PurchaseOrderCustom(models.Model):
    _name = 'purchase.order.custom'
    _inherit = 'base.document'

    vendor_id = fields.Many2one('res.partner')
```

---

## 4. Delegation Inheritance (_inherits)

### 4.1 Basic Delegation

**Use case**: Model "has-a" relationship (composition)

```python
class ResUsers(models.Model):
    _name = 'res.users'
    _inherits = {'res.partner': 'partner_id'}

    partner_id = fields.Many2one(
        'res.partner',
        required=True,
        ondelete='restrict'
    )

    # User-specific fields
    login = fields.Char(required=True)
    password = fields.Char()

    # Transparent access: user.name, user.email (from res.partner)
```

### 4.2 How It Works

```python
# Create user — partner created automatically
user = self.env['res.users'].create({
    'login': 'john',
    'name': 'John Doe',           # Stored in res.partner
    'email': 'john@example.com',  # Stored in res.partner
})

# Transparent field access
print(user.name)            # From res.partner
print(user.email)           # From res.partner
print(user.login)           # From res.users
print(user.partner_id.name) # Same as user.name
```

### 4.3 Multiple Delegation

```python
class MyModel(models.Model):
    _name = 'my.model'
    _inherits = {
        'res.partner': 'partner_id',
        'res.company': 'company_id',
    }

    partner_id = fields.Many2one('res.partner', required=True, ondelete='cascade')
    company_id = fields.Many2one('res.company', required=True, ondelete='cascade')
    # Has transparent fields from both res.partner and res.company
```

---

## 5. Mixin Classes

### 5.1 Common Mixins

```python
# Mail mixin — adds messaging + chatter
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['mail.thread', 'mail.activity.mixin']

    name = fields.Char(tracking=True)
    state = fields.Selection([('draft', 'Draft'), ('done', 'Done')], tracking=True)

# Portal mixin — adds portal access for external users
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['portal.mixin']

    def _compute_access_url(self):
        super()._compute_access_url()
        for record in self:
            record.access_url = f'/my/models/{record.id}'

# Image mixin — adds image_1920, image_1024, image_512, image_256, image_128
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['image.mixin']
```

### 5.2 Create Custom Mixin

```python
# File: models/mixins.py
class SequenceMixin(models.AbstractModel):
    _name = 'sequence.mixin'
    _description = 'Sequence Mixin'

    name = fields.Char(string='Number', readonly=True, copy=False, default='/')

    @api.model_create_multi
    def create(self, vals_list):
        for vals in vals_list:
            if vals.get('name', '/') == '/':
                vals['name'] = self.env['ir.sequence'].next_by_code(self._get_sequence_code()) or '/'
        return super().create(vals_list)

    def _get_sequence_code(self):
        """Override trong model kế thừa"""
        return self._name

# Dùng mixin
class SaleOrderCustom(models.Model):
    _name = 'sale.order.custom'
    _inherit = ['sequence.mixin']

    def _get_sequence_code(self):
        return 'sale.order.custom'
```

---

## 6. View Inheritance

### 6.1 Extend Form View

```xml
<record id="view_partner_form_inherit_mymodule" model="ir.ui.view">
    <field name="name">res.partner.form.inherit.mymodule</field>
    <field name="model">res.partner</field>
    <field name="inherit_id" ref="base.view_partner_form"/>
    <field name="arch" type="xml">
        <!-- Thêm field sau email -->
        <field name="email" position="after">
            <field name="tax_id"/>
        </field>

        <!-- Thêm page mới trong notebook -->
        <xpath expr="//notebook" position="inside">
            <page string="Custom Info">
                <group>
                    <field name="custom_field"/>
                </group>
            </page>
        </xpath>

        <!-- Chỉ thay đổi attribute — không replace element -->
        <field name="phone" position="attributes">
            <attribute name="required">1</attribute>
        </field>
    </field>
</record>
```

### 6.2 Extend List View (Odoo 19: `<list>`, không dùng `<tree>`)

```xml
<record id="view_partner_list_inherit_mymodule" model="ir.ui.view">
    <field name="name">res.partner.list.inherit.mymodule</field>
    <field name="model">res.partner</field>
    <field name="inherit_id" ref="base.view_partner_tree"/>
    <field name="arch" type="xml">
        <!-- Thêm column sau phone -->
        <field name="phone" position="after">
            <field name="tax_id"/>
        </field>
    </field>
</record>
```

### 6.3 XPath Position Values

| Position | Hành vi |
|----------|---------|
| `after` | Chèn sau node được chọn |
| `before` | Chèn trước node được chọn |
| `inside` | Chèn vào cuối nội dung node |
| `replace` | Thay thế hoàn toàn node (tránh dùng) |
| `attributes` | Sửa attributes của node |

### 6.4 XPath Examples

```xml
<!-- Theo name -->
<xpath expr="//field[@name='partner_id']" position="after">
    <field name="custom_field"/>
</xpath>

<!-- Theo page name -->
<xpath expr="//page[@name='sales_tab']" position="inside">
    <field name="custom_field"/>
</xpath>

<!-- Theo button name -->
<xpath expr="//button[@name='action_confirm']" position="before">
    <button name="action_custom" type="object" string="Custom"/>
</xpath>

<!-- Ẩn field -->
<xpath expr="//field[@name='phone']" position="attributes">
    <attribute name="invisible">1</attribute>
</xpath>
```

---

## 7. Inheritance Patterns

### 7.1 Add Tracking (Chatter)

```python
class SaleOrder(models.Model):
    _inherit = 'sale.order'

    # Thêm tracking cho field có sẵn
    state = fields.Selection(tracking=True)

    # Tracking cho field mới
    custom_field = fields.Char(tracking=True)
```

### 7.2 Add Computed Field

```python
class ResPartner(models.Model):
    _inherit = 'res.partner'

    invoice_count = fields.Integer(
        compute='_compute_invoice_count',
        string='Invoice Count'
    )

    def _compute_invoice_count(self):
        for partner in self:
            partner.invoice_count = self.env['account.move'].search_count([
                ('partner_id', '=', partner.id),
                ('move_type', '=', 'out_invoice')
            ])
```

### 7.3 Add Constraint

```python
class SaleOrder(models.Model):
    _inherit = 'sale.order'

    @api.constrains('amount_total')
    def _check_amount_total(self):
        for order in self:
            if order.amount_total < 0:
                raise ValidationError('Total amount cannot be negative!')
```

### 7.4 Extend Sale Order (Full Example)

```python
class SaleOrder(models.Model):
    _inherit = 'sale.order'

    custom_discount = fields.Float(string='Custom Discount')

    @api.depends('order_line.price_total', 'custom_discount')
    def _compute_amount_total(self):
        super()._compute_amount_total()
        for order in self:
            order.amount_total -= order.custom_discount

    def action_custom_confirm(self):
        self.ensure_one()
        # Custom logic trước
        return self.action_confirm()
```

### 7.5 Extend Product Template

```python
class ProductTemplate(models.Model):
    _inherit = 'product.template'

    manufacturer_id = fields.Many2one('res.partner', string='Manufacturer')
    warranty_months = fields.Integer(string='Warranty (months)')

    is_under_warranty = fields.Boolean(
        compute='_compute_is_under_warranty',
        string='Under Warranty'
    )

    @api.depends('create_date', 'warranty_months')
    def _compute_is_under_warranty(self):
        from dateutil.relativedelta import relativedelta
        today = fields.Date.today()
        for product in self:
            if product.create_date and product.warranty_months:
                expiry = product.create_date.date() + relativedelta(months=product.warranty_months)
                product.is_under_warranty = today <= expiry
            else:
                product.is_under_warranty = False
```

---

## 8. Best Practices

### DO

```python
# Extension inheritance để thêm features
class ResPartner(models.Model):
    _inherit = 'res.partner'
    custom_field = fields.Char()

# Classical cho model chuyên biệt
class ProductVariant(models.Model):
    _name = 'product.variant'
    _inherit = 'product.template'

# Delegation cho composition
class HrEmployee(models.Model):
    _name = 'hr.employee'
    _inherits = {'res.partner': 'partner_id'}

# Luôn gọi super() khi override
@api.model_create_multi
def create(self, vals_list):
    # custom logic
    return super().create(vals_list)

def write(self, vals):
    result = super().write(vals)
    # post-write logic
    return result
```

### DON'T

```python
# Không sửa trực tiếp file Odoo core
# Không edit odoo/addons/base/models/res_partner.py

# Không bỏ super()
@api.model_create_multi
def create(self, vals_list):
    # Thiếu super() — sẽ không lưu record!
    return self.env['my.model'].browse(1)

# Không nhầm _inherits với _inherit
class MyModel(models.Model):
    _name = 'my.model'
    _inherits = 'res.partner'  # Sai! _inherits phải là dict

# Không dùng @api.model + create(self, vals) đơn trong Odoo 19
# Odoo 19 dùng @api.model_create_multi
@api.model
def create(self, vals):  # Deprecated pattern
    return super().create(vals)
```

---

## 9. Debugging Inheritance

### 9.1 Check Inherited Fields

```python
# Trong Python shell (odoo shell)
model = env['res.partner']
print(list(model._fields.keys()))  # Tất cả fields kể cả inherited

# Kiểm tra field có tồn tại không
print('tax_id' in model._fields)
```

### 9.2 Check Inheritance Chain

```python
# Delegation links
print(env['res.users']._inherits)
# {'res.partner': 'partner_id'}

# MRO (Method Resolution Order)
print(env['res.partner'].__class__.__mro__)
```

### 9.3 Check View Inheritance

```python
# Tìm tất cả view inherit từ một view cha
views = env['ir.ui.view'].search([
    ('inherit_id.xml_id', '=', 'base.view_partner_form')
])
for v in views:
    print(v.name, v.module)
```

---

## 10. Odoo 19 Specific Notes

- **`<tree>` → `<list>`**: Odoo 19 dùng `<list>` thay cho `<tree>` trong view definitions. Tuy nhiên khi inheritance, `inherit_id` của view tree cũ vẫn hoạt động.
- **`@api.model_create_multi`**: Bắt buộc dùng thay vì `@api.model` + `create(vals)` đơn.
- **`name_get()` deprecated**: Dùng `_rec_name` + `display_name` compute thay thế.
- **`read_group()` deprecated**: Dùng `_read_group()` (new API) trong Odoo 17+.
- **`attrs` deprecated**: Dùng `invisible`, `required`, `readonly` trực tiếp như attribute trong view XML.

---

**Status**: Complete
**Last Updated**: 2026-06-15

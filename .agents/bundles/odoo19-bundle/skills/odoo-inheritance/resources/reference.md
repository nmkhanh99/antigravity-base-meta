# Inheritance Skill - Odoo 19

## 1. Inheritance Types Overview

Odoo có 3 loại inheritance chính:

| Type | Syntax | Purpose | Creates New Model? |
|------|--------|---------|-------------------|
| **Classical** | `_inherit` + `_name` | Create specialized model | ✅ Yes |
| **Extension** | `_inherit` only | Extend existing model | ❌ No |
| **Delegation** | `_inherits` | Composition (has-a) | ✅ Yes |

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
                # Check logic
                pass
    
    # Override existing method
    @api.model
    def create(self, vals):
        # Generate customer code
        if not vals.get('customer_code'):
            vals['customer_code'] = self.env['ir.sequence'].next_by_code('res.partner')
        return super().create(vals)
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
    
    # Override compute method
    @api.depends('order_line.price_total', 'custom_discount')
    def _compute_amount_total(self):
        super()._compute_amount_total()
        for order in self:
            # Add custom logic
            order.amount_total -= order.custom_discount
```

---

## 3. Classical Inheritance

### 3.1 Basic Classical Inheritance

**Use case**: Create new model based on existing one

```python
class ProductTemplate(models.Model):
    _name = 'product.template'
    _description = 'Product Template'
    
    name = fields.Char(required=True)
    type = fields.Selection([...])

# Classical inheritance - creates new model
class ProductProduct(models.Model):
    _name = 'product.product'
    _inherit = 'product.template'
    _description = 'Product Variant'
    
    # Inherits all fields from product.template
    # Plus adds new fields
    barcode = fields.Char(string='Barcode')
    default_code = fields.Char(string='Internal Reference')
```

### 3.2 Real-world Example

```python
# Base model
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

# Specialized documents
class Invoice(models.Model):
    _name = 'account.invoice'
    _inherit = 'base.document'
    
    # Has all base.document fields + methods
    partner_id = fields.Many2one('res.partner')
    amount_total = fields.Float()

class PurchaseOrder(models.Model):
    _name = 'purchase.order'
    _inherit = 'base.document'
    
    # Has all base.document fields + methods
    vendor_id = fields.Many2one('res.partner')
    order_line_ids = fields.One2many(...)
```

---

## 4. Delegation Inheritance (_inherits)

### 4.1 Basic Delegation

**Use case**: Model "has-a" relationship (composition)

```python
class ResUsers(models.Model):
    _name = 'res.users'
    _inherits = {'res.partner': 'partner_id'}
    
    # Link to partner
    partner_id = fields.Many2one(
        'res.partner',
        required=True,
        ondelete='restrict'
    )
    
    # User-specific fields
    login = fields.Char(required=True)
    password = fields.Char()
    
    # Can access partner fields directly
    # user.name, user.email, user.phone (from res.partner)
```

### 4.2 How It Works

```python
# Create user
user = self.env['res.users'].create({
    'login': 'john',
    'password': 'secret',
    'name': 'John Doe',      # Stored in res.partner
    'email': 'john@example.com',  # Stored in res.partner
})

# Access partner fields
print(user.name)   # From res.partner
print(user.email)  # From res.partner
print(user.login)  # From res.users

# Behind the scenes
print(user.partner_id.name)  # Same as user.name
```

### 4.3 Multiple Delegation

```python
class MyModel(models.Model):
    _name = 'my.model'
    _inherits = {
        'res.partner': 'partner_id',
        'res.company': 'company_id',
    }
    
    partner_id = fields.Many2one('res.partner', required=True)
    company_id = fields.Many2one('res.company', required=True)
    
    # Has fields from both res.partner and res.company
```

---

## 5. Mixin Classes

### 5.1 Common Mixins

```python
# Mail mixin - adds messaging features
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['mail.thread', 'mail.activity.mixin']
    
    name = fields.Char(tracking=True)  # Track changes
    state = fields.Selection([...], tracking=True)

# Portal mixin - adds portal access
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['portal.mixin']
    
    def _compute_access_url(self):
        super()._compute_access_url()
        for record in self:
            record.access_url = f'/my/models/{record.id}'

# Image mixin - adds image fields
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['image.mixin']
    
    # Automatically adds: image_1920, image_1024, image_512, image_256, image_128
```

### 5.2 Create Custom Mixin

```python
# File: models/mixins.py
class SequenceMixin(models.AbstractModel):
    _name = 'sequence.mixin'
    _description = 'Sequence Mixin'
    
    name = fields.Char(string='Number', readonly=True, copy=False)
    
    @api.model
    def create(self, vals):
        if vals.get('name', '/') == '/':
            sequence_code = self._get_sequence_code()
            vals['name'] = self.env['ir.sequence'].next_by_code(sequence_code)
        return super().create(vals)
    
    def _get_sequence_code(self):
        """Override in inherited model"""
        return self._name

# Use mixin
class SaleOrder(models.Model):
    _name = 'sale.order'
    _inherit = ['sequence.mixin']
    
    def _get_sequence_code(self):
        return 'sale.order'
```

---

## 6. View Inheritance

### 6.1 Extend Form View

```xml
<record id="view_partner_form_inherit" model="ir.ui.view">
    <field name="name">res.partner.form.inherit</field>
    <field name="model">res.partner</field>
    <field name="inherit_id" ref="base.view_partner_form"/>
    <field name="arch" type="xml">
        <!-- Add field after email -->
        <field name="email" position="after">
            <field name="tax_id"/>
        </field>
        
        <!-- Add new page in notebook -->
        <xpath expr="//notebook" position="inside">
            <page string="Custom Info">
                <group>
                    <field name="custom_field"/>
                </group>
            </page>
        </xpath>
    </field>
</record>
```

### 6.2 Extend Tree View

```xml
<record id="view_partner_tree_inherit" model="ir.ui.view">
    <field name="name">res.partner.tree.inherit</field>
    <field name="model">res.partner</field>
    <field name="inherit_id" ref="base.view_partner_tree"/>
    <field name="arch" type="xml">
        <!-- Add column -->
        <field name="phone" position="after">
            <field name="tax_id"/>
        </field>
    </field>
</record>
```

### 6.3 Multiple Inheritance

```xml
<!-- Inherit from multiple views -->
<record id="view_partner_form_inherit_1" model="ir.ui.view">
    <field name="inherit_id" ref="base.view_partner_form"/>
    <field name="arch" type="xml">
        <!-- Changes 1 -->
    </field>
</record>

<record id="view_partner_form_inherit_2" model="ir.ui.view">
    <field name="inherit_id" ref="base.view_partner_form"/>
    <field name="arch" type="xml">
        <!-- Changes 2 -->
    </field>
</record>
```

---

## 7. Inheritance Patterns

### 7.1 Add Tracking (Chatter)

```python
class SaleOrder(models.Model):
    _inherit = 'sale.order'
    
    # Add tracking to existing field
    state = fields.Selection(tracking=True)
    
    # Track custom field
    custom_field = fields.Char(tracking=True)
```

### 7.2 Add Compute Field

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

### 7.4 Add Default Value

```python
class SaleOrder(models.Model):
    _inherit = 'sale.order'
    
    # Override default
    state = fields.Selection(default='draft')
    
    # Add default to new field
    custom_field = fields.Char(default='Default Value')
```

---

## 8. Best Practices

### DO ✅

```python
# Use extension inheritance for adding features
class ResPartner(models.Model):
    _inherit = 'res.partner'
    custom_field = fields.Char()

# Use classical for specialized models
class ProductVariant(models.Model):
    _name = 'product.variant'
    _inherit = 'product.template'

# Use delegation for composition
class ResUsers(models.Model):
    _name = 'res.users'
    _inherits = {'res.partner': 'partner_id'}

# Call super() when overriding
@api.model
def create(self, vals):
    result = super().create(vals)
    # Custom logic
    return result
```

### DON'T ❌

```python
# Don't modify core without inheritance
# ❌ Editing odoo/addons/base/models/res_partner.py

# Don't forget super()
def create(self, vals):
    # ❌ Missing super() call
    return self.env['my.model'].browse(1)

# Don't use wrong inheritance type
class MyModel(models.Model):
    _name = 'my.model'
    _inherits = 'res.partner'  # ❌ Should be _inherit for extension

# Don't create circular inheritance
class ModelA(models.Model):
    _inherit = 'model.b'  # ❌ If model.b inherits model.a
```

---

## 9. Debugging Inheritance

### 9.1 Check Inherited Fields

```python
# In Python shell
model = self.env['res.partner']
print(model._fields.keys())  # All fields including inherited

# Check if field is inherited
print('tax_id' in model._fields)
```

### 9.2 Check Inheritance Chain

```python
# Check what models inherit from this
print(self.env['res.partner']._inherits)

# Check MRO (Method Resolution Order)
print(self.env['res.partner'].__class__.__mro__)
```

---

## 10. Common Patterns

### 10.1 Extend Sale Order

```python
class SaleOrder(models.Model):
    _inherit = 'sale.order'
    
    # Add custom fields
    custom_discount = fields.Float()
    
    # Override compute
    @api.depends('order_line.price_total', 'custom_discount')
    def _compute_amount_total(self):
        super()._compute_amount_total()
        for order in self:
            order.amount_total -= order.custom_discount
    
    # Add custom action
    def action_custom_confirm(self):
        self.ensure_one()
        # Custom logic
        return self.action_confirm()
```

### 10.2 Extend Product

```python
class ProductTemplate(models.Model):
    _inherit = 'product.template'
    
    # Add fields
    manufacturer_id = fields.Many2one('res.partner')
    warranty_months = fields.Integer()
    
    # Add compute
    is_under_warranty = fields.Boolean(
        compute='_compute_is_under_warranty'
    )
    
    @api.depends('create_date', 'warranty_months')
    def _compute_is_under_warranty(self):
        for product in self:
            # Logic
            pass
```

---

**Status**: ✅ Complete  
**Last Updated**: 24/01/2026

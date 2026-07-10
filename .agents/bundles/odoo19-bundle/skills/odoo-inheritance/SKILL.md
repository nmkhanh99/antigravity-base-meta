---
name: odoo-inheritance
description: Hướng dẫn 3 loại Inheritance trong Odoo 19 - Classical, Extension, Delegation và view inheritance patterns. Use when the user asks to extend model, inherit view, override method, add fields to existing model, or use delegation inheritance.
---

# Odoo 19 Inheritance

## Goal
Giúp agent sử dụng đúng loại inheritance phù hợp khi mở rộng models và views trong Odoo 19.

## When to use this skill
- "kế thừa model", "extend model", "inherit model"
- "thêm field vào model có sẵn", "override method"
- "delegation inheritance", "_inherits"
- "view inheritance", "xpath"
- "_inherit", "_name"

## Instructions

### 1. Extension Inheritance (phổ biến nhất)
Thêm fields/methods vào model có sẵn, **không tạo model mới**.
```python
class ResPartner(models.Model):
    _inherit = 'res.partner'  # No _name → extend existing

    tax_id = fields.Char(string='Tax ID')
    credit_limit = fields.Float(string='Credit Limit')

    def action_custom(self):
        # Override or extend existing methods
        result = super().action_custom()
        return result
```

### 2. Classical Inheritance
Tạo model mới **copy từ model cha**, có bảng riêng.
```python
class CustomPartner(models.Model):
    _name = 'custom.partner'       # New model name
    _inherit = 'res.partner'       # Copy from parent
    _description = 'Custom Partner'

    custom_field = fields.Char()
```

### 3. Delegation Inheritance
Tạo model mới, **link đến model cha** qua Many2one. Truy cập transparent field cha.
```python
class Employee(models.Model):
    _name = 'hr.employee'
    _inherits = {'res.partner': 'partner_id'}  # Note: _inherits (plural)

    partner_id = fields.Many2one('res.partner', required=True, ondelete='cascade')
    department_id = fields.Many2one('hr.department')
    # Can access partner fields directly: employee.name, employee.email
```

### 4. Multiple Inheritance (Mixins)
```python
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['mail.thread', 'mail.activity.mixin', 'portal.mixin']
```

### 5. Method Override Patterns
```python
# Extend (call super)
def write(self, vals):
    if 'state' in vals:
        vals['state_date'] = fields.Datetime.now()
    return super().write(vals)

# Odoo 19: @api.model_create_multi
@api.model_create_multi
def create(self, vals_list):
    for vals in vals_list:
        vals['name'] = self.env['ir.sequence'].next_by_code('my.model')
    return super().create(vals_list)
```

### 6. View Inheritance
```xml
<record id="view_inherit" model="ir.ui.view">
    <field name="inherit_id" ref="base.view_partner_form"/>
    <field name="arch" type="xml">
        <!-- position: after, before, inside, replace, attributes -->
        <field name="email" position="after">
            <field name="tax_id"/>
        </field>
        <xpath expr="//page[@name='sales']" position="inside">
            <field name="credit_limit"/>
        </xpath>
    </field>
</record>
```

## Constraints
- `_inherit` (singular) = model inheritance. `_inherits` (plural) = delegation.
- Extension: KHÔNG khai báo `_name` mới (giữ tên model gốc).
- Luôn gọi `super()` khi override CRUD methods.

## Best practices
- Ưu tiên Extension inheritance cho việc thêm fields vào model có sẵn.
- Dùng mixins (`mail.thread`, `portal.mixin`) qua multiple inheritance.
- Dùng `position="attributes"` để chỉ thay đổi attributes, không replace element.

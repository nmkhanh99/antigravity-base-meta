---
name: odoo-inheritance
description: Hướng dẫn 3 loại Inheritance trong Odoo 19 - Classical, Extension, Delegation và view inheritance patterns. Use when the user asks to extend model, inherit view, override method, add fields to existing model, or use delegation inheritance.
---

# Odoo 19 Inheritance

## Goal
Giúp agent sử dụng đúng loại inheritance phù hợp khi mở rộng models và views trong Odoo 19, tránh các lỗi phổ biến về `_inherit` vs `_inherits` và cách gọi `super()`.

## When to use this skill
- "kế thừa model", "extend model", "inherit model"
- "thêm field vào model có sẵn", "override method"
- "delegation inheritance", "_inherits"
- "view inheritance", "xpath"
- "_inherit", "_name"
- "mixin", "mail.thread", "portal.mixin"
- "classical inheritance", "create specialized model"

## Instructions

### Bước 1 — Xác định loại inheritance phù hợp

| Loại | Cú pháp | Mục đích | Tạo model mới? |
|------|---------|---------|----------------|
| **Extension** | `_inherit` only | Thêm field/method vào model có sẵn | Không |
| **Classical** | `_inherit` + `_name` | Tạo model mới copy từ cha | Có |
| **Delegation** | `_inherits` (plural) | Composition (has-a relationship) | Có |

### Bước 2 — Áp dụng pattern đúng

**Extension Inheritance** (phổ biến nhất — thêm field vào model có sẵn):
```python
class ResPartner(models.Model):
    _inherit = 'res.partner'  # Không có _name → extend model gốc

    tax_id = fields.Char(string='Tax ID')
    credit_limit = fields.Float(string='Credit Limit')

    # Override CRUD — luôn gọi super()
    @api.model_create_multi
    def create(self, vals_list):
        for vals in vals_list:
            if not vals.get('customer_code'):
                vals['customer_code'] = self.env['ir.sequence'].next_by_code('res.partner')
        return super().create(vals_list)
```

**Classical Inheritance** (tạo model mới từ model cha):
```python
class CustomPartner(models.Model):
    _name = 'custom.partner'       # Model mới
    _inherit = 'res.partner'       # Copy fields/methods từ cha
    _description = 'Custom Partner'

    custom_field = fields.Char()
```

**Delegation Inheritance** (composition — truy cập transparent field cha):
```python
class HrEmployee(models.Model):
    _name = 'hr.employee'
    _inherits = {'res.partner': 'partner_id'}  # _inherits (plural)

    partner_id = fields.Many2one('res.partner', required=True, ondelete='cascade')
    department_id = fields.Many2one('hr.department')
    # Truy cập field cha trực tiếp: employee.name, employee.email
```

**Multiple Inheritance / Mixins**:
```python
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['mail.thread', 'mail.activity.mixin', 'portal.mixin']

    name = fields.Char(tracking=True)
    state = fields.Selection([('draft', 'Draft'), ('done', 'Done')], tracking=True)
```

### Bước 3 — View Inheritance

```xml
<record id="view_partner_form_inherit" model="ir.ui.view">
    <field name="name">res.partner.form.inherit.mymodule</field>
    <field name="model">res.partner</field>
    <field name="inherit_id" ref="base.view_partner_form"/>
    <field name="arch" type="xml">
        <!-- Thêm field sau email -->
        <field name="email" position="after">
            <field name="tax_id"/>
        </field>
        <!-- Dùng xpath cho selector phức tạp -->
        <xpath expr="//notebook" position="inside">
            <page string="Custom Info">
                <group>
                    <field name="credit_limit"/>
                </group>
            </page>
        </xpath>
        <!-- Chỉ sửa attribute, không replace element -->
        <field name="phone" position="attributes">
            <attribute name="required">1</attribute>
        </field>
    </field>
</record>
```

### Bước 4 — Override Method Patterns

```python
# Write — extend ghi dữ liệu
def write(self, vals):
    if 'state' in vals:
        vals['state_date'] = fields.Datetime.now()
    return super().write(vals)

# Odoo 19: dùng @api.model_create_multi (không dùng @api.model + create đơn lẻ)
@api.model_create_multi
def create(self, vals_list):
    for vals in vals_list:
        vals['name'] = self.env['ir.sequence'].next_by_code('my.model') or '/'
    return super().create(vals_list)
```

### Bước 5 — Custom Mixin

```python
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
        return self._name
```

## Constraints
- `_inherit` (singular) = model inheritance. `_inherits` (plural) = delegation — **không nhầm lẫn**.
- Extension inheritance: **KHÔNG khai báo `_name` mới** — nếu thêm `_name` sẽ thành Classical.
- **Luôn gọi `super()`** khi override `create`, `write`, `unlink`, `read`.
- Odoo 19: dùng `@api.model_create_multi` thay vì `@api.model` + `create(self, vals)` đơn lẻ.
- **Không sửa trực tiếp** file trong `odoo/addons/` — phải dùng inheritance.
- Delegation inheritance (`_inherits`): field link (`partner_id`) phải `required=True`.
- View inheritance: ưu tiên `position="attributes"` khi chỉ cần thay đổi attribute, không dùng `position="replace"` nếu tránh được.

## References
- https://www.odoo.com/documentation/19.0/developer/tutorials/server_framework_101/04_basicmodel.html
- https://www.odoo.com/documentation/19.0/developer/reference/backend/orm.html#inheritance-and-extension
- https://www.odoo.com/documentation/19.0/developer/reference/backend/views.html#inheritance

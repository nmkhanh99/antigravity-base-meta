# Models Skill - Odoo 19 Reference Guide

## 1. Model Definition

### 1.1 Basic Model Structure

```python
from odoo import models, fields, api
from odoo.exceptions import ValidationError

class MyModel(models.Model):
    _name = 'my.module.mymodel'
    _description = 'My Model Description'
    _order = 'name'          # Default sorting
    _rec_name = 'name'       # Field used for display_name

    name = fields.Char(string='Name', required=True, index=True)
    description = fields.Text(string='Description')
    active = fields.Boolean(string='Active', default=True)
```

### 1.2 Model Attributes & Inheritance

```python
class MyModel(models.Model):
    _name = 'my.model'
    _description = 'My Model'
    _inherit = ['mail.thread', 'mail.activity.mixin']  # Multiple inheritance
    _order = 'sequence, name'
    _rec_name = 'display_name'

    # Odoo 19: Declarative constraints (preferred — biến phải bắt đầu bằng _)
    _name_uniq = models.UniqueIndex('(name)', 'Name must be unique!')
    _check_amount = models.Constraint('CHECK(amount >= 0)', 'Amount must be positive!')

    # Index chỉ tạo DB index (không phải constraint)
    _partner_date_idx = models.Index('partner_id', 'date')

    # Automatic fields added by Odoo (không cần khai báo):
    # id, create_date, create_uid, write_date, write_uid
```

---

## 2. Field Types

### 2.1 Basic Fields

```python
# Text fields
name = fields.Char(string='Name', size=100, required=True)
description = fields.Text(string='Description')
html_content = fields.Html(string='HTML Content')

# Numeric fields
quantity = fields.Integer(string='Quantity', default=0)
price = fields.Float(string='Price', digits=(16, 2))
amount = fields.Monetary(string='Amount', currency_field='currency_id')

# Boolean
active = fields.Boolean(string='Active', default=True)

# Date/Time
date = fields.Date(string='Date', default=fields.Date.today)
datetime_field = fields.Datetime(string='DateTime', default=fields.Datetime.now)

# Selection (dropdown)
state = fields.Selection([
    ('draft', 'Draft'),
    ('confirmed', 'Confirmed'),
    ('done', 'Done'),
], string='Status', default='draft')

# Binary
image = fields.Binary(string='Image')
attachment = fields.Binary(string='Attachment', attachment=True)
```

### 2.2 Relational Fields

```python
# Many2one (Foreign Key)
partner_id = fields.Many2one(
    'res.partner',
    string='Customer',
    required=True,
    ondelete='cascade',   # 'cascade' | 'restrict' | 'set null'
    index=True
)

# One2many (Reverse of Many2one)
line_ids = fields.One2many(
    'my.model.line',   # Related model
    'order_id',        # Field in related model pointing back
    string='Lines'
)

# Many2many
tag_ids = fields.Many2many(
    'my.tag',
    'my_model_tag_rel',  # Relation table name
    'model_id',          # Column for this model
    'tag_id',            # Column for related model
    string='Tags'
)
```

### 2.3 Computed Fields

```python
class SaleOrder(models.Model):
    _name = 'sale.order'

    line_ids = fields.One2many('sale.order.line', 'order_id')

    # Computed + stored (ghi vào DB)
    total_amount = fields.Float(
        string='Total',
        compute='_compute_total_amount',
        store=True
    )

    @api.depends('line_ids.price_subtotal')
    def _compute_total_amount(self):
        for record in self:
            record.total_amount = sum(record.line_ids.mapped('price_subtotal'))

    # Computed không stored (tính mỗi lần đọc)
    display_name = fields.Char(compute='_compute_display_name')

    @api.depends('name', 'partner_id.name')
    def _compute_display_name(self):
        for record in self:
            record.display_name = f"{record.name} - {record.partner_id.name}"
```

### 2.4 Related Fields

```python
partner_id = fields.Many2one('res.partner')
partner_email = fields.Char(
    related='partner_id.email',
    string='Partner Email',
    store=True,      # Optional: lưu vào DB
    readonly=False   # Optional: cho phép sửa
)
```

---

## 3. ORM Methods

### 3.1 CRUD Operations

```python
# CREATE — single record
partner = self.env['res.partner'].create({
    'name': 'John Doe',
    'email': 'john@example.com',
})

# CREATE — multiple records (preferred)
partners = self.env['res.partner'].create([
    {'name': 'Partner 1'},
    {'name': 'Partner 2'},
])

# READ — search
partners = self.env['res.partner'].search([
    ('is_company', '=', True),
    ('country_id.code', '=', 'US')
], limit=10, order='name')

# READ — browse by ID
partner = self.env['res.partner'].browse(partner_id)

# READ — search_fetch (Odoo 19, ít SQL hơn search_read)
partners = self.env['res.partner'].search_fetch(
    [('is_company', '=', True)],
    ['name', 'email', 'phone'],
    limit=10,
)
# Trả về recordset đã prefetch fields

# READ — fetch (browse + prefetch in one call)
partners = self.env['res.partner'].browse(ids)
partners.fetch(['name', 'email'])

# UPDATE
partner.write({
    'phone': '+1234567890',
    'email': 'newemail@example.com',
})

# DELETE
partner.unlink()
```

### 3.2 Search Domains

```python
# Operators cơ bản
[('name', '=', 'John')]
[('age', '>', 18)]
[('email', '!=', False)]
[('name', 'in', ['John', 'Jane'])]
[('name', 'not in', ['Admin'])]
[('description', 'like', '%odoo%')]
[('description', 'ilike', '%ODOO%')]   # Case-insensitive

# Logical operators
['|', ('name', '=', 'John'), ('name', '=', 'Jane')]   # OR
['&', ('age', '>', 18), ('country', '=', 'US')]        # AND (mặc định)
['!', ('active', '=', False)]                           # NOT

# Related fields
[('partner_id.country_id.code', '=', 'US')]

# Odoo 19: Dynamic dates
[('date_order', '>=', 'today')]   # Evaluates to today
[('write_date', '>=', 'now')]     # Evaluates to current datetime

# Odoo 19: 'any'/'not any' cho x2many
[('line_ids', 'any', [('product_id.type', '=', 'consu')])]
[('line_ids', 'not any', [('qty_delivered', '>', 0)])]

# Odoo 19: Date granularity
[('create_date.month_number', '=', 1)]   # January
[('create_date.year', '=', 2024)]         # Year 2024
```

### 3.3 Domain Class (Odoo 19)

```python
from odoo.osv.expression import Domain

company_domain = Domain('is_company', '=', True)
country_domain = Domain('country_id.code', '=', 'US')

combined = company_domain & country_domain   # AND
either = company_domain | country_domain      # OR
negated = ~company_domain                     # NOT

partners = self.env['res.partner'].search(combined)
```

### 3.4 Aggregation

```python
# Count
count = self.env['res.partner'].search_count([('is_company', '=', True)])

# Odoo 19: _read_group (private/backend)
result = self.env['sale.order']._read_group(
    [('state', '=', 'sale')],
    groupby=['partner_id'],
    aggregates=['amount_total:sum'],
)

# Odoo 19: formatted_read_group (public API)
result = self.env['sale.order'].formatted_read_group(
    [('state', '=', 'sale')],
    ['amount_total:sum'],
    ['partner_id'],
)

# DEPRECATED — KHÔNG dùng:
# self.env['sale.order'].read_group(...)
```

---

## 4. Decorators & API

### 4.1 @api.depends

```python
@api.depends('quantity', 'price_unit')
def _compute_price_subtotal(self):
    for record in self:
        record.price_subtotal = record.quantity * record.price_unit
```

### 4.2 @api.onchange

```python
@api.onchange('partner_id')
def _onchange_partner_id(self):
    if self.partner_id:
        self.payment_term_id = self.partner_id.property_payment_term_id
        return {
            'warning': {
                'title': 'Warning',
                'message': 'Partner changed!'
            }
        }
```

### 4.3 @api.constrains

```python
@api.constrains('start_date', 'end_date')
def _check_dates(self):
    for record in self:
        if record.start_date and record.end_date:
            if record.start_date > record.end_date:
                raise ValidationError('Start date must be before end date!')
```

### 4.4 @api.model_create_multi (Odoo 19 — thay thế @api.model cho create)

```python
@api.model_create_multi
def create(self, vals_list):
    for vals in vals_list:
        if vals.get('name', '/') == '/':
            vals['name'] = self.env['ir.sequence'].next_by_code('my.model') or '/'
    return super().create(vals_list)
```

### 4.5 @api.private (Odoo 19)

```python
@api.private
def _internal_computation(self):
    """Không thể gọi qua XML-RPC/JSON-RPC"""
    return self.amount_total * self.tax_rate
```

---

## 5. Constraints

### 5.1 Declarative Constraints — Odoo 19 (preferred)

```python
from odoo import models

class MyModel(models.Model):
    _name = 'my.model'

    # Tên biến PHẢI bắt đầu bằng _
    # UniqueIndex: single column
    _name_uniq = models.UniqueIndex('(name)', 'Name must be unique!')

    # UniqueIndex: multiple columns
    _code_company_uniq = models.UniqueIndex('(code, company_id)', 'Code must be unique per company!')

    # Constraint: CHECK constraint
    _quantity_positive = models.Constraint(
        'CHECK(quantity >= 0)',
        'Quantity must be positive!',
    )

    # Index: chỉ tạo DB index, không phải constraint
    _partner_date_idx = models.Index('partner_id', 'date')
```

### 5.2 Legacy SQL Constraints (vẫn hoạt động nhưng không nên dùng)

```python
# DEPRECATED trong Odoo 19 — dùng declarative API ở trên
_sql_constraints = [
    ('name_unique', 'unique(name)', 'Name must be unique!'),
    ('quantity_positive', 'CHECK(quantity >= 0)', 'Quantity must be positive!'),
]
```

### 5.3 Python Constraints

```python
from odoo.exceptions import ValidationError

@api.constrains('quantity')
def _check_quantity(self):
    for record in self:
        if record.quantity < 0:
            raise ValidationError('Quantity cannot be negative!')
```

---

## 6. Default Values

```python
class MyModel(models.Model):
    _name = 'my.model'

    # Static default
    active = fields.Boolean(default=True)

    # Function default
    date = fields.Date(default=fields.Date.today)

    # Lambda default
    user_id = fields.Many2one('res.users', default=lambda self: self.env.user)

    # Method default
    company_id = fields.Many2one(
        'res.company',
        default=lambda self: self.env.company
    )
```

---

## 7. Override CRUD Methods

### 7.1 Create

```python
@api.model_create_multi
def create(self, vals_list):
    for vals in vals_list:
        if vals.get('name', '/') == '/':
            vals['name'] = self.env['ir.sequence'].next_by_code('my.model') or '/'
    records = super().create(vals_list)
    records._send_notification()
    return records
```

### 7.2 Write

```python
def write(self, vals):
    if 'state' in vals and vals['state'] == 'done':
        vals['done_date'] = fields.Datetime.now()
    result = super().write(vals)
    if 'partner_id' in vals:
        self._update_related_records()
    return result
```

### 7.3 Unlink

```python
def unlink(self):
    if any(record.state == 'done' for record in self):
        raise ValidationError('Cannot delete confirmed records!')
    return super().unlink()
```

### 7.4 Custom Display Name (thay thế name_get)

```python
# Odoo 19: dùng _compute_display_name thay vì name_get()
@api.depends('code', 'name')
def _compute_display_name(self):
    for record in self:
        record.display_name = f"[{record.code}] {record.name}"

# DEPRECATED — KHÔNG dùng trong Odoo 19:
# def name_get(self):
#     ...
```

---

## 8. Context & Environment

```python
# Access environment
self.env           # Current environment
self.env.user      # Current user (res.users record)
self.env.company   # Current company
self.env.companies # All active companies for user
self.env.lang      # Current language code
self.env.context   # Context dict (read-only)
self.env.cr        # DB cursor
self.env.uid       # Current user ID (int)

# DEPRECATED — KHÔNG dùng trong Odoo 19:
# self._cr      → self.env.cr
# self._uid     → self.env.uid
# self._context → self.env.context

# Thay đổi context
records = self.with_context(lang='en_US', tz='UTC').search([])

# Sudo (bypass access rights)
partner = self.env['res.partner'].sudo().browse(partner_id)

# Thay đổi company
records = self.with_company(company_id).search([])
```

---

## 9. Best Practices

### DO — Nên làm

```python
# Dùng ORM (không raw SQL)
partners = self.env['res.partner'].search([('is_company', '=', True)])

# Batch write
partners.write({'active': False})

# mapped, filtered, sorted
emails = partners.mapped('email')
active_partners = partners.filtered(lambda p: p.active)
sorted_partners = partners.sorted('name')

# Đúng @api.depends
@api.depends('line_ids.price_subtotal')
def _compute_total(self):
    for rec in self:
        rec.total = sum(rec.line_ids.mapped('price_subtotal'))

# search_fetch thay search_read
partners = self.env['res.partner'].search_fetch(
    [('is_company', '=', True)],
    ['name', 'email'],
)
```

### DON'T — Không nên làm

```python
# Raw SQL
self.env.cr.execute("SELECT * FROM res_partner")  # KHÔNG

# Loop write
for partner in partners:
    partner.write({'active': False})   # KHÔNG — dùng batch

# Thiếu @api.depends
def _compute_total(self):             # KHÔNG — mất @api.depends
    pass

# Browse trong vòng lặp
for pid in partner_ids:
    p = self.env['res.partner'].browse(pid)  # KHÔNG — dùng browse(partner_ids)

# read_group (deprecated)
self.env['sale.order'].read_group(...)   # KHÔNG — dùng _read_group

# name_get (deprecated)
def name_get(self): ...                  # KHÔNG — dùng _compute_display_name
```

---

## 10. Common Patterns

### 10.1 Sequence Generation

```python
@api.model_create_multi
def create(self, vals_list):
    for vals in vals_list:
        if vals.get('name', '/') == '/':
            vals['name'] = self.env['ir.sequence'].next_by_code('my.model') or '/'
    return super().create(vals_list)
```

### 10.2 State Management

```python
state = fields.Selection([
    ('draft', 'Draft'),
    ('confirmed', 'Confirmed'),
    ('done', 'Done'),
    ('cancel', 'Cancelled'),
], default='draft')

def action_confirm(self):
    self.write({'state': 'confirmed'})

def action_done(self):
    self.write({'state': 'done', 'done_date': fields.Datetime.now()})

def action_cancel(self):
    self.write({'state': 'cancel'})
```

### 10.3 Multi-company

```python
company_id = fields.Many2one(
    'res.company',
    string='Company',
    required=True,
    default=lambda self: self.env.company
)

# Thêm record rule trong security XML để lọc theo company
```

### 10.4 Chaining Environment Modifiers

```python
# sudo + with_company + with_context
record = self.env['my.model'].sudo().with_company(company_id).with_context(
    lang='vi_VN'
).search([('state', '=', 'draft')])
```

---

## 11. Deprecated API Summary (Odoo 19)

| Deprecated | Thay thế |
|---|---|
| `name_get()` | `_compute_display_name` |
| `_sql_constraints` list | `models.UniqueIndex` / `models.Constraint` |
| `read_group()` | `_read_group()` hoặc `formatted_read_group()` |
| `search_read()` | `search_fetch()` (ưu tiên) |
| `@api.model` trên `create` | `@api.model_create_multi` |
| `self._cr` | `self.env.cr` |
| `self._uid` | `self.env.uid` |
| `self._context` | `self.env.context` |
| `<tree>` tag in views | `<list>` tag |
| `attrs="{}"` in views | Direct modifiers: `invisible="..."` |

---

**Status**: Complete
**Odoo Version**: 19.0
**Last Updated**: 2026-06-15

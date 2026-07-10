# Models Skill - Odoo 19

## 1. Model Definition

### 1.1 Basic Model Structure

```python
from odoo import models, fields, api
from odoo.exceptions import ValidationError

class MyModel(models.Model):
    _name = 'my.module.mymodel'
    _description = 'My Model Description'
    _order = 'name'  # Default sorting
    _rec_name = 'name'  # Field used for display_name
    
    # Fields
    name = fields.Char(string='Name', required=True, index=True)
    description = fields.Text(string='Description')
    active = fields.Boolean(string='Active', default=True)
```

### 1.2 Model Attributes

```python
class MyModel(models.Model):
    _name = 'my.model'
    _description = 'My Model'
    _inherit = ['mail.thread', 'mail.activity.mixin']  # Multiple inheritance
    _order = 'sequence, name'
    _rec_name = 'display_name'
    
    # Odoo 19: Declarative constraints (preferred)
    name_uniq = models.UniqueIndex('name')
    
    # ⚠️ Legacy: _sql_constraints still works but prefer declarative API
    # _sql_constraints = [
    #     ('name_uniq', 'unique(name)', 'Name must be unique!')
    # ]
    
    # Automatic fields (added by Odoo)
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
datetime = fields.Datetime(string='DateTime', default=fields.Datetime.now)

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
    ondelete='cascade',  # or 'restrict', 'set null'
    index=True
)

# One2many (Reverse of Many2one)
line_ids = fields.One2many(
    'my.model.line',  # Related model
    'order_id',       # Field in related model pointing back
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
    
    # Computed field (stored)
    total_amount = fields.Float(
        string='Total',
        compute='_compute_total_amount',
        store=True  # Store in database
    )
    
    @api.depends('line_ids.price_subtotal')
    def _compute_total_amount(self):
        for record in self:
            record.total_amount = sum(record.line_ids.mapped('price_subtotal'))
    
    # Computed field (non-stored)
    display_name = fields.Char(
        compute='_compute_display_name'
    )
    
    @api.depends('name', 'partner_id.name')
    def _compute_display_name(self):
        for record in self:
            record.display_name = f"{record.name} - {record.partner_id.name}"
```

### 2.4 Related Fields

```python
# Shortcut to access related model's field
partner_id = fields.Many2one('res.partner')
partner_email = fields.Char(
    related='partner_id.email',
    string='Partner Email',
    store=True,  # Optional: store in DB
    readonly=False  # Optional: make editable
)
```

---

## 3. ORM Methods

### 3.1 CRUD Operations

```python
# CREATE
partner = self.env['res.partner'].create({
    'name': 'John Doe',
    'email': 'john@example.com',
})

# Multiple records
partners = self.env['res.partner'].create([
    {'name': 'Partner 1'},
    {'name': 'Partner 2'},
])

# READ (Search)
partners = self.env['res.partner'].search([
    ('is_company', '=', True),
    ('country_id.code', '=', 'US')
], limit=10, order='name')

# Browse (by ID)
partner = self.env['res.partner'].browse(partner_id)

# Search + Read (legacy - still works)
partner_data = self.env['res.partner'].search_read(
    [('is_company', '=', True)],
    ['name', 'email', 'phone']
)

# Odoo 19: search_fetch (optimized - fewer SQL queries)
partners = self.env['res.partner'].search_fetch(
    [('is_company', '=', True)],
    ['name', 'email', 'phone'],
    limit=10,
)
# Returns recordset with pre-fetched fields

# Odoo 19: fetch (like browse + read in one call)
partners = self.env['res.partner'].browse(ids)
partners.fetch(['name', 'email'])  # Pre-loads fields

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
# Basic operators
[('name', '=', 'John')]
[('age', '>', 18)]
[('email', '!=', False)]
[('name', 'in', ['John', 'Jane'])]
[('name', 'not in', ['Admin'])]
[('description', 'like', '%odoo%')]
[('description', 'ilike', '%ODOO%')]  # Case-insensitive

# Logical operators
['|', ('name', '=', 'John'), ('name', '=', 'Jane')]  # OR
['&', ('age', '>', 18), ('country', '=', 'US')]      # AND (default)
['!', ('active', '=', False)]                         # NOT

# Related fields
[('partner_id.country_id.code', '=', 'US')]

# Date ranges
[('create_date', '>=', '2024-01-01')]
[('create_date', '<=', fields.Date.today())]

# Odoo 19: Dynamic dates in domains
[('date_order', '>=', 'today')]  # Dynamic: evaluates to today
[('write_date', '>=', 'now')]    # Dynamic: evaluates to current datetime

# Odoo 19: 'any'/'not any' operators (for x2many fields)
[('line_ids', 'any', [('product_id.type', '=', 'consu')])]  # Has any line with consumable
[('line_ids', 'not any', [('qty_delivered', '>', 0)])]       # No line delivered

# Date granularity in domains
[('create_date.month_number', '=', 1)]  # January
[('create_date.year', '=', 2024)]       # Year 2024
```

### 3.3 Domain Class (Odoo 19)

```python
from odoo.osv.expression import Domain

# Build domains programmatically
domain = Domain('is_company', '=', True)

# Combine domains with & (AND) and | (OR)
company_domain = Domain('is_company', '=', True)
country_domain = Domain('country_id.code', '=', 'US')
combined = company_domain & country_domain

# OR combination
either_domain = company_domain | country_domain

# Negate with ~
not_company = ~company_domain

# Use in search
partners = self.env['res.partner'].search(combined)
```

### 3.3 Aggregation

```python
# Count
count = self.env['res.partner'].search_count([('is_company', '=', True)])

# ⚠️ DEPRECATED: read_group (use _read_group or formatted_read_group)
# result = self.env['sale.order'].read_group(...)

# Odoo 19: _read_group (backend / private)
result = self.env['sale.order']._read_group(
    [('state', '=', 'sale')],       # Domain
    groupby=['partner_id'],          # Group by
    aggregates=['amount_total:sum'], # Aggregation
)

# Odoo 19: formatted_read_group (public API, formatted output)
result = self.env['sale.order'].formatted_read_group(
    [('state', '=', 'sale')],
    ['amount_total:sum'],
    ['partner_id'],
)
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
        # Return warning
        return {
            'warning': {
                'title': 'Warning',
                'message': 'Partner changed!'
            }
        }
```

### 4.3 @api.constrains

```python
@api.constrains('quantity')
def _check_quantity(self):
    for record in self:
        if record.quantity < 0:
            raise ValidationError('Quantity cannot be negative!')
```

### 4.4 @api.model

```python
@api.model
def create(self, vals):
    # Custom logic before create
    if 'name' not in vals:
        vals['name'] = self.env['ir.sequence'].next_by_code('my.model')
    return super().create(vals)
```

### 4.5 @api.model_create_multi

```python
@api.model_create_multi
def create(self, vals_list):
    # Batch create optimization
    for vals in vals_list:
        if 'code' not in vals:
            vals['code'] = self.env['ir.sequence'].next_by_code('my.model')
    return super().create(vals_list)
```

### 4.6 @api.private (Odoo 19)

```python
@api.private
def _internal_computation(self):
    """Not callable via XML-RPC/JSON-RPC"""
    return self.amount_total * self.tax_rate
```

---

## 5. Constraints

### 5.1 Declarative Constraints (Odoo 19 — preferred)

```python
from odoo import models

class MyModel(models.Model):
    _name = 'my.model'
    
    # UniqueIndex: enforces unique constraint
    name_unique = models.UniqueIndex('name')
    code_company_unique = models.UniqueIndex('code', 'company_id')
    
    # Constraint: enforces CHECK constraint
    quantity_positive = models.Constraint(
        'CHECK(quantity >= 0)',
        'Quantity must be positive!',
    )
    
    # Index: creates database index (no constraint)
    partner_date_idx = models.Index('partner_id', 'date')
```

### 5.2 Legacy SQL Constraints

```python
# ⚠️ Still works but prefer declarative API above
_sql_constraints = [
    ('name_unique', 'unique(name)', 'Name must be unique!'),
    ('quantity_positive', 'CHECK(quantity >= 0)', 'Quantity must be positive!'),
]

### 5.3 Python Constraints

```python
from odoo.exceptions import ValidationError

@api.constrains('start_date', 'end_date')
def _check_dates(self):
    for record in self:
        if record.start_date and record.end_date:
            if record.start_date > record.end_date:
                raise ValidationError('Start date must be before end date!')
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
    company_id = fields.Many2one('res.company', default=lambda self: self._default_company())
    
    def _default_company(self):
        return self.env.company
```

---

## 7. Override Methods

### 7.1 Create

```python
@api.model_create_multi
def create(self, vals_list):
    for vals in vals_list:
        # Generate sequence
        if 'name' not in vals or vals['name'] == '/':
            vals['name'] = self.env['ir.sequence'].next_by_code('my.model') or '/'
    
    records = super().create(vals_list)
    
    # Post-create logic
    records._send_notification()
    
    return records
```

### 7.2 Write

```python
def write(self, vals):
    # Pre-write logic
    if 'state' in vals and vals['state'] == 'done':
        vals['done_date'] = fields.Datetime.now()
    
    result = super().write(vals)
    
    # Post-write logic
    if 'partner_id' in vals:
        self._update_related_records()
    
    return result
```

### 7.3 Unlink

```python
def unlink(self):
    # Check before delete
    if any(record.state == 'done' for record in self):
        raise ValidationError('Cannot delete confirmed records!')
    
    return super().unlink()
```

### 7.4 Custom Display Name (replaces name_get)

```python
# Odoo 19: Use _compute_display_name instead of name_get()
@api.depends('code', 'name')
def _compute_display_name(self):
    for record in self:
        record.display_name = f"[{record.code}] {record.name}"

# ⚠️ DEPRECATED: name_get() — do NOT use in Odoo 19
# def name_get(self):
#     result = []
#     for record in self:
#         name = f"[{record.code}] {record.name}"
#         result.append((record.id, name))
#     return result
```

---

## 8. Context & Environment

```python
# Access environment
self.env  # Current environment
self.env.user  # Current user
self.env.company  # Current company
self.env.companies  # All user companies
self.env.lang  # Current language
self.env.context  # Context dictionary
self.env.cr  # Database cursor
self.env.uid  # Current user ID

# ⚠️ DEPRECATED: Do NOT use (Odoo 19)
# self._cr      → use self.env.cr
# self._uid     → use self.env.uid
# self._context → use self.env.context

# Modify context
records = self.with_context(lang='en_US', tz='UTC').search([])

# Sudo (bypass access rights)
partner = self.env['res.partner'].sudo().browse(partner_id)

# Change company
records = self.with_company(company_id).search([])
```

---

## 9. Best Practices

### DO ✅
```python
# Use ORM
partners = self.env['res.partner'].search([('is_company', '=', True)])

# Batch operations
partners.write({'active': False})

# Use mapped
emails = partners.mapped('email')

# Use filtered
active_partners = partners.filtered(lambda p: p.active)

# Proper depends
@api.depends('line_ids.price_subtotal')
def _compute_total(self):
    pass
```

### DON'T ❌
```python
# Don't use raw SQL
self.env.cr.execute("SELECT * FROM res_partner")

# Don't loop for updates
for partner in partners:
    partner.write({'active': False})  # Use batch instead

# Don't forget depends
def _compute_total(self):  # Missing @api.depends!
    pass

# Don't use browse in loops
for partner_id in partner_ids:
    partner = self.env['res.partner'].browse(partner_id)  # Use browse(partner_ids)
```

---

## 10. Common Patterns

### 10.1 Sequence Generation

```python
@api.model_create_multi
def create(self, vals_list):
    for vals in vals_list:
        if vals.get('name', '/') == '/':
            vals['name'] = self.env['ir.sequence'].next_by_code('my.model')
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
```

### 10.3 Multi-company

```python
company_id = fields.Many2one(
    'res.company',
    default=lambda self: self.env.company,
    required=True
)

# Record rule will be added in security
```

---

**Status**: ✅ Complete  
**Last Updated**: 23/02/2026

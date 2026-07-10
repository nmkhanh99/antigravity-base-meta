---
name: odoo-multicompany
description: Hướng dẫn cấu hình Multi-Company trong Odoo 19 - company-dependent fields, record rules, inter-company transactions. Use when the user asks about multi-company setup, company rules, or cross-company data in Odoo 19.
---

# Odoo 19 Multi-Company

## Goal
Giúp agent cấu hình và quản lý multi-company trong Odoo 19: company-dependent fields, record rules, data isolation.

## When to use this skill
- "multi-company", "đa công ty"
- "company rule", "company_id"
- "inter-company", "cross-company"
- "company-dependent field"

## Instructions

### 1. Company Field Pattern
```python
class MyModel(models.Model):
    _name = 'my.model'

    company_id = fields.Many2one(
        'res.company',
        string='Company',
        default=lambda self: self.env.company,
        required=True,
        index=True,
    )
```

### 2. Multi-Company Record Rule
```xml
<record id="rule_multi_company" model="ir.rule">
    <field name="name">My Model: Multi-Company</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="domain_force">
        ['|', ('company_id', '=', False), ('company_id', 'in', company_ids)]
    </field>
    <field name="global" eval="True"/>
</record>
```

### 3. Company-Dependent Fields (Properties)
```python
# Value varies per company (stored in ir.property)
discount_rate = fields.Float(company_dependent=True)
default_warehouse = fields.Many2one('stock.warehouse', company_dependent=True)
```

### 4. Switching Company Context
```python
# Force specific company
record.with_company(company_id).action_confirm()

# Access current company
current = self.env.company
all_companies = self.env.companies  # All user's companies
```

### 5. Inter-Company Patterns
```python
def _create_intercompany_order(self):
    """Create mirror order in target company"""
    target_company = self.env['res.company'].browse(target_id)
    self.with_company(target_company).sudo().create({
        'name': f'IC/{self.name}',
        'company_id': target_company.id,
    })
```

## Constraints
- Models với `company_id` PHẢI có multi-company record rule.
- KHÔNG trộn data giữa các companies trong cùng recordset.
- Dùng `with_company()` khi cần thao tác cross-company.

## Best practices
- Luôn thêm `index=True` cho `company_id` field.
- Record rule dùng `company_ids` (plural) để support multi-company users.
- `company_dependent=True` cho fields có giá trị khác nhau per company.

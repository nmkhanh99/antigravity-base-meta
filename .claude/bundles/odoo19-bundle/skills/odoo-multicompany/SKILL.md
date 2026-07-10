---
name: odoo-multicompany
description: Hướng dẫn cấu hình Multi-Company trong Odoo 19 - company-dependent fields, record rules, inter-company transactions, Strategy Pattern. Use when the user asks about multi-company setup, company rules, cross-company data, or company-specific business logic in Odoo 19.
---

# Odoo 19 Multi-Company

## Goal
Giúp agent cấu hình và quản lý multi-company trong Odoo 19: company-dependent fields, record rules, data isolation, và Strategy Pattern cho business logic khác nhau per company.

## When to use this skill
- "multi-company", "đa công ty"
- "company rule", "company_id"
- "inter-company", "cross-company"
- "company-dependent field"
- "strategy pattern odoo"
- "logic khác nhau theo công ty"
- "cấu hình per company"

## Instructions

### Bước 1 — Thêm company_id field
Mọi model cần data isolation phải có `company_id`:
```python
company_id = fields.Many2one(
    'res.company',
    string='Company',
    default=lambda self: self.env.company,
    required=True,
    index=True,
)
```
Luôn thêm `index=True` để tối ưu query performance.

### Bước 2 — Tạo Multi-Company Record Rule
Sau khi thêm `company_id`, bắt buộc tạo record rule trong file security XML:
```xml
<record id="rule_my_model_multi_company" model="ir.rule">
    <field name="name">My Model: Multi-Company</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="domain_force">
        ['|', ('company_id', '=', False), ('company_id', 'in', company_ids)]
    </field>
    <field name="global" eval="True"/>
</record>
```
Dùng `company_ids` (plural) để support users thuộc nhiều companies.

### Bước 3 — Company-Dependent Fields
Khi giá trị field cần khác nhau per company (stored trong `ir.property`):
```python
discount_rate = fields.Float(company_dependent=True)
default_warehouse = fields.Many2one('stock.warehouse', company_dependent=True)
```

### Bước 4 — Cross-Company Operations
```python
# Force specific company context
record.with_company(company_id).action_confirm()

# Access company info
current = self.env.company        # Current company
all_user_companies = self.env.companies  # All companies user belongs to
```

### Bước 5 — Strategy Pattern (khi logic khác nhau per company)
Dùng khi mỗi company có business logic riêng (pricing, tax, workflow).
Xem `references/GUIDE.md` để có implementation đầy đủ.

Pattern tổng quát:
1. Định nghĩa Strategy Interface (AbstractModel)
2. Implement Concrete Strategies (mỗi strategy là một Model)
3. Thêm strategy selection vào `res.company`
4. Dùng factory method để lấy strategy instance

### Bước 6 — Inter-Company Transactions
```python
def _create_intercompany_record(self):
    target_company = self.env['res.company'].browse(target_id)
    self.with_company(target_company).sudo().create({
        'name': f'IC/{self.name}',
        'company_id': target_company.id,
    })
```
Dùng `sudo()` khi tạo record ở company khác để bypass record rules.

## Constraints
- Models với `company_id` PHẢI có multi-company record rule — không có rule = data leak giữa companies.
- KHÔNG trộn records của các companies khác nhau trong cùng một recordset.
- Dùng `with_company()` khi cần thao tác cross-company, KHÔNG thay đổi `env.company` trực tiếp.
- Strategy Pattern: KHÔNG hardcode company name để chọn strategy — dùng factory method qua config field.
- `company_dependent=True` chỉ dùng cho fields cần giá trị riêng per company, KHÔNG dùng cho tất cả fields.
- Trong Odoo 19, tránh `name_get()` deprecated — dùng `_rec_name` hoặc `display_name` compute field.

## References
- [Odoo 19 Multi-Company Docs](https://www.odoo.com/documentation/19.0/applications/general/companies.html)
- [Strategy Pattern Guide](https://refactoring.guru/design-patterns/strategy)
- [Odoo ORM — with_company()](https://www.odoo.com/documentation/19.0/developer/reference/backend/orm.html)
- Chi tiết implementation: `references/GUIDE.md`

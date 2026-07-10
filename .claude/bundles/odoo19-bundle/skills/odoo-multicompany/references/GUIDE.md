---
name: Multi-Company Strategy Pattern
description: Hướng dẫn toàn diện về Multi-Company customization sử dụng Strategy Pattern trong Odoo 19
---

# Multi-Company Strategy Pattern — Odoo 19

## Mục Lục

1. [Architecture Overview](#architecture-overview)
2. [Company Field & Record Rule](#company-field--record-rule)
3. [Strategy Pattern Implementation](#strategy-pattern-implementation)
4. [Pricing Strategy](#pricing-strategy)
5. [Tax Strategy](#tax-strategy)
6. [Workflow Strategy](#workflow-strategy)
7. [Integration Examples](#integration-examples)
8. [Testing](#testing)
9. [Checklist](#checklist)

---

## Architecture Overview

```
┌─────────────────────────────────────────────┐
│   Company A (Vietnam)                       │
│   - Pricing Strategy: Discount-based        │
│   - Tax: Vietnam VAT (10%)                  │
│   - Workflow: 3-step approval               │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│   Company B (Singapore)                     │
│   - Pricing Strategy: Premium pricing       │
│   - Tax: Singapore GST (9%)                 │
│   - Workflow: 2-step approval               │
└─────────────────────────────────────────────┘

           ▼ Strategy Pattern
┌─────────────────────────────────────────────┐
│   ResCompany.get_current_strategy(type)     │
│   → factory method → concrete strategy      │
└─────────────────────────────────────────────┘
```

---

## Company Field & Record Rule

### Model với company_id

```python
# models/my_model.py
from odoo import models, fields

class MyModel(models.Model):
    _name = 'my.model'
    _description = 'My Model'

    name = fields.Char(required=True)
    company_id = fields.Many2one(
        'res.company',
        string='Company',
        default=lambda self: self.env.company,
        required=True,
        index=True,
    )
    # Company-dependent field: giá trị khác nhau per company
    discount_rate = fields.Float(company_dependent=True)
```

### Security XML

```xml
<!-- security/security.xml -->
<odoo>
    <record id="rule_my_model_multi_company" model="ir.rule">
        <field name="name">My Model: Multi-Company</field>
        <field name="model_id" ref="model_my_model"/>
        <field name="domain_force">
            ['|', ('company_id', '=', False), ('company_id', 'in', company_ids)]
        </field>
        <field name="global" eval="True"/>
    </record>
</odoo>
```

### ACL (ir.model.access.csv)

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_my_model_user,my.model user,model_my_model,base.group_user,1,1,1,0
access_my_model_manager,my.model manager,model_my_model,base.group_system,1,1,1,1
```

---

## Strategy Pattern Implementation

### Module Structure

```
multi_company_strategy/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   ├── res_company.py          # Strategy selection fields
│   ├── pricing_strategy.py     # Pricing strategies
│   ├── tax_strategy.py         # Tax strategies
│   └── workflow_strategy.py    # Workflow strategies
├── views/
│   └── res_company_views.xml
├── security/
│   ├── security.xml
│   └── ir.model.access.csv
└── tests/
    └── test_pricing_strategy.py
```

### __manifest__.py

```python
{
    'name': 'Multi-Company Strategy',
    'version': '19.0.1.0.0',
    'category': 'Technical',
    'depends': ['base', 'account', 'sale'],
    'data': [
        'security/security.xml',
        'security/ir.model.access.csv',
        'views/res_company_views.xml',
    ],
    'installable': True,
    'auto_install': False,
}
```

### Company Extension

```python
# models/res_company.py
from odoo import models, fields, api

class ResCompany(models.Model):
    _inherit = 'res.company'

    pricing_strategy_id = fields.Many2one(
        'pricing.strategy.config',
        string='Pricing Strategy',
    )
    tax_strategy_id = fields.Many2one(
        'tax.strategy.config',
        string='Tax Strategy',
    )
    workflow_strategy_id = fields.Many2one(
        'workflow.strategy.config',
        string='Workflow Strategy',
    )
    discount_percentage = fields.Float(string='Default Discount %')
    approval_levels = fields.Integer(string='Approval Levels', default=1)
    volume_tiers = fields.One2many(
        'volume.tier', 'company_id', string='Volume Tiers'
    )

    @api.model
    def get_current_strategy(self, strategy_type):
        """Factory method: lấy strategy instance theo type."""
        company = self.env.company
        field_map = {
            'pricing': 'pricing_strategy_id',
            'tax': 'tax_strategy_id',
            'workflow': 'workflow_strategy_id',
        }
        config = company[field_map[strategy_type]]
        if not config:
            raise ValueError(
                f'No {strategy_type} strategy configured for {company.name}'
            )
        return config.get_strategy_instance()
```

---

## Pricing Strategy

```python
# models/pricing_strategy.py
from odoo import models, fields, api


class PricingStrategyConfig(models.Model):
    _name = 'pricing.strategy.config'
    _description = 'Pricing Strategy Configuration'

    name = fields.Char(required=True)
    strategy_type = fields.Selection([
        ('discount', 'Discount-Based'),
        ('premium', 'Premium Pricing'),
        ('volume', 'Volume-Based'),
    ], required=True)
    active = fields.Boolean(default=True)
    base_discount = fields.Float(string='Base Discount %')
    premium_multiplier = fields.Float(string='Premium Multiplier', default=1.0)

    def get_strategy_instance(self):
        strategy_map = {
            'discount': 'pricing.strategy.discount',
            'premium': 'pricing.strategy.premium',
            'volume': 'pricing.strategy.volume',
        }
        model_name = strategy_map.get(self.strategy_type)
        if not model_name:
            raise ValueError(f'Unknown strategy type: {self.strategy_type}')
        return self.env[model_name].with_context(strategy_config=self)


class PricingStrategyBase(models.AbstractModel):
    _name = 'pricing.strategy.base'
    _description = 'Pricing Strategy Base'

    @api.model
    def calculate_price(self, product, quantity, partner, **kwargs):
        """
        Returns:
            dict: {unit_price, total_price, discount, applied_rules}
        """
        raise NotImplementedError('Subclasses must implement calculate_price()')

    @api.model
    def _get_base_price(self, product):
        return product.list_price


class DiscountPricingStrategy(models.Model):
    _name = 'pricing.strategy.discount'
    _inherit = 'pricing.strategy.base'
    _description = 'Discount Pricing Strategy'

    @api.model
    def calculate_price(self, product, quantity, partner, **kwargs):
        base_price = self._get_base_price(product)
        company = self.env.company
        discount_pct = company.discount_percentage or 0.0

        if hasattr(partner, 'discount_percentage') and partner.discount_percentage:
            discount_pct = max(discount_pct, partner.discount_percentage)

        unit_price = base_price * (1 - discount_pct / 100.0)
        return {
            'unit_price': unit_price,
            'total_price': unit_price * quantity,
            'discount': discount_pct,
            'applied_rules': [f'Company discount: {discount_pct}%'],
        }


class PremiumPricingStrategy(models.Model):
    _name = 'pricing.strategy.premium'
    _inherit = 'pricing.strategy.base'
    _description = 'Premium Pricing Strategy'

    @api.model
    def calculate_price(self, product, quantity, partner, **kwargs):
        base_price = self._get_base_price(product)
        config = self.env.context.get('strategy_config')
        multiplier = config.premium_multiplier if config else 1.2
        unit_price = base_price * multiplier
        return {
            'unit_price': unit_price,
            'total_price': unit_price * quantity,
            'discount': 0.0,
            'applied_rules': [f'Premium multiplier: {multiplier}'],
        }


class VolumePricingStrategy(models.Model):
    _name = 'pricing.strategy.volume'
    _inherit = 'pricing.strategy.base'
    _description = 'Volume Pricing Strategy'

    @api.model
    def calculate_price(self, product, quantity, partner, **kwargs):
        base_price = self._get_base_price(product)
        company = self.env.company
        tier = self.env['volume.tier'].search([
            ('company_id', '=', company.id),
            ('min_quantity', '<=', quantity),
            ('max_quantity', '>=', quantity),
        ], limit=1)
        discount_pct = tier.discount_percentage if tier else 0.0
        unit_price = base_price * (1 - discount_pct / 100.0)
        return {
            'unit_price': unit_price,
            'total_price': unit_price * quantity,
            'discount': discount_pct,
            'applied_rules': [
                f'Volume tier: {tier.name}' if tier else 'No tier applied',
                f'Discount: {discount_pct}%',
            ],
        }


class VolumeTier(models.Model):
    _name = 'volume.tier'
    _description = 'Volume Pricing Tier'
    _order = 'min_quantity'

    company_id = fields.Many2one('res.company', required=True, ondelete='cascade')
    name = fields.Char(required=True)
    min_quantity = fields.Float(required=True)
    max_quantity = fields.Float(required=True)
    discount_percentage = fields.Float(required=True)

    _sql_constraints = [
        ('check_quantity', 'CHECK(max_quantity > min_quantity)',
         'Max quantity must be greater than min quantity'),
    ]
```

---

## Tax Strategy

```python
# models/tax_strategy.py
from odoo import models, fields, api


class TaxStrategyConfig(models.Model):
    _name = 'tax.strategy.config'
    _description = 'Tax Strategy Configuration'

    name = fields.Char(required=True)
    strategy_type = fields.Selection([
        ('vat_vn', 'Vietnam VAT'),
        ('gst_sg', 'Singapore GST'),
        ('vat_th', 'Thailand VAT'),
    ], required=True)

    def get_strategy_instance(self):
        strategy_map = {
            'vat_vn': 'tax.strategy.vat.vn',
            'gst_sg': 'tax.strategy.gst.sg',
            'vat_th': 'tax.strategy.vat.th',
        }
        return self.env[strategy_map[self.strategy_type]]


class TaxStrategyBase(models.AbstractModel):
    _name = 'tax.strategy.base'
    _description = 'Tax Strategy Base'

    @api.model
    def calculate_tax(self, amount, product, partner):
        """
        Returns:
            dict: {tax_amount, tax_rate, total_amount, tax_name}
        """
        raise NotImplementedError()


class VietnamVATStrategy(models.Model):
    _name = 'tax.strategy.vat.vn'
    _inherit = 'tax.strategy.base'
    _description = 'Vietnam VAT Strategy'

    @api.model
    def calculate_tax(self, amount, product, partner):
        vat_rate = 0.10  # Default 10%
        categ = product.categ_id
        if hasattr(categ, 'is_essential_goods') and categ.is_essential_goods:
            vat_rate = 0.05
        elif hasattr(categ, 'is_export') and categ.is_export:
            vat_rate = 0.0
        tax_amount = amount * vat_rate
        return {
            'tax_amount': tax_amount,
            'tax_rate': vat_rate * 100,
            'total_amount': amount + tax_amount,
            'tax_name': f'VAT {vat_rate * 100:.0f}%',
        }


class SingaporeGSTStrategy(models.Model):
    _name = 'tax.strategy.gst.sg'
    _inherit = 'tax.strategy.base'
    _description = 'Singapore GST Strategy'

    @api.model
    def calculate_tax(self, amount, product, partner):
        gst_rate = 0.09
        if (hasattr(product, 'is_gst_exempt') and product.is_gst_exempt) or \
           (hasattr(partner, 'is_gst_exempt') and partner.is_gst_exempt):
            gst_rate = 0.0
        tax_amount = amount * gst_rate
        return {
            'tax_amount': tax_amount,
            'tax_rate': gst_rate * 100,
            'total_amount': amount + tax_amount,
            'tax_name': f'GST {gst_rate * 100:.0f}%',
        }
```

---

## Workflow Strategy

```python
# models/workflow_strategy.py
from odoo import models, fields, api


class WorkflowStrategyConfig(models.Model):
    _name = 'workflow.strategy.config'
    _description = 'Workflow Strategy Configuration'

    name = fields.Char(required=True)
    strategy_type = fields.Selection([
        ('single', 'Single Approval'),
        ('double', 'Double Approval'),
        ('dynamic', 'Dynamic Approval'),
    ], required=True)

    def get_strategy_instance(self):
        strategy_map = {
            'single': 'workflow.strategy.single',
            'double': 'workflow.strategy.double',
            'dynamic': 'workflow.strategy.dynamic',
        }
        return self.env[strategy_map[self.strategy_type]]


class WorkflowStrategyBase(models.AbstractModel):
    _name = 'workflow.strategy.base'
    _description = 'Workflow Strategy Base'

    @api.model
    def get_approvers(self, record, current_level=0):
        raise NotImplementedError()

    @api.model
    def validate_approval(self, record):
        raise NotImplementedError()


class SingleApprovalStrategy(models.Model):
    _name = 'workflow.strategy.single'
    _inherit = 'workflow.strategy.base'
    _description = 'Single Approval Strategy'

    @api.model
    def get_approvers(self, record, current_level=0):
        if current_level > 0:
            return self.env['res.users']
        manager = getattr(record.user_id, 'parent_id', False)
        return manager.user_id if manager else self.env['res.users']

    @api.model
    def validate_approval(self, record):
        return record.approval_level >= 1


class DoubleApprovalStrategy(models.Model):
    _name = 'workflow.strategy.double'
    _inherit = 'workflow.strategy.base'
    _description = 'Double Approval Strategy'

    @api.model
    def get_approvers(self, record, current_level=0):
        if current_level == 0:
            manager = getattr(record.user_id, 'parent_id', False)
            return manager.user_id if manager else self.env['res.users']
        elif current_level == 1:
            dept = getattr(record.user_id, 'department_id', False)
            return dept.manager_id.user_id if dept else self.env['res.users']
        return self.env['res.users']

    @api.model
    def validate_approval(self, record):
        return record.approval_level >= 2


class DynamicApprovalStrategy(models.Model):
    _name = 'workflow.strategy.dynamic'
    _inherit = 'workflow.strategy.base'
    _description = 'Dynamic Approval Strategy'

    THRESHOLDS = [
        (1000, 1),    # amount < 1000: 1 approval
        (10000, 2),   # amount < 10000: 2 approvals
        (float('inf'), 3),  # amount >= 10000: 3 approvals
    ]

    @api.model
    def _required_levels(self, record):
        amount = getattr(record, 'amount_total', 0)
        for threshold, levels in self.THRESHOLDS:
            if amount < threshold:
                return levels
        return 3

    @api.model
    def get_approvers(self, record, current_level=0):
        required = self._required_levels(record)
        if current_level >= required:
            return self.env['res.users']
        if current_level == 0:
            manager = getattr(record.user_id, 'parent_id', False)
            return manager.user_id if manager else self.env['res.users']
        elif current_level == 1:
            dept = getattr(record.user_id, 'department_id', False)
            return dept.manager_id.user_id if dept else self.env['res.users']
        elif current_level == 2:
            return self.env.ref('account.group_account_manager').users[:1]
        return self.env['res.users']

    @api.model
    def validate_approval(self, record):
        return record.approval_level >= self._required_levels(record)
```

---

## Integration Examples

### Sale Order với Multi-Company Strategies

```python
# models/sale_order.py
from odoo import models, fields, api
from odoo.exceptions import UserError


class SaleOrder(models.Model):
    _inherit = 'sale.order'

    approval_level = fields.Integer(default=0)
    approver_ids = fields.Many2many('res.users', string='Approvers')

    @api.onchange('order_line', 'partner_id')
    def _onchange_apply_pricing_strategy(self):
        if not self.order_line:
            return
        strategy = self.env.company.get_current_strategy('pricing')
        for line in self.order_line:
            result = strategy.calculate_price(
                product=line.product_id,
                quantity=line.product_uom_qty,
                partner=self.partner_id,
            )
            line.price_unit = result['unit_price']
            line.discount = result['discount']

    def action_confirm(self):
        workflow_strategy = self.env.company.get_current_strategy('workflow')
        if not workflow_strategy.validate_approval(self):
            approvers = workflow_strategy.get_approvers(self, self.approval_level)
            if approvers:
                self.approver_ids = [(6, 0, approvers.ids)]
                self.message_post(
                    body=f'Approval required from: {", ".join(approvers.mapped("name"))}',
                    partner_ids=approvers.partner_id.ids,
                )
                return False
        return super().action_confirm()

    def action_approve(self):
        self.ensure_one()
        if self.env.user not in self.approver_ids:
            raise UserError('You are not authorized to approve this order.')
        self.approval_level += 1
        return self.action_confirm()
```

### Inter-Company Record Creation

```python
def _create_intercompany_record(self):
    """Tạo mirror record ở company khác."""
    target_company = self.env['res.company'].browse(self.target_company_id.id)
    # sudo() cần thiết để bypass record rules của target company
    self.with_company(target_company).sudo().create({
        'name': f'IC/{self.name}',
        'company_id': target_company.id,
        'origin': self.name,
    })
```

---

## Testing

```python
# tests/test_pricing_strategy.py
from odoo.tests import TransactionCase


class TestPricingStrategy(TransactionCase):

    def setUp(self):
        super().setUp()
        self.company = self.env['res.company'].create({
            'name': 'Test Company VN',
            'discount_percentage': 15.0,
        })
        self.strategy_config = self.env['pricing.strategy.config'].create({
            'name': 'Discount Strategy Test',
            'strategy_type': 'discount',
        })
        self.company.pricing_strategy_id = self.strategy_config

    def test_discount_calculation(self):
        product = self.env['product.product'].create({
            'name': 'Test Product',
            'list_price': 100.0,
        })
        partner = self.env['res.partner'].create({'name': 'Test Partner'})

        strategy = self.company.with_company(self.company).get_current_strategy('pricing')
        result = strategy.calculate_price(product, 10, partner)

        self.assertAlmostEqual(result['discount'], 15.0)
        self.assertAlmostEqual(result['unit_price'], 85.0)
        self.assertAlmostEqual(result['total_price'], 850.0)

    def test_record_rule_isolation(self):
        """Records của company khác không được hiển thị."""
        other_company = self.env['res.company'].create({'name': 'Other Company'})
        record_other = self.env['my.model'].with_company(other_company).sudo().create({
            'name': 'Other Company Record',
            'company_id': other_company.id,
        })
        records_visible = self.env['my.model'].with_company(self.company).search([])
        self.assertNotIn(record_other, records_visible)
```

---

## Checklist

- [ ] `company_id` field với `index=True` trên mọi model cần isolation
- [ ] Multi-company record rule trong security XML (dùng `company_ids`)
- [ ] ACL đầy đủ trong `ir.model.access.csv`
- [ ] Strategy interfaces (AbstractModel) đã định nghĩa
- [ ] Concrete strategies implemented và đăng ký trong factory
- [ ] `res.company` có strategy selection fields
- [ ] Factory method `get_current_strategy()` hoạt động
- [ ] Inter-company operations dùng `with_company().sudo()`
- [ ] Tests: unit test từng strategy + record rule isolation
- [ ] `company_dependent=True` chỉ dùng cho fields thực sự cần per-company value

---

## Anti-Patterns (Không làm)

| Anti-Pattern | Thay bằng |
|---|---|
| `if company.name == 'VN': ...` | Factory method + config field |
| Hardcode discount = 10.0 | `company.discount_percentage` |
| `env['res.company'].search([])` để lấy tất cả companies | `self.env.companies` |
| Quên `sudo()` khi tạo record ở company khác | `with_company(c).sudo().create(...)` |
| Dùng `company_dependent=True` cho mọi field | Chỉ dùng khi thực sự cần giá trị khác per company |

---
name: Multi-Company Strategy Pattern
description: Hướng dẫn toàn diện về Multi-Company customization sử dụng Strategy Pattern trong Odoo 19
---

# Multi-Company Strategy Pattern Skill

## 📋 Mục Lục

1. [Multi-Company Overview](#multi-company-overview)
2. [Strategy Pattern in Odoo](#strategy-pattern-in-odoo)
3. [Implementation Guide](#implementation-guide)
4. [Real-World Examples](#real-world-examples)
5. [Best Practices](#best-practices)

---

## Multi-Company Overview

### Multi-Company Architecture

```
┌─────────────────────────────────────────────┐
│   Company A (Vietnam)                       │
│   - Pricing Strategy: Discount-based       │
│   - Tax Calculation: Vietnam VAT           │
│   - Workflow: 3-step approval               │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│   Company B (Singapore)                     │
│   - Pricing Strategy: Premium pricing      │
│   - Tax Calculation: Singapore GST         │
│   - Workflow: 2-step approval               │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│   Company C (Thailand)                      │
│   - Pricing Strategy: Volume-based         │
│   - Tax Calculation: Thailand VAT          │
│   - Workflow: Single approval               │
└─────────────────────────────────────────────┘

           ▼
┌─────────────────────────────────────────────┐
│   Strategy Pattern                          │
│   - Select strategy based on company       │
│   - Execute company-specific logic         │
└─────────────────────────────────────────────┘
```

---

## Strategy Pattern in Odoo

### Pattern Structure

```python
# Strategy Interface (Abstract Base)
class PricingStrategy(models.AbstractModel):
    _name = 'pricing.strategy'
    _description = 'Pricing Strategy Interface'
    
    def calculate_price(self, product, quantity, partner):
        """Calculate price based on strategy"""
        raise NotImplementedError()

# Concrete Strategies
class DiscountPricingStrategy(models.Model):
    _name = 'pricing.strategy.discount'
    _inherit = 'pricing.strategy'
    
    def calculate_price(self, product, quantity, partner):
        # Discount-based logic
        pass

class PremiumPricingStrategy(models.Model):
    _name = 'pricing.strategy.premium'
    _inherit = 'pricing.strategy'
    
    def calculate_price(self, product, quantity, partner):
        # Premium pricing logic
        pass

# Context (Company)
class ResCompany(models.Model):
    _inherit = 'res.company'
    
    pricing_strategy = fields.Selection([
        ('discount', 'Discount Strategy'),
        ('premium', 'Premium Strategy'),
        ('volume', 'Volume Strategy'),
    ])
    
    def get_pricing_strategy(self):
        """Get pricing strategy instance"""
        strategy_map = {
            'discount': 'pricing.strategy.discount',
            'premium': 'pricing.strategy.premium',
            'volume': 'pricing.strategy.volume',
        }
        return self.env[strategy_map[self.pricing_strategy]]
```

---

## Implementation Guide

### 1. Base Strategy Module

#### Module Structure

```
multi_company_strategy/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   ├── res_company.py
│   ├── strategy_base.py
│   ├── pricing_strategy.py
│   ├── tax_strategy.py
│   └── workflow_strategy.py
├── views/
│   ├── res_company_views.xml
│   └── strategy_config_views.xml
├── security/
│   └── ir.model.access.csv
└── data/
    └── default_strategies.xml
```

---

### 2. Company Extension

```python
# models/res_company.py
from odoo import models, fields, api

class ResCompany(models.Model):
    _inherit = 'res.company'
    
    # Strategy Selections
    pricing_strategy_id = fields.Many2one(
        'pricing.strategy.config',
        string='Pricing Strategy',
        help='Select pricing strategy for this company'
    )
    
    tax_strategy_id = fields.Many2one(
        'tax.strategy.config',
        string='Tax Strategy',
        help='Select tax calculation strategy'
    )
    
    workflow_strategy_id = fields.Many2one(
        'workflow.strategy.config',
        string='Workflow Strategy',
        help='Select approval workflow strategy'
    )
    
    # Strategy Parameters
    discount_percentage = fields.Float(
        string='Default Discount %',
        help='Used by discount pricing strategy'
    )
    
    volume_tiers = fields.One2many(
        'volume.tier',
        'company_id',
        string='Volume Tiers',
        help='Used by volume pricing strategy'
    )
    
    approval_levels = fields.Integer(
        string='Approval Levels',
        default=1,
        help='Number of approval levels required'
    )
    
    @api.model
    def get_current_strategy(self, strategy_type):
        """Get current company's strategy instance"""
        company = self.env.company
        
        strategy_field_map = {
            'pricing': 'pricing_strategy_id',
            'tax': 'tax_strategy_id',
            'workflow': 'workflow_strategy_id',
        }
        
        strategy_config = company[strategy_field_map[strategy_type]]
        
        if not strategy_config:
            raise ValueError(f'No {strategy_type} strategy configured for {company.name}')
        
        return strategy_config.get_strategy_instance()
```

---

### 3. Pricing Strategy Implementation

```python
# models/pricing_strategy.py
from odoo import models, fields, api
from abc import ABC, abstractmethod

class PricingStrategyConfig(models.Model):
    """Configuration for pricing strategies"""
    _name = 'pricing.strategy.config'
    _description = 'Pricing Strategy Configuration'
    
    name = fields.Char(string='Strategy Name', required=True)
    strategy_type = fields.Selection([
        ('discount', 'Discount-Based'),
        ('premium', 'Premium Pricing'),
        ('volume', 'Volume-Based'),
        ('dynamic', 'Dynamic Pricing'),
        ('custom', 'Custom Strategy'),
    ], string='Strategy Type', required=True)
    
    active = fields.Boolean(default=True)
    
    # Strategy-specific parameters
    base_discount = fields.Float(string='Base Discount %')
    premium_multiplier = fields.Float(string='Premium Multiplier', default=1.0)
    
    def get_strategy_instance(self):
        """Factory method to get strategy instance"""
        strategy_map = {
            'discount': self.env['pricing.strategy.discount'],
            'premium': self.env['pricing.strategy.premium'],
            'volume': self.env['pricing.strategy.volume'],
            'dynamic': self.env['pricing.strategy.dynamic'],
        }
        
        strategy = strategy_map.get(self.strategy_type)
        if not strategy:
            raise ValueError(f'Unknown strategy type: {self.strategy_type}')
        
        return strategy.with_context(strategy_config=self)


class PricingStrategyBase(models.AbstractModel):
    """Base class for all pricing strategies"""
    _name = 'pricing.strategy.base'
    _description = 'Pricing Strategy Base'
    
    @api.model
    def calculate_price(self, product, quantity, partner, **kwargs):
        """
        Calculate price based on strategy
        
        Args:
            product: product.product record
            quantity: float
            partner: res.partner record
            **kwargs: additional parameters
            
        Returns:
            dict: {
                'unit_price': float,
                'total_price': float,
                'discount': float,
                'applied_rules': list,
            }
        """
        raise NotImplementedError('Subclasses must implement calculate_price()')
    
    @api.model
    def _get_base_price(self, product):
        """Get base price from product"""
        return product.list_price
    
    @api.model
    def _apply_pricelist(self, product, quantity, partner):
        """Apply pricelist if exists"""
        pricelist = partner.property_product_pricelist
        if pricelist:
            return pricelist._get_product_price(product, quantity, partner)
        return product.list_price


class DiscountPricingStrategy(models.Model):
    """Discount-based pricing strategy"""
    _name = 'pricing.strategy.discount'
    _inherit = 'pricing.strategy.base'
    _description = 'Discount Pricing Strategy'
    
    @api.model
    def calculate_price(self, product, quantity, partner, **kwargs):
        """Calculate price with discount"""
        base_price = self._get_base_price(product)
        company = self.env.company
        
        # Get discount percentage from company config
        discount_pct = company.discount_percentage or 0.0
        
        # Apply partner-specific discount if exists
        if partner.discount_percentage:
            discount_pct = max(discount_pct, partner.discount_percentage)
        
        # Calculate discounted price
        discount_amount = base_price * (discount_pct / 100.0)
        unit_price = base_price - discount_amount
        total_price = unit_price * quantity
        
        return {
            'unit_price': unit_price,
            'total_price': total_price,
            'discount': discount_pct,
            'applied_rules': [f'Company discount: {discount_pct}%'],
        }


class PremiumPricingStrategy(models.Model):
    """Premium pricing strategy"""
    _name = 'pricing.strategy.premium'
    _inherit = 'pricing.strategy.base'
    _description = 'Premium Pricing Strategy'
    
    @api.model
    def calculate_price(self, product, quantity, partner, **kwargs):
        """Calculate premium price"""
        base_price = self._get_base_price(product)
        
        # Get strategy config
        config = self.env.context.get('strategy_config')
        multiplier = config.premium_multiplier if config else 1.2
        
        # Apply premium multiplier
        unit_price = base_price * multiplier
        
        # Additional premium for VIP customers
        if partner.is_vip:
            unit_price *= 0.95  # 5% VIP discount on premium
        
        total_price = unit_price * quantity
        
        return {
            'unit_price': unit_price,
            'total_price': total_price,
            'discount': 0.0,
            'applied_rules': [f'Premium multiplier: {multiplier}'],
        }


class VolumePricingStrategy(models.Model):
    """Volume-based pricing strategy"""
    _name = 'pricing.strategy.volume'
    _inherit = 'pricing.strategy.base'
    _description = 'Volume Pricing Strategy'
    
    @api.model
    def calculate_price(self, product, quantity, partner, **kwargs):
        """Calculate price based on volume tiers"""
        base_price = self._get_base_price(product)
        company = self.env.company
        
        # Get applicable volume tier
        tier = self.env['volume.tier'].search([
            ('company_id', '=', company.id),
            ('min_quantity', '<=', quantity),
            ('max_quantity', '>=', quantity),
        ], limit=1)
        
        discount_pct = tier.discount_percentage if tier else 0.0
        
        # Calculate price
        discount_amount = base_price * (discount_pct / 100.0)
        unit_price = base_price - discount_amount
        total_price = unit_price * quantity
        
        return {
            'unit_price': unit_price,
            'total_price': total_price,
            'discount': discount_pct,
            'applied_rules': [
                f'Volume tier: {tier.name}' if tier else 'No tier applied',
                f'Discount: {discount_pct}%'
            ],
        }


class VolumeTier(models.Model):
    """Volume pricing tiers"""
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

### 4. Tax Strategy Implementation

```python
# models/tax_strategy.py
from odoo import models, fields, api

class TaxStrategyConfig(models.Model):
    """Configuration for tax strategies"""
    _name = 'tax.strategy.config'
    _description = 'Tax Strategy Configuration'
    
    name = fields.Char(required=True)
    strategy_type = fields.Selection([
        ('vat_vn', 'Vietnam VAT'),
        ('gst_sg', 'Singapore GST'),
        ('vat_th', 'Thailand VAT'),
        ('sales_tax_us', 'US Sales Tax'),
        ('custom', 'Custom Tax'),
    ], required=True)
    
    def get_strategy_instance(self):
        """Get tax strategy instance"""
        strategy_map = {
            'vat_vn': self.env['tax.strategy.vat.vn'],
            'gst_sg': self.env['tax.strategy.gst.sg'],
            'vat_th': self.env['tax.strategy.vat.th'],
            'sales_tax_us': self.env['tax.strategy.sales.us'],
        }
        return strategy_map.get(self.strategy_type)


class TaxStrategyBase(models.AbstractModel):
    """Base tax strategy"""
    _name = 'tax.strategy.base'
    _description = 'Tax Strategy Base'
    
    @api.model
    def calculate_tax(self, amount, product, partner):
        """Calculate tax amount"""
        raise NotImplementedError()


class VietnamVATStrategy(models.Model):
    """Vietnam VAT calculation"""
    _name = 'tax.strategy.vat.vn'
    _inherit = 'tax.strategy.base'
    _description = 'Vietnam VAT Strategy'
    
    @api.model
    def calculate_tax(self, amount, product, partner):
        """Calculate Vietnam VAT (10% or 5%)"""
        
        # Determine VAT rate based on product category
        vat_rate = 0.10  # Default 10%
        
        if product.categ_id.is_essential_goods:
            vat_rate = 0.05  # Essential goods: 5%
        elif product.categ_id.is_export:
            vat_rate = 0.0  # Export: 0%
        
        tax_amount = amount * vat_rate
        
        return {
            'tax_amount': tax_amount,
            'tax_rate': vat_rate * 100,
            'total_amount': amount + tax_amount,
            'tax_name': f'VAT {vat_rate * 100}%',
        }


class SingaporeGSTStrategy(models.Model):
    """Singapore GST calculation"""
    _name = 'tax.strategy.gst.sg'
    _inherit = 'tax.strategy.base'
    _description = 'Singapore GST Strategy'
    
    @api.model
    def calculate_tax(self, amount, product, partner):
        """Calculate Singapore GST (9%)"""
        
        gst_rate = 0.09  # Singapore GST: 9%
        
        # Check if GST-exempt
        if product.is_gst_exempt or partner.is_gst_exempt:
            gst_rate = 0.0
        
        tax_amount = amount * gst_rate
        
        return {
            'tax_amount': tax_amount,
            'tax_rate': gst_rate * 100,
            'total_amount': amount + tax_amount,
            'tax_name': f'GST {gst_rate * 100}%',
        }
```

---

### 5. Workflow Strategy Implementation

```python
# models/workflow_strategy.py
from odoo import models, fields, api

class WorkflowStrategyConfig(models.Model):
    """Workflow strategy configuration"""
    _name = 'workflow.strategy.config'
    _description = 'Workflow Strategy Configuration'
    
    name = fields.Char(required=True)
    strategy_type = fields.Selection([
        ('single', 'Single Approval'),
        ('double', 'Double Approval'),
        ('triple', 'Triple Approval'),
        ('dynamic', 'Dynamic Approval'),
    ], required=True)
    
    def get_strategy_instance(self):
        """Get workflow strategy instance"""
        strategy_map = {
            'single': self.env['workflow.strategy.single'],
            'double': self.env['workflow.strategy.double'],
            'triple': self.env['workflow.strategy.triple'],
            'dynamic': self.env['workflow.strategy.dynamic'],
        }
        return strategy_map.get(self.strategy_type)


class WorkflowStrategyBase(models.AbstractModel):
    """Base workflow strategy"""
    _name = 'workflow.strategy.base'
    _description = 'Workflow Strategy Base'
    
    @api.model
    def get_approvers(self, record, current_level=0):
        """Get list of approvers for current level"""
        raise NotImplementedError()
    
    @api.model
    def validate_approval(self, record):
        """Validate if approval is complete"""
        raise NotImplementedError()


class SingleApprovalStrategy(models.Model):
    """Single approval workflow"""
    _name = 'workflow.strategy.single'
    _inherit = 'workflow.strategy.base'
    _description = 'Single Approval Strategy'
    
    @api.model
    def get_approvers(self, record, current_level=0):
        """Get manager as approver"""
        if current_level > 0:
            return self.env['res.users']
        
        # Get record's responsible user's manager
        manager = record.user_id.parent_id if hasattr(record, 'user_id') else False
        
        return manager.user_id if manager else self.env['res.users']
    
    @api.model
    def validate_approval(self, record):
        """Check if approved by manager"""
        return record.approval_level >= 1


class DoubleApprovalStrategy(models.Model):
    """Double approval workflow"""
    _name = 'workflow.strategy.double'
    _inherit = 'workflow.strategy.base'
    _description = 'Double Approval Strategy'
    
    @api.model
    def get_approvers(self, record, current_level=0):
        """Get approvers for each level"""
        if current_level == 0:
            # Level 1: Direct manager
            manager = record.user_id.parent_id
            return manager.user_id if manager else self.env['res.users']
        
        elif current_level == 1:
            # Level 2: Department head
            department = record.user_id.department_id
            return department.manager_id.user_id if department else self.env['res.users']
        
        return self.env['res.users']
    
    @api.model
    def validate_approval(self, record):
        """Check if approved by both levels"""
        return record.approval_level >= 2


class DynamicApprovalStrategy(models.Model):
    """Dynamic approval based on amount"""
    _name = 'workflow.strategy.dynamic'
    _inherit = 'workflow.strategy.base'
    _description = 'Dynamic Approval Strategy'
    
    @api.model
    def get_approvers(self, record, current_level=0):
        """Get approvers based on amount"""
        amount = record.amount_total if hasattr(record, 'amount_total') else 0
        
        # Define approval thresholds
        if amount < 1000:
            # Small amount: manager only
            if current_level == 0:
                return record.user_id.parent_id.user_id
        
        elif amount < 10000:
            # Medium amount: manager + department head
            if current_level == 0:
                return record.user_id.parent_id.user_id
            elif current_level == 1:
                return record.user_id.department_id.manager_id.user_id
        
        else:
            # Large amount: manager + department head + CFO
            if current_level == 0:
                return record.user_id.parent_id.user_id
            elif current_level == 1:
                return record.user_id.department_id.manager_id.user_id
            elif current_level == 2:
                # Get CFO
                cfo = self.env.ref('base.group_system').users.filtered(
                    lambda u: u.has_group('account.group_account_manager')
                )[:1]
                return cfo
        
        return self.env['res.users']
    
    @api.model
    def validate_approval(self, record):
        """Validate based on amount"""
        amount = record.amount_total if hasattr(record, 'amount_total') else 0
        
        if amount < 1000:
            return record.approval_level >= 1
        elif amount < 10000:
            return record.approval_level >= 2
        else:
            return record.approval_level >= 3
```

---

## Real-World Examples

### Example 1: Sale Order with Multi-Company Strategies

```python
# models/sale_order.py
from odoo import models, fields, api

class SaleOrder(models.Model):
    _inherit = 'sale.order'
    
    approval_level = fields.Integer(default=0)
    approver_ids = fields.Many2many('res.users', string='Approvers')
    
    @api.onchange('order_line', 'partner_id')
    def _onchange_apply_pricing_strategy(self):
        """Apply company's pricing strategy"""
        if not self.order_line:
            return
        
        # Get pricing strategy
        strategy = self.env.company.get_current_strategy('pricing')
        
        for line in self.order_line:
            # Calculate price using strategy
            result = strategy.calculate_price(
                product=line.product_id,
                quantity=line.product_uom_qty,
                partner=self.partner_id
            )
            
            # Update line
            line.price_unit = result['unit_price']
            line.discount = result['discount']
    
    def action_confirm(self):
        """Override to apply workflow strategy"""
        # Get workflow strategy
        workflow_strategy = self.env.company.get_current_strategy('workflow')
        
        # Check if approval is needed
        if not workflow_strategy.validate_approval(self):
            # Get next approvers
            approvers = workflow_strategy.get_approvers(self, self.approval_level)
            
            if approvers:
                # Request approval
                self.approver_ids = [(6, 0, approvers.ids)]
                self.message_post(
                    body=f'Approval required from: {", ".join(approvers.mapped("name"))}',
                    partner_ids=approvers.partner_id.ids
                )
                return False
        
        # Proceed with confirmation
        return super().action_confirm()
    
    def action_approve(self):
        """Approve order"""
        self.ensure_one()
        
        # Check if current user is approver
        if self.env.user not in self.approver_ids:
            raise UserError('You are not authorized to approve this order')
        
        # Increment approval level
        self.approval_level += 1
        
        # Try to confirm again
        return self.action_confirm()


class SaleOrderLine(models.Model):
    _inherit = 'sale.order.line'
    
    @api.depends('product_uom_qty', 'price_unit', 'tax_id')
    def _compute_amount(self):
        """Override to use tax strategy"""
        for line in self:
            # Get base amount
            price = line.price_unit * (1 - (line.discount or 0.0) / 100.0)
            amount_untaxed = price * line.product_uom_qty
            
            # Apply tax strategy
            tax_strategy = self.env.company.get_current_strategy('tax')
            
            tax_result = tax_strategy.calculate_tax(
                amount=amount_untaxed,
                product=line.product_id,
                partner=line.order_id.partner_id
            )
            
            # Update amounts
            line.price_subtotal = amount_untaxed
            line.price_tax = tax_result['tax_amount']
            line.price_total = tax_result['total_amount']
```

---

### Example 2: Invoice with Company-Specific Logic

```python
# models/account_move.py
from odoo import models, api

class AccountMove(models.Model):
    _inherit = 'account.move'
    
    @api.model
    def _get_tax_strategy(self):
        """Get company's tax strategy"""
        return self.env.company.get_current_strategy('tax')
    
    def _recompute_tax_lines(self):
        """Override to use tax strategy"""
        self.ensure_one()
        
        # Get tax strategy
        tax_strategy = self._get_tax_strategy()
        
        # Remove existing tax lines
        self.line_ids.filtered(lambda l: l.tax_line_id).unlink()
        
        # Recalculate taxes using strategy
        for line in self.invoice_line_ids:
            if not line.tax_ids:
                continue
            
            tax_result = tax_strategy.calculate_tax(
                amount=line.price_subtotal,
                product=line.product_id,
                partner=self.partner_id
            )
            
            # Create tax line
            self.env['account.move.line'].create({
                'move_id': self.id,
                'name': tax_result['tax_name'],
                'debit': tax_result['tax_amount'] if self.move_type == 'out_invoice' else 0,
                'credit': tax_result['tax_amount'] if self.move_type == 'in_invoice' else 0,
                'tax_line_id': line.tax_ids[0].id,
            })
```

---

## Best Practices

### 1. Strategy Selection

```python
# ✅ GOOD: Use factory pattern
def get_strategy(self, strategy_type):
    """Factory method to get strategy"""
    config = self.env.company[f'{strategy_type}_strategy_id']
    return config.get_strategy_instance()

# ❌ BAD: Hardcode strategy selection
if self.env.company.name == 'Company A':
    strategy = self.env['pricing.strategy.discount']
elif self.env.company.name == 'Company B':
    strategy = self.env['pricing.strategy.premium']
```

### 2. Strategy Parameters

```python
# ✅ GOOD: Store parameters in company
class ResCompany(models.Model):
    _inherit = 'res.company'
    
    discount_percentage = fields.Float()
    volume_tiers = fields.One2many('volume.tier', 'company_id')

# ❌ BAD: Hardcode parameters
discount = 10.0  # Don't do this!
```

### 3. Strategy Testing

```python
# tests/test_pricing_strategy.py
from odoo.tests import TransactionCase

class TestPricingStrategy(TransactionCase):
    
    def setUp(self):
        super().setUp()
        
        # Create test company
        self.company = self.env['res.company'].create({
            'name': 'Test Company',
            'discount_percentage': 15.0,
        })
        
        # Create strategy config
        self.strategy_config = self.env['pricing.strategy.config'].create({
            'name': 'Discount Strategy',
            'strategy_type': 'discount',
        })
        
        self.company.pricing_strategy_id = self.strategy_config
    
    def test_discount_calculation(self):
        """Test discount pricing strategy"""
        product = self.env['product.product'].create({
            'name': 'Test Product',
            'list_price': 100.0,
        })
        
        partner = self.env['res.partner'].create({
            'name': 'Test Partner',
        })
        
        # Get strategy
        strategy = self.company.get_current_strategy('pricing')
        
        # Calculate price
        result = strategy.calculate_price(product, 10, partner)
        
        # Assert
        self.assertEqual(result['discount'], 15.0)
        self.assertEqual(result['unit_price'], 85.0)
        self.assertEqual(result['total_price'], 850.0)
```

---

## 🎯 Checklist

- [ ] Strategy interfaces defined
- [ ] Concrete strategies implemented
- [ ] Company configuration added
- [ ] Factory methods created
- [ ] Real-world models integrated
- [ ] Tests written
- [ ] Documentation complete
- [ ] Multi-company tested

---

## 📚 Tài Liệu Tham Khảo

- [Strategy Pattern](https://refactoring.guru/design-patterns/strategy)
- [Odoo Multi-Company](https://www.odoo.com/documentation/19.0/applications/general/companies.html)
- [Design Patterns in Python](https://python-patterns.guide/)

---
name: Odoo Wizards
description: Hướng dẫn toàn diện về Wizards (TransientModel) trong Odoo 19 - tạo popup, xử lý batch operations, và user interactions
---

# Odoo 19 Wizards — Reference Guide

## Mục Lục

1. [TransientModel Basics](#transientmodel-basics)
2. [Wizard Views](#wizard-views)
3. [Wizard Actions](#wizard-actions)
4. [Multi-Step Wizards](#multi-step-wizards)
5. [Batch Operations](#batch-operations)
6. [Best Practices](#best-practices)
7. [Ví Dụ Thực Tế](#ví-dụ-thực-tế)

---

## TransientModel Basics

### 1. Basic Wizard Structure

```python
# wizards/my_wizard.py
from odoo import models, fields, api
from odoo.exceptions import UserError, ValidationError

class MyWizard(models.TransientModel):
    _name = 'my.module.wizard'
    _description = 'My Wizard'

    name = fields.Char(string='Name', required=True)
    partner_id = fields.Many2one('res.partner', string='Partner')
    line_ids = fields.One2many('my.module.wizard.line', 'wizard_id', string='Lines')

    def action_confirm(self):
        """Process wizard and close"""
        self.ensure_one()
        # Business logic here
        return {'type': 'ir.actions.act_window_close'}

    def action_confirm_and_view(self):
        """Process and open result record"""
        self.ensure_one()
        record = self.env['my.model'].create({'name': self.name})
        return {
            'type': 'ir.actions.act_window',
            'res_model': 'my.model',
            'res_id': record.id,
            'view_mode': 'form',
            'target': 'current',
        }
```

### 2. Wizard với Default Values từ Context

```python
class PartnerWizard(models.TransientModel):
    _name = 'partner.wizard'
    _description = 'Partner Wizard'

    partner_id = fields.Many2one('res.partner', string='Partner', required=True)
    email = fields.Char(string='Email')
    phone = fields.Char(string='Phone')

    @api.model
    def default_get(self, fields_list):
        """Set default values from context"""
        res = super().default_get(fields_list)

        if self.env.context.get('active_id'):
            partner = self.env['res.partner'].browse(self.env.context['active_id'])
            res.update({
                'partner_id': partner.id,
                'email': partner.email,
                'phone': partner.phone,
            })

        return res

    def action_update_partner(self):
        self.ensure_one()
        self.partner_id.write({
            'email': self.email,
            'phone': self.phone,
        })
        return {'type': 'ir.actions.act_window_close'}
```

### 3. Wizard với Computed Fields và Wizard Lines

```python
class OrderWizard(models.TransientModel):
    _name = 'order.wizard'
    _description = 'Order Wizard'

    order_id = fields.Many2one('sale.order', string='Order')
    line_ids = fields.One2many('order.wizard.line', 'wizard_id', string='Order Lines')
    total_amount = fields.Float(string='Total', compute='_compute_total_amount')

    @api.depends('line_ids.subtotal')
    def _compute_total_amount(self):
        for wizard in self:
            wizard.total_amount = sum(wizard.line_ids.mapped('subtotal'))

    def action_create_order(self):
        self.ensure_one()
        order = self.env['sale.order'].create({
            'partner_id': self.order_id.partner_id.id,
            'order_line': [
                (0, 0, {
                    'product_id': line.product_id.id,
                    'product_uom_qty': line.quantity,
                    'price_unit': line.price_unit,
                })
                for line in self.line_ids
            ]
        })
        return {
            'type': 'ir.actions.act_window',
            'res_model': 'sale.order',
            'res_id': order.id,
            'view_mode': 'form',
            'target': 'current',
        }


class OrderWizardLine(models.TransientModel):
    _name = 'order.wizard.line'
    _description = 'Order Wizard Line'

    wizard_id = fields.Many2one('order.wizard', required=True, ondelete='cascade')
    product_id = fields.Many2one('product.product', string='Product', required=True)
    quantity = fields.Float(string='Quantity', default=1.0)
    price_unit = fields.Float(string='Unit Price')
    subtotal = fields.Float(string='Subtotal', compute='_compute_subtotal')

    @api.depends('quantity', 'price_unit')
    def _compute_subtotal(self):
        for line in self:
            line.subtotal = line.quantity * line.price_unit

    @api.onchange('product_id')
    def _onchange_product_id(self):
        if self.product_id:
            self.price_unit = self.product_id.list_price
```

---

## Wizard Views

### 1. Basic Wizard Form View

Lưu ý Odoo 19: dùng `<list>` thay vì `<tree>`, dùng `invisible=` thay vì `attrs=`.

```xml
<!-- views/my_wizard_views.xml -->
<odoo>
    <record id="view_my_wizard_form" model="ir.ui.view">
        <field name="name">my.module.wizard.form</field>
        <field name="model">my.module.wizard</field>
        <field name="arch" type="xml">
            <form string="My Wizard">
                <group>
                    <field name="partner_id"/>
                    <field name="name"/>
                </group>
                <field name="line_ids">
                    <list editable="bottom">
                        <field name="product_id"/>
                        <field name="quantity"/>
                    </list>
                </field>
                <footer>
                    <button string="Confirm" name="action_confirm"
                            type="object" class="btn-primary"/>
                    <button string="Cancel" class="btn-secondary" special="cancel"/>
                </footer>
            </form>
        </field>
    </record>
</odoo>
```

### 2. Wizard với Notebook (Tabs)

```xml
<record id="view_order_wizard_form" model="ir.ui.view">
    <field name="name">order.wizard.form</field>
    <field name="model">order.wizard</field>
    <field name="arch" type="xml">
        <form string="Create Order">
            <sheet>
                <group>
                    <field name="order_id"/>
                </group>
                <notebook>
                    <page string="Order Lines">
                        <field name="line_ids">
                            <list editable="bottom">
                                <field name="product_id"/>
                                <field name="quantity"/>
                                <field name="price_unit"/>
                            </list>
                        </field>
                    </page>
                    <page string="Summary">
                        <group>
                            <field name="total_amount"/>
                        </group>
                    </page>
                </notebook>
            </sheet>
            <footer>
                <button string="Create Order" type="object"
                        name="action_create_order" class="btn-primary"/>
                <button string="Cancel" special="cancel"/>
            </footer>
        </form>
    </field>
</record>
```

### 3. Wizard với Conditional Fields (Odoo 19 syntax)

```xml
<record id="view_conditional_wizard_form" model="ir.ui.view">
    <field name="name">conditional.wizard.form</field>
    <field name="model">conditional.wizard</field>
    <field name="arch" type="xml">
        <form string="Conditional Wizard">
            <sheet>
                <group>
                    <field name="action_type"/>
                    <!-- Odoo 17+: dùng invisible= trực tiếp, không dùng attrs= -->
                    <field name="approval_note"
                           invisible="action_type != 'approve'"/>
                    <field name="rejection_reason"
                           invisible="action_type != 'reject'"
                           required="action_type == 'reject'"/>
                </group>
            </sheet>
            <footer>
                <button string="Submit" type="object"
                        name="action_submit" class="btn-primary"/>
                <button string="Cancel" special="cancel"/>
            </footer>
        </form>
    </field>
</record>
```

---

## Wizard Actions

### 1. Window Action để Mở Wizard

```xml
<!-- actions/wizard_actions.xml -->
<odoo>
    <record id="action_my_wizard" model="ir.actions.act_window">
        <field name="name">My Wizard</field>
        <field name="res_model">my.module.wizard</field>
        <field name="view_mode">form</field>
        <field name="target">new</field>
        <!-- Binding để hiện trong Action menu của list/form view -->
        <field name="binding_model_id" ref="model_res_partner"/>
        <field name="binding_view_types">form,list</field>
    </record>
</odoo>
```

### 2. Button trong Form View Cha

```xml
<record id="view_partner_form_inherit" model="ir.ui.view">
    <field name="name">res.partner.form.inherit.my_module</field>
    <field name="model">res.partner</field>
    <field name="inherit_id" ref="base.view_partner_form"/>
    <field name="arch" type="xml">
        <xpath expr="//header" position="inside">
            <button name="%(action_my_wizard)d"
                    string="Open Wizard"
                    type="action"
                    class="btn-primary"/>
        </xpath>
    </field>
</record>
```

### 3. Mở Wizard từ Python với Context

```python
def action_open_wizard(self):
    return {
        'type': 'ir.actions.act_window',
        'name': 'My Wizard',
        'res_model': 'my.module.wizard',
        'view_mode': 'form',
        'target': 'new',
        'context': {
            'default_partner_id': self.partner_id.id,
            'active_ids': self.ids,
            'active_model': self._name,
        },
    }
```

### 4. Return Actions phổ biến

```python
# Đóng popup
return {'type': 'ir.actions.act_window_close'}

# Reload trang
return {'type': 'ir.actions.client', 'tag': 'reload'}

# Mở record cụ thể
return {
    'type': 'ir.actions.act_window',
    'res_model': 'sale.order',
    'res_id': order.id,
    'view_mode': 'form',
    'target': 'current',
}

# Hiện notification rồi đóng
return {
    'type': 'ir.actions.client',
    'tag': 'display_notification',
    'params': {
        'title': 'Success',
        'message': 'Operation completed successfully',
        'type': 'success',  # 'success' | 'warning' | 'danger' | 'info'
        'sticky': False,
        'next': {'type': 'ir.actions.act_window_close'},
    }
}
```

---

## Multi-Step Wizards

### 1. State-Based Multi-Step Wizard

```python
class MultiStepWizard(models.TransientModel):
    _name = 'multi.step.wizard'
    _description = 'Multi-Step Wizard'

    state = fields.Selection([
        ('step1', 'Step 1: Basic Info'),
        ('step2', 'Step 2: Details'),
        ('step3', 'Step 3: Confirmation'),
    ], default='step1', required=True)

    # Step 1 fields
    name = fields.Char(string='Name')
    email = fields.Char(string='Email')

    # Step 2 fields
    address = fields.Text(string='Address')
    phone = fields.Char(string='Phone')

    # Step 3 — summary computed
    summary = fields.Html(string='Summary', compute='_compute_summary')

    @api.depends('name', 'email', 'address', 'phone')
    def _compute_summary(self):
        for wizard in self:
            wizard.summary = (
                f'<h3>Summary</h3>'
                f'<p><b>Name:</b> {wizard.name}</p>'
                f'<p><b>Email:</b> {wizard.email}</p>'
                f'<p><b>Address:</b> {wizard.address}</p>'
                f'<p><b>Phone:</b> {wizard.phone}</p>'
            )

    def action_next(self):
        self.ensure_one()
        if self.state == 'step1':
            if not self.name or not self.email:
                raise ValidationError('Name and Email are required')
            self.state = 'step2'
        elif self.state == 'step2':
            self.state = 'step3'
        # Re-open same wizard record
        return {
            'type': 'ir.actions.act_window',
            'res_model': self._name,
            'res_id': self.id,
            'view_mode': 'form',
            'target': 'new',
        }

    def action_previous(self):
        self.ensure_one()
        if self.state == 'step3':
            self.state = 'step2'
        elif self.state == 'step2':
            self.state = 'step1'
        return {
            'type': 'ir.actions.act_window',
            'res_model': self._name,
            'res_id': self.id,
            'view_mode': 'form',
            'target': 'new',
        }

    def action_confirm(self):
        self.ensure_one()
        partner = self.env['res.partner'].create({
            'name': self.name,
            'email': self.email,
            'street': self.address,
            'phone': self.phone,
        })
        return {
            'type': 'ir.actions.act_window',
            'res_model': 'res.partner',
            'res_id': partner.id,
            'view_mode': 'form',
            'target': 'current',
        }
```

### 2. Multi-Step Wizard View

```xml
<record id="view_multi_step_wizard_form" model="ir.ui.view">
    <field name="name">multi.step.wizard.form</field>
    <field name="model">multi.step.wizard</field>
    <field name="arch" type="xml">
        <form string="Multi-Step Wizard">
            <sheet>
                <!-- Step 1 -->
                <group invisible="state != 'step1'">
                    <field name="name" required="state == 'step1'"/>
                    <field name="email" required="state == 'step1'"/>
                </group>

                <!-- Step 2 -->
                <group invisible="state != 'step2'">
                    <field name="address"/>
                    <field name="phone"/>
                </group>

                <!-- Step 3 -->
                <group invisible="state != 'step3'">
                    <field name="summary" nolabel="1"/>
                </group>

                <field name="state" invisible="1"/>
            </sheet>
            <footer>
                <button string="Previous" type="object" name="action_previous"
                        invisible="state == 'step1'"/>
                <button string="Next" type="object" name="action_next"
                        class="btn-primary" invisible="state == 'step3'"/>
                <button string="Confirm" type="object" name="action_confirm"
                        class="btn-primary" invisible="state != 'step3'"/>
                <button string="Cancel" special="cancel"/>
            </footer>
        </form>
    </field>
</record>
```

---

## Batch Operations

### 1. Batch Update Wizard

```python
class BatchUpdateWizard(models.TransientModel):
    _name = 'batch.update.wizard'
    _description = 'Batch Update Wizard'

    partner_ids = fields.Many2many(
        'res.partner',
        string='Partners',
        default=lambda self: self.env.context.get('active_ids', [])
    )
    field_to_update = fields.Selection([
        ('category', 'Category'),
        ('user', 'Salesperson'),
    ], string='Field to Update', required=True)

    category_id = fields.Many2one('res.partner.category', string='Category')
    user_id = fields.Many2one('res.users', string='Salesperson')

    def action_update(self):
        self.ensure_one()
        if not self.partner_ids:
            raise UserError('No partners selected')

        values = {}
        if self.field_to_update == 'category':
            values['category_id'] = [(4, self.category_id.id)]
        elif self.field_to_update == 'user':
            values['user_id'] = self.user_id.id

        self.partner_ids.write(values)

        return {
            'type': 'ir.actions.client',
            'tag': 'display_notification',
            'params': {
                'title': 'Success',
                'message': f'{len(self.partner_ids)} partners updated',
                'type': 'success',
                'next': {'type': 'ir.actions.act_window_close'},
            }
        }
```

### 2. Batch Process Pattern (simple)

```python
class BatchWizard(models.TransientModel):
    _name = 'batch.wizard'
    _description = 'Batch Wizard'

    def action_process(self):
        active_ids = self.env.context.get('active_ids', [])
        records = self.env['my.model'].browse(active_ids)
        for record in records:
            record.action_confirm()
        return {'type': 'ir.actions.act_window_close'}
```

---

## Best Practices

### Validation và Error Handling

```python
class ValidatedWizard(models.TransientModel):
    _name = 'validated.wizard'
    _description = 'Validated Wizard'

    amount = fields.Float(string='Amount', required=True)
    date = fields.Date(string='Date', required=True)

    @api.constrains('amount')
    def _check_amount(self):
        for wizard in self:
            if wizard.amount <= 0:
                raise ValidationError('Amount must be greater than 0')

    @api.constrains('date')
    def _check_date(self):
        for wizard in self:
            if wizard.date < fields.Date.today():
                raise ValidationError('Date cannot be in the past')

    def action_process(self):
        self.ensure_one()
        try:
            self._do_processing()
            return {
                'type': 'ir.actions.client',
                'tag': 'display_notification',
                'params': {
                    'title': 'Success',
                    'message': 'Processing completed',
                    'type': 'success',
                }
            }
        except Exception as e:
            raise UserError(f'Processing failed: {str(e)}')
```

### ir.model.access.csv cho Wizard

Mọi TransientModel đều cần entry trong `security/ir.model.access.csv`:

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_my_wizard,my.module.wizard,model_my_module_wizard,,1,1,1,1
access_my_wizard_line,my.module.wizard.line,model_my_module_wizard_line,,1,1,1,1
```

### __manifest__.py — khai báo files

```python
{
    'name': 'My Module',
    'version': '19.0.1.0.0',
    'data': [
        'security/ir.model.access.csv',
        'views/my_wizard_views.xml',
        'wizards/my_wizard_actions.xml',
    ],
}
```

---

## Ví Dụ Thực Tế

### Invoice Payment Wizard

```python
class InvoicePaymentWizard(models.TransientModel):
    _name = 'invoice.payment.wizard'
    _description = 'Invoice Payment Wizard'

    invoice_id = fields.Many2one('account.move', string='Invoice', required=True)
    amount = fields.Float(string='Payment Amount', required=True)
    payment_date = fields.Date(string='Payment Date', default=fields.Date.today)
    journal_id = fields.Many2one('account.journal', string='Payment Journal', required=True)

    @api.model
    def default_get(self, fields_list):
        res = super().default_get(fields_list)
        if self.env.context.get('active_id'):
            invoice = self.env['account.move'].browse(self.env.context['active_id'])
            res.update({
                'invoice_id': invoice.id,
                'amount': invoice.amount_residual,
            })
        return res

    def action_register_payment(self):
        self.ensure_one()
        payment = self.env['account.payment'].create({
            'payment_type': 'inbound',
            'partner_type': 'customer',
            'partner_id': self.invoice_id.partner_id.id,
            'amount': self.amount,
            'date': self.payment_date,
            'journal_id': self.journal_id.id,
        })
        payment.action_post()
        return {
            'type': 'ir.actions.act_window',
            'res_model': 'account.payment',
            'res_id': payment.id,
            'view_mode': 'form',
            'target': 'current',
        }
```

### Mass Mailing Wizard

```python
class MassMailingWizard(models.TransientModel):
    _name = 'mass.mailing.wizard'
    _description = 'Mass Mailing Wizard'

    partner_ids = fields.Many2many(
        'res.partner',
        string='Recipients',
        default=lambda self: self.env.context.get('active_ids', [])
    )
    subject = fields.Char(string='Subject', required=True)
    body = fields.Html(string='Message', required=True)
    template_id = fields.Many2one('mail.template', string='Email Template')

    @api.onchange('template_id')
    def _onchange_template_id(self):
        if self.template_id:
            self.subject = self.template_id.subject
            self.body = self.template_id.body_html

    def action_send_mail(self):
        self.ensure_one()
        for partner in self.partner_ids:
            if partner.email:
                self.env['mail.mail'].create({
                    'subject': self.subject,
                    'body_html': self.body,
                    'email_to': partner.email,
                }).send()
        return {
            'type': 'ir.actions.client',
            'tag': 'display_notification',
            'params': {
                'title': 'Emails Sent',
                'message': f'{len(self.partner_ids)} emails sent successfully',
                'type': 'success',
                'next': {'type': 'ir.actions.act_window_close'},
            }
        }
```

---

## Checklist

Khi tạo Wizards, đảm bảo:

- [ ] Kế thừa từ `models.TransientModel`
- [ ] Có `_name` và `_description` rõ ràng
- [ ] Form view có footer với buttons
- [ ] Sử dụng `target="new"` trong action
- [ ] Implement `default_get()` nếu cần default values từ context
- [ ] Validation đầy đủ cho user input (`@api.constrains` hoặc check trong action)
- [ ] Return appropriate action sau khi xử lý
- [ ] Error handling với `UserError`/`ValidationError`
- [ ] Xử lý context (`active_id`, `active_ids`)
- [ ] Entry trong `ir.model.access.csv`
- [ ] Dùng `<list>` thay vì `<tree>` (Odoo 17+)
- [ ] Dùng `invisible=` thay vì `attrs=` (Odoo 17+)

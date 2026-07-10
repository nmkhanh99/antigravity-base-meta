---
name: Odoo Wizards
description: Hướng dẫn toàn diện về Wizards (TransientModel) trong Odoo - tạo popup, xử lý batch operations, và user interactions
---

# Odoo Wizards Skill

## 📋 Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [TransientModel Basics](#transientmodel-basics)
3. [Wizard Views](#wizard-views)
4. [Wizard Actions](#wizard-actions)
5. [Multi-Step Wizards](#multi-step-wizards)
6. [Batch Operations](#batch-operations)
7. [Best Practices](#best-practices)
8. [Ví Dụ Thực Tế](#ví-dụ-thực-tế)

---

## Tổng Quan

### Wizards Là Gì?

Wizards (hay TransientModels) là các models tạm thời trong Odoo, được sử dụng để:
- Thu thập input từ user qua popup/dialog
- Xử lý batch operations trên nhiều records
- Thực hiện các tác vụ phức tạp cần nhiều bước
- Tạo temporary data không cần lưu lâu dài

### Đặc Điểm của TransientModel

- ✅ Tự động xóa sau một khoảng thời gian (mặc định: 1 giờ)
- ✅ Không có menu riêng
- ✅ Được mở qua actions từ buttons/menu items
- ✅ Thường có form view dạng popup
- ✅ Kế thừa từ `models.TransientModel`

### Khi Nào Sử Dụng Wizards?

- ✅ Confirm actions (xác nhận trước khi xóa, approve, etc.)
- ✅ Collect additional data (nhập thông tin bổ sung)
- ✅ Batch operations (xử lý nhiều records cùng lúc)
- ✅ Multi-step processes (quy trình nhiều bước)
- ✅ Report parameters (nhập tham số cho báo cáo)
- ✅ Import/Export wizards

---

## TransientModel Basics

### 1. Basic Wizard Structure

```python
# models/wizard_example.py
from odoo import models, fields, api
from odoo.exceptions import UserError, ValidationError

class ExampleWizard(models.TransientModel):
    _name = 'example.wizard'
    _description = 'Example Wizard'
    
    # Fields
    name = fields.Char(string='Name', required=True)
    description = fields.Text(string='Description')
    date = fields.Date(string='Date', default=fields.Date.today)
    
    # Action method
    def action_confirm(self):
        """Execute wizard action"""
        self.ensure_one()
        
        # Your business logic here
        # ...
        
        return {'type': 'ir.actions.act_window_close'}
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
        
        # Get active record from context
        if self.env.context.get('active_id'):
            partner = self.env['res.partner'].browse(
                self.env.context['active_id']
            )
            res.update({
                'partner_id': partner.id,
                'email': partner.email,
                'phone': partner.phone,
            })
        
        return res
    
    def action_update_partner(self):
        """Update partner information"""
        self.ensure_one()
        
        self.partner_id.write({
            'email': self.email,
            'phone': self.phone,
        })
        
        return {'type': 'ir.actions.act_window_close'}
```

### 3. Wizard với Computed Fields

```python
class OrderWizard(models.TransientModel):
    _name = 'order.wizard'
    _description = 'Order Wizard'
    
    order_id = fields.Many2one('sale.order', string='Order')
    line_ids = fields.One2many(
        'order.wizard.line', 'wizard_id', 
        string='Order Lines'
    )
    total_amount = fields.Float(
        string='Total', 
        compute='_compute_total_amount'
    )
    
    @api.depends('line_ids.subtotal')
    def _compute_total_amount(self):
        for wizard in self:
            wizard.total_amount = sum(wizard.line_ids.mapped('subtotal'))
    
    def action_create_order(self):
        """Create order from wizard"""
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
        """Auto-fill price when product changes"""
        if self.product_id:
            self.price_unit = self.product_id.list_price
```

---

## Wizard Views

### 1. Basic Wizard Form View

```xml
<!-- views/wizard_views.xml -->
<odoo>
    <record id="view_example_wizard_form" model="ir.ui.view">
        <field name="name">example.wizard.form</field>
        <field name="model">example.wizard</field>
        <field name="arch" type="xml">
            <form string="Example Wizard">
                <sheet>
                    <group>
                        <field name="name"/>
                        <field name="description"/>
                        <field name="date"/>
                    </group>
                </sheet>
                <footer>
                    <button string="Confirm" 
                            type="object" 
                            name="action_confirm" 
                            class="btn-primary"/>
                    <button string="Cancel" 
                            class="btn-secondary" 
                            special="cancel"/>
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
                <button string="Create Order" 
                        type="object" 
                        name="action_create_order" 
                        class="btn-primary"/>
                <button string="Cancel" special="cancel"/>
            </footer>
        </form>
    </field>
</record>
```

### 3. Wizard với Conditional Fields

```xml
<record id="view_conditional_wizard_form" model="ir.ui.view">
    <field name="name">conditional.wizard.form</field>
    <field name="model">conditional.wizard</field>
    <field name="arch" type="xml">
        <form string="Conditional Wizard">
            <sheet>
                <group>
                    <field name="action_type"/>
                    
                    <!-- Show only if action_type == 'approve' -->
                    <field name="approval_note" 
                           invisible="action_type != 'approve'"/>
                    
                    <!-- Show only if action_type == 'reject' -->
                    <field name="rejection_reason" 
                           invisible="action_type != 'reject'" 
                           required="action_type == 'reject'"/>
                </group>
            </sheet>
            <footer>
                <button string="Submit" 
                        type="object" 
                        name="action_submit" 
                        class="btn-primary"/>
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
    <record id="action_example_wizard" model="ir.actions.act_window">
        <field name="name">Example Wizard</field>
        <field name="res_model">example.wizard</field>
        <field name="view_mode">form</field>
        <field name="target">new</field>  <!-- Open as popup -->
        <field name="binding_model_id" ref="model_res_partner"/>
        <field name="binding_view_types">form,list</field>
    </record>
</odoo>
```

### 2. Button để Mở Wizard từ Form View

```xml
<!-- In partner form view -->
<record id="view_partner_form_inherit" model="ir.ui.view">
    <field name="name">res.partner.form.inherit</field>
    <field name="model">res.partner</field>
    <field name="inherit_id" ref="base.view_partner_form"/>
    <field name="arch" type="xml">
        <xpath expr="//header" position="inside">
            <button name="%(action_example_wizard)d" 
                    string="Open Wizard" 
                    type="action" 
                    class="btn-primary"/>
        </xpath>
    </field>
</record>
```

### 3. Wizard Return Actions

```python
class ReturnActionWizard(models.TransientModel):
    _name = 'return.action.wizard'
    _description = 'Return Action Wizard'
    
    def action_close(self):
        """Close wizard"""
        return {'type': 'ir.actions.act_window_close'}
    
    def action_reload(self):
        """Close and reload parent view"""
        return {
            'type': 'ir.actions.client',
            'tag': 'reload',
        }
    
    def action_open_record(self):
        """Close and open specific record"""
        record = self.env['sale.order'].create({
            'partner_id': self.partner_id.id,
        })
        
        return {
            'type': 'ir.actions.act_window',
            'res_model': 'sale.order',
            'res_id': record.id,
            'view_mode': 'form',
            'target': 'current',
        }
    
    def action_show_notification(self):
        """Show notification and close"""
        return {
            'type': 'ir.actions.client',
            'tag': 'display_notification',
            'params': {
                'title': 'Success',
                'message': 'Operation completed successfully',
                'type': 'success',
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
    
    # Step 3 fields (computed)
    summary = fields.Html(string='Summary', compute='_compute_summary')
    
    @api.depends('name', 'email', 'address', 'phone')
    def _compute_summary(self):
        for wizard in self:
            wizard.summary = f"""
                <h3>Summary</h3>
                <p><b>Name:</b> {wizard.name}</p>
                <p><b>Email:</b> {wizard.email}</p>
                <p><b>Address:</b> {wizard.address}</p>
                <p><b>Phone:</b> {wizard.phone}</p>
            """
    
    def action_next(self):
        """Move to next step"""
        self.ensure_one()
        
        if self.state == 'step1':
            if not self.name or not self.email:
                raise ValidationError('Name and Email are required')
            self.state = 'step2'
        elif self.state == 'step2':
            self.state = 'step3'
        
        return {
            'type': 'ir.actions.act_window',
            'res_model': self._name,
            'res_id': self.id,
            'view_mode': 'form',
            'target': 'new',
        }
    
    def action_previous(self):
        """Move to previous step"""
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
        """Final confirmation"""
        self.ensure_one()
        
        # Create partner
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
                    <group>
                        <field name="name" required="state == 'step1'"/>
                        <field name="email" required="state == 'step1'"/>
                    </group>
                </group>
                
                <!-- Step 2 -->
                <group invisible="state != 'step2'">
                    <group>
                        <field name="address"/>
                        <field name="phone"/>
                    </group>
                </group>
                
                <!-- Step 3 -->
                <group invisible="state != 'step3'">
                    <field name="summary" nolabel="1"/>
                </group>
                
                <field name="state" invisible="1"/>
            </sheet>
            <footer>
                <!-- Previous button (not on step 1) -->
                <button string="Previous" 
                        type="object" 
                        name="action_previous" 
                        invisible="state == 'step1'"/>
                
                <!-- Next button (not on last step) -->
                <button string="Next" 
                        type="object" 
                        name="action_next" 
                        class="btn-primary"
                        invisible="state == 'step3'"/>
                
                <!-- Confirm button (only on last step) -->
                <button string="Confirm" 
                        type="object" 
                        name="action_confirm" 
                        class="btn-primary"
                        invisible="state != 'step3'"/>
                
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
        ('tag', 'Tags'),
    ], string='Field to Update', required=True)
    
    # Update values
    category_id = fields.Many2one('res.partner.category', string='Category')
    user_id = fields.Many2one('res.users', string='Salesperson')
    tag_ids = fields.Many2many('res.partner.category', string='Tags')
    
    def action_update(self):
        """Apply batch update"""
        self.ensure_one()
        
        if not self.partner_ids:
            raise UserError('No partners selected')
        
        values = {}
        if self.field_to_update == 'category':
            values['category_id'] = [(4, self.category_id.id)]
        elif self.field_to_update == 'user':
            values['user_id'] = self.user_id.id
        elif self.field_to_update == 'tag':
            values['category_id'] = [(6, 0, self.tag_ids.ids)]
        
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

### 2. Batch Delete Wizard

```python
class BatchDeleteWizard(models.TransientModel):
    _name = 'batch.delete.wizard'
    _description = 'Batch Delete Wizard'
    
    record_ids = fields.Many2many(
        'sale.order',
        string='Orders to Delete',
        default=lambda self: self.env.context.get('active_ids', [])
    )
    confirm_text = fields.Char(string='Type DELETE to confirm')
    
    @api.constrains('confirm_text')
    def _check_confirm_text(self):
        for wizard in self:
            if wizard.confirm_text and wizard.confirm_text != 'DELETE':
                raise ValidationError('Please type DELETE to confirm')
    
    def action_delete(self):
        """Delete selected records"""
        self.ensure_one()
        
        if self.confirm_text != 'DELETE':
            raise UserError('Please type DELETE to confirm')
        
        count = len(self.record_ids)
        self.record_ids.unlink()
        
        return {
            'type': 'ir.actions.client',
            'tag': 'display_notification',
            'params': {
                'title': 'Deleted',
                'message': f'{count} orders deleted',
                'type': 'warning',
                'next': {'type': 'ir.actions.act_window_close'},
            }
        }
```

---

## Best Practices

### 1. Validation và Error Handling

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
        """Process with error handling"""
        self.ensure_one()
        
        try:
            # Business logic
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

### 2. Progress Tracking

```python
class ProgressWizard(models.TransientModel):
    _name = 'progress.wizard'
    _description = 'Progress Wizard'
    
    total_records = fields.Integer(string='Total Records')
    processed_records = fields.Integer(string='Processed Records')
    progress = fields.Float(
        string='Progress (%)', 
        compute='_compute_progress'
    )
    
    @api.depends('total_records', 'processed_records')
    def _compute_progress(self):
        for wizard in self:
            if wizard.total_records > 0:
                wizard.progress = (wizard.processed_records / wizard.total_records) * 100
            else:
                wizard.progress = 0.0
    
    def action_process_batch(self):
        """Process records in batches"""
        self.ensure_one()
        
        records = self.env['sale.order'].search([('state', '=', 'draft')])
        self.total_records = len(records)
        self.processed_records = 0
        
        for record in records:
            # Process record
            record.action_confirm()
            
            # Update progress
            self.processed_records += 1
            
            # Commit every 100 records
            if self.processed_records % 100 == 0:
                self.env.cr.commit()
        
        return {'type': 'ir.actions.act_window_close'}
```

---

## Ví Dụ Thực Tế

### 1. Invoice Payment Wizard

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
        """Register payment for invoice"""
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
        
        # Reconcile with invoice
        (payment.line_ids + self.invoice_id.line_ids).filtered(
            lambda line: line.account_id == payment.destination_account_id
        ).reconcile()
        
        return {
            'type': 'ir.actions.act_window',
            'res_model': 'account.payment',
            'res_id': payment.id,
            'view_mode': 'form',
            'target': 'current',
        }
```

### 2. Mass Mailing Wizard

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
        """Load template content"""
        if self.template_id:
            self.subject = self.template_id.subject
            self.body = self.template_id.body_html
    
    def action_send_mail(self):
        """Send emails to all recipients"""
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

## 🎯 Checklist

Khi tạo Wizards, đảm bảo:

- [ ] Kế thừa từ `models.TransientModel`
- [ ] Có `_name` và `_description` rõ ràng
- [ ] Form view có footer với buttons
- [ ] Sử dụng `target="new"` trong action
- [ ] Implement `default_get()` nếu cần default values
- [ ] Validation đầy đủ cho user input
- [ ] Return appropriate action sau khi xử lý
- [ ] Error handling và user feedback
- [ ] Xử lý context (active_id, active_ids)
- [ ] Clean up transient data nếu cần

---

## 📚 Tài Liệu Tham Khảo

- [Odoo TransientModel Documentation](https://www.odoo.com/documentation/19.0/developer/reference/backend/orm.html#transient-models)
- [Wizards Best Practices](https://www.odoo.com/documentation/19.0/developer/tutorials/backend.html#wizards)

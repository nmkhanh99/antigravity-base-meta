---
name: Odoo Email Automation
description: Hướng dẫn toàn diện về email templates, notifications, và automated email workflows trong Odoo
---

# Odoo Email Automation Skill

## 📋 Mục Lục

1. [Email Templates](#email-templates)
2. [Automated Emails](#automated-emails)
3. [Email Notifications](#email-notifications)
4. [Email Queue](#email-queue)

---

## Email Templates

### Basic Email Template

```xml
<record id="email_template_order_confirmation" model="mail.template">
    <field name="name">Order Confirmation</field>
    <field name="model_id" ref="sale.model_sale_order"/>
    <field name="subject">Order ${object.name} Confirmed</field>
    <field name="email_from">${object.company_id.email or ''}</field>
    <field name="email_to">${object.partner_id.email}</field>
    <field name="body_html"><![CDATA[
<table border="0" cellpadding="0" cellspacing="0" style="width:100%">
    <tr>
        <td style="padding:20px">
            <h2>Order Confirmation</h2>
            <p>Dear ${object.partner_id.name},</p>
            <p>Your order <strong>${object.name}</strong> has been confirmed.</p>
            
            <h3>Order Details:</h3>
            <table border="1" cellpadding="5" style="width:100%">
                <tr>
                    <th>Product</th>
                    <th>Quantity</th>
                    <th>Price</th>
                </tr>
                <t t-foreach="object.order_line" t-as="line">
                    <tr>
                        <td>${line.product_id.name}</td>
                        <td>${line.product_uom_qty}</td>
                        <td>${line.price_unit}</td>
                    </tr>
                </t>
            </table>
            
            <p><strong>Total: ${object.amount_total} ${object.currency_id.name}</strong></p>
            
            <p>Thank you for your business!</p>
        </td>
    </tr>
</table>
    ]]></field>
</record>
```

### Template with Attachments

```xml
<record id="email_template_with_attachment" model="mail.template">
    <field name="name">Invoice Email</field>
    <field name="model_id" ref="account.model_account_move"/>
    <field name="subject">Invoice ${object.name}</field>
    <field name="email_to">${object.partner_id.email}</field>
    <field name="body_html"><![CDATA[
<p>Dear ${object.partner_id.name},</p>
<p>Please find attached your invoice.</p>
    ]]></field>
    <field name="report_template" ref="account.account_invoices"/>
    <field name="report_name">Invoice_${object.name}</field>
</record>
```

---

## Automated Emails

### Send Email on State Change

```python
from odoo import models, api

class SaleOrder(models.Model):
    _inherit = 'sale.order'
    
    def action_confirm(self):
        """Send confirmation email on order confirmation"""
        res = super().action_confirm()
        
        # Send email
        template = self.env.ref('my_module.email_template_order_confirmation')
        for order in self:
            template.send_mail(order.id, force_send=True)
        
        return res
```

### Scheduled Email

```python
def send_daily_report(self):
    """Send daily report email"""
    # Get data
    orders = self.env['sale.order'].search([
        ('date_order', '>=', fields.Date.today()),
    ])
    
    # Prepare email
    template = self.env.ref('my_module.email_template_daily_report')
    
    # Send to managers
    managers = self.env['res.users'].search([
        ('groups_id', 'in', self.env.ref('sales_team.group_sale_manager').id)
    ])
    
    for manager in managers:
        template.with_context(
            orders=orders,
        ).send_mail(
            manager.partner_id.id,
            force_send=True
        )
```

---

## Email Notifications

### Activity Notifications

```python
def create_activity_with_email(self):
    """Create activity with email notification"""
    self.activity_schedule(
        'mail.mail_activity_data_todo',
        summary='Follow up required',
        note='Please follow up on this order',
        user_id=self.user_id.id,
    )
```

### Message Post

```python
def notify_users(self):
    """Send notification to followers"""
    self.message_post(
        body='Order has been updated',
        subject='Order Update',
        message_type='notification',
        subtype_xmlid='mail.mt_comment',
        partner_ids=self.message_partner_ids.ids,
    )
```

---

## Email Queue

### Queue Management

```python
def send_bulk_emails(self):
    """Send emails in bulk"""
    partners = self.env['res.partner'].search([
        ('customer_rank', '>', 0)
    ])
    
    template = self.env.ref('my_module.email_template_newsletter')
    
    # Queue emails
    for partner in partners:
        template.send_mail(
            partner.id,
            force_send=False,  # Queue instead of sending immediately
        )
    
    # Process queue
    self.env['mail.mail'].sudo().process_email_queue()
```

---

## 🎯 Best Practices

- ✅ Use templates for reusable emails
- ✅ Queue bulk emails
- ✅ Handle bounces
- ✅ Test templates thoroughly
- ✅ Respect opt-out preferences
- ✅ Monitor email delivery

---

## 📚 Tài Liệu Tham Khảo

- [Odoo Email](https://www.odoo.com/documentation/19.0/developer/reference/backend/mixins.html#mail)
- [Mail Templates](https://www.odoo.com/documentation/19.0/developer/reference/backend/mixins.html#mail-templates)

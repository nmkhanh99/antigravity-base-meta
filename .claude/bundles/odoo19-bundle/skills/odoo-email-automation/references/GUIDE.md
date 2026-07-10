---
name: Odoo Email Automation - Reference Guide
description: Hướng dẫn toàn diện về email templates, notifications, và automated email workflows trong Odoo 19
---

# Odoo 19 Email Automation - Reference Guide

## Mục Lục

1. [Email Templates](#email-templates)
2. [Automated Emails](#automated-emails)
3. [Email Notifications](#email-notifications)
4. [Email Queue](#email-queue)
5. [Best Practices](#best-practices)

---

## Email Templates

### Lưu ý Syntax Odoo 19
Odoo 19 dùng **Jinja2** (`{{ object.field }}`), KHÔNG dùng Mako (`${object.field}`).  
Mako đã bị deprecated từ Odoo 14 và bị loại bỏ hoàn toàn.

### Basic Email Template

```xml
<!-- data/mail_templates.xml -->
<odoo>
    <data noupdate="1">
        <record id="email_template_order_confirmation" model="mail.template">
            <field name="name">Order Confirmation</field>
            <field name="model_id" ref="sale.model_sale_order"/>
            <field name="subject">Order {{ object.name }} Confirmed</field>
            <field name="email_from">{{ object.company_id.email or user.email }}</field>
            <field name="email_to">{{ object.partner_id.email }}</field>
            <field name="body_html" type="html">
                <table border="0" cellpadding="0" cellspacing="0" style="width:100%">
                    <tr>
                        <td style="padding:20px">
                            <h2>Order Confirmation</h2>
                            <p>Dear {{ object.partner_id.name }},</p>
                            <p>Your order <strong>{{ object.name }}</strong> has been confirmed.</p>

                            <h3>Order Details:</h3>
                            <table border="1" cellpadding="5" style="width:100%">
                                <tr>
                                    <th>Product</th>
                                    <th>Quantity</th>
                                    <th>Price</th>
                                </tr>
                                {% for line in object.order_line %}
                                <tr>
                                    <td>{{ line.product_id.name }}</td>
                                    <td>{{ line.product_uom_qty }}</td>
                                    <td>{{ line.price_unit }}</td>
                                </tr>
                                {% endfor %}
                            </table>

                            <p><strong>Total: {{ format_amount(object.amount_total, object.currency_id) }}</strong></p>
                            <p>Thank you for your business!</p>
                        </td>
                    </tr>
                </table>
            </field>
        </record>
    </data>
</odoo>
```

### Template with Report Attachment

```xml
<record id="email_template_with_attachment" model="mail.template">
    <field name="name">Invoice Email</field>
    <field name="model_id" ref="account.model_account_move"/>
    <field name="subject">Invoice {{ object.name }}</field>
    <field name="email_from">{{ object.company_id.email or user.email }}</field>
    <field name="email_to">{{ object.partner_id.email }}</field>
    <field name="body_html" type="html">
        <p>Dear {{ object.partner_id.name }},</p>
        <p>Please find attached your invoice.</p>
    </field>
    <!-- Attach PDF report -->
    <field name="report_template_ids" eval="[(4, ref('account.account_invoices'))]"/>
    <field name="report_name">Invoice_{{ object.name }}</field>
</record>
```

> **Odoo 19 note**: Field `report_template` (Many2one) đã được thay bằng `report_template_ids` (Many2many) từ Odoo 17+. Dùng `eval="[(4, ref(...))]"` để add.

---

## Automated Emails

### Send Email on State Change

```python
# models/my_model.py
from odoo import models, api

class SaleOrder(models.Model):
    _inherit = 'sale.order'

    def action_confirm(self):
        """Send confirmation email on order confirmation"""
        res = super().action_confirm()

        template = self.env.ref('my_module.email_template_order_confirmation')
        for order in self:
            template.send_mail(order.id, force_send=True)

        return res
```

### Scheduled Email (Cron-based)

```python
# models/my_model.py
from odoo import models, fields, api

class MyModel(models.Model):
    _name = 'my.model'

    def send_daily_report(self):
        """Send daily report email — gọi từ ir.cron"""
        orders = self.env['sale.order'].search([
            ('date_order', '>=', fields.Date.today()),
            ('state', '=', 'sale'),
        ])

        template = self.env.ref('my_module.email_template_daily_report')
        manager_group = self.env.ref('sales_team.group_sale_manager')

        managers = self.env['res.users'].search([
            ('groups_id', 'in', manager_group.id)
        ])

        for manager in managers:
            template.with_context(
                orders=orders,
            ).send_mail(
                manager.partner_id.id,
                force_send=True,
            )
```

```xml
<!-- data/ir_cron.xml -->
<record id="ir_cron_send_daily_report" model="ir.cron">
    <field name="name">Send Daily Sales Report</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="state">code</field>
    <field name="code">model.send_daily_report()</field>
    <field name="interval_number">1</field>
    <field name="interval_type">days</field>
    <field name="numbercall">-1</field>
    <field name="active">True</field>
</record>
```

---

## Email Notifications

### Post Message vào Chatter

```python
def notify_users(self):
    """Post notification cho followers"""
    self.message_post(
        body='Order has been updated',
        subject='Order Update',
        message_type='notification',        # 'notification' | 'comment' | 'email'
        subtype_xmlid='mail.mt_comment',    # 'mail.mt_note' (internal) | 'mail.mt_comment' (public)
        partner_ids=self.message_partner_ids.ids,
    )
```

### Lên Lịch Activity

```python
from odoo import models, fields

class MyModel(models.Model):
    _inherit = ['mail.thread', 'mail.activity.mixin']

    def create_followup_activity(self):
        self.activity_schedule(
            'mail.mail_activity_data_todo',
            date_deadline=fields.Date.today(),
            summary='Follow up required',
            note='Please follow up on this order',
            user_id=self.user_id.id,
        )
```

### Field Tracking

```python
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['mail.thread', 'mail.activity.mixin']
    _description = 'My Model'

    state = fields.Selection([
        ('draft', 'Draft'),
        ('confirmed', 'Confirmed'),
        ('done', 'Done'),
    ], default='draft', tracking=True)      # Tự log mọi thay đổi state

    partner_id = fields.Many2one('res.partner', tracking=True)
    amount_total = fields.Monetary(tracking=True)
```

---

## Email Queue

### Bulk Email với Queue

```python
def send_bulk_emails(self):
    """Gửi email hàng loạt qua queue"""
    partners = self.env['res.partner'].search([
        ('customer_rank', '>', 0),
        ('email', '!=', False),
    ])

    template = self.env.ref('my_module.email_template_newsletter')

    # Đẩy vào queue (force_send=False)
    for partner in partners:
        template.send_mail(
            partner.id,
            force_send=False,
        )

    # Xử lý queue ngay (hoặc để ir.cron tự xử lý)
    self.env['mail.mail'].sudo().process_email_queue()
```

### Kiểm Tra Email Queue

```python
# Xem email đang pending trong queue
pending = self.env['mail.mail'].search([
    ('state', '=', 'outgoing'),
])

# Retry failed emails
failed = self.env['mail.mail'].search([
    ('state', '=', 'exception'),
])
failed.send()
```

---

## Best Practices

| Practice | Chi tiết |
|---|---|
| Dùng templates | Tái sử dụng qua `mail.template`, không hardcode HTML trong Python |
| Queue bulk email | `force_send=False` + `process_email_queue()` cho gửi hàng loạt |
| Tracking fields | `tracking=True` trên fields quan trọng thay vì tự log |
| Kiểm tra email | Luôn check `partner_id.email != False` trước khi gửi |
| Noupdate | Dùng `<data noupdate="1">` cho templates production |
| Jinja2 syntax | Dùng `{{ object.field }}` — KHÔNG dùng `${object.field}` |
| Multi-company | Dùng `object.company_id.email or user.email` cho email_from |
| Opt-out | Kiểm tra `partner_id.opt_out` cho mass mailing |

---

## Manifest Setup

```python
# __manifest__.py
{
    'name': 'My Module',
    'version': '19.0.1.0.0',
    'depends': ['mail', 'base'],
    'data': [
        'security/ir.model.access.csv',
        'data/mail_templates.xml',   # Templates trước views
        'views/my_model_views.xml',
    ],
}
```

---

## Tài Liệu Tham Khảo

- [Odoo 19 Mail Mixins](https://www.odoo.com/documentation/19.0/developer/reference/backend/mixins.html)
- [Mail Templates Reference](https://www.odoo.com/documentation/19.0/developer/reference/backend/mixins.html#mail-templates)

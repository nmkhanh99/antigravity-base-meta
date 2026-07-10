---
name: odoo-email-automation
description: Hướng dẫn tạo email templates, notifications, và automated email workflows trong Odoo 19. Use when the user asks to send email, create email template, setup notifications, or automate email in Odoo 19.
---

# Odoo 19 Email Automation

## Goal
Giúp agent tạo email templates, automated notifications, và email workflows trong Odoo 19.

## When to use this skill
- "gửi email", "email template", "send email"
- "notification", "mail template"
- "automated email", "email workflow"

## Instructions

### 1. Email Template (XML)
```xml
<record id="email_template_confirm" model="mail.template">
    <field name="name">My Model: Confirmation</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="subject">Confirmation - {{ object.name }}</field>
    <field name="email_from">{{ (object.company_id.email or user.email) }}</field>
    <field name="email_to">{{ object.partner_id.email }}</field>
    <field name="body_html" type="html">
        <div style="margin: 0; padding: 0;">
            <p>Dear {{ object.partner_id.name }},</p>
            <p>Your order <strong>{{ object.name }}</strong> has been confirmed.</p>
            <p>Total: {{ format_amount(object.amount_total, object.currency_id) }}</p>
        </div>
    </field>
</record>
```

### 2. Send Email from Python
```python
def action_send_email(self):
    template = self.env.ref('my_module.email_template_confirm')
    for record in self:
        template.send_mail(record.id, force_send=True)
```

### 3. Post Message (Chatter)
```python
def action_confirm(self):
    self.write({'state': 'confirmed'})
    self.message_post(
        body="Order confirmed",
        message_type='notification',
        subtype_xmlid='mail.mt_note',
    )
```

### 4. Activity Scheduling
```python
self.activity_schedule(
    'mail.mail_activity_data_todo',
    date_deadline=fields.Date.today(),
    summary='Review this record',
    user_id=self.user_id.id,
)
```

### 5. Mail Thread Integration
```python
class MyModel(models.Model):
    _inherit = ['mail.thread', 'mail.activity.mixin']

    state = fields.Selection([...], tracking=True)  # Auto-track changes
    partner_id = fields.Many2one('res.partner', tracking=True)
```

## Constraints
- Email templates dùng Jinja2 syntax: `{{ object.field }}`.
- KHÔNG hardcode email addresses.
- Luôn kế thừa `mail.thread` nếu model cần chatter/tracking.

## Best practices
- Dùng `tracking=True` trên fields quan trọng để auto-log changes.
- Dùng `force_send=True` cho emails cần gửi ngay.
- Đọc `resources/reference.md` cho report attachment, mass mailing patterns.

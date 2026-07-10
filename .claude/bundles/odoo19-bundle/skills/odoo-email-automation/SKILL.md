---
name: odoo-email-automation
description: Hướng dẫn tạo email templates, notifications, và automated email workflows trong Odoo 19. Use when the user asks to send email, create email template, setup notifications, or automate email in Odoo 19.
---

# Odoo 19 Email Automation

## Goal
Giúp agent tạo email templates, automated notifications, và email workflows trong Odoo 19 theo đúng chuẩn mail.thread và mail.template.

## When to use this skill
- "gửi email", "email template", "send email"
- "notification", "mail template"
- "automated email", "email workflow"
- "chatter", "message_post", "activity"
- "tracking field", "mail.thread"

## Instructions

### 1. Kế thừa Mail Thread cho Model
Trước khi dùng email/chatter, model phải kế thừa `mail.thread` và `mail.activity.mixin`:

```python
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['mail.thread', 'mail.activity.mixin']

    state = fields.Selection([
        ('draft', 'Draft'),
        ('confirmed', 'Confirmed'),
    ], tracking=True)  # tracking=True → tự log changes vào chatter
    partner_id = fields.Many2one('res.partner', tracking=True)
```

### 2. Email Template (XML)
Tạo file `data/mail_templates.xml`, dùng Jinja2 `{{ object.field }}` (Odoo 14+):

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
            <p>Your record <strong>{{ object.name }}</strong> has been confirmed.</p>
        </div>
    </field>
</record>
```

> Xem GUIDE.md §Email Templates cho template nâng cao (attachment, loop order lines).

### 3. Gửi Email từ Python
```python
def action_send_email(self):
    template = self.env.ref('my_module.email_template_confirm')
    for record in self:
        template.send_mail(record.id, force_send=True)
```

### 4. Post Message vào Chatter
```python
def action_confirm(self):
    self.write({'state': 'confirmed'})
    self.message_post(
        body="Order confirmed",
        message_type='notification',
        subtype_xmlid='mail.mt_note',
    )
```

### 5. Lên Lịch Activity
```python
self.activity_schedule(
    'mail.mail_activity_data_todo',
    date_deadline=fields.Date.today(),
    summary='Review this record',
    user_id=self.user_id.id,
)
```

### 6. Gửi Bulk Email (Queue)
Dùng `force_send=False` để đẩy vào queue thay vì gửi ngay:
```python
for record in records:
    template.send_mail(record.id, force_send=False)
self.env['mail.mail'].sudo().process_email_queue()
```

> Xem GUIDE.md §Email Queue và §Scheduled Email cho patterns nâng cao.

## Constraints
- Dùng Jinja2 syntax `{{ object.field }}` trong templates — KHÔNG dùng `${object.field}` (Mako, deprecated từ Odoo 14).
- KHÔNG hardcode email addresses trong template.
- Luôn kế thừa `mail.thread` nếu model cần chatter/tracking.
- Dùng `tracking=True` trên fields quan trọng để auto-log changes.
- `force_send=True` chỉ dùng khi cần gửi ngay (transactional); bulk dùng queue.
- Thêm `data/mail_templates.xml` vào `__manifest__.py` key `data`.

## References
- https://www.odoo.com/documentation/19.0/developer/reference/backend/mixins.html
- https://www.odoo.com/documentation/19.0/developer/reference/backend/mixins.html#mail-templates

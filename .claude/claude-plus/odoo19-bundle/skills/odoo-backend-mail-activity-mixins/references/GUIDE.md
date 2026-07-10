# Odoo Backend Mail & Activity Mixins — Reference Guide

## Model Pattern

```python
from odoo import fields, models
from odoo.tools import Markup


class MyModel(models.Model):
    _name = "my.model"
    _description = "My Model"
    _inherit = ["mail.thread", "mail.activity.mixin"]

    name = fields.Char(required=True, tracking=True)
    state = fields.Selection(
        [("draft", "Draft"), ("confirmed", "Confirmed"), ("done", "Done")],
        default="draft",
        tracking=True,
    )
    partner_id = fields.Many2one("res.partner", tracking=True)

    def _track_subtype(self, initial_values):
        """Map field changes to specific mail subtypes."""
        self.ensure_one()
        if "state" in initial_values and self.state == "done":
            return self.env.ref("my_module.mt_my_model_done")
        return super()._track_subtype(initial_values)
```

## View Pattern

```xml
<record id="view_my_model_form" model="ir.ui.view">
    <field name="name">my.model.form</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <form string="My Model">
            <header>
                <button name="action_confirm" string="Confirm" type="object"
                        invisible="state != 'draft'"/>
                <field name="state" widget="statusbar"/>
            </header>
            <sheet>
                <group>
                    <field name="name"/>
                    <field name="partner_id"/>
                </group>
            </sheet>
            <chatter/>
        </form>
    </field>
</record>
```

> Odoo 19: dùng `<chatter/>` thay cho `<div class="oe_chatter">` (deprecated).

## message_post() — Odoo 19

**Tất cả arguments phải là keyword-only (breaking change Odoo 19).**

```python
from odoo.tools import Markup

# HTML body — dùng Markup() để tránh auto-escape
record.message_post(
    body=Markup("<p>Hello <b>world</b></p>"),
    subject="Update",
    partner_ids=[partner.id],
    subtype_xmlid="mail.mt_comment",
)

# Plain text body — str bị auto-escape, an toàn
record.message_post(
    body="Simple text message",
    subtype_xmlid="mail.mt_note",
)

# Odoo 19 experimental: gửi email đến địa chỉ ngoài partner records
record.message_post(
    body=Markup("<p>Notification</p>"),
    outgoing_email_to="external@example.com,another@example.com",
)
```

### message_type hợp lệ trong message_post()
| message_type | Mô tả |
|---|---|
| `comment` | Comment hiển thị trong chatter |
| `email` | Email gửi/nhận qua mail gateway |
| `email_outgoing` | Email đang gửi |
| `sms` | SMS |

> `user_notification` bị CẤM trong `message_post()` — dùng `message_notify()` thay thế.

## message_notify() — Gửi thông báo không gắn record

```python
# Dùng khi không có record ID, hoặc cần user_notification type
self.env["mail.thread"].message_notify(
    partner_ids=[partner.id],
    body=Markup("<p>You have a new task</p>"),
    subject="New Task Notification",
    record_name="My Record",
    model="my.model",
    res_id=record.id,
)
```

## Followers — Subscribe / Unsubscribe

```python
# Subscribe partner với subtype cụ thể
record.message_subscribe(
    partner_ids=[partner.id],
    subtype_ids=[self.env.ref("mail.mt_comment").id],
)

# Unsubscribe partner
record.message_unsubscribe(partner_ids=[partner.id])

# Kiểm tra followers hiện tại
followers = record.message_follower_ids
partner_followers = record.message_partner_ids
```

## Schedule Activity

```python
# Schedule activity cơ bản
record.activity_schedule(
    "mail.mail_activity_data_todo",
    date_deadline=fields.Date.today(),
    summary="Review needed",
    note="Please review this record.",
)

# Schedule với user cụ thể
record.activity_schedule(
    "mail.mail_activity_data_email",
    date_deadline=fields.Date.context_today(self) + relativedelta(days=3),
    summary="Send follow-up email",
    user_id=responsible_user.id,
)

# Activity types phổ biến (xmlid)
# mail.mail_activity_data_todo       → To-Do
# mail.mail_activity_data_call       → Phone Call
# mail.mail_activity_data_email      → Email
# mail.mail_activity_data_meeting    → Meeting
# mail.mail_activity_data_warning    → Warning (icon alert)
```

## Custom Message Subtype

```xml
<!-- data/mail_data.xml -->
<record id="mt_my_model_done" model="mail.message.subtype">
    <field name="name">Done</field>
    <field name="res_model">my.model</field>
    <field name="default" eval="True"/>
    <field name="description">My model marked as done</field>
</record>
```

```python
# Trong model: map state change → subtype
def _track_subtype(self, initial_values):
    self.ensure_one()
    if "state" in initial_values and self.state == "done":
        return self.env.ref("my_module.mt_my_model_done")
    return super()._track_subtype(initial_values)
```

## Context Keys — Mail Behavior

| Context key | Hiệu lực |
|---|---|
| `mail_create_nosubscribe` | Không auto-subscribe creator khi tạo record |
| `mail_create_nolog` | Không log message "Document created" tự động |
| `mail_notrack` | Vô hiệu hóa field tracking tại create/write |
| `tracking_disable` | Vô hiệu hóa TOÀN BỘ MailThread features (tracking, subscribe, post, ...) |
| `mail_notify_author_mention` | Notify tác giả nếu xuất hiện trong partner_ids (Odoo 19 mới) |

```python
# Ví dụ: tạo record không log, không subscribe creator
record = self.env["my.model"].with_context(
    mail_create_nosubscribe=True,
    mail_create_nolog=True,
).create(vals)
```

## _mail_post_access — Cho phép user read-only post message

```python
class MyModel(models.Model):
    _name = "my.model"
    _inherit = ["mail.thread", "mail.activity.mixin"]
    _mail_post_access = "read"  # mặc định là 'write'
```

## Website Published Mixin

```python
class MyPage(models.Model):
    _name = "my.page"
    _inherit = ["website.published.multi.mixin", "website.seo.metadata"]
    _description = "My Page"

    name = fields.Char(required=True)
    website_url = fields.Char(compute="_compute_website_url")

    def _compute_website_url(self):
        for record in self:
            record.website_url = f"/my-page/{record.id}"

    def _compute_can_publish(self):
        """Override để custom logic ai được publish."""
        for record in self:
            record.can_publish = self.env.user.has_group("base.group_system")
```

> `website.published.multi.mixin` kế thừa cả `website.published.mixin` và `website.multi.mixin` — dùng khi site có multi-website.

## FRD Checklist {#frd-checklist}

| Hạng mục | Bắt buộc mô tả trong FRD |
|---|---|
| Mixins | `mail.thread`, `mail.activity.mixin` — lý do dùng từng mixin |
| Tracking fields | Field nào `tracking=True`, điều kiện ghi log, subtype liên quan |
| Activities | Activity type (xmlid), actor phụ trách, deadline, trigger event, điều kiện complete |
| Notifications | Người nhận (partner_ids / follower), subtype, channel (email / internal note) |
| Context flags | Context key nào cần dùng và khi nào |
| Security | Ai được xem chatter, ai được post message, record rules ảnh hưởng |
| Tests | Test tracking message, test tạo/hoàn thành activity, test access rights chatter |

## activity_ids — Fields trên mail.activity.mixin

| Field | Mô tả |
|---|---|
| `activity_ids` | One2many đến `mail.activity` |
| `activity_state` | `overdue` / `today` / `planned` — tính từ deadline sớm nhất |
| `activity_exception_decoration` | `warning` hoặc `danger` nếu có activity loại exception |
| `activity_summary` | Summary của activity gần nhất |
| `activity_type_id` | Type của activity gần nhất |

## Domain Composition — Odoo 19

```python
# Odoo 19: dùng Domain class thay vì list manipulation thủ công
from odoo.fields import Domain

domain = Domain.AND([
    [("state", "=", "done")],
    [("partner_id", "!=", False)],
])

domain_or = Domain.OR([
    [("state", "=", "draft")],
    [("state", "=", "confirmed")],
])
```

## Access Check — Odoo 19

```python
# Odoo 19: dùng check_access() thay cho check_access_rights() + check_access_rule()
record.check_access("write")   # thay cho check_access_rights + check_access_rule riêng lẻ
```

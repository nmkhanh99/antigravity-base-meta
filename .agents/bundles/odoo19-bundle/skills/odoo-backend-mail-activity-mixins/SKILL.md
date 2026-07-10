---
name: odoo-backend-mail-activity-mixins
description: Hướng dẫn mail/thread/activity mixins trong Odoo 19 - mail.thread, mail.activity.mixin, chatter, field tracking, message subtypes, notifications và scheduled activities. Use when FRD/code needs chatter, activity tracking, notifications, or mail integration on business models.
---

# Odoo Backend Mail & Activity Mixins

## Goal
Giúp agent đặc tả/tạo chatter, tracking và activities đúng Odoo 19 cho business model.

## Source-first khi viết FRD
Trước khi thêm chatter/activity vào FRD, dùng `odoo-backend-source-analysis` để kiểm tra model đã `_inherit` `mail.thread`/`mail.activity.mixin` chưa, field nào đã `tracking=True`, và view đã có `<chatter/>` chưa.

## Khi nào dùng
- Model cần chatter, followers, message log.
- Field cần `tracking=True`.
- User cần schedule/complete activities.
- Custom notification/subtype/mail behavior.

## Model Pattern
```python
from odoo import fields, models


class MyModel(models.Model):
    _name = "my.model"
    _description = "My Model"
    _inherit = ["mail.thread", "mail.activity.mixin"]

    name = fields.Char(required=True, tracking=True)
    state = fields.Selection(
        [("draft", "Draft"), ("done", "Done")],
        default="draft",
        tracking=True,
    )
```

## View Pattern
```xml
<form string="My Model">
    <sheet>
        <group>
            <field name="name"/>
            <field name="state"/>
        </group>
    </sheet>
    <chatter/>
</form>
```

## FRD Checklist
| Hạng mục | Bắt buộc mô tả |
|----------|----------------|
| Mixins | `mail.thread`, `mail.activity.mixin`, lý do dùng |
| Tracking fields | Field nào `tracking=True`, khi nào ghi log |
| Activities | Activity type, actor, deadline, trigger, completion |
| Notifications | Người nhận, subtype, email/internal note |
| Security | Ai được xem chatter/activity, record rules liên quan |
| Tests | Tracking message, activity creation/done, access rights |

## Notes
- Activities là record `mail.activity`, hiển thị phía trên message history trong chatter.
- Nếu chỉ cần log thay đổi field, `mail.thread` + `tracking=True` là đủ.
- Nếu cần to-do/reminder gắn record, thêm `mail.activity.mixin`.

## References
- Odoo 19 Mixins and Useful Classes: https://www.odoo.com/documentation/19.0/developer/reference/backend/mixins.html

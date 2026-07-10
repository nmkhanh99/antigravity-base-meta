---
name: odoo-wizards
description: Hướng dẫn tạo Wizards (TransientModel) trong Odoo 19 - popup dialogs, batch operations, multi-step wizards. Use when the user asks to create wizard, build popup, batch process, or implement TransientModel in Odoo 19.
---

# Odoo 19 Wizards

## Goal
Giúp agent tạo wizards (TransientModel) cho user interactions, batch operations, confirmation dialogs và multi-step workflows trong Odoo 19.

## When to use this skill
- "tạo wizard", "create wizard", "popup", "dialog"
- "TransientModel", "batch operation"
- "confirmation dialog", "multi-step wizard"
- "action popup", "selection wizard"
- "xử lý hàng loạt", "xác nhận hành động"

## Instructions

### Bước 1 — Tạo Wizard Model (TransientModel)

Tất cả wizards phải kế thừa `models.TransientModel`. Xem mẫu cơ bản và nâng cao trong `references/GUIDE.md#transientmodel-basics`.

Các pattern phổ biến:
- **Basic wizard**: thu thập input, xử lý, đóng popup
- **Wizard với `default_get`**: load giá trị từ record đang mở qua `active_id`/`active_ids` trong context
- **Wizard có wizard line**: dùng One2many với một TransientModel con

### Bước 2 — Tạo Form View với footer

Wizard view là form view thông thường. Footer bắt buộc phải có:
- Button action chính với `class="btn-primary"`
- Button Cancel với `special="cancel"` (không cần Python method)

Xem các mẫu view trong `references/GUIDE.md#wizard-views`.

### Bước 3 — Tạo Action để mở Wizard

```xml
<record id="action_my_wizard" model="ir.actions.act_window">
    <field name="name">My Wizard</field>
    <field name="res_model">my.module.wizard</field>
    <field name="view_mode">form</field>
    <field name="target">new</field>
</record>
```

Mở wizard từ Python (truyền context):
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

### Bước 4 — Return Action sau khi xử lý

| Mục đích | Return value |
|----------|-------------|
| Đóng popup | `{'type': 'ir.actions.act_window_close'}` |
| Mở record mới | `{'type': 'ir.actions.act_window', 'res_model': ..., 'res_id': ..., 'view_mode': 'form', 'target': 'current'}` |
| Reload trang | `{'type': 'ir.actions.client', 'tag': 'reload'}` |
| Hiện notification | `{'type': 'ir.actions.client', 'tag': 'display_notification', 'params': {...}}` |

### Bước 5 — Multi-Step Wizard (nếu cần)

Dùng `state` Selection field để điều hướng giữa các bước. Button Next/Previous trả về action mở lại cùng wizard record (`res_id=self.id`). Xem mẫu đầy đủ trong `references/GUIDE.md#multi-step-wizards`.

### Bước 6 — Batch Operations (nếu cần)

Lấy danh sách records từ `self.env.context.get('active_ids', [])`. Xem pattern trong `references/GUIDE.md#batch-operations`.

## Constraints
- Wizard models **bắt buộc** dùng `TransientModel` — không dùng `Model`.
- Wizard data tự động bị xóa sau khoảng thời gian ngắn (transient).
- `special="cancel"` cho nút Cancel — không cần Python method.
- Luôn dùng `ensure_one()` trong action methods.
- Dùng `target='new'` để mở popup, `target='current'` để navigate full page.
- Phải có `ir.model.access.csv` entry cho mọi TransientModel.
- Dùng `invisible=` (Odoo 17+) thay vì `attrs=` — `attrs` đã bị deprecated.
- Trong form view, dùng `<list>` thay vì `<tree>` cho editable lines (Odoo 17+).

## References
- https://www.odoo.com/documentation/19.0/developer/reference/backend/orm.html#transient-models
- https://www.odoo.com/documentation/19.0/developer/tutorials/backend.html#wizards
- references/GUIDE.md — Code examples đầy đủ

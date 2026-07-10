---
name: odoo-backend-mail-activity-mixins
description: Hướng dẫn mail/thread/activity mixins trong Odoo 19 - mail.thread, mail.activity.mixin, chatter, field tracking, message subtypes, notifications và scheduled activities. Use when FRD/code needs chatter, activity tracking, notifications, or mail integration on business models.
---

# Odoo Backend Mail & Activity Mixins

## Goal
Giúp agent đặc tả và tạo chatter, field tracking, scheduled activities và notifications đúng chuẩn Odoo 19 cho business model, bao gồm kiểm tra code hiện tại trước khi thêm vào FRD.

## When to use this skill
- FRD/model cần chatter, followers, message log trên record.
- Field cần `tracking=True` để ghi lại thay đổi.
- User cần schedule/complete activities (to-do, reminder, call, email...).
- Cần custom notification, message subtype, hoặc email behavior.
- Cần tích hợp mail gateway (reply-by-email, bounce detection).
- Cần publish mixin (website.published.mixin) cho record có webpage.

## Instructions

### Bước 1 — Kiểm tra source trước khi viết FRD
Dùng skill `odoo-backend-source-analysis` để kiểm tra:
- Model đã `_inherit` `mail.thread` / `mail.activity.mixin` chưa.
- Field nào đã có `tracking=True`.
- View đã có `<chatter/>` chưa.
- Tránh duplicate mixin hoặc tracking.

### Bước 2 — Chọn mixin phù hợp
| Nhu cầu | Mixin cần thêm |
|---------|----------------|
| Chỉ log thay đổi field | `mail.thread` |
| Chatter + followers + messaging | `mail.thread` |
| To-do / reminder / activity | `mail.thread` + `mail.activity.mixin` |
| Website publish toggle | `website.published.mixin` hoặc `website.published.multi.mixin` |
| Website SEO | `website.seo.metadata` |

### Bước 3 — Implement model
Xem chi tiết tại `references/GUIDE.md#model-pattern`.

Quy tắc bắt buộc:
- `_inherit` dùng list: `["mail.thread", "mail.activity.mixin"]`.
- Thêm `tracking=True` vào field cần theo dõi.
- Override `_track_subtype(initial_values)` nếu cần map field changes sang subtype cụ thể.

### Bước 4 — Thêm chatter vào view
```xml
<form>
    <sheet>...</sheet>
    <chatter/>
</form>
```
Odoo 19 dùng `<chatter/>` (không phải `<div class="oe_chatter">` của phiên bản cũ).

### Bước 5 — message_post() (Odoo 19 breaking change)
`message_post()` trong Odoo 19 **bắt buộc keyword-only arguments** (có `*` separator).
- `body` là `str` sẽ bị auto-escape; dùng `Markup()` nếu cần HTML.
- `message_type='user_notification'` bị cấm trong `message_post()` — dùng `message_notify()` thay thế.
- Tham số mới Odoo 19: `outgoing_email_to` (experimental) — comma-separated emails ngoài partner records.

### Bước 6 — Schedule activity
```python
record.activity_schedule(
    'mail.mail_activity_data_todo',
    date_deadline=fields.Date.today(),
    summary='Review needed',
    note='Please review this record.',
)
```
`activity_schedule()` tạo record `mail.activity` với `automated=True` mặc định.

### Bước 7 — Điền FRD Checklist
Xem bảng checklist tại `references/GUIDE.md#frd-checklist`.

## Constraints
- **KHÔNG** dùng `<div class="oe_chatter">` — deprecated, thay bằng `<chatter/>`.
- **KHÔNG** truyền positional args cho `message_post()` — Odoo 19 yêu cầu keyword-only.
- **KHÔNG** dùng `message_type='user_notification'` trong `message_post()` — dùng `message_notify()`.
- **KHÔNG** post trên `mail.thread` trực tiếp hoặc record chưa có ID — dùng `message_notify()`.
- **KHÔNG** dùng `check_access_rights()` / `check_access_rule()` riêng lẻ — Odoo 19 dùng `check_access()`.
- **KHÔNG** dùng list manipulation thủ công cho domain — dùng `Domain.AND()` / `Domain.OR()`.
- Context key `tracking_disable` sẽ vô hiệu hóa TOÀN BỘ MailThread features, không chỉ tracking.
- Context key `mail_notrack` chỉ vô hiệu hóa field tracking tại create/write.
- Context key `mail_create_nosubscribe` ngăn auto-subscribe creator khi tạo record.

## References
- Odoo 19 Mixins and Useful Classes: https://www.odoo.com/documentation/19.0/developer/reference/backend/mixins.html
- Chi tiết code examples: `references/GUIDE.md`

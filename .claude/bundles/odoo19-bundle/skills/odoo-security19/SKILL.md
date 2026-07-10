---
name: odoo-security19
description: Thiết lập security Odoo 19 đúng chuẩn — groups, ACL, record rules, multi-company, field visibility, SQL injection prevention. Kích hoạt khi user nói "thiết lập security", "tạo groups", "access rights", "record rules", "multi-company security", "phân quyền", "ir.model.access".
---

# Odoo Security Guide (v19)

## Goal
Tạo security đầy đủ cho module Odoo 19 — security groups, ACL, record rules, multi-company, view security — đúng chuẩn v19.

**Input**: Tên model, yêu cầu phân quyền  
**Output**: security.xml, ir.model.access.csv, model security patterns

## When to use this skill
- "tạo security groups", "tạo phân quyền"
- "tạo record rules multi-company"
- "field visibility theo group"
- "SQL injection prevention"
- Review security module trước deploy

## Instructions

### Bước 1 — Security Groups (security.xml)
Tạo module category, user group, manager group, multi-company record rule và optional own-record rule.
Xem template đầy đủ: `references/GUIDE.md#security-groups-xml`

Key points:
- Manager group dùng `implied_ids` để kế thừa User group
- Multi-company rule dùng `allowed_company_ids` (v18+, không phải `company_ids`)
- Global rules áp dụng cho tất cả groups

### Bước 2 — Access Rights (ir.model.access.csv)
Format: `id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink`
Xem template: `references/GUIDE.md#acl-csv`

Quy tắc đặt `model_id`: `model_` + tên model thay `.` bằng `_`
- `my.module.my.model` → `model_my_module_my_model`

### Bước 3 — Model Security Patterns (v19)
Model chuẩn với `_check_company_auto`, type annotations, và manual permission check.
Xem template: `references/GUIDE.md#model-security-pattern`

Key points:
- `_check_company_auto = True` bắt buộc cho multi-company
- `check_company=True` trên relational fields (partner_id, user_id)
- Field nhạy cảm dùng `groups=` attribute
- Manual check: `check_access_rights()` + `check_access_rule()`

### Bước 4 — SQL Security (v19)
SQL() builder bắt buộc trong v19 để tránh SQL injection.
Xem examples: `references/GUIDE.md#sql-security`

### Bước 5 — View Security (v17+ syntax)
Dùng `invisible=` expressions và `groups=` attributes trên fields/buttons.
Xem examples: `references/GUIDE.md#view-security`

### Bước 6 — Audit Log Pattern
Immutable audit log model: write() và unlink() raise UserError.
Xem template: `references/GUIDE.md#audit-log`

### Bước 7 — Security Checklist (v19)
```
□ TẤT CẢ models có ir.model.access.csv entries
□ _check_company_auto = True cho multi-company models
□ check_company=True trên relational fields
□ allowed_company_ids trong record rules (không phải company_ids)
□ Type annotations trên tất cả fields và method signatures
□ SQL() builder cho tất cả raw SQL
□ Views dùng invisible= (không phải attrs=)
□ KHÔNG dùng sudo() bừa bãi
□ Field nhạy cảm có groups= attribute
```

## Constraints
- **KHÔNG** dùng raw string SQL — luôn dùng `SQL()` builder
- **KHÔNG** bỏ qua ACL cho model mới
- **KHÔNG** dùng `company_ids` trong record rules (v18+: `allowed_company_ids`)
- Luôn test với `sudo()` để xác nhận là security issue khi có AccessError

## Best practices
- Thứ tự phân quyền: Users (read/write/create) → Managers (+ unlink)
- Record rules nên có domain rõ ràng, tránh quá phức tạp
- Kiểm tra `has_group()` trước khi thực hiện sensitive operations
- Audit log là immutable — không cho phép write/unlink
- Dùng `check_access_rights()` + `check_access_rule()` khi cần kiểm tra thủ công

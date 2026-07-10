---
name: odoo-security
description: Hướng dẫn thiết lập bảo mật Odoo 19 - ACL, Record Rules, Groups, Field-level security, Portal access. Use when the user asks to add security, create groups, define access rights, record rules, or protect fields/methods in Odoo 19.
---

# Odoo 19 Security

## Goal
Giúp agent cấu hình đầy đủ security layers trong Odoo 19: ACL (ir.model.access.csv), Groups (dùng `res.groups.privilege`), Record Rules, field-level và method-level security theo đúng API Odoo 19.

## When to use this skill
- "thêm security", "tạo group", "phân quyền"
- "access rights", "record rules", "ACL"
- "ir.model.access.csv", "ir.rule"
- "portal access", "multi-company rule"
- "field security", "sudo", "check_access"

## Instructions

### Bước 1 — Tạo Groups (security/security.xml)

Odoo 19 dùng `res.groups.privilege` thay cho `ir.module.category`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <record id="privilege_my_module" model="res.groups.privilege">
        <field name="name">My Module Access</field>
    </record>

    <record id="group_my_module_user" model="res.groups">
        <field name="name">User</field>
        <field name="privilege_id" ref="privilege_my_module"/>
        <field name="implied_ids" eval="[(4, ref('base.group_user'))]"/>
    </record>

    <record id="group_my_module_manager" model="res.groups">
        <field name="name">Manager</field>
        <field name="privilege_id" ref="privilege_my_module"/>
        <field name="implied_ids" eval="[(4, ref('group_my_module_user'))]"/>
        <field name="users" eval="[(4, ref('base.user_root')), (4, ref('base.user_admin'))]"/>
    </record>
</odoo>
```

### Bước 2 — ACL (security/ir.model.access.csv)

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_my_model_user,my.model.user,model_my_model,my_module.group_my_module_user,1,1,1,0
access_my_model_manager,my.model.manager,model_my_model,my_module.group_my_module_manager,1,1,1,1
access_my_model_portal,my.model.portal,model_my_model,base.group_portal,1,0,0,0
```

### Bước 3 — Record Rules

```xml
<!-- User chỉ thấy record của mình -->
<record id="my_model_user_rule" model="ir.rule">
    <field name="name">My Model: User sees own records</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="domain_force">[('user_id', '=', user.id)]</field>
    <field name="groups" eval="[(4, ref('group_my_module_user'))]"/>
</record>

<!-- Multi-company global rule -->
<record id="my_model_company_rule" model="ir.rule">
    <field name="name">My Model: Multi-Company</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="domain_force">['|', ('company_id', '=', False), ('company_id', 'in', company_ids)]</field>
    <field name="global" eval="True"/>
</record>
```

### Bước 4 — Method-level Security (Odoo 19 API)

```python
from odoo.exceptions import AccessError

class MyModel(models.Model):
    _name = 'my.model'

    def action_confirm(self):
        # Odoo 19: check_access() kết hợp rights + rules (thay vì 2 lần gọi riêng)
        self.check_access('write')
        # logic...

    def _can_edit(self):
        # Odoo 19: has_access() trả về bool, không raise exception
        return self.has_access('write')

    @api.private
    def _internal_method(self):
        # Không thể gọi qua XML-RPC / JSON-RPC
        pass
```

### Bước 5 — Thứ tự trong __manifest__.py

```python
'data': [
    'security/security.xml',         # Groups trước
    'security/ir.model.access.csv',  # ACL sau
    # ... views, data
],
```

### Bước 6 — Field-level Security

```python
# Trong model
sensitive_field = fields.Char(groups='my_module.group_my_module_manager')
```

```xml
<!-- Trong view -->
<field name="internal_notes" groups="my_module.group_my_module_manager"/>
<button name="action_admin" string="Admin" groups="base.group_system"/>
```

## Constraints
- Mọi model PHẢI có ít nhất 1 dòng trong ir.model.access.csv.
- KHÔNG dùng `sudo()` mà không kiểm tra quyền trước.
- KHÔNG hardcode user ID (ví dụ `user.id == 1`).
- KHÔNG dùng `ir.module.category` cho group category — dùng `res.groups.privilege` (Odoo 19).
- KHÔNG gọi `check_access_rights()` và `check_access_rule()` riêng lẻ — dùng `check_access()` (Odoo 19 combined API).

## References
- GUIDE.md: `references/GUIDE.md` — full code examples, portal mixin, security testing patterns, checklist

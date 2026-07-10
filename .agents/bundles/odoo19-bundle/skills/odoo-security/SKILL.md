---
name: odoo-security
description: Hướng dẫn thiết lập bảo mật Odoo 19 - ACL, Record Rules, Groups, Field-level security, Portal access. Use when the user asks to add security, create groups, define access rights, record rules, or protect fields/methods in Odoo 19.
---

# Odoo 19 Security

## Goal
Giúp agent cấu hình đầy đủ security layers trong Odoo 19: ACL (ir.model.access.csv), Groups, Record Rules, field-level và method-level security.

## When to use this skill
- "thêm security", "tạo group", "phân quyền"
- "access rights", "record rules", "ACL"
- "ir.model.access.csv", "ir.rule"
- "portal access", "multi-company rule"
- "field security", "sudo", "check_access"

## Instructions

### 1. Groups (security.xml) — Odoo 19 dùng `res.groups.privilege`
```xml
<record id="privilege_my_module" model="res.groups.privilege">
    <field name="name">My Module Access</field>
</record>
<record id="group_user" model="res.groups">
    <field name="name">User</field>
    <field name="privilege_id" ref="privilege_my_module"/>
    <field name="implied_ids" eval="[(4, ref('base.group_user'))]"/>
</record>
<record id="group_manager" model="res.groups">
    <field name="name">Manager</field>
    <field name="privilege_id" ref="privilege_my_module"/>
    <field name="implied_ids" eval="[(4, ref('group_user'))]"/>
</record>
```

### 2. ACL (ir.model.access.csv)
```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_my_model_user,my.model.user,model_my_model,my_module.group_user,1,1,1,0
access_my_model_manager,my.model.manager,model_my_model,my_module.group_manager,1,1,1,1
```

### 3. Record Rules
```xml
<!-- User sees own records -->
<record id="rule_user" model="ir.rule">
    <field name="model_id" ref="model_my_model"/>
    <field name="domain_force">[('user_id','=',user.id)]</field>
    <field name="groups" eval="[(4, ref('group_user'))]"/>
</record>
<!-- Multi-company (global) -->
<record id="rule_company" model="ir.rule">
    <field name="model_id" ref="model_my_model"/>
    <field name="domain_force">['|',('company_id','=',False),('company_id','in',company_ids)]</field>
    <field name="global" eval="True"/>
</record>
```

### 4. Odoo 19 Key Changes
```python
# Combined check (replaces check_access_rights + check_access_rule)
self.check_access('write')

# Boolean check (no exception)
if self.has_access('write'):
    ...

# Prevent RPC access
@api.private
def _internal_method(self):
    pass
```

### 5. Manifest Order
```python
'data': [
    'security/security.xml',        # Groups first
    'security/ir.model.access.csv', # Then ACL
]
```

## Constraints
- Mọi model PHẢI có ir.model.access.csv.
- KHÔNG dùng sudo() mà không kiểm tra quyền trước.
- KHÔNG hardcode user IDs.

## Best practices
- Dùng `res.groups.privilege` thay `ir.module.category` cho group categories.
- Dùng `check_access()` combined API thay vì separate calls.
- Dùng `@api.private` cho internal methods.
- Đọc `resources/reference.md` cho portal access, security testing patterns.

# Security Skill - Odoo 19 Reference Guide

## 1. Security Layers Overview

Odoo có 4 layers bảo mật (áp dụng theo thứ tự):

| Layer | Cơ chế | Phạm vi |
|---|---|---|
| 1. ACL | ir.model.access.csv | Model-level CRUD |
| 2. Record Rules | ir.rule (domain) | Row-level filter |
| 3. Field Security | groups= trên field/view | Ẩn/readonly field |
| 4. Method Security | check_access(), @api.private | Python code |

---

## 2. Access Control Lists (ACL)

### 2.1 ir.model.access.csv

File: `security/ir.model.access.csv`

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_my_model_user,my.model.user,model_my_model,my_module.group_my_module_user,1,1,1,0
access_my_model_manager,my.model.manager,model_my_model,my_module.group_my_module_manager,1,1,1,1
access_my_model_portal,my.model.portal,model_my_model,base.group_portal,1,0,0,0
access_my_model_public,my.model.public,model_my_model,,1,0,0,0
```

**Columns:**
- `id`: Unique identifier (snake_case)
- `name`: Human-readable name (dấu chấm)
- `model_id:id`: `model_` + tên model thay `.` bằng `_`
- `group_id:id`: Để trống = public access
- `perm_*`: 1 = có quyền, 0 = không có quyền

### 2.2 Đăng ký trong __manifest__.py

```python
{
    'data': [
        'security/security.xml',          # Groups trước
        'security/ir.model.access.csv',   # ACL sau
    ],
}
```

---

## 3. Groups (Odoo 19)

### 3.1 Khai báo Groups với res.groups.privilege

File: `security/security.xml`

> **Odoo 19**: Dùng `res.groups.privilege` thay cho `ir.module.category` để phân nhóm các groups.

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <!-- Privilege (category) cho module -->
    <record id="privilege_my_module" model="res.groups.privilege">
        <field name="name">My Module Access</field>
    </record>

    <!-- User Group -->
    <record id="group_my_module_user" model="res.groups">
        <field name="name">User</field>
        <field name="privilege_id" ref="privilege_my_module"/>
        <field name="implied_ids" eval="[(4, ref('base.group_user'))]"/>
    </record>

    <!-- Manager Group kế thừa User -->
    <record id="group_my_module_manager" model="res.groups">
        <field name="name">Manager</field>
        <field name="privilege_id" ref="privilege_my_module"/>
        <field name="implied_ids" eval="[(4, ref('group_my_module_user'))]"/>
        <field name="users" eval="[(4, ref('base.user_root')), (4, ref('base.user_admin'))]"/>
    </record>

    <!-- Admin Group kế thừa Manager -->
    <record id="group_my_module_admin" model="res.groups">
        <field name="name">Administrator</field>
        <field name="privilege_id" ref="privilege_my_module"/>
        <field name="implied_ids" eval="[(4, ref('group_my_module_manager'))]"/>
    </record>
</odoo>
```

### 3.2 Group Hierarchy

```
base.group_user (Employee)
    └── group_my_module_user
            └── group_my_module_manager
                    └── group_my_module_admin
```

---

## 4. Record Rules

### 4.1 User-based Rule

```xml
<record id="my_model_user_rule" model="ir.rule">
    <field name="name">My Model: User sees own records</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="domain_force">[('user_id', '=', user.id)]</field>
    <field name="groups" eval="[(4, ref('my_module.group_my_module_user'))]"/>
    <field name="perm_read" eval="True"/>
    <field name="perm_write" eval="True"/>
    <field name="perm_create" eval="True"/>
    <field name="perm_unlink" eval="False"/>
</record>
```

### 4.2 Multi-Company Rule (Global)

```xml
<record id="my_model_company_rule" model="ir.rule">
    <field name="name">My Model: Multi-Company</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="domain_force">
        ['|', ('company_id', '=', False), ('company_id', 'in', company_ids)]
    </field>
    <field name="global" eval="True"/>
</record>
```

### 4.3 Manager sees all / User sees own + team

```xml
<!-- Manager rule: full visibility -->
<record id="my_model_manager_rule" model="ir.rule">
    <field name="name">My Model: Manager sees all</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="domain_force">[(1, '=', 1)]</field>
    <field name="groups" eval="[(4, ref('my_module.group_my_module_manager'))]"/>
</record>

<!-- User rule: own + team -->
<record id="my_model_user_rule" model="ir.rule">
    <field name="name">My Model: User sees own and team</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="domain_force">
        ['|', ('user_id', '=', user.id), ('team_id.member_ids', 'in', [user.id])]
    </field>
    <field name="groups" eval="[(4, ref('my_module.group_my_module_user'))]"/>
</record>
```

### 4.4 State-based Rule

```xml
<!-- Chỉ cho phép edit record ở trạng thái draft -->
<record id="my_model_draft_rule" model="ir.rule">
    <field name="name">My Model: Write draft only</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="domain_force">[('state', '=', 'draft')]</field>
    <field name="groups" eval="[(4, ref('my_module.group_my_module_user'))]"/>
    <field name="perm_read" eval="False"/>
    <field name="perm_write" eval="True"/>
    <field name="perm_create" eval="False"/>
    <field name="perm_unlink" eval="False"/>
</record>
```

### 4.5 Portal Rule

```xml
<record id="my_model_portal_rule" model="ir.rule">
    <field name="name">My Model: Portal Access</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="domain_force">[('partner_id.user_ids', 'in', [user.id])]</field>
    <field name="groups" eval="[(4, ref('base.group_portal'))]"/>
</record>
```

---

## 5. Field-level Security

### 5.1 groups= trên field (model)

```python
class MyModel(models.Model):
    _name = 'my.model'

    # Chỉ manager thấy
    internal_notes = fields.Text(
        string='Internal Notes',
        groups='my_module.group_my_module_manager'
    )

    # Nhiều groups (OR)
    sensitive_data = fields.Char(
        groups='base.group_system,my_module.group_my_module_manager'
    )
```

### 5.2 groups= trong view

```xml
<form>
    <!-- Ẩn field -->
    <field name="internal_notes" groups="my_module.group_my_module_manager"/>

    <!-- Ẩn cả group -->
    <group string="Admin Info" groups="base.group_system">
        <field name="admin_field1"/>
        <field name="admin_field2"/>
    </group>

    <!-- Ẩn button -->
    <button name="action_admin" string="Admin Action"
            groups="base.group_system"/>

    <!-- Readonly nếu KHÔNG phải manager -->
    <field name="amount" readonly="1" groups="!my_module.group_my_module_manager"/>
</form>
```

---

## 6. Method-level Security (Odoo 19 API)

### 6.1 check_access() — Combined API (Odoo 19)

```python
from odoo.exceptions import AccessError

class MyModel(models.Model):
    _name = 'my.model'

    def action_confirm(self):
        # Odoo 19: check_access() kết hợp check_access_rights() + check_access_rule()
        # Thay thế cho việc gọi riêng lẻ 2 methods cũ
        self.check_access('write')
        # logic xử lý...

    def action_delete_all(self):
        self.check_access('unlink')
        return self.unlink()
```

### 6.2 has_access() — Boolean check (Odoo 19)

```python
def _can_user_edit(self):
    # Odoo 19: has_access() trả về bool, không raise exception
    return self.has_access('write')

def get_display_data(self):
    if self.has_access('read'):
        return self._get_full_data()
    return self._get_public_data()
```

### 6.3 @api.private — Ngăn RPC access (Odoo 19)

```python
from odoo import api

class MyModel(models.Model):
    _name = 'my.model'

    @api.private
    def _compute_secret_token(self):
        """Không thể gọi qua XML-RPC / JSON-RPC"""
        pass

    @api.private
    def _sync_internal_state(self):
        pass
```

### 6.4 has_group() — Kiểm tra group membership

```python
def my_method(self):
    if not self.env.user.has_group('my_module.group_my_module_manager'):
        raise AccessError('Only managers can perform this action')

    # Admin check
    if self.env.user.has_group('base.group_system'):
        # Admin-only logic
        pass
```

### 6.5 sudo() — Dùng đúng cách

```python
# SAI: không kiểm tra quyền trước
def dangerous_method(self):
    self.sudo().unlink()  # Bất kỳ ai cũng có thể xóa!

# ĐÚNG: kiểm tra trước rồi mới sudo
def safe_method(self):
    if not self.env.user.has_group('base.group_system'):
        raise AccessError('Only admins can perform this action')
    self.sudo().unlink()

# ĐÚNG: sudo để tạo system record với quyền đặc biệt
def create_audit_log(self, message):
    """Tạo audit log bất kể quyền user hiện tại"""
    self.env['my.audit.log'].sudo().create({
        'user_id': self.env.user.id,
        'message': message,
    })
```

---

## 7. Portal Access

### 7.1 ACL + Rule cho Portal

```xml
<!-- ACL cho portal users (trong ir.model.access.csv) -->
access_my_model_portal,my.model.portal,model_my_model,base.group_portal,1,0,0,0

<!-- Record rule: chỉ thấy record của mình -->
<record id="my_model_portal_rule" model="ir.rule">
    <field name="name">My Model: Portal Access</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="domain_force">[('partner_id.user_ids', 'in', [user.id])]</field>
    <field name="groups" eval="[(4, ref('base.group_portal'))]"/>
</record>
```

### 7.2 Portal Mixin

```python
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['portal.mixin', 'mail.thread']

    def _compute_access_url(self):
        super()._compute_access_url()
        for record in self:
            record.access_url = f'/my/models/{record.id}'

    def _get_portal_return_action(self):
        self.ensure_one()
        return self.env.ref('my_module.action_portal_my_model')
```

---

## 8. Common Security Patterns

### 8.1 User-based

```python
# Model field
user_id = fields.Many2one('res.users', default=lambda self: self.env.user)
```
```xml
<!-- Record rule -->
<field name="domain_force">[('user_id', '=', user.id)]</field>
```

### 8.2 Company-based

```python
# Model field
company_id = fields.Many2one(
    'res.company',
    default=lambda self: self.env.company,
    required=True,
)
```
```xml
<!-- Record rule -->
<field name="domain_force">
    ['|', ('company_id', '=', False), ('company_id', 'in', company_ids)]
</field>
```

### 8.3 Team-based

```python
team_id = fields.Many2one('crm.team')
```
```xml
<field name="domain_force">
    ['|', ('user_id', '=', user.id), ('team_id.member_ids', 'in', [user.id])]
</field>
```

---

## 9. Security Testing

### 9.1 Test ACL và Record Rules

```python
from odoo.tests import TransactionCase
from odoo.exceptions import AccessError

class TestMyModelSecurity(TransactionCase):

    def setUp(self):
        super().setUp()
        self.user = self.env['res.users'].create({
            'name': 'Test User',
            'login': 'test_user_security',
            'email': 'test@example.com',
            'groups_id': [(6, 0, [
                self.env.ref('my_module.group_my_module_user').id
            ])]
        })
        self.manager = self.env['res.users'].create({
            'name': 'Test Manager',
            'login': 'test_manager_security',
            'email': 'manager@example.com',
            'groups_id': [(6, 0, [
                self.env.ref('my_module.group_my_module_manager').id
            ])]
        })

    def test_user_cannot_delete(self):
        """User thường không được xóa record"""
        record = self.env['my.model'].create({'name': 'Test'})
        with self.assertRaises(AccessError):
            record.with_user(self.user).unlink()

    def test_manager_can_delete(self):
        """Manager có thể xóa record"""
        record = self.env['my.model'].create({'name': 'Test'})
        record.with_user(self.manager).unlink()  # Không raise

    def test_user_sees_only_own_records(self):
        """User chỉ thấy record của mình"""
        own_record = self.env['my.model'].with_user(self.user).create({
            'name': 'Own Record',
            'user_id': self.user.id,
        })
        other_record = self.env['my.model'].create({
            'name': 'Other Record',
            'user_id': self.manager.id,
        })

        visible = self.env['my.model'].with_user(self.user).search([])
        self.assertIn(own_record, visible)
        self.assertNotIn(other_record, visible)
```

---

## 10. Security Checklist

### Trước khi deploy

- [ ] Tất cả models có ir.model.access.csv
- [ ] Record rules cho multi-user scenarios
- [ ] Sensitive fields có groups= attribute
- [ ] Portal access đúng nếu cần
- [ ] Multi-company rules nếu applicable
- [ ] Method-level checks cho critical operations
- [ ] Viết và pass security tests
- [ ] Không sudo() mà không check quyền trước
- [ ] Không hardcode user IDs
- [ ] Audit log cho sensitive operations

### Common Vulnerabilities

| Lỗi | Hậu quả | Fix |
|---|---|---|
| Thiếu ACL | Model accessible với mọi người | Thêm ir.model.access.csv |
| Thiếu Record Rules | Users thấy tất cả records | Thêm ir.rule với domain phù hợp |
| sudo() không check | Ai cũng bypass security | Check has_group() trước sudo() |
| Hardcode user ID | Brittle, security risk | Dùng has_group() |
| Thiếu field-level security | Sensitive data exposed | groups= trên field + view |

---

## 11. API Changes Summary (Odoo 19 vs trước)

| Cũ (Odoo ≤ 17) | Mới (Odoo 19) | Ghi chú |
|---|---|---|
| `ir.module.category` | `res.groups.privilege` | Group categories |
| `check_access_rights('write')` + `check_access_rule('write')` | `check_access('write')` | Combined call |
| Không có | `has_access('write')` | Returns bool, no exception |
| Không có | `@api.private` | Block RPC access |

---

**Status**: Complete — Odoo 19 Compatible
**Last Updated**: 2026-06-15

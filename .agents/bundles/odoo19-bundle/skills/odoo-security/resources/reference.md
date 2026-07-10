# Security Skill - Odoo 19

## 1. Security Layers Overview

Odoo có 4 layers bảo mật:

1. **Access Control Lists (ACL)** - Model-level permissions (CRUD)
2. **Record Rules** - Row-level security (domain filters)
3. **Field-level Security** - Hide/readonly specific fields
4. **Method-level Security** - Python code security checks

---

## 2. Access Control Lists (ACL)

### 2.1 ir.model.access.csv

File: `security/ir.model.access.csv`

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_my_model_user,my.model.user,model_my_model,base.group_user,1,1,1,0
access_my_model_manager,my.model.manager,model_my_model,base.group_system,1,1,1,1
access_my_model_public,my.model.public,model_my_model,,1,0,0,0
```

**Columns:**
- `id`: Unique identifier
- `name`: Human-readable name
- `model_id:id`: Model reference (`model_` + model name with underscores)
- `group_id:id`: Group reference (empty = public access)
- `perm_read`: Read permission (1 = yes, 0 = no)
- `perm_write`: Write permission
- `perm_create`: Create permission
- `perm_unlink`: Delete permission

### 2.2 Common Patterns

```csv
# User can read/write, not create/delete
access_my_model_user,my.model.user,model_my_model,base.group_user,1,1,0,0

# Manager has full access
access_my_model_manager,my.model.manager,model_my_model,my_module.group_manager,1,1,1,1

# Public read-only
access_my_model_public,my.model.public,model_my_model,,1,0,0,0

# Portal user limited access
access_my_model_portal,my.model.portal,model_my_model,base.group_portal,1,0,0,0
```

### 2.3 Register in __manifest__.py

```python
{
    'data': [
        'security/security.xml',  # Groups first
        'security/ir.model.access.csv',  # Then ACL
    ],
}
```

---

## 3. Groups

### 3.1 Define Groups

File: `security/security.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <!-- Odoo 19: Use res.groups.privilege for group categories -->
    <record id="privilege_my_module" model="res.groups.privilege">
        <field name="name">My Module Access</field>
    </record>

    <!-- User Group -->
    <record id="group_my_module_user" model="res.groups">
        <field name="name">User</field>
        <field name="privilege_id" ref="privilege_my_module"/>
        <field name="implied_ids" eval="[(4, ref('base.group_user'))]"/>
    </record>

    <!-- Manager Group -->
    <record id="group_my_module_manager" model="res.groups">
        <field name="name">Manager</field>
        <field name="privilege_id" ref="privilege_my_module"/>
        <field name="implied_ids" eval="[(4, ref('group_my_module_user'))]"/>
        <field name="users" eval="[(4, ref('base.user_root')), (4, ref('base.user_admin'))]"/>
    </record>
</odoo>
```

> **Odoo 19 Note**: Use `res.groups.privilege` model instead of `ir.module.category` for custom group categories.
```

### 3.2 Group Hierarchy

```xml
<!-- Base group -->
<record id="group_user" model="res.groups">
    <field name="name">User</field>
</record>

<!-- Manager inherits User permissions -->
<record id="group_manager" model="res.groups">
    <field name="name">Manager</field>
    <field name="implied_ids" eval="[(4, ref('group_user'))]"/>
</record>

<!-- Admin inherits Manager permissions -->
<record id="group_admin" model="res.groups">
    <field name="name">Administrator</field>
    <field name="implied_ids" eval="[(4, ref('group_manager'))]"/>
</record>
```

---

## 4. Record Rules

### 4.1 Basic Record Rule

File: `security/security.xml`

```xml
<record id="my_model_user_rule" model="ir.rule">
    <field name="name">My Model: User can see own records</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="domain_force">[('user_id', '=', user.id)]</field>
    <field name="groups" eval="[(4, ref('base.group_user'))]"/>
    <field name="perm_read" eval="True"/>
    <field name="perm_write" eval="True"/>
    <field name="perm_create" eval="True"/>
    <field name="perm_unlink" eval="False"/>
</record>
```

### 4.2 Multi-Company Rule

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

### 4.3 Complex Domain Rules

```xml
<!-- Manager can see all, User can see own + team -->
<record id="my_model_manager_rule" model="ir.rule">
    <field name="name">My Model: Manager sees all</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="domain_force">[(1, '=', 1)]</field>
    <field name="groups" eval="[(4, ref('group_manager'))]"/>
</record>

<record id="my_model_user_rule" model="ir.rule">
    <field name="name">My Model: User sees own and team</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="domain_force">
        ['|', ('user_id', '=', user.id), ('team_id.member_ids', 'in', [user.id])]
    </field>
    <field name="groups" eval="[(4, ref('base.group_user'))]"/>
</record>
```

### 4.4 Global vs Group Rules

```xml
<!-- Global rule (applies to everyone) -->
<record id="my_model_global_rule" model="ir.rule">
    <field name="name">My Model: Global Rule</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="domain_force">[('active', '=', True)]</field>
    <field name="global" eval="True"/>
</record>

<!-- Group-specific rule -->
<record id="my_model_portal_rule" model="ir.rule">
    <field name="name">My Model: Portal users see published only</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="domain_force">[('is_published', '=', True)]</field>
    <field name="groups" eval="[(4, ref('base.group_portal'))]"/>
</record>
```

---

## 5. Field-level Security

### 5.1 Groups Attribute in Fields

```python
class MyModel(models.Model):
    _name = 'my.model'
    
    # Only managers can see this field
    internal_notes = fields.Text(
        string='Internal Notes',
        groups='my_module.group_manager'
    )
    
    # Multiple groups (OR condition)
    sensitive_data = fields.Char(
        groups='base.group_system,my_module.group_manager'
    )
```

### 5.2 Groups in Views

```xml
<form>
    <!-- Hide field for non-managers -->
    <field name="internal_notes" groups="my_module.group_manager"/>
    
    <!-- Hide entire group -->
    <group string="Admin Info" groups="base.group_system">
        <field name="admin_field1"/>
        <field name="admin_field2"/>
    </group>
    
    <!-- Hide button -->
    <button name="action_admin" string="Admin Action" 
            groups="base.group_system"/>
</form>
```

### 5.3 Readonly based on Groups

```xml
<field name="amount" 
       readonly="1" 
       groups="!my_module.group_manager"/>
```

---

## 6. Method-level Security

### 6.1 Check Access Rights

```python
from odoo.exceptions import AccessError

class MyModel(models.Model):
    _name = 'my.model'
    
    def my_custom_method(self):
        # Odoo 19: Combined rights + rules check
        self.check_access('write')
        
        # Legacy (still works): separate checks
        # self.check_access_rights('write')
        # self.check_access_rule('write')
        
        # Your logic here
        pass
    
    def delete_records(self):
        # Check delete permission
        try:
            self.check_access('unlink')  # Odoo 19: combined API
        except AccessError:
            raise AccessError('You do not have permission to delete records')
        
        return self.unlink()
    
    def _can_user_edit(self):
        """Check if user can edit without raising"""
        # Odoo 19: has_access returns bool instead of raising
        return self.has_access('write')
```

### 6.2 Check Group Membership

```python
def my_method(self):
    # Check if user is in group
    if not self.env.user.has_group('my_module.group_manager'):
        raise AccessError('Only managers can perform this action')
    
    # Alternative
    if self.env.user.has_group('base.group_system'):
        # Admin-only logic
        pass
```

### 6.3 Sudo (Bypass Security)

```python
# Use with caution!
def create_system_record(self):
    # This bypasses all security checks
    record = self.sudo().create({
        'name': 'System Record',
    })
    return record

# Better: Check permission first
def create_system_record(self):
    if not self.env.user.has_group('base.group_system'):
        raise AccessError('Only admins can create system records')
    
    record = self.sudo().create({
        'name': 'System Record',
    })
    return record
```

---

## 7. Portal Access

### 7.1 Portal User Access

```xml
<!-- ACL for portal users -->
<record id="access_my_model_portal" model="ir.model.access">
    <field name="name">my.model.portal</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="group_id" ref="base.group_portal"/>
    <field name="perm_read" eval="True"/>
    <field name="perm_write" eval="False"/>
    <field name="perm_create" eval="False"/>
    <field name="perm_unlink" eval="False"/>
</record>

<!-- Record rule: Portal users see only their records -->
<record id="my_model_portal_rule" model="ir.rule">
    <field name="name">My Model: Portal Access</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="domain_force">[('partner_id.user_ids', 'in', [user.id])]</field>
    <field name="groups" eval="[(4, ref('base.group_portal'))]"/>
</record>
```

### 7.2 Portal Mixin

```python
from odoo.addons.portal.controllers.portal import CustomerPortal, pager

class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['portal.mixin']
    
    def _compute_access_url(self):
        super()._compute_access_url()
        for record in self:
            record.access_url = f'/my/models/{record.id}'
```

---

## 8. Common Security Patterns

### 8.1 User-based Access

```python
# Model
user_id = fields.Many2one('res.users', default=lambda self: self.env.user)

# Record Rule
<field name="domain_force">[('user_id', '=', user.id)]</field>
```

### 8.2 Company-based Access

```python
# Model
company_id = fields.Many2one('res.company', default=lambda self: self.env.company)

# Record Rule
<field name="domain_force">
    ['|', ('company_id', '=', False), ('company_id', 'in', company_ids)]
</field>
```

### 8.3 Team-based Access

```python
# Model
team_id = fields.Many2one('crm.team')

# Record Rule
<field name="domain_force">
    ['|', ('user_id', '=', user.id), ('team_id.member_ids', 'in', [user.id])]
</field>
```

### 8.4 State-based Access

```xml
<!-- Users can only edit draft records -->
<record id="my_model_draft_rule" model="ir.rule">
    <field name="name">My Model: Edit draft only</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="domain_force">[('state', '=', 'draft')]</field>
    <field name="groups" eval="[(4, ref('base.group_user'))]"/>
    <field name="perm_write" eval="True"/>
    <field name="perm_read" eval="False"/>
</record>
```

---

## 9. Security Testing

### 9.1 Test Access Rights

```python
from odoo.tests import TransactionCase
from odoo.exceptions import AccessError

class TestMyModelSecurity(TransactionCase):
    
    def setUp(self):
        super().setUp()
        self.user = self.env['res.users'].create({
            'name': 'Test User',
            'login': 'test_user',
            'groups_id': [(6, 0, [self.env.ref('base.group_user').id])]
        })
    
    def test_user_cannot_delete(self):
        """Test that regular users cannot delete records"""
        record = self.env['my.model'].create({'name': 'Test'})
        
        with self.assertRaises(AccessError):
            record.with_user(self.user).unlink()
    
    def test_user_can_read_own_records(self):
        """Test that users can read their own records"""
        record = self.env['my.model'].with_user(self.user).create({
            'name': 'Test',
            'user_id': self.user.id
        })
        
        # Should not raise error
        record.with_user(self.user).read(['name'])
```

---

## 10. Security Checklist

### Before Deployment ✅

- [ ] All models have ACL defined
- [ ] Record rules implemented for multi-user scenarios
- [ ] Sensitive fields protected with groups
- [ ] Portal access properly restricted
- [ ] Multi-company rules if applicable
- [ ] Method-level checks for critical operations
- [ ] Security tests written and passing
- [ ] No sudo() without proper checks
- [ ] No hardcoded user IDs
- [ ] Audit log for sensitive operations

### Common Vulnerabilities ❌

- Missing ACL (model accessible to everyone)
- Missing record rules (users see all records)
- Sudo without permission check
- Hardcoded admin bypass
- Missing field-level security
- No audit trail for critical operations

---

## 11. Best Practices

### DO ✅

```python
# Check permissions before sensitive operations (Odoo 19: combined API)
def delete_all(self):
    self.check_access('unlink')  # Combined rights + rules check
    return self.unlink()

# Odoo 19: Use @api.private to prevent RPC access
@api.private
def _internal_method(self):
    """Cannot be called via XML-RPC/JSON-RPC"""
    pass

# Use groups for field visibility
sensitive_field = fields.Char(groups='base.group_system')

# Implement proper record rules
<field name="domain_force">[('user_id', '=', user.id)]</field>

# Test security
def test_security(self):
    with self.assertRaises(AccessError):
        record.with_user(normal_user).unlink()
```

### DON'T ❌

```python
# Don't use sudo without checks
def dangerous_method(self):
    self.sudo().unlink()  # ❌ Anyone can delete!

# Don't hardcode user checks
if self.env.user.id == 1:  # ❌ Hardcoded admin ID
    pass

# Don't forget ACL
# ❌ No ir.model.access.csv for model

# Don't expose sensitive data
# ❌ No groups on sensitive fields
```

---

**Status**: ✅ Complete  
**Last Updated**: 23/02/2026

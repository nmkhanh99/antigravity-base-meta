# Odoo 19 Data Setup — Reference Guide

## Cấu trúc file chuẩn

```
my_module/
├── __manifest__.py          # security/security.xml phải đứng trước views
├── security/
│   ├── security.xml         # Groups + Record Rules
│   └── ir.model.access.csv  # ACL
└── data/
    ├── ir_sequence_data.xml # Sequences (noupdate="1")
    └── data.xml             # Default config records (noupdate="1")
```

Thứ tự trong `__manifest__.py`:
```python
'data': [
    'security/security.xml',
    'security/ir.model.access.csv',
    'data/ir_sequence_data.xml',
    'data/data.xml',
    'views/my_model_views.xml',
    'views/menu_views.xml',
],
```

---

## Security Groups & Privileges

> **Odoo 19**: Dùng `res.groups.privilege` thay vì `ir.module.category` (deprecated).

```xml
<!-- security/security.xml -->
<odoo>
    <data noupdate="1">

        <!-- Privilege (thay ir.module.category) -->
        <record id="privilege_my_module" model="res.groups.privilege">
            <field name="name">My Module</field>
        </record>

        <!-- Group: User -->
        <record id="group_my_module_user" model="res.groups">
            <field name="name">User</field>
            <field name="privilege_id" ref="privilege_my_module"/>
            <field name="implied_ids" eval="[(4, ref('base.group_user'))]"/>
        </record>

        <!-- Group: Manager -->
        <record id="group_my_module_manager" model="res.groups">
            <field name="name">Manager</field>
            <field name="privilege_id" ref="privilege_my_module"/>
            <field name="implied_ids" eval="[(4, ref('group_my_module_user'))]"/>
            <field name="users" eval="[(4, ref('base.user_admin'))]"/>
        </record>

    </data>
</odoo>
```

---

## Record Rules

```xml
<!-- Trong security/security.xml, cùng <data noupdate="1"> -->
<record id="rule_my_model_company" model="ir.rule">
    <field name="name">My Model: Multi-Company</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="domain_force">[('company_id', 'in', company_ids)]</field>
    <field name="groups" eval="[(4, ref('group_my_module_user'))]"/>
</record>

<!-- Rule chỉ cho phép xem record của chính mình -->
<record id="rule_my_model_own" model="ir.rule">
    <field name="name">My Model: Own Records</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="domain_force">[('user_id', '=', user.id)]</field>
    <field name="groups" eval="[(4, ref('group_my_module_user'))]"/>
    <field name="perm_read" eval="True"/>
    <field name="perm_write" eval="True"/>
    <field name="perm_create" eval="True"/>
    <field name="perm_unlink" eval="False"/>
</record>
```

**Lưu ý `model_id` ref**: dùng `model_<tên_model_dấu_chấm_thành_gạch_dưới>`.
Ví dụ: model `stock.picking` → ref `model_stock_picking`.

---

## Sequences

```xml
<!-- data/ir_sequence_data.xml -->
<odoo>
    <data noupdate="1">

        <!-- Sequence dùng chung (không gắn company) -->
        <record id="seq_my_model" model="ir.sequence">
            <field name="name">My Model</field>
            <field name="code">my.model</field>
            <field name="prefix">MY/%(year)s/%(month)s/</field>
            <field name="padding">5</field>
            <field name="company_id" eval="False"/>
        </record>

        <!-- Sequence gắn theo company -->
        <record id="seq_my_model_company" model="ir.sequence">
            <field name="name">My Model (Company)</field>
            <field name="code">my.model.company</field>
            <field name="prefix">MY-%(year)s-</field>
            <field name="padding">4</field>
        </record>

    </data>
</odoo>
```

Dùng sequence trong model:
```python
# models/my_model.py
name = fields.Char(default=lambda self: _('New'), required=True)

@api.model_create_multi
def create(self, vals_list):
    for vals in vals_list:
        if vals.get('name', _('New')) == _('New'):
            vals['name'] = self.env['ir.sequence'].next_by_code('my.model') or _('New')
    return super().create(vals_list)
```

---

## Menus & Window Actions

```xml
<!-- views/menu_views.xml -->
<odoo>

    <!-- Window Action -->
    <record id="action_my_model" model="ir.actions.act_window">
        <field name="name">My Models</field>
        <field name="res_model">my.model</field>
        <field name="view_mode">list,form,kanban</field>
        <field name="context">{}</field>
    </record>

    <!-- Action cho config (chỉ manager) -->
    <record id="action_my_config" model="ir.actions.act_window">
        <field name="name">Configuration</field>
        <field name="res_model">my.config</field>
        <field name="view_mode">list,form</field>
    </record>

    <!-- Menu Root -->
    <menuitem id="menu_my_module_root"
              name="My Module"
              sequence="50"/>

    <!-- Menu chính -->
    <menuitem id="menu_my_model"
              name="My Models"
              parent="menu_my_module_root"
              action="action_my_model"
              sequence="10"/>

    <!-- Menu Configuration (chỉ manager) -->
    <menuitem id="menu_my_module_config"
              name="Configuration"
              parent="menu_my_module_root"
              sequence="90"
              groups="group_my_module_manager"/>

    <menuitem id="menu_my_config"
              name="Settings"
              parent="menu_my_module_config"
              action="action_my_config"
              sequence="10"/>

</odoo>
```

**Quy tắc sequence menu**: 10 (tính năng chính), 90 (cấu hình/admin).

---

## ACL (ir.model.access.csv)

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_my_model_user,my.model.user,model_my_model,my_module.group_my_module_user,1,1,1,0
access_my_model_manager,my.model.manager,model_my_model,my_module.group_my_module_manager,1,1,1,1
access_my_config_manager,my.config.manager,model_my_config,my_module.group_my_module_manager,1,1,1,1
```

Cột `model_id:id`: `model_<model_name>` (dấu chấm thành gạch dưới).
Cột `group_id:id`: `<module_name>.<group_xml_id>`.

---

## Default Configuration Data

```xml
<!-- data/data.xml -->
<odoo>
    <data noupdate="1">

        <record id="config_default" model="my.config">
            <field name="name">Default Config</field>
            <field name="active" eval="True"/>
            <field name="company_id" ref="base.main_company"/>
        </record>

    </data>
</odoo>
```

---

## i18n (Đa ngôn ngữ)

**Nguyên tắc**: Trường `name` trong XML dùng tiếng Anh. Dịch qua `i18n/vi.po`.

```po
# i18n/vi.po
#. module: my_module
#: model:res.groups,name:my_module.group_my_module_user
msgid "User"
msgstr "Người dùng"

#: model:res.groups,name:my_module.group_my_module_manager
msgid "Manager"
msgstr "Quản lý"

#: model:ir.sequence,name:my_module.seq_my_model
msgid "My Model"
msgstr "Mô hình của tôi"

#: model:ir.actions.act_window,name:my_module.action_my_model
msgid "My Models"
msgstr "Danh sách mô hình"
```

Cấu trúc thư mục:
```
my_module/
└── i18n/
    ├── vi.po    # Tiếng Việt
    └── en.po    # (tuỳ chọn, thường không cần)
```

---

## Checklist trước khi deploy

- [ ] `noupdate="1"` cho Groups, Sequences, Record Rules, System Parameters
- [ ] `security/security.xml` đứng trước views trong manifest
- [ ] ACL có đủ cho mọi model mới
- [ ] Trường `name` XML dùng tiếng Anh, bản dịch trong `i18n/vi.po`
- [ ] `model_id` ref dùng đúng định dạng `model_<name_with_underscores>`
- [ ] Menu `groups` dùng XML ID đầy đủ (có module prefix)
- [ ] Sequences có `company_id eval="False"` nếu dùng chung
- [ ] Dùng `res.groups.privilege` (không dùng `ir.module.category`)

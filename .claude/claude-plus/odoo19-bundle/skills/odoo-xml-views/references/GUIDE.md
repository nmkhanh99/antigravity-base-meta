# Odoo XML View Patterns — Full Code Reference

## attrs-migration

| attrs Domain | v17+ Expression |
|---|---|
| `[('field', '=', 'value')]` | `field == 'value'` |
| `[('field', '!=', 'value')]` | `field != 'value'` |
| `[('field', '=', True)]` | `field` |
| `[('field', '=', False)]` | `not field` |
| `[('field', 'in', ['a','b'])]` | `field in ('a', 'b')` |
| `['&', A, B]` | `A and B` |
| `['|', A, B]` | `A or B` |

```xml
<!-- ❌ v14-v16 REMOVED -->
<field name="partner_id" attrs="{'invisible': [('state', '=', 'draft')], 'readonly': [('state', '!=', 'draft')]}"/>

<!-- ✅ v17+ -->
<field name="partner_id" invisible="state == 'draft'" readonly="state != 'draft'"/>

<!-- Nested, context, group check -->
<field name="x" invisible="state == 'draft' or (type == 'service' and qty == 0)"/>
<field name="x" invisible="context.get('hide_field')"/>
<field name="x" invisible="not user_has_groups('my_module.group_manager')"/>
<field name="x" invisible="parent.state != 'draft'"/>  <!-- trong One2many -->
```

## form

```xml
<record id="my_model_view_form" model="ir.ui.view">
    <field name="name">my.module.my.model.form</field>
    <field name="model">my.module.my.model</field>
    <field name="arch" type="xml">
        <form string="My Model">
            <header>
                <button name="action_confirm" string="Confirm" type="object"
                        class="btn-primary" invisible="state != 'draft'"/>
                <button name="action_cancel" string="Cancel" type="object"
                        invisible="state not in ('draft', 'confirmed')"/>
                <button name="action_draft" string="Reset" type="object"
                        invisible="state != 'cancelled'"/>
                <field name="state" widget="statusbar" statusbar_visible="draft,confirmed,done"/>
            </header>
            <sheet>
                <div class="oe_button_box" name="button_box">
                    <button name="action_view_invoices" type="object"
                            class="oe_stat_button" icon="fa-pencil-square-o">
                        <field name="invoice_count" widget="statinfo" string="Invoices"/>
                    </button>
                </div>
                <widget name="web_ribbon" title="Archived" bg_color="bg-danger" invisible="active"/>
                <div class="oe_title"><h1><field name="name" placeholder="Name..."/></h1></div>
                <group>
                    <group string="General">
                        <field name="partner_id"/>
                        <field name="date"/>
                        <field name="user_id"/>
                    </group>
                    <group string="Details">
                        <field name="company_id" groups="base.group_multi_company"/>
                        <field name="currency_id" invisible="1"/>
                        <field name="amount"/>
                    </group>
                </group>
                <notebook>
                    <page string="Lines" name="lines">
                        <field name="line_ids">
                            <list editable="bottom">
                                <field name="sequence" widget="handle"/>
                                <field name="name"/>
                                <field name="quantity"/>
                                <field name="price_unit"/>
                                <field name="subtotal" readonly="1"/>
                            </list>
                        </field>
                    </page>
                    <page string="Notes" name="notes">
                        <field name="notes" placeholder="Internal notes..."/>
                    </page>
                </notebook>
            </sheet>
            <div class="oe_chatter">
                <field name="message_follower_ids"/>
                <field name="activity_ids"/>
                <field name="message_ids"/>
            </div>
        </form>
    </field>
</record>
```

## list

```xml
<record id="my_model_view_list" model="ir.ui.view">
    <field name="name">my.module.my.model.list</field>
    <field name="model">my.module.my.model</field>
    <field name="arch" type="xml">
        <list decoration-danger="state == 'cancelled'" decoration-warning="state == 'draft'"
              decoration-success="state == 'done'" default_order="date desc">
            <field name="sequence" widget="handle"/>
            <field name="name"/>
            <field name="partner_id"/>
            <field name="state" widget="badge"
                   decoration-success="state == 'done'" decoration-info="state == 'confirmed'"/>
            <field name="amount" sum="Total"/>
            <field name="company_id" column_invisible="True"/>
            <field name="internal_notes" optional="hide"/>
        </list>
    </field>
</record>
```

## search

> **Odoo 19**: `<group expand="0" string="Group By">` đã bị xoá. Đặt `<filter>` group-by trực tiếp trong `<search>`, không dùng `<group>` wrapper.

```xml
<record id="my_model_view_search" model="ir.ui.view">
    <field name="name">my.module.my.model.search</field>
    <field name="model">my.module.my.model</field>
    <field name="arch" type="xml">
        <search string="Search My Model">
            <field name="name"/>
            <field name="partner_id"/>
            <separator/>
            <filter name="draft" string="Draft" domain="[('state', '=', 'draft')]"/>
            <filter name="my_records" string="My Records" domain="[('user_id', '=', uid)]"/>
            <filter name="today" string="Today"
                    domain="[('date', '=', context_today().strftime('%Y-%m-%d'))]"/>
            <!-- ✅ Odoo 19: group-by filters đặt trực tiếp, không dùng <group> wrapper -->
            <filter name="group_state" string="Status" context="{'group_by': 'state'}"/>
            <filter name="group_partner" string="Partner" context="{'group_by': 'partner_id'}"/>
            <filter name="group_date" string="Date" context="{'group_by': 'date:month'}"/>
            <searchpanel>
                <field name="state" icon="fa-filter" enable_counters="1"/>
            </searchpanel>
        </search>
    </field>
</record>

<!-- ❌ Odoo 19 KHÔNG dùng -->
<!--
<group expand="0" string="Group By">
    <filter name="group_state" .../>
</group>
-->
```

## kanban

```xml
<record id="my_model_view_kanban" model="ir.ui.view">
    <field name="name">my.module.my.model.kanban</field>
    <field name="model">my.module.my.model</field>
    <field name="arch" type="xml">
        <kanban default_group_by="state">
            <field name="name"/><field name="state"/><field name="color"/>
            <templates>
                <t t-name="kanban-box">
                    <div t-attf-class="oe_kanban_card oe_kanban_global_click #{kanban_color(record.color.raw_value)}">
                        <div class="oe_kanban_content">
                            <div class="o_kanban_record_top">
                                <strong class="o_kanban_record_title"><field name="name"/></strong>
                            </div>
                            <div class="o_kanban_record_body"><field name="partner_id"/></div>
                            <div class="o_kanban_record_bottom">
                                <div class="oe_kanban_bottom_left">
                                    <field name="priority" widget="priority"/>
                                </div>
                                <div class="oe_kanban_bottom_right">
                                    <field name="user_id" widget="many2one_avatar_user"/>
                                </div>
                            </div>
                        </div>
                    </div>
                </t>
            </templates>
        </kanban>
    </field>
</record>
```

## inheritance

```xml
<!-- QUAN TRỌNG: Luôn đọc parent view trước khi viết xpath -->
<record id="view_partner_form_inherit_my_module" model="ir.ui.view">
    <field name="name">res.partner.form.inherit.my_module</field>
    <field name="model">res.partner</field>
    <field name="inherit_id" ref="base.view_partner_form"/>
    <field name="arch" type="xml">
        <xpath expr="//field[@name='email']" position="after">
            <field name="x_custom_field"/>
        </xpath>
        <xpath expr="//field[@name='name']" position="attributes">
            <attribute name="required">1</attribute>
        </xpath>
        <xpath expr="//notebook" position="inside">
            <page string="Custom" name="custom">
                <group><field name="x_custom_field"/></group>
            </page>
        </xpath>
    </field>
</record>
```

XPath positions: `after`, `before`, `replace`, `inside`, `attributes`

XPath expressions: `//field[@name='x']`, `//group[@name='x']`, `//page[@name='x']`, `//button[@name='x']`, `//notebook`, `//sheet`

## actions

```xml
<record id="my_model_action" model="ir.actions.act_window">
    <field name="name">My Models</field>
    <field name="res_model">my.module.my.model</field>
    <field name="view_mode">list,form,kanban</field>
    <field name="domain">[('active', '=', True)]</field>
    <field name="context">{'search_default_my_records': 1}</field>
    <field name="help" type="html">
        <p class="o_view_nocontent_smiling_face">Create your first record</p>
    </field>
</record>

<menuitem id="my_module_menu_root" name="My Module" sequence="10"
          web_icon="my_module,static/description/icon.png"/>
<menuitem id="my_module_menu_main" name="Main Menu" parent="my_module_menu_root" sequence="10"/>
<menuitem id="my_model_menu" name="My Models" parent="my_module_menu_main"
          action="my_model_action" sequence="10"/>
```

## delete-control

> **Odoo 17+/19**: Attribute `delete` trên `<list>` CHỈ nhận giá trị cứng `"0"`/`"1"` — **KHÔNG** support dynamic expression như `delete="state == 'draft'"` hay `delete="context.get('can_delete')"`.

**Pattern chuẩn: 2 list view riêng + chọn view trong action method**

```xml
<!-- View cho draft: có delete -->
<record id="my_model_view_list_draft" model="ir.ui.view">
    <field name="name">my.model.list.draft</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <list editable="bottom" create="0" delete="1">
            <!-- fields -->
        </list>
    </field>
</record>

<!-- View cho các state khác: không delete, readonly -->
<record id="my_model_view_list" model="ir.ui.view">
    <field name="name">my.model.list</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <list create="0" delete="0">
            <!-- fields readonly="1" -->
        </list>
    </field>
</record>
```

```python
# Action method chọn view theo state
def action_view_lines(self):
    self.ensure_one()
    if self.state == 'draft':
        list_view = self.env.ref('my_module.my_model_view_list_draft')
    else:
        list_view = self.env.ref('my_module.my_model_view_list')
    form_view = self.env.ref('my_module.my_model_view_form')
    return {
        'type': 'ir.actions.act_window',
        'res_model': 'my.model',
        'view_mode': 'list,form',
        'views': [(list_view.id, 'list'), (form_view.id, 'form')],
        'domain': [('parent_id', '=', self.id)],
        'context': {'default_parent_id': self.id},
    }
```

**Bảo vệ backend: luôn override unlink() kèm theo**
```python
def unlink(self):
    if any(rec.parent_id.state != 'draft' for rec in self):
        raise ValidationError(_('Chỉ có thể xoá khi ở trạng thái Draft.'))
    return super().unlink()
```

Tham khảo Odoo base: `point_of_sale/models/pos_order_line.py`, `hr_holidays/models/hr_leave.py`

> **Cũng áp dụng cho**: `create`, `edit` — các attribute này cũng chỉ nhận giá trị cứng.

## widgets

| Widget | Field Types | Purpose |
|---|---|---|
| `statusbar` | Selection | Status bar |
| `badge` | Selection | Colored badge |
| `priority` | Selection | Star rating |
| `many2one_avatar_user` | Many2one | User avatar |
| `many2many_tags` | Many2many | Tag chips |
| `monetary` | Float | Currency |
| `handle` | Integer | Drag sort |
| `boolean_toggle` | Boolean | Toggle |
| `progressbar` | Float/Int | Progress |
| `html` | Html | Rich text |
| `statinfo` | Integer | Stat button counter |

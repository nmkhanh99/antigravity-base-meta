# Odoo Backend Views — Guide

## view-types

### Bảng loại view hợp lệ trong Odoo 19

| View type | Root tag | Ghi chú |
|-----------|----------|---------|
| `list` | `<list>` | KHÔNG dùng `<tree>` — đã bị rename từ v17 |
| `form` | `<form>` | |
| `kanban` | `<kanban>` | Template dùng `t-name="card"` |
| `search` | `<search>` | |
| `calendar` | `<calendar>` | |
| `graph` | `<graph>` | |
| `pivot` | `<pivot>` | |
| `qweb` | QWeb template | Bắt buộc có field `key` non-null |

---

## modifiers

### VIEW_MODIFIERS hợp lệ trong Odoo 19
`column_invisible`, `invisible`, `readonly`, `required`

### Quy tắc viết modifier

```xml
<!-- VALID v19: Python expression trực tiếp -->
<field name="foo" invisible="state == 'cancel'"/>
<field name="bar" readonly="state in ('done', 'cancel')"/>
<field name="baz" required="type == 'contact' and not is_company"/>

<!-- column_invisible: ẩn toàn bộ cột trong list view -->
<field name="state" column_invisible="True"/>
<field name="company_id" column_invisible="not context.get('show_company')"/>

<!-- Field kỹ thuật không hiển thị nhưng dùng trong expression -->
<field name="state" invisible="True"/>
<field name="foo" invisible="state == 'draft'"/>

<!-- INVALID v19: attrs bị xóa — raises ValidationError -->
<!-- <field name="foo" attrs="{'invisible': [('state','=','draft')]}"/> -->

<!-- INVALID v19: states bị xóa — raises ValidationError -->
<!-- <field name="foo" states="draft"/> -->
```

---

## inheritance

### Mẫu view inherit chuẩn

```xml
<record id="view_my_model_form_inherit" model="ir.ui.view">
    <field name="name">my.model.form.inherit</field>
    <field name="model">my.model</field>
    <field name="inherit_id" ref="base_module.view_my_model_form"/>
    <field name="arch" type="xml">
        <!-- Thêm field sau field name -->
        <xpath expr="//field[@name='name']" position="after">
            <field name="x_custom_field" invisible="state != 'active'"/>
        </xpath>

        <!-- Thêm field vào group có name ổn định -->
        <xpath expr="//group[@name='main_group']" position="inside">
            <field name="x_state_reason" invisible="state != 'cancel'"/>
        </xpath>

        <!-- Sửa attribute của node -->
        <xpath expr="//form" position="attributes">
            <attribute name="create">0</attribute>
        </xpath>

        <!-- Xóa field -->
        <xpath expr="//field[@name='old_field']" position="replace"/>

        <!-- Thêm trước node -->
        <xpath expr="//button[@name='action_confirm']" position="before">
            <button name="action_custom" string="Custom" type="object"/>
        </xpath>
    </field>
</record>
```

### Các giá trị `position` hợp lệ
- `inside` — thêm vào bên trong node (default nếu không ghi)
- `replace` — thay thế node; nếu body rỗng → xóa node
- `after` — thêm ngay sau node
- `before` — thêm ngay trước node
- `attributes` — sửa attribute của node, dùng `<attribute name="...">` bên trong

---

## form-view

### Cấu trúc form view đầy đủ

```xml
<record id="view_my_model_form" model="ir.ui.view">
    <field name="name">my.model.form</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <form string="My Model">
            <!-- Smart buttons -->
            <div class="oe_button_box" name="button_box">
                <button name="action_view_orders"
                        type="object"
                        class="oe_stat_button"
                        icon="fa-shopping-cart">
                    <field name="order_count" string="Orders" widget="statinfo"/>
                </button>
            </div>

            <!-- Status bar -->
            <header>
                <button name="action_confirm" string="Confirm"
                        type="object" class="btn-primary"
                        invisible="state != 'draft'"/>
                <field name="state" widget="statusbar"
                       statusbar_visible="draft,confirm,done"/>
            </header>

            <sheet>
                <!-- Field ẩn dùng trong modifier expression -->
                <field name="state" invisible="True"/>

                <group name="main_group">
                    <group name="left_group" string="General Info">
                        <field name="name"/>
                        <field name="code"/>
                        <field name="partner_id"/>
                    </group>
                    <group name="right_group" string="Details">
                        <field name="date_start"/>
                        <field name="date_end" readonly="state == 'done'"/>
                        <field name="company_id" groups="base.group_multi_company"/>
                    </group>
                </group>

                <!-- Tabs -->
                <notebook>
                    <page name="tab_lines" string="Lines">
                        <field name="line_ids">
                            <list editable="bottom">
                                <field name="product_id"/>
                                <field name="qty"/>
                                <field name="price_unit"/>
                            </list>
                        </field>
                    </page>
                    <page name="tab_notes" string="Notes">
                        <field name="note" widget="html"/>
                    </page>
                </notebook>
            </sheet>

            <!-- Chatter -->
            <chatter/>
        </form>
    </field>
</record>
```

---

## list-view

### List view với đầy đủ tính năng Odoo 19

```xml
<record id="view_my_model_list" model="ir.ui.view">
    <field name="name">my.model.list</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <list string="My Models"
              sample="1"
              multi_edit="1"
              editable="bottom"
              decoration-muted="not active"
              decoration-danger="state == 'cancel'"
              decoration-success="state == 'done'">
            <field name="name"/>
            <field name="partner_id"/>
            <field name="date_start"/>
            <field name="state" widget="badge"
                   decoration-success="state == 'done'"
                   decoration-warning="state == 'draft'"/>
            <field name="active" column_invisible="True"/>
            <field name="company_id" optional="hide"
                   groups="base.group_multi_company"/>
            <field name="email" optional="show"/>
        </list>
    </field>
</record>
```

### Thuộc tính `optional`
- `optional="show"` — hiển thị mặc định, user có thể ẩn.
- `optional="hide"` — ẩn mặc định, user có thể bật.

### Decoration attributes trên `<list>` root
`decoration-bf`, `decoration-it`, `decoration-danger`, `decoration-info`,
`decoration-muted`, `decoration-primary`, `decoration-success`, `decoration-warning`
— nhận Python expression.

---

## kanban-view

### Kanban view với template `t-name="card"` (Odoo 19)

```xml
<record id="view_my_model_kanban" model="ir.ui.view">
    <field name="name">my.model.kanban</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <kanban sample="1" default_group_by="state">
            <field name="active"/>
            <field name="color"/>
            <templates>
                <t t-name="card" class="flex-row">
                    <widget name="web_ribbon" title="Archived"
                            bg_color="text-bg-danger" invisible="active"/>
                    <main>
                        <field name="name" class="fw-bold fs-5"/>
                        <div t-if="record.partner_id.raw_value">
                            <field name="partner_id"/>
                        </div>
                        <field name="state" widget="badge"/>
                    </main>
                    <footer>
                        <field name="user_id" widget="many2one_avatar_user"/>
                    </footer>
                </t>
            </templates>
        </kanban>
    </field>
</record>
```

---

## search-view

### Search view với filter và group by

```xml
<record id="view_my_model_search" model="ir.ui.view">
    <field name="name">my.model.search</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <search string="Search My Model">
            <!-- Search fields -->
            <field name="name"
                   filter_domain="['|', ('name', 'ilike', self), ('code', 'ilike', self)]"/>
            <field name="partner_id"/>

            <!-- Filters -->
            <filter name="draft" string="Draft" domain="[('state', '=', 'draft')]"/>
            <filter name="done" string="Done" domain="[('state', '=', 'done')]"/>
            <separator/>
            <filter name="my_records" string="My Records"
                    domain="[('user_id', '=', uid)]"/>
            <filter name="active_test" string="Archived"
                    domain="[('active', '=', False)]"/>

            <!-- Group by -->
            <group expand="0" string="Group By">
                <filter name="group_by_state" string="Status"
                        context="{'group_by': 'state'}"/>
                <filter name="group_by_partner" string="Partner"
                        context="{'group_by': 'partner_id'}"/>
                <filter name="group_by_date" string="Date"
                        context="{'group_by': 'date_start:month'}"/>
            </group>

            <!-- Search panel (nếu cần) -->
            <!-- <searchpanel view_types="list,kanban"> -->
            <!--     <field name="state" select="one"/> -->
            <!-- </searchpanel> -->
        </search>
    </field>
</record>
```

---

## calendar-view

### Calendar view

```xml
<record id="view_my_model_calendar" model="ir.ui.view">
    <field name="name">my.model.calendar</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <calendar string="My Calendar"
                  date_start="date_start"
                  date_stop="date_end"
                  color="user_id"
                  mode="month"
                  form_view_id="%(view_my_model_form)d"
                  quick_create="0">
            <field name="name"/>
            <field name="partner_id"/>
            <field name="state"/>
        </calendar>
    </field>
</record>
```

---

## graph-view

### Graph view

```xml
<record id="view_my_model_graph" model="ir.ui.view">
    <field name="name">my.model.graph</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <graph string="My Graph" type="bar" stacked="0" sample="1">
            <field name="date_start" type="col"/>
            <field name="state" type="row"/>
            <field name="amount" type="measure"/>
        </graph>
    </field>
</record>
```

Thuộc tính `type` trên field: `row`, `col`, `measure`.
Graph type: `bar`, `pie`, `line`.

---

## pivot-view

### Pivot view

```xml
<record id="view_my_model_pivot" model="ir.ui.view">
    <field name="name">my.model.pivot</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <pivot string="My Pivot" sample="1" disable_linking="0">
            <field name="state" type="row"/>
            <field name="date_start" type="col" interval="month"/>
            <field name="amount" type="measure"/>
        </pivot>
    </field>
</record>
```

---

## actions-menus

### ir.actions.act_window

```xml
<record id="action_my_model" model="ir.actions.act_window">
    <field name="name">My Models</field>
    <field name="res_model">my.model</field>
    <field name="view_mode">list,form,kanban,search</field>
    <field name="domain">[]</field>
    <field name="context">{'default_state': 'draft'}</field>
    <field name="target">current</field>
</record>

<!-- Menu item -->
<menuitem id="menu_my_model_root"
          name="My Module"
          sequence="10"/>

<menuitem id="menu_my_model"
          name="My Models"
          parent="menu_my_model_root"
          action="action_my_model"
          sequence="10"
          groups="my_module.group_my_user"/>
```

### target values
- `current` — mở trong cùng tab (default).
- `new` — mở popup dialog.
- `fullscreen` — mở toàn màn hình.
- `main` — mở và reset breadcrumb.

---

## ir-ui-view-fields

### Các field quan trọng của ir.ui.view

| Field | Type | Ghi chú |
|-------|------|---------|
| `name` | Char | Bắt buộc |
| `model` | Char | Tên model (dotted) |
| `type` | Selection | list/form/graph/pivot/calendar/kanban/search/qweb |
| `arch` | Text (computed) | Đọc arch từ đây |
| `arch_db` | Text | Stored blob, dùng để write |
| `inherit_id` | Many2one ir.ui.view | View cha được inherit |
| `mode` | Selection | primary / extension (auto từ inherit_id) |
| `priority` | Integer | default=16; thấp hơn = ưu tiên cao hơn |
| `active` | Boolean | default=True |
| `group_ids` | Many2many res.groups | Chỉ dùng trên primary view |
| `key` | Char | Bắt buộc cho QWeb view |

### Quy tắc mode
- `mode='primary'`: base view, có thể resolve độc lập.
- `mode='extension'`: extend view cha gần nhất. Tự động set khi `inherit_id` được ghi.
- Extension view KHÔNG được có `group_ids` — dùng `groups` attribute trong arch.

---

## common-patterns

### Button trong form view

```xml
<!-- Button server action -->
<button name="action_confirm" string="Confirm"
        type="object" class="btn-primary"
        invisible="state != 'draft'"
        confirm="Bạn chắc chắn muốn xác nhận?"/>

<!-- Button mở act_window -->
<button name="%(action_my_model)d" string="View Models"
        type="action"/>

<!-- Button URL -->
<button name="open_url" string="Open" type="object" icon="fa-external-link"/>
```

### Field with widget

```xml
<field name="state" widget="statusbar" statusbar_visible="draft,confirm,done"/>
<field name="state" widget="badge"
       decoration-success="state == 'done'"
       decoration-warning="state == 'in_progress'"
       decoration-danger="state == 'cancel'"/>
<field name="partner_id" widget="many2one_avatar"/>
<field name="user_id" widget="many2one_avatar_user"/>
<field name="image" widget="image" class="oe_avatar"/>
<field name="note" widget="html"/>
<field name="amount" widget="monetary" options="{'currency_field': 'currency_id'}"/>
```

### Domain và Context trong view

```xml
<!-- Domain lọc records trong field relational -->
<field name="product_id" domain="[('type', '=', 'product'), ('sale_ok', '=', True)]"/>

<!-- Context truyền default value và ngữ cảnh -->
<field name="order_ids" context="{'default_partner_id': id}"/>

<!-- Domain với uid -->
<filter name="my_tasks" string="My Tasks" domain="[('user_id', '=', uid)]"/>
```

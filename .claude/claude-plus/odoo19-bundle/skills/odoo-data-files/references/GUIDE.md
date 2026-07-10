# Data Files Skill - Odoo 19 Reference Guide

## 1. Data Files Overview

Data files trong Odoo dùng để:
- Load master data (sequences, categories, default records)
- Configure system (email templates, cron jobs, server actions)
- Provide demo data for testing

---

## 2. Basic Data File Structure

### 2.1 Simple Data File

File: `data/data.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <data noupdate="0">
        <!-- Records here -->
        <record id="my_record_1" model="my.model">
            <field name="name">Record 1</field>
            <field name="description">Description</field>
            <field name="active" eval="True"/>
        </record>
    </data>
</odoo>
```

### 2.2 noupdate Flag

```xml
<!-- noupdate="0": Update on module upgrade (default for views) -->
<odoo>
    <data noupdate="0">
        <record id="view_form" model="ir.ui.view">
            <!-- Will be updated on upgrade -->
        </record>
    </data>
</odoo>

<!-- noupdate="1": Don't update on module upgrade (for production data) -->
<odoo>
    <data noupdate="1">
        <record id="default_category" model="product.category">
            <field name="name">Default Category</field>
            <!-- Won't be updated on upgrade -->
        </record>
    </data>
</odoo>
```

---

## 3. Field Types in XML

### 3.1 Simple Fields

```xml
<record id="my_record" model="my.model">
    <!-- Char/Text -->
    <field name="name">My Name</field>
    <field name="description">Long description here</field>

    <!-- Integer/Float -->
    <field name="sequence">10</field>
    <field name="price" eval="99.99"/>

    <!-- Boolean -->
    <field name="active" eval="True"/>
    <field name="is_company" eval="False"/>

    <!-- Date/Datetime -->
    <field name="date" eval="time.strftime('%Y-%m-%d')"/>
    <field name="datetime_field" eval="datetime.now()"/>

    <!-- Selection -->
    <field name="state">draft</field>
</record>
```

### 3.2 Relational Fields

```xml
<record id="my_record" model="my.model">
    <!-- Many2one (reference by XML ID) -->
    <field name="partner_id" ref="base.res_partner_1"/>

    <!-- Many2one (search) -->
    <field name="partner_id" search="[('name', '=', 'Admin')]"/>

    <!-- Many2one (eval) -->
    <field name="company_id" eval="obj().env.ref('base.main_company').id"/>

    <!-- Many2many (replace all) -->
    <field name="tag_ids" eval="[(6, 0, [ref('tag_1'), ref('tag_2')])]"/>

    <!-- Many2many (add) -->
    <field name="group_ids" eval="[(4, ref('base.group_user'))]"/>
</record>
```

### 3.3 Eval Expressions

```xml
<record id="my_record" model="my.model">
    <!-- Python expression -->
    <field name="date" eval="(datetime.now() + timedelta(days=7)).strftime('%Y-%m-%d')"/>

    <!-- Reference to other record -->
    <field name="user_id" eval="ref('base.user_admin')"/>

    <!-- Get current company -->
    <field name="company_id" eval="obj().env.company.id"/>

    <!-- List/Tuple for M2M -->
    <field name="tag_ids" eval="[(6, 0, [ref('tag_1'), ref('tag_2')])]"/>
</record>
```

---

## 4. Sequences

### 4.1 Define Sequence

File: `data/sequence.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <data noupdate="1">
        <record id="seq_my_model" model="ir.sequence">
            <field name="name">My Model Sequence</field>
            <field name="code">my.model</field>
            <field name="prefix">MM/%(year)s/</field>
            <field name="padding">5</field>
            <field name="number_next">1</field>
            <field name="number_increment">1</field>
            <field name="implementation">standard</field>
        </record>
    </data>
</odoo>
```

**Result**: MM/2026/00001, MM/2026/00002, ...

**Prefix variables available**:
- `%(year)s` — 4-digit year (e.g. 2026)
- `%(month)s` — 2-digit month (e.g. 06)
- `%(day)s` — 2-digit day (e.g. 15)
- `%(y)s` — 2-digit year (e.g. 26)

### 4.2 Use Sequence in Model (Odoo 19)

```python
class MyModel(models.Model):
    _name = 'my.model'

    name = fields.Char(readonly=True, copy=False, default='/')

    @api.model_create_multi
    def create(self, vals_list):
        for vals in vals_list:
            if vals.get('name', '/') == '/':
                vals['name'] = self.env['ir.sequence'].next_by_code('my.model')
        return super().create(vals_list)
```

> Odoo 19: Dùng `@api.model_create_multi` thay vì `@api.model` + override `create` đơn lẻ.

---

## 5. Master Data

### 5.1 Product Categories (Hierarchy)

```xml
<odoo>
    <data noupdate="1">
        <record id="category_parent" model="product.category">
            <field name="name">Electronics</field>
            <field name="parent_id" ref="product.product_category_all"/>
        </record>

        <record id="category_child_1" model="product.category">
            <field name="name">Computers</field>
            <field name="parent_id" ref="category_parent"/>
        </record>

        <record id="category_child_2" model="product.category">
            <field name="name">Laptops</field>
            <field name="parent_id" ref="category_child_1"/>
        </record>
    </data>
</odoo>
```

### 5.2 Partners

```xml
<odoo>
    <data noupdate="1">
        <record id="partner_demo_1" model="res.partner">
            <field name="name">Demo Customer</field>
            <field name="email">demo@example.com</field>
            <field name="phone">+1234567890</field>
            <field name="is_company" eval="True"/>
            <field name="customer_rank">1</field>
        </record>
    </data>
</odoo>
```

---

## 6. Configuration Data

### 6.1 System Parameters

```xml
<odoo>
    <data noupdate="1">
        <record id="param_default_discount" model="ir.config_parameter">
            <field name="key">my_module.default_discount</field>
            <field name="value">10.0</field>
        </record>
    </data>
</odoo>
```

**Usage in code:**
```python
discount = float(self.env['ir.config_parameter'].sudo().get_param(
    'my_module.default_discount', default='0.0'
))
```

### 6.2 Cron Jobs (Scheduled Actions)

File: `data/cron.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <data noupdate="1">
        <record id="ir_cron_my_task" model="ir.cron">
            <field name="name">My Module: Scheduled Task</field>
            <field name="model_id" ref="model_my_model"/>
            <field name="state">code</field>
            <field name="code">model.my_scheduled_method()</field>
            <field name="interval_number">1</field>
            <field name="interval_type">days</field>
            <field name="numbercall">-1</field>
            <field name="active" eval="True"/>
        </record>
    </data>
</odoo>
```

**interval_type values**: `minutes`, `hours`, `days`, `weeks`, `months`

**Method in model:**
```python
def my_scheduled_method(self):
    records = self.search([('state', '=', 'pending')])
    records.write({'state': 'done'})
```

---

## 7. Email Templates

### 7.1 Basic Email Template

File: `data/mail_template.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <data noupdate="1">
        <record id="email_template_order_confirmation" model="mail.template">
            <field name="name">Order Confirmation</field>
            <field name="model_id" ref="sale.model_sale_order"/>
            <field name="subject">Order ${object.name} Confirmed</field>
            <field name="email_from">${object.company_id.email}</field>
            <field name="email_to">${object.partner_id.email}</field>
            <field name="body_html" type="html">
<![CDATA[
<div style="font-family: Arial, sans-serif;">
    <h2>Hello ${object.partner_id.name},</h2>
    <p>Your order <strong>${object.name}</strong> has been confirmed.</p>
    <p>Total Amount: ${object.amount_total} ${object.currency_id.symbol}</p>
    <p>Thank you for your business!</p>
</div>
]]>
            </field>
        </record>
    </data>
</odoo>
```

### 7.2 Send Email from Code

```python
template = self.env.ref('my_module.email_template_order_confirmation')
template.send_mail(self.id, force_send=True)
```

---

## 8. Server Actions

### 8.1 Python Code Action

File: `data/server_action.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <data noupdate="1">
        <record id="action_server_confirm_orders" model="ir.actions.server">
            <field name="name">Confirm Selected Orders</field>
            <field name="model_id" ref="sale.model_sale_order"/>
            <field name="binding_model_id" ref="sale.model_sale_order"/>
            <field name="state">code</field>
            <field name="code">
for record in records:
    if record.state == 'draft':
        record.action_confirm()
            </field>
        </record>
    </data>
</odoo>
```

> `binding_model_id` gắn action vào Action menu của list/form view cho model đó.

### 8.2 Create Record Action

```xml
<record id="action_server_create_task" model="ir.actions.server">
    <field name="name">Create Task from Order</field>
    <field name="model_id" ref="sale.model_sale_order"/>
    <field name="state">object_create</field>
    <field name="crud_model_id" ref="project.model_project_task"/>
</record>
```

---

## 9. Demo Data

### 9.1 Demo Data File

File: `demo/demo.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <data noupdate="1">
        <!-- Demo products -->
        <record id="product_demo_1" model="product.product">
            <field name="name">Demo Product 1</field>
            <field name="list_price">100.0</field>
            <field name="type">consu</field>
        </record>

        <record id="product_demo_2" model="product.product">
            <field name="name">Demo Product 2</field>
            <field name="list_price">200.0</field>
            <field name="type">service</field>
        </record>

        <!-- Demo orders -->
        <record id="order_demo_1" model="sale.order">
            <field name="partner_id" ref="base.res_partner_1"/>
            <field name="date_order" eval="datetime.now()"/>
        </record>

        <record id="order_line_demo_1" model="sale.order.line">
            <field name="order_id" ref="order_demo_1"/>
            <field name="product_id" ref="product_demo_1"/>
            <field name="product_uom_qty">5</field>
        </record>
    </data>
</odoo>
```

### 9.2 Register Demo Data in Manifest

```python
# __manifest__.py
{
    'demo': [
        'demo/demo.xml',
    ],
}
```

---

## 10. CSV Data Files

### 10.1 CSV Format (ACL only)

File: `security/ir.model.access.csv`

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_my_model_user,my.model user,model_my_model,base.group_user,1,1,1,0
access_my_model_manager,my.model manager,model_my_model,base.group_system,1,1,1,1
```

### 10.2 CSV for Simple Master Data

File: `data/product_categories.csv`

```csv
id,name,parent_id:id
category_electronics,Electronics,
category_computers,Computers,category_electronics
category_laptops,Laptops,category_computers
```

```python
# __manifest__.py
{
    'data': [
        'data/product_categories.csv',
    ],
}
```

---

## 11. Function Fields in XML

### 11.1 Using ref()

```xml
<!-- Reference by XML ID -->
<field name="partner_id" ref="base.res_partner_1"/>

<!-- Reference in eval -->
<field name="user_id" eval="ref('base.user_admin')"/>
```

### 11.2 Using obj()

```xml
<!-- Access environment -->
<field name="company_id" eval="obj().env.company.id"/>

<!-- Access current user -->
<field name="user_id" eval="obj().env.user.id"/>
```

### 11.3 Using time/datetime

```xml
<!-- Current date -->
<field name="date" eval="time.strftime('%Y-%m-%d')"/>

<!-- Date arithmetic -->
<field name="deadline" eval="(datetime.now() + timedelta(days=30)).strftime('%Y-%m-%d')"/>
```

---

## 12. Many2many Operations

| Command | Syntax | Mô tả |
|---------|--------|-------|
| Replace all | `(6, 0, [ids])` | Xóa hết, set mới |
| Add | `(4, id)` | Thêm 1 record |
| Remove | `(3, id)` | Bỏ liên kết 1 record |
| Clear all | `(5,)` | Xóa hết liên kết |
| Unlink+delete | `(2, id)` | Xóa record khỏi DB |

```xml
<!-- Replace all -->
<field name="tag_ids" eval="[(6, 0, [ref('tag_1'), ref('tag_2')])]"/>

<!-- Add -->
<field name="group_ids" eval="[(4, ref('base.group_user'))]"/>

<!-- Remove -->
<field name="tag_ids" eval="[(3, ref('tag_old'))]"/>

<!-- Clear all -->
<field name="tag_ids" eval="[(5,)]"/>
```

---

## 13. __manifest__.py — Data Loading Order

```python
{
    'name': 'My Module',
    'version': '19.0.1.0.0',
    'data': [
        # 1. Security first (groups, ACL)
        'security/security.xml',
        'security/ir.model.access.csv',
        # 2. Data files
        'data/sequence.xml',
        'data/data.xml',
        'data/cron.xml',
        'data/mail_template.xml',
        'data/server_action.xml',
        # 3. Views last
        'views/my_model_views.xml',
        'views/menu.xml',
    ],
    'demo': [
        'demo/demo.xml',
    ],
}
```

---

## 14. Best Practices

### DO

```xml
<!-- noupdate="1" cho production data -->
<data noupdate="1">
    <record id="default_category" model="product.category">
        <field name="name">Default</field>
    </record>
</data>

<!-- Dùng XML ID có nghĩa -->
<record id="email_template_order_confirmation" model="mail.template">

<!-- Dùng ref() cho references -->
<field name="partner_id" ref="base.res_partner_1"/>

<!-- model_id dùng underscore notation -->
<field name="model_id" ref="model_my_model"/>
```

### DON'T

```xml
<!-- KHÔNG dùng noupdate="0" cho production data -->
<data noupdate="0">
    <record id="important_config" model="ir.config_parameter">
        <!-- Sẽ bị overwrite khi upgrade! -->
    </record>
</data>

<!-- KHÔNG hardcode IDs -->
<field name="partner_id" eval="1"/>  <!-- Dùng ref() thay thế -->

<!-- KHÔNG bỏ quên noupdate cho cron/sequence -->
<odoo>
    <record id="my_cron" model="ir.cron">
        <!-- Thiếu noupdate="1"! -->
    </record>
</odoo>
```

---

## 15. Common model_id References

| Model | ref value |
|-------|-----------|
| `my.model` | `model_my_model` |
| `sale.order` | `sale.model_sale_order` |
| `purchase.order` | `purchase.model_purchase_order` |
| `account.move` | `account.model_account_move` |
| `res.partner` | `base.model_res_partner` |
| `project.task` | `project.model_project_task` |

Format: `{module}.model_{model_name_with_underscores}`

---

**Status**: Complete
**Odoo Version**: 19.0
**Last Updated**: 2026-06-15

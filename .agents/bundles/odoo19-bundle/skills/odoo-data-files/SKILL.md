---
name: odoo-data-files
description: Hướng dẫn tạo Data Files trong Odoo 19 - XML/CSV data, noupdate, sequences, server actions, scheduled actions. Use when the user asks to create data file, add default data, define sequence, configure server action, or setup scheduled action.
---

# Odoo 19 Data Files

## Goal
Giúp agent tạo và quản lý data files (XML/CSV) cho default data, sequences, server actions, và scheduled actions.

## When to use this skill
- "tạo data file", "default data", "demo data"
- "sequence", "auto number"
- "server action", "automated action"
- "scheduled action", "cron job"
- "noupdate", "ref()"

## Instructions

### 1. XML Data File
```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <!-- Regular data (will be updated on module upgrade) -->
    <record id="my_default_record" model="my.model">
        <field name="name">Default Record</field>
        <field name="partner_id" ref="base.main_partner"/>
        <field name="active" eval="True"/>
        <field name="date" eval="datetime.now().strftime('%Y-%m-%d')"/>
    </record>
</odoo>
```

### 2. Noupdate Data (won't be overwritten on upgrade)
```xml
<odoo noupdate="1">
    <record id="sequence_my_model" model="ir.sequence">
        <field name="name">My Model Sequence</field>
        <field name="code">my.model</field>
        <field name="prefix">MY/%(year)s/</field>
        <field name="padding">5</field>
    </record>
</odoo>
```

### 3. CSV Data
```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_my_model,my.model.user,model_my_model,base.group_user,1,1,1,0
```

### 4. Server Action
```xml
<record id="action_set_done" model="ir.actions.server">
    <field name="name">Set Done</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="state">code</field>
    <field name="code">
        for rec in records:
            rec.write({'state': 'done'})
    </field>
</record>
```

### 5. Scheduled Action (Cron)
```xml
<record id="cron_cleanup" model="ir.cron">
    <field name="name">My Model: Cleanup</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="state">code</field>
    <field name="code">model._cron_cleanup()</field>
    <field name="interval_number">1</field>
    <field name="interval_type">days</field>
    <field name="numbercall">-1</field>
    <field name="active" eval="True"/>
</record>
```

### 6. Common eval Expressions
```xml
<field name="value" eval="True"/>
<field name="list" eval="[(4, ref('other_record'))]"/>
<field name="date" eval="(datetime.now() + relativedelta(days=30)).strftime('%Y-%m-%d')"/>
<field name="users" eval="[(6, 0, [ref('base.user_admin')])]"/>
```

## Constraints
- Data cho security/sequences/cron PHẢI dùng `noupdate="1"`.
- KHÔNG hardcode IDs — luôn dùng `ref('module.xml_id')`.
- CSV files chỉ dùng cho ir.model.access (ACL).

## Best practices
- Group data files: `data/`, `demo/` (demo chỉ load khi install with demo).
- Sequence prefix dùng `%(year)s`, `%(month)s` cho dynamic values.
- Đọc `resources/reference.md` cho mail templates, automated actions patterns.

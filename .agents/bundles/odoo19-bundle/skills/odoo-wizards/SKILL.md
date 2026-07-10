---
name: odoo-wizards
description: Hướng dẫn tạo Wizards (TransientModel) trong Odoo 19 - popup dialogs, batch operations, multi-step wizards. Use when the user asks to create wizard, build popup, batch process, or implement TransientModel in Odoo 19.
---

# Odoo 19 Wizards

## Goal
Giúp agent tạo wizards (TransientModel) cho user interactions, batch operations, confirmation dialogs và multi-step workflows.

## When to use this skill
- "tạo wizard", "create wizard", "popup"
- "TransientModel", "batch operation"
- "confirmation dialog", "multi-step wizard"
- "action popup", "selection wizard"

## Instructions

### 1. Basic Wizard Model
```python
from odoo import models, fields, api

class MyWizard(models.TransientModel):
    _name = 'my.module.wizard'
    _description = 'My Wizard'

    name = fields.Char(required=True)
    partner_id = fields.Many2one('res.partner')
    line_ids = fields.One2many('my.module.wizard.line', 'wizard_id')

    def action_confirm(self):
        """Process wizard and close"""
        self.ensure_one()
        # Business logic here
        return {'type': 'ir.actions.act_window_close'}

    def action_confirm_and_view(self):
        """Process and open result"""
        self.ensure_one()
        record = self.env['my.model'].create({'name': self.name})
        return {
            'type': 'ir.actions.act_window',
            'res_model': 'my.model',
            'res_id': record.id,
            'view_mode': 'form',
            'target': 'current',
        }
```

### 2. Wizard View (Form as dialog)
```xml
<record id="view_my_wizard_form" model="ir.ui.view">
    <field name="name">my.module.wizard.form</field>
    <field name="model">my.module.wizard</field>
    <field name="arch" type="xml">
        <form string="My Wizard">
            <group>
                <field name="partner_id"/>
                <field name="name"/>
            </group>
            <field name="line_ids">
                <list editable="bottom">
                    <field name="product_id"/>
                    <field name="quantity"/>
                </list>
            </field>
            <footer>
                <button string="Confirm" name="action_confirm"
                        type="object" class="btn-primary"/>
                <button string="Cancel" class="btn-secondary" special="cancel"/>
            </footer>
        </form>
    </field>
</record>
```

### 3. Action to Open Wizard
```xml
<record id="action_my_wizard" model="ir.actions.act_window">
    <field name="name">My Wizard</field>
    <field name="res_model">my.module.wizard</field>
    <field name="view_mode">form</field>
    <field name="target">new</field>  <!-- Opens as popup -->
</record>
```

### 4. Open Wizard from Python (with context)
```python
def action_open_wizard(self):
    return {
        'type': 'ir.actions.act_window',
        'name': 'My Wizard',
        'res_model': 'my.module.wizard',
        'view_mode': 'form',
        'target': 'new',
        'context': {
            'default_partner_id': self.partner_id.id,
            'active_ids': self.ids,
            'active_model': self._name,
        },
    }
```

### 5. Batch Processing Pattern
```python
class BatchWizard(models.TransientModel):
    _name = 'batch.wizard'

    def action_process(self):
        active_ids = self.env.context.get('active_ids', [])
        records = self.env['my.model'].browse(active_ids)
        for record in records:
            record.action_confirm()
        return {'type': 'ir.actions.act_window_close'}
```

## Constraints
- Wizard models PHẢI dùng `TransientModel` (không phải `Model`).
- Wizard data tự động bị xóa sau 1-2 ngày (transient).
- Footer buttons: `special="cancel"` cho nút Cancel (không cần Python method).

## Best practices
- Truyền context (`active_ids`, `active_model`) khi mở wizard từ button.
- Dùng `target='new'` để mở popup, `target='current'` để mở full page.
- Dùng `ensure_one()` trong action methods.

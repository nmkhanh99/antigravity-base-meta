# Odoo Report Patterns — Full Code Reference

## action-template

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <!-- Report Action -->
    <record id="report_my_model" model="ir.actions.report">
        <field name="name">My Report</field>
        <field name="model">my.module.my.model</field>
        <field name="report_type">qweb-pdf</field>
        <field name="report_name">my_module.report_my_model_document</field>
        <field name="report_file">my_module.report_my_model_document</field>
        <field name="print_report_name">'MyReport - %s' % object.name</field>
        <field name="binding_model_id" ref="my_module.model_my_module_my_model"/>
        <field name="binding_type">report</field>
        <field name="paperformat_id" ref="base.paperformat_euro"/>
    </record>

    <!-- QWeb Template -->
    <template id="report_my_model_document">
        <t t-call="web.html_container">
            <t t-foreach="docs" t-as="doc">
                <t t-call="web.external_layout">
                    <div class="page">
                        <h2><span t-field="doc.name"/></h2>
                        <div class="row mt-4">
                            <div class="col-6">
                                <strong>Customer:</strong> <span t-field="doc.partner_id.name"/>
                            </div>
                            <div class="col-6 text-end">
                                <strong>Date:</strong> <span t-field="doc.date"/>
                            </div>
                        </div>

                        <table class="table table-sm mt-4">
                            <thead>
                                <tr>
                                    <th>Description</th>
                                    <th class="text-end">Qty</th>
                                    <th class="text-end">Price</th>
                                    <th class="text-end">Total</th>
                                </tr>
                            </thead>
                            <tbody>
                                <t t-foreach="doc.line_ids" t-as="line">
                                    <tr>
                                        <td><span t-field="line.name"/></td>
                                        <td class="text-end"><span t-field="line.quantity"/></td>
                                        <td class="text-end">
                                            <span t-field="line.price_unit"
                                                  t-options='{"widget":"monetary","display_currency":doc.currency_id}'/>
                                        </td>
                                        <td class="text-end">
                                            <span t-field="line.subtotal"
                                                  t-options='{"widget":"monetary","display_currency":doc.currency_id}'/>
                                        </td>
                                    </tr>
                                </t>
                            </tbody>
                            <tfoot>
                                <tr>
                                    <td colspan="3" class="text-end"><strong>Total:</strong></td>
                                    <td class="text-end">
                                        <strong><span t-field="doc.amount_total"
                                              t-options='{"widget":"monetary","display_currency":doc.currency_id}'/></strong>
                                    </td>
                                </tr>
                            </tfoot>
                        </table>

                        <div t-if="doc.notes" class="mt-4">
                            <strong>Notes:</strong>
                            <p t-field="doc.notes"/>
                        </div>
                    </div>
                </t>
            </t>
        </t>
    </template>
</odoo>
```

## directives

```xml
<!-- t-field với formatting options -->
<span t-field="doc.date" t-options='{"format": "dd/MM/yyyy"}'/>
<span t-field="doc.amount_total" t-options='{"widget": "monetary", "display_currency": doc.currency_id}'/>
<span t-field="doc.quantity" t-options='{"precision": 2}'/>
<div t-field="doc.partner_id" t-options='{"widget": "contact", "fields": ["address", "phone", "email"]}'/>

<!-- Numbered rows with loop variables -->
<t t-foreach="doc.line_ids" t-as="line">
    <tr>
        <td><t t-esc="line_index + 1"/></td>
        <td><span t-field="line.name"/></td>
        <td t-if="line_last"><strong>Last row</strong></td>
    </tr>
</t>

<!-- Conditional row styling -->
<tr t-att-class="'table-danger' if line.amount &lt; 0 else ''"/>
<div t-attf-class="alert alert-#{doc.state == 'done' and 'success' or 'warning'}"/>
```

## report-model

```python
# report/my_report.py
from __future__ import annotations
from typing import Any, Optional
from odoo import api, models


class MyReport(models.AbstractModel):
    _name = 'report.my_module.report_my_model_document'
    _description = 'My Report'

    @api.model
    def _get_report_values(self, docids: list[int], data: Optional[dict[str, Any]] = None) -> dict[str, Any]:
        docs = self.env['my.module.my.model'].browse(docids)
        return {
            'doc_ids': docids,
            'doc_model': 'my.module.my.model',
            'docs': docs,
            'data': data,
            'totals': {'amount': sum(docs.mapped('amount_total')), 'count': len(docs)},
            'company': self.env.company,
        }
```

## paper-format

```xml
<record id="paperformat_my_module" model="report.paperformat">
    <field name="name">My Module Format</field>
    <field name="default" eval="False"/>
    <field name="format">A4</field>
    <field name="orientation">Portrait</field>
    <field name="margin_top">40</field>
    <field name="margin_bottom">20</field>
    <field name="margin_left">7</field>
    <field name="margin_right">7</field>
    <field name="header_spacing">35</field>
    <field name="dpi">90</field>
</record>
<!-- Standard: base.paperformat_euro (A4), base.paperformat_us (Letter) -->
```

## print-python

```python
# Single record
def action_print(self) -> dict[str, Any]:
    return self.env.ref('my_module.report_my_model').report_action(self)

# Multiple records
def action_print_selected(self) -> dict[str, Any]:
    return self.env.ref('my_module.report_my_model').report_action(self.ids)

# Với custom data (truyền sang _get_report_values)
def action_print_with_data(self) -> dict[str, Any]:
    data = {'date_from': str(self.date_from), 'date_to': str(self.date_to)}
    return self.env.ref('my_module.report_my_model').report_action(self, data=data)
```

## inheritance

```xml
<template id="report_invoice_document_inherit" inherit_id="account.report_invoice_document">
    <xpath expr="//div[@name='invoice_address']" position="after">
        <div class="col-6">
            <strong>Custom Field:</strong>
            <span t-field="o.x_custom_field"/>
        </div>
    </xpath>
</template>
```

## css

```xml
<!-- Inline styles -->
<style>
    .my-report-table { width: 100%; border-collapse: collapse; }
    .my-report-table th { background-color: #f5f5f5; border-bottom: 2px solid #333; }
    .page-break { page-break-after: always; }
</style>

<!-- Multi-page với page break -->
<t t-foreach="docs" t-as="doc">
    <div class="page"><!-- Content --></div>
    <t t-if="not doc_last"><div class="page-break"/></t>
</t>
```

```python
# External SCSS trong manifest
'assets': {
    'web.report_assets_common': ['my_module/static/src/scss/report.scss'],
},
```

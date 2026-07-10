# QWeb Template Patterns — Full Code Reference

## directives

```xml
<!-- t-if / t-elif / t-else -->
<t t-if="record.state == 'draft'"><span class="badge bg-secondary">Draft</span></t>
<t t-elif="record.state == 'confirmed'"><span class="badge bg-primary">Confirmed</span></t>
<t t-else=""><span class="badge bg-danger">Other</span></t>

<!-- t-foreach với loop variables: _index, _size, _first, _last, _odd, _even -->
<t t-foreach="lines" t-as="line">
    <tr>
        <td t-esc="line_index + 1"/>
        <td t-esc="line.name"/>
        <td t-if="line_last">Last</td>
    </tr>
</t>

<!-- t-set variable / expression -->
<t t-set="total" t-value="sum(line.amount for line in lines)"/>

<!-- t-att-{attr}: dynamic value | t-attf-{attr}: format string -->
<div t-att-class="record.state"/>
<div t-attf-class="card state-#{record.state}"/>

<!-- Striped table -->
<t t-foreach="items" t-as="item">
    <tr t-attf-class="#{item_odd and 'table-light' or ''}"><td t-esc="item.name"/></tr>
</t>

<!-- t-call với params -->
<t t-call="my_module.address_block">
    <t t-set="partner" t-value="record.partner_id"/>
</t>
```

## report

```xml
<!-- Standard report structure -->
<template id="report_my_document">
    <t t-call="web.html_container">
        <t t-foreach="docs" t-as="doc">
            <t t-call="web.external_layout">
                <div class="page">
                    <h1 t-field="doc.name"/>
                    <table class="table table-sm">
                        <thead>
                            <tr>
                                <th>Product</th>
                                <th class="text-end">Qty</th>
                                <th class="text-end">Price</th>
                                <th class="text-end">Subtotal</th>
                            </tr>
                        </thead>
                        <tbody>
                            <t t-foreach="doc.line_ids" t-as="line">
                                <tr>
                                    <td t-esc="line.product_id.name"/>
                                    <td class="text-end" t-esc="line.quantity"/>
                                    <td class="text-end" t-field="line.price_unit"
                                        t-options='{"widget":"monetary","display_currency":doc.currency_id}'/>
                                    <td class="text-end" t-field="line.price_subtotal"
                                        t-options='{"widget":"monetary","display_currency":doc.currency_id}'/>
                                </tr>
                            </t>
                        </tbody>
                        <tfoot>
                            <tr>
                                <td colspan="3" class="text-end"><strong>Total:</strong></td>
                                <td class="text-end" t-field="doc.amount_total"
                                    t-options='{"widget":"monetary","display_currency":doc.currency_id}'/>
                            </tr>
                        </tfoot>
                    </table>
                </div>
            </t>
        </t>
    </t>
</template>
```

## kanban

```xml
<kanban>
    <field name="name"/><field name="state"/><field name="color"/>
    <templates>
        <t t-name="kanban-box">
            <div t-attf-class="oe_kanban_card oe_kanban_global_click #{kanban_color(record.color.raw_value)}">
                <div class="oe_kanban_content">
                    <div class="o_kanban_record_top">
                        <div class="o_kanban_record_headings">
                            <strong class="o_kanban_record_title"><field name="name"/></strong>
                        </div>
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
```

## website

```xml
<!-- Website page -->
<template id="my_page" name="My Page">
    <t t-call="website.layout">
        <div id="wrap" class="oe_structure">
            <section class="container py-5">
                <t t-foreach="records" t-as="record">
                    <div class="card mb-3">
                        <div class="card-body">
                            <h5 t-esc="record.name"/>
                            <p t-raw="record.description"/>
                        </div>
                    </div>
                </t>
            </section>
        </div>
    </t>
</template>

<!-- Portal template -->
<template id="portal_my_records" name="My Records">
    <t t-call="portal.portal_layout">
        <t t-set="breadcrumbs_searchbar" t-value="True"/>
        <t t-call="portal.portal_searchbar"><t t-set="title">My Records</t></t>
        <t t-if="records">
            <t t-foreach="records" t-as="record">
                <div class="card mb-2">
                    <div class="card-body">
                        <a t-attf-href="/my/records/#{record.id}"><t t-esc="record.name"/></a>
                    </div>
                </div>
            </t>
        </t>
        <t t-else=""><p>No records found.</p></t>
    </t>
</template>
```

## inheritance

```xml
<!-- Extend existing template -->
<template id="my_inherit" inherit_id="web.layout">
    <xpath expr="//head" position="inside">
        <link rel="stylesheet" href="/my_module/static/src/css/custom.css"/>
    </xpath>
</template>

<!-- Extend report -->
<template id="report_invoice_inherit" inherit_id="account.report_invoice_document">
    <xpath expr="//div[@name='invoice_address']" position="after">
        <div class="col-6">
            <strong>Custom:</strong>
            <span t-field="o.x_custom_field"/>
        </div>
    </xpath>
</template>
```

## expressions

```xml
<!-- String operations -->
<t t-esc="record.name.upper()"/>
<t t-esc="record.name[:20] + '...' if len(record.name) > 20 else record.name"/>
<t t-esc="', '.join(record.tag_ids.mapped('name'))"/>

<!-- Number formatting -->
<t t-esc="'%.2f' % record.amount"/>
<t t-esc="'{:,.2f}'.format(record.amount)"/>

<!-- Date formatting -->
<t t-esc="format_date(env, record.date)"/>
<t t-esc="record.date.strftime('%d/%m/%Y') if record.date else ''"/>

<!-- t-field options -->
<span t-field="record.amount" t-options='{"widget": "monetary", "display_currency": doc.currency_id}'/>
<span t-field="record.date" t-options='{"format": "dd/MM/yyyy"}'/>
<div t-field="record.partner_id" t-options='{"widget": "contact", "fields": ["address", "phone", "email"]}'/>
```

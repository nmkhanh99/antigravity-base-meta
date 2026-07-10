# QWeb Frontend Templates — Odoo 19 Reference Guide

## FRD Checklist

| Hạng mục | Bắt buộc mô tả |
|----------|----------------|
| Template | `t-name`, file path, owner component/view |
| Type | New / Inherit / OWL / Kanban / Website / Portal |
| Parent | `t-inherit` target, mode (`primary`/`extension`), XPath |
| Data | Context/props/state variables dùng trong template |
| Directives | `t-if`, `t-foreach`, `t-out`, `t-att`, `t-call`, `t-set` |
| Safety | Nguồn HTML, escaping/sanitizing, không dùng raw user input |

---

## Pattern 1 — OWL Component Template

```xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
    <t t-name="my_module.MyDashboard">
        <div class="o_my_dashboard">
            <t t-if="state.loading">
                <span>Loading...</span>
            </t>
            <t t-else="">
                <t t-foreach="state.records" t-as="record" t-key="record.id">
                    <button type="button" t-on-click="() => this.openRecord(record.id)">
                        <span t-out="record.name"/>
                    </button>
                </t>
            </t>
        </div>
    </t>
</templates>
```

Lưu ý:
- `t-key` bắt buộc trong mọi `t-foreach` của OWL template
- `t-out` auto-escape; dùng `Markup` từ Python nếu cần render HTML an toàn
- `t-on-click` với arrow function để giữ đúng `this` context

---

## Pattern 2 — Template Inheritance (extension)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
    <t t-inherit="base_module.TargetTemplateName" t-inherit-mode="extension">
        <xpath expr="//div[contains(@class, 'o_target')]" position="inside">
            <span t-out="props.extraLabel"/>
        </xpath>
    </t>
</templates>
```

Lưu ý:
- `t-inherit-mode="extension"` — sửa template gốc in-place
- `t-inherit-mode="primary"` — tạo template mới kế thừa từ gốc (có tên riêng)
- Không dùng `t-extend` + `t-jquery` (deprecated)

---

## Pattern 3 — t-out, t-if, t-foreach cơ bản

```xml
<!-- Conditional output -->
<t t-if="condition">
    <p t-out="value"/>
</t>

<!-- Loop với index-based parity (thay thế {as}_even/_odd deprecated) -->
<t t-foreach="items" t-as="item">
    <li t-attf-class="row {{ item_index % 2 == 0 and 'even' or 'odd' }}">
        <t t-out="item.name"/>
    </li>
</t>
```

Biến tự động trong `t-foreach`:
| Biến | Ý nghĩa |
|------|---------|
| `{as}_index` | Index 0-based |
| `{as}_size` | Tổng số phần tử |
| `{as}_first` | Boolean, là phần tử đầu |
| `{as}_last` | Boolean, là phần tử cuối |
| `{as}_value` | Alias của biến lặp |

Deprecated (không dùng): `{as}_all`, `{as}_parity`, `{as}_even`, `{as}_odd`

---

## Pattern 4 — t-set với body (safe HTML capture)

```xml
<!-- Capture safe HTML content không bị double-escape -->
<t t-set="content"><em>Nội dung <strong>quan trọng</strong></em></t>
<div>
    <t t-out="content"/>   <!-- render đúng, không escape lần 2 -->
</div>
```

---

## Pattern 5 — t-call với magic variable '0'

```xml
<!-- Caller -->
<t t-call="my_module.CardTemplate">
    <em>Nội dung inject vào callee</em>
</t>

<!-- Callee (my_module.CardTemplate) -->
<t t-name="my_module.CardTemplate">
    <div class="o_card">
        <t t-out="0"/>   <!-- render body từ caller -->
    </div>
</t>
```

Lưu ý: `t-set` bên trong `t-call` body là local scope — không visible sau khi call kết thúc.

---

## Pattern 6 — t-att dynamic attributes

```xml
<!-- Dict nhiều attr -->
<div t-att="{'class': 'o_box', 'data-id': record.id}"/>

<!-- Pair [name, value] cho một attr -->
<div t-att="['data-active', isActive and 'true' or 'false']"/>

<!-- Format string (t-attf) — Jinja style -->
<div t-attf-class="o_row {{ state.selected and 'selected' or '' }}"/>
```

---

## Pattern 7 — t-field (Python/server-side only)

```xml
<!-- Chỉ dùng trong QWeb Python templates (website, reports) -->
<span t-field="record.date_deadline" t-options='{"widget": "date"}'/>
<span t-field="record.amount_total" t-options='{"widget": "monetary", "display_currency": record.currency_id}'/>
```

KHÔNG dùng `t-field` trong OWL JS templates.

---

## Pattern 8 — Safe HTML với markupsafe (Python side)

```python
from markupsafe import Markup, escape

# Trusted HTML — render với t-out không bị escape
safe_html = Markup("<b>Nội dung an toàn</b>")

# Escape thủ công khi cần
escaped = escape(user_input)
combined = Markup("<p>{}</p>").format(escaped)
```

```xml
<!-- Template nhận giá trị Markup — t-out sẽ không double-escape -->
<div t-out="safe_html"/>
```

---

## Pattern 9 — OWL Dropdown với useDropdownState

```javascript
// Component JS
import { useDropdownState } from "@web/core/dropdown/dropdown_hooks";

setup() {
    this.dropdownState = useDropdownState({
        onOpen: () => this.onDropdownOpen(),
        onClose: () => this.onDropdownClose(),
    });
}
```

```xml
<!-- Template XML -->
<Dropdown state="dropdownState" menuClass="'o_my_menu'">
    <t t-set-slot="default">
        <button>Toggle Menu</button>
    </t>
    <t t-set-slot="content">
        <DropdownItem onSelected="() => this.doAction()">Action 1</DropdownItem>
    </t>
</Dropdown>
```

---

## Pattern 10 — OWL Notebook (tabs)

```xml
<Notebook defaultPage="'tab_info'" onPageUpdate="(page) => this.onTabChange(page)">
    <t t-set-slot="tab_info" title="'Thông tin'" isVisible="true">
        <div>Nội dung tab 1</div>
    </t>
    <t t-set-slot="tab_detail" title="'Chi tiết'" isVisible="true">
        <div>Nội dung tab 2</div>
    </t>
</Notebook>
```

---

## Pattern 11 — Kanban template

```xml
<kanban>
    <field name="name"/>
    <field name="state"/>
    <templates>
        <t t-name="kanban-card">
            <div class="o_kanban_card_header">
                <span class="o_kanban_record_title" t-out="record.name.value"/>
            </div>
            <div class="o_kanban_card_content">
                <span t-out="record.state.value"/>
            </div>
        </t>
    </templates>
</kanban>
```

---

## Deprecated — Không dùng trong Odoo 19

| Deprecated | Thay bằng |
|-----------|-----------|
| `t-raw="value"` | `t-out="value"` với `Markup` nếu cần HTML |
| `t-esc="value"` | `t-out="value"` (preferred) |
| `t-extend` + `t-jquery` | `t-inherit` + `t-inherit-mode` + XPath |
| `{as}_even`, `{as}_odd` | `{as}_index % 2 == 0` |
| `{as}_parity`, `{as}_all` | `{as}_index` arithmetic |
| Iterate integers in `t-foreach` | `t-foreach="range(n)"` |

---

## Python render helpers

```python
# Trong controller/website route
response = http.request.render('my_module.template_id', {
    'key': value,
    'records': records,
})

# Trong ir.qweb
html = request.env['ir.qweb']._render('my_module.template_id', values)
```

Context tự động có sẵn trong Python templates:
`request`, `debug`, `quote_plus`, `json`, `time`, `datetime`, `relativedelta`, `keep_query`

---

## OWL Hooks thường dùng

```javascript
import { useService } from "@web/core/utils/hooks";
import { useBus } from "@web/core/utils/hooks";
import { useAutofocus } from "@web/core/utils/hooks";

setup() {
    this.orm = useService("orm");
    this.action = useService("action");
    // useBus(bus, eventName, callback)
    // useAutofocus({ refName: "myInput", selectAll: true })
}
```

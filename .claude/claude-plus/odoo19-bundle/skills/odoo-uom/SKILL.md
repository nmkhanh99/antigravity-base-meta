---
name: odoo-uom
description: Unit of Measure (UoM) Odoo 19 — tạo danh mục và đơn vị đo, chuyển đổi số lượng/giá, UoM trên sản phẩm, stock moves, float_compare/float_round, view pattern với groups="uom.group_uom". Kích hoạt khi user nói "đơn vị đo", "UoM", "unit of measure", "chuyển đổi đơn vị", "_compute_quantity", "float_compare".
---

# Unit of Measure (UoM) Patterns (Odoo 19)

## Goal
Implement UoM đúng chuẩn Odoo 19 — tạo category/unit, chuyển đổi số lượng và giá, tích hợp model tùy chỉnh.

**Input**: Mô tả nhu cầu UoM, model target  
**Output**: Python + XML + Data file đầy đủ

## When to use this skill
- "thêm UoM vào model", "chuyển đổi đơn vị đo"
- "tạo danh mục UoM tùy chỉnh"
- "tính giá theo UoM mua/bán"
- "float_compare precision"

## Instructions

### Bước 1 — UoM Category và cấu trúc
```
UoM Category: Weight → kg (reference, factor=1), g (smaller, factor_inv=1000), lb (bigger, factor=0.453592)
UoM Category: Unit → Unit(s) (reference), Dozen (factor_inv=12), Hundred (factor_inv=100)
```
Xem data XML: `references/GUIDE.md#data-xml`

### Bước 2 — Tạo UoM Category và Unit (Python)
Xem template: `references/GUIDE.md#create-uom`  
`uom_type`: `reference` | `smaller` (factor_inv: bao nhiêu unit = 1 ref) | `bigger` (factor: 1 ref = bao nhiêu unit)

### Bước 3 — Chuyển đổi số lượng
`from_uom._compute_quantity(qty, to_uom, round=True, rounding_method='HALF-UP')`  
Chỉ chuyển trong cùng category. Xem examples: `references/GUIDE.md#convert-qty`

### Bước 4 — Chuyển đổi giá
`from_uom._compute_price(price, to_uom)` — xem `references/GUIDE.md#convert-price`

### Bước 5 — Float comparison đúng chuẩn
```python
from odoo.tools import float_compare, float_round
result = float_compare(qty1, qty2, precision_rounding=uom.rounding)
# Returns: -1, 0, 1. KHÔNG dùng == trực tiếp với float.
```

### Bước 6 — Model tùy chỉnh với UoM
Xem template: `references/GUIDE.md#custom-model`  
Key: `digits='Product Unit of Measure'`, `domain="[('category_id', '=', product_uom_category_id)]"`, `_onchange_product_id` set UoM mặc định.

### Bước 7 — UoM trên Sale/Purchase Order Line
`_onchange_product_uom` tính lại giá khi đổi UoM. Xem `references/GUIDE.md#sale-line`

### Bước 8 — Stock Move với UoM
Xem `references/GUIDE.md#stock-move` — `product_uom_qty` + `product_uom` fields.

### Bước 9 — View XML pattern
```xml
<field name="quantity" class="oe_inline"/>
<field name="product_uom_id" class="oe_inline" options="{'no_create': True}" groups="uom.group_uom"/>
<field name="product_uom_category_id" invisible="1"/>
```
`groups="uom.group_uom"` — ẩn khi chưa bật Multi-UoM.

### Bước 10 — Data XML (noupdate="1")
Xem template: `references/GUIDE.md#data-xml`

### Bước 11 — UoM chuẩn Odoo (external IDs)
Xem `references/GUIDE.md#standard-uom` — `uom.product_uom_unit/dozen/kgm/gram/hour/day/litre`

## Constraints
- **Chỉ** chuyển đổi trong cùng UoM category
- **PHẢI** dùng `float_compare` — không dùng `==` cho float
- View UoM fields **phải** có `groups="uom.group_uom"`

## Best practices
- Set `rounding` phù hợp: kg → 0.001, cái → 1.0, giờ → 0.25
- Dùng `digits='Product Unit of Measure'` cho Float quantity field
- `noupdate="1"` trong data XML để không override khi upgrade

# UoM Patterns — Full Code Reference

## create-uom

```python
from odoo import fields

# Tạo category
category = self.env['uom.category'].create({'name': 'Volume'})

# Reference unit (factor = 1)
liter = self.env['uom.uom'].create({
    'name': 'Liter', 'category_id': category.id,
    'uom_type': 'reference', 'rounding': 0.001,
})

# Smaller unit: factor_inv = bao nhiêu unit = 1 reference
milliliter = self.env['uom.uom'].create({
    'name': 'Milliliter', 'category_id': category.id,
    'uom_type': 'smaller', 'factor_inv': 1000, 'rounding': 1,
})

# Bigger unit: factor = 1 unit = bao nhiêu reference
gallon = self.env['uom.uom'].create({
    'name': 'Gallon', 'category_id': category.id,
    'uom_type': 'bigger', 'factor': 3.78541, 'rounding': 0.01,
})
```

## convert-qty

```python
# Chuyển số lượng — chỉ trong cùng category
from_uom = self.env.ref('uom.product_uom_dozen')
to_uom = self.env.ref('uom.product_uom_unit')
units = from_uom._compute_quantity(2, to_uom)  # = 24

# Với rounding method
units_rounded = from_uom._compute_quantity(2.5, to_uom, round=True, rounding_method='HALF-UP')
units_up = from_uom._compute_quantity(10.3, to_uom, round=True, rounding_method='UP')
```

## convert-price

```python
# Chuyển đơn giá
price_per_liter = gallon._compute_price(10, liter)  # $10/gallon → $/liter
```

## custom-model

```python
class InventoryAdjustment(models.Model):
    _name = 'inventory.adjustment'
    _description = 'Inventory Adjustment'

    product_id: int = fields.Many2one('product.product', required=True)
    quantity: float = fields.Float(digits='Product Unit of Measure')
    product_uom_id: int = fields.Many2one('uom.uom', domain="[('category_id', '=', product_uom_category_id)]")
    product_uom_category_id: int = fields.Many2one(related='product_id.uom_id.category_id')
    product_qty: float = fields.Float(compute='_compute_product_qty', store=True)

    @api.depends('quantity', 'product_uom_id', 'product_id')
    def _compute_product_qty(self) -> None:
        for record in self:
            if record.product_uom_id and record.product_id:
                record.product_qty = record.product_uom_id._compute_quantity(record.quantity, record.product_id.uom_id)
            else:
                record.product_qty = record.quantity

    @api.onchange('product_id')
    def _onchange_product_id(self) -> None:
        if self.product_id:
            self.product_uom_id = self.product_id.uom_id
```

## sale-line

```python
class SaleOrderLineExtend(models.Model):
    _inherit = 'sale.order.line'

    @api.onchange('product_uom')
    def _onchange_product_uom(self) -> None:
        if self.product_id and self.product_uom:
            self.price_unit = self.product_id.uom_id._compute_price(self.product_id.lst_price, self.product_uom)
```

## stock-move

```python
def create_stock_move(self, product, qty, uom, location_src, location_dest):
    return self.env['stock.move'].create({
        'name': product.name,
        'product_id': product.id,
        'product_uom_qty': qty,
        'product_uom': uom.id,
        'location_id': location_src.id,
        'location_dest_id': location_dest.id,
    })

def get_stock_in_uom(self, product, uom, location=None):
    qty = product.with_context(location=location.id).qty_available if location else product.qty_available
    return product.uom_id._compute_quantity(qty, uom)
```

## data-xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo noupdate="1">
    <record id="uom_categ_length" model="uom.category">
        <field name="name">Length</field>
    </record>
    <record id="uom_meter" model="uom.uom">
        <field name="name">Meter</field>
        <field name="category_id" ref="uom_categ_length"/>
        <field name="uom_type">reference</field>
        <field name="rounding">0.01</field>
    </record>
    <record id="uom_centimeter" model="uom.uom">
        <field name="name">Centimeter</field>
        <field name="category_id" ref="uom_categ_length"/>
        <field name="uom_type">smaller</field>
        <field name="factor_inv">100</field>
        <field name="rounding">1</field>
    </record>
    <record id="uom_kilometer" model="uom.uom">
        <field name="name">Kilometer</field>
        <field name="category_id" ref="uom_categ_length"/>
        <field name="uom_type">bigger</field>
        <field name="factor">1000</field>
        <field name="rounding">0.001</field>
    </record>
</odoo>
```

## standard-uom

```python
# Unit
unit   = self.env.ref('uom.product_uom_unit')
dozen  = self.env.ref('uom.product_uom_dozen')

# Weight
kg     = self.env.ref('uom.product_uom_kgm')
gram   = self.env.ref('uom.product_uom_gram')
lb     = self.env.ref('uom.product_uom_lb')
oz     = self.env.ref('uom.product_uom_oz')

# Time
hour   = self.env.ref('uom.product_uom_hour')
day    = self.env.ref('uom.product_uom_day')

# Volume
litre  = self.env.ref('uom.product_uom_litre')
```

# Translation & i18n — Full Code Reference

## _function

```python
from odoo import _

# Simple
message = _("This is translatable")

# %s — với 1 giá trị
message = _("Record %s has been created") % name

# Nhiều giá trị
message = _("Created %s records in %s seconds") % (count, time)

# Named placeholders
message = _("Record %(name)s created by %(user)s") % {'name': record.name, 'user': self.env.user.name}

# ❌ SAI — f-strings không được extract
# _(f"Record {name} created")

# ❌ SAI — concatenation
# _("Hello ") + name + _("!")

# ✅ ĐÚNG — chuỗi đơn với placeholder
_("Hello %s!") % name

# Log messages KHÔNG cần dịch
_logger.info("Record %s processed", record.id)  # Không dùng _()

# UI messages PHẢI dịch
record.message_post(body=_("Record processed."))
```

## with-context-lang

```python
# Đọc field theo ngôn ngữ
name_fr = self.with_context(lang='fr_FR').name
name_de = self.with_context(lang='de_DE').name

# Ghi field cho ngôn ngữ cụ thể
self.with_context(lang='fr_FR').write({'name': 'Nom Français'})

# Lấy ngôn ngữ user hiện tại
lang = self.env.user.lang or self.env.context.get('lang', 'en_US')
```

## dynamic-selection

```python
class MyModel(models.Model):
    _name = 'my.model'

    # Dynamic selection — labels dùng _()
    @api.model
    def _get_type_selection(self) -> list[tuple[str, str]]:
        return [('type_a', _('Type A')), ('type_b', _('Type B'))]

    type: str = fields.Selection(selection='_get_type_selection', string='Type')

    def get_translated_status(self) -> str:
        selection = self._fields['state'].selection
        if callable(selection): selection = selection(self)
        return dict(selection).get(self.state, self.state)
```

## po-format

```po
# Translation for Odoo module my_module — Vietnamese
msgid ""
msgstr ""
"Project-Id-Version: Odoo 19.0\n"
"Language: vi\n"
"Content-Type: text/plain; charset=UTF-8\n"

#. module: my_module
#: model:ir.model,name:my_module.model_my_model
msgid "My Model"
msgstr "Mô hình của tôi"

#. module: my_module
#: code:addons/my_module/models/my_model.py:0
#, python-format
msgid "Record %s has been created"
msgstr "Bản ghi %s đã được tạo"
```

```bash
# Export POT template
./odoo-bin -d mydb --i18n-export=/tmp/my_module.pot --modules=my_module

# Import PO file
./odoo-bin -d mydb --i18n-import=/tmp/vi.po --language=vi
```

## report-lang

```xml
<!-- Product name theo ngôn ngữ partner trong report -->
<td t-esc="line.product_id.with_context(lang=doc.partner_id.lang).name"/>
```

```python
# Gửi email theo ngôn ngữ partner
lang = partner.lang or 'en_US'
template.with_context(lang=lang).send_mail(record.id)
```

## test-translation

```python
def test_translation(self) -> None:
    record = self.env['my.model'].create({'name': 'English Name'})
    record.with_context(lang='fr_FR').write({'name': 'Nom Français'})
    self.assertEqual(record.name, 'English Name')
    self.assertEqual(record.with_context(lang='fr_FR').name, 'Nom Français')
```

## translation-table

| Thành phần | Dịch được | Cách dịch |
|---|---|---|
| Field labels | Có | `string=` attribute |
| Help text | Có | `help=` attribute |
| Selection labels | Có | Giá trị tuple |
| Nội dung field | Tuỳ chọn | `translate=True` |
| Error messages | Có | `_()` |
| Button/Menu labels | Có | `string=` trong XML |
| Text trong report | Có | Text XML tĩnh |
| Log messages | **Không** | Giữ tiếng Anh |

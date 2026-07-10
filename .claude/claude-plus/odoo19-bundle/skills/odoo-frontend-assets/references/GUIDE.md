# Odoo Frontend Assets — Reference Guide

## Bundle Chính

| Bundle | Khi dùng |
|--------|----------|
| `web.assets_backend` | Web client backend: views, actions, widgets, OWL templates |
| `web.assets_common` | Framework low-level (chứa boot.js) — dùng thận trọng |
| `web.assets_frontend` | Website frontend (portal, eCommerce) |
| `web.assets_unit_tests` | JS tests trong `static/tests/` |
| `web.reports_assets_common` | CSS/font cho QWeb PDF reports |

## Manifest Pattern — Tất cả 7 Operations

```python
# __manifest__.py
'assets': {
    'web.assets_backend': [
        # --- append (mặc định) ---
        'my_addon/static/src/js/file.js',
        ('append', 'my_addon/static/src/js/another.js'),  # tương đương

        # --- prepend: thêm đầu bundle ---
        ('prepend', 'my_addon/static/src/css/override.scss'),

        # --- before: chèn TRƯỚC một file target ---
        ('before', 'web/static/src/css/target.scss', 'my_addon/static/src/css/new.scss'),

        # --- after: chèn SAU một file target ---
        ('after', 'web/static/src/css/list_view.scss', 'my_addon/static/src/css/custom.scss'),

        # --- include: nhúng sub-bundle (prefix _ theo convention) ---
        ('include', 'web._primary_variables'),

        # --- remove: xóa file khỏi bundle ---
        ('remove', 'web/static/src/js/unwanted.js'),

        # --- replace: thay thế file tại vị trí cũ ---
        ('replace', 'web/static/src/js/boot.js', 'my_addon/static/src/js/boot.js'),
    ],
}
```

## Pattern Thông Dụng

```python
# Khai báo OWL component (JS + XML + SCSS)
'assets': {
    'web.assets_backend': [
        'my_module/static/src/components/my_widget/my_widget.js',
        'my_module/static/src/components/my_widget/my_widget.xml',
        'my_module/static/src/components/my_widget/my_widget.scss',
    ],
}

# Dùng glob pattern cho toàn bộ thư mục
'assets': {
    'web.assets_backend': [
        'my_module/static/src/js/**/*',
        'my_module/static/src/xml/**/*',
    ],
    'web.assets_unit_tests': [
        'my_module/static/tests/**/*',
    ],
}

# CSS cho QWeb PDF report
'assets': {
    'web.reports_assets_common': [
        'my_module/static/src/css/report.scss',
    ],
}
```

## Lazy Loading

### `useAssets` — OWL hook (dùng trong `setup()`)

```javascript
import { useAssets } from "@web/core/assets";

class MyComponent extends Component {
    setup() {
        // Load trước khi component render
        useAssets({
            jsLibs: ["/my_module/static/lib/chart/chart.js"],
            cssLibs: ["/my_module/static/lib/chart/chart.css"],
        });
    }
}
```

### `loadAssets` — One-shot async

```javascript
import { loadAssets } from "@web/core/assets";

async function onButtonClick() {
    await loadAssets({
        jsLibs: ["/web/static/lib/stacktracejs/stacktrace.js"],
        cssLibs: ["/my_module/static/src/css/extra.css"],
    });
    // Sau khi load xong mới dùng thư viện
    doSomethingWithLib();
}
```

**Khi nào lazy load?**
- Thư viện nặng (chart, PDF viewer, rich text editor) chỉ dùng trong một màn hình.
- Asset điều kiện: chỉ cần khi user thực hiện thao tác cụ thể.
- Tránh làm nặng bundle chính cho tất cả người dùng.

## ir.asset — Database-driven Asset Management

Dùng khi cần load asset động mà không cần restart server.

```xml
<!-- data/ir_asset_data.xml -->
<odoo noupdate="1">
    <record id="my_dynamic_asset" model="ir.asset">
        <field name="name">My Custom Dynamic Asset</field>
        <field name="bundle">web.assets_backend</field>
        <field name="path">my_addon/static/src/js/dynamic.js</field>
        <field name="directive">append</field>
        <!-- sequence < 16 → áp dụng TRƯỚC manifest assets -->
        <!-- sequence >= 16 (default) → áp dụng SAU manifest assets -->
        <field name="sequence">16</field>
        <field name="active">True</field>
    </record>
</odoo>
```

**ir.asset fields:**
| Field | Giá trị | Mô tả |
|-------|---------|-------|
| `name` | string | Tên định danh |
| `bundle` | string | Tên bundle target |
| `directive` | append/prepend/before/after/include/remove/replace | Mặc định: `append` |
| `path` | string | Path đến asset file |
| `target` | string | File target (cho before/after/replace) |
| `active` | boolean | Mặc định: `True` |
| `sequence` | integer | Mặc định: `16` — ảnh hưởng thứ tự load |

## Thứ Tự Load

1. `ir.asset` records có `sequence < 16` (TRƯỚC manifest)
2. Assets trong `__manifest__.py` (theo dependency order của modules)
3. `ir.asset` records có `sequence >= 16` (SAU manifest)

## File Deduplication

Nếu cùng một file xuất hiện nhiều lần trong bundle (từ nhiều module hoặc nhiều directive), **chỉ lần đầu được giữ lại**. Các lần sau bị bỏ qua.

## FRD Checklist

Điền bảng này cho mỗi asset cần khai báo trong FRD:

| Hạng mục | Bắt buộc mô tả |
|----------|----------------|
| Bundle | `web.assets_backend`, `web.reports_assets_common`, ... |
| Files | JS/XML/SCSS path đầy đủ, glob nếu dùng |
| Operation | append/prepend/before/after/include/remove/replace |
| Order | Lý do thứ tự load (nếu dùng prepend/before/after) |
| Lazy loading | Asset nào lazy-load, hook nào (`useAssets`/`loadAssets`), trigger |
| Dependency | Module phụ thuộc để bundle target tồn tại |
| ir.asset | Có cần dynamic/conditional load không? Sequence nào? |

## Lưu Ý Quan Trọng Odoo 19

- Key `qweb`, `css`, `js` trong manifest là **legacy Odoo < 15** — không dùng trong Odoo 19.
- XML QWeb template được serve qua `/web/webclient/qweb/` từ filesystem — không bundle vào JS.
- SCSS compile server-side bằng libsass — variables/mixins phải được include đúng thứ tự.
- Compiled bundles lưu dưới dạng `ir.attachment` — tự regenerate khi addon thay đổi.
- `boot.js` trong `web.assets_common` là bắt buộc cho module system — không xóa/replace nếu không có lý do rõ ràng.

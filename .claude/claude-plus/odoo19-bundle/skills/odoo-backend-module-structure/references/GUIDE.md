# Odoo Backend Module Structure — Reference Guide

## directory-layout

```text
my_module/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   └── my_model.py
├── wizards/
│   ├── __init__.py
│   └── my_wizard.py
├── controllers/
│   ├── __init__.py
│   └── main.py
├── security/
│   ├── security.xml          # Groups/categories
│   └── ir.model.access.csv   # ACL — bắt buộc cho mọi model mới
├── data/
│   └── sequence.xml          # noupdate="1" cho production data
├── views/
│   ├── my_model_views.xml
│   └── menu_views.xml
├── report/
│   └── report_action.xml
├── static/
│   ├── description/
│   │   └── icon.png
│   └── src/
│       ├── js/
│       ├── xml/
│       └── scss/
├── i18n/
│   └── vi.po
└── tests/
    ├── __init__.py
    └── test_my_model.py
```

---

## manifest-pattern

```python
# __manifest__.py — Odoo 19 chuẩn
{
    # Bắt buộc (thiếu → warning + fallback)
    "name": "My Module",
    "author": "My Company",
    "license": "LGPL-3",

    # Metadata
    "version": "19.0.1.0.0",        # Bắt buộc đúng format 19.x.y.z (2-5 phần)
    "category": "Operations",
    "summary": "Short description",  # Nếu thiếu, Odoo đọc README file
    "website": "https://example.com",

    # Dependencies — KHÔNG để rỗng []; Odoo 19 ép thành ['base'] và log warning
    "depends": ["base", "mail"],

    # Data files — thứ tự quan trọng (xem Load Order bên dưới)
    "data": [
        "security/security.xml",
        "security/ir.model.access.csv",
        "data/sequence.xml",
        "views/my_model_views.xml",
        "views/menu_views.xml",
        "report/report_action.xml",
    ],

    # Demo data — chỉ load khi chế độ demo bật
    "demo": ["demo/demo.xml"],

    # Assets — dùng key 'assets', KHÔNG dùng 'qweb'/'css'/'js' (deprecated < Odoo 15)
    "assets": {
        "web.assets_backend": [
            "my_module/static/src/js/**/*.js",
            "my_module/static/src/xml/**/*.xml",
            "my_module/static/src/scss/**/*.scss",
        ],
        "web.assets_unit_tests": [
            "my_module/static/tests/**/*",
        ],
    },

    # External dependencies
    "external_dependencies": {
        "python": ["requests>=2.28", "lxml"],  # Tên PyPI hợp lệ (PEP 508), không phải tên import
        "bin": ["wkhtmltopdf"],
    },

    # Lifecycle hooks — nhận 'env' (không phải cr, uid như Odoo <= 16)
    "pre_init_hook": "pre_init",
    "post_init_hook": "post_init",
    "uninstall_hook": "uninstall",

    # auto_install: False | True | list[str] (list phải subset của depends)
    "auto_install": False,

    "installable": True,
    "application": False,
}
```

### Hook definitions trong `__init__.py` (Odoo 19)

```python
# __init__.py

def pre_init(env):
    """Chạy trước khi model được khởi tạo."""
    pass

def post_init(env):
    """Chạy sau khi module được cài đặt."""
    pass

def uninstall(env):
    """Chạy khi module bị gỡ cài đặt."""
    pass
```

### auto_install dạng list

```python
# Tự động cài khi cả 'sale' và 'stock' đều được cài (cả hai phải có trong 'depends')
"depends": ["sale", "stock", "base"],
"auto_install": ["sale", "stock"],
```

### Programmatic manifest access (Odoo 19 preferred API)

```python
# Dùng Manifest.for_addon() — load_manifest() và get_modules_with_version() deprecated từ 19.0
from odoo.modules.module import Manifest

manifest = Manifest.for_addon("sale")
version = manifest.version
depends = manifest["depends"]
```

---

## asset-bundles

### Bảng Bundle Phổ Biến

| Bundle | Khi dùng |
|--------|----------|
| `web.assets_backend` | Web client backend: actions, views, field widgets, OWL templates |
| `web.assets_frontend` | Website/portal frontend |
| `web.assets_common` | Low-level assets chung backend/website/POS — dùng thận trọng |
| `web.assets_unit_tests` | JS unit tests chạy qua `/web/tests` |
| `web.reports_assets_common` | CSS/font cho QWeb PDF reports |

### 7 Bundle Directives

```python
"assets": {
    "web.assets_backend": [
        # 1. append (default) — thêm vào cuối bundle
        "my_module/static/src/js/main.js",

        # 2. prepend — thêm vào đầu bundle
        ("prepend", "my_module/static/src/scss/override.scss"),

        # 3. before — chèn trước một file cụ thể
        ("before", "web/static/src/css/target.scss", "my_module/static/src/css/new.scss"),

        # 4. after — chèn sau một file cụ thể
        ("after", "web/static/src/css/list_view.scss", "my_module/static/src/css/custom.scss"),

        # 5. include — nhúng sub-bundle (prefix _ theo convention Odoo 19)
        ("include", "web._primary_variables"),

        # 6. remove — xóa file khỏi bundle
        ("remove", "web/static/src/js/unwanted.js"),

        # 7. replace — thay thế file tại cùng vị trí
        ("replace", "web/static/src/js/boot.js", "my_module/static/src/js/boot.js"),
    ],
}
```

### Lazy Loading (OWL component — Odoo 19)

```javascript
import { loadAssets, useAssets } from "@web/core/assets";

// One-shot async (bên ngoài component)
await loadAssets({
    jsLibs: ["/web/static/lib/chart/chart.js"],
    cssLibs: ["/my_module/static/src/css/extra.css"],
});

// OWL hook (bên trong setup() của component)
useAssets({
    jsLibs: ["/web/static/lib/chart/chart.js"],
});
```

### ir.asset record (database-driven, không cần restart)

```xml
<record id="my_dynamic_asset" model="ir.asset">
    <field name="name">My Custom Asset</field>
    <field name="bundle">web.assets_backend</field>
    <field name="path">my_module/static/src/js/dynamic.js</field>
    <field name="directive">append</field>
    <field name="sequence">10</field>  <!-- < 16: load TRƯỚC manifest assets -->
</record>
```

**Lưu ý loading order:**
- `ir.asset` records với `sequence < 16` → load trước manifest assets.
- Manifest assets → load theo thứ tự dependency module.
- `ir.asset` records với `sequence >= 16` → load sau manifest assets.
- File trùng lặp: chỉ giữ lần xuất hiện đầu tiên.

---

## frd-stg-checklist

```
[ ] Module name (display) và technical name (snake_case, max 256 ký tự word chars)
[ ] Owner/maintainer và license (LGPL-3 / OPL-1 / ...)
[ ] depends: danh sách kèm lý do cho từng dependency
[ ] Không để depends rỗng (Odoo 19 ép về ['base'] + warning)
[ ] version: đúng format 19.x.y.z
[ ] author: có giá trị (thiếu → warning)
[ ] data files: thứ tự đúng load order (security → data → views → report)
[ ] noupdate="1" cho production data trong <odoo noupdate="1">
[ ] assets bundles và file paths (glob patterns nếu cần)
[ ] external_dependencies: dùng tên PyPI (PEP 508), không phải tên import
[ ] Hooks: pre_init/post_init/uninstall nhận env (không phải cr, uid)
[ ] Security baseline: groups định nghĩa trong security.xml
[ ] ACL: ir.model.access.csv cho mọi model mới
[ ] Test strategy: test file layout trong tests/
[ ] auto_install: False / True / list (nếu list → phải subset của depends)
```

---

## load-order-detail

```
data/: [
    # 1. Groups/categories — phải đứng đầu để ACL tham chiếu được
    "security/security.xml",

    # 2. ACL — sau security.xml vì tham chiếu group
    "security/ir.model.access.csv",

    # 3. Sequences, config data
    "data/ir_sequence.xml",
    "data/config.xml",

    # 4. Views, actions, menus
    "views/my_model_views.xml",
    "views/menu_views.xml",

    # 5. Reports/templates
    "report/report_my_model_template.xml",
    "report/report_my_model_action.xml",
]
```

**Production data với noupdate:**

```xml
<odoo noupdate="1">
    <record id="seq_my_module" model="ir.sequence">
        <field name="name">My Module Sequence</field>
        <field name="code">my.module.seq</field>
        <field name="prefix">MYM/%(year)s/</field>
        <field name="padding">5</field>
    </record>
</odoo>
```

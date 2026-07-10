---
name: odoo-backend-module-structure
description: Hướng dẫn cấu trúc module Odoo 19 phía backend - __manifest__.py, depends, data/demo/assets, load order, Python package layout, security/views/report/static/tests. Use when creating, reviewing, or specifying Odoo module structure and manifest.
---

# Odoo Backend Module Structure

## Goal
Giúp agent tạo/spec module Odoo 19 có manifest, dependency, data load order và static assets đúng chuẩn.

## Source-first khi viết FRD
Trước khi viết FRD-STG hoặc quyết định dependency/load order, dùng `odoo-backend-source-analysis` để đọc `__manifest__.py` của custom module và các base modules liên quan. Không thêm dependency vào FRD nếu chưa có lý do từ source/BRD/PRD.

## Khi nào dùng
- Tạo/scaffold module, review `__manifest__.py`.
- Viết FRD-STG kiến trúc module.
- Xác định dependency, load order, assets bundle, data/demo/test layout.

## Cấu trúc chuẩn
```text
my_module/
├── __init__.py
├── __manifest__.py
├── models/
├── wizards/
├── controllers/
├── security/
│   ├── security.xml
│   └── ir.model.access.csv
├── data/
├── views/
├── report/
├── static/
│   ├── description/icon.png
│   └── src/
│       ├── js/
│       ├── xml/
│       └── scss/
├── i18n/
└── tests/
```

## Manifest Pattern
```python
{
    "name": "My Module",
    "version": "19.0.1.0.0",
    "category": "Operations",
    "license": "LGPL-3",
    "depends": ["base", "mail"],
    "data": [
        "security/security.xml",
        "security/ir.model.access.csv",
        "data/sequence.xml",
        "views/my_model_views.xml",
        "views/menu_views.xml",
        "report/report_action.xml",
    ],
    "demo": ["demo/demo.xml"],
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
    "installable": True,
    "application": False,
}
```

## Load Order
1. Groups/categories in `security/security.xml`.
2. ACL in `security/ir.model.access.csv`.
3. Record rules and metadata dependencies.
4. Sequences/config data in `data/`.
5. Views/actions/menus.
6. Reports/templates.

## Asset Bundles Cần Biết
| Bundle | Khi dùng |
|--------|----------|
| `web.assets_backend` | Web client backend: actions, views, field widgets, OWL templates |
| `web.assets_common` | Low-level assets chung cho backend/website/POS, dùng thận trọng |
| `web.assets_unit_tests` | JS unit tests chạy qua `/web/tests` |
| `web.reports_assets_common` | CSS/font cho QWeb PDF reports |

## FRD-STG Checklist
- Module name, technical name, owner, license.
- Depends kèm lý do cho từng dependency.
- Data files, load order, `noupdate`.
- Assets bundles và file paths.
- Security baseline, groups/rules.
- Test strategy và test file layout.

## References
- Odoo 19 Module manifests: https://www.odoo.com/documentation/19.0/developer/reference/backend/module.html
- Odoo 19 Assets: https://www.odoo.com/documentation/19.0/developer/reference/frontend/assets.html

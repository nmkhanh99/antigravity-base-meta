---
name: odoo-frontend-assets
description: Đặc tả và triển khai frontend assets trong Odoo 19 - __manifest__.py assets bundles, JS/XML/SCSS paths, append/prepend/before/after/include/remove/replace, lazy loading bằng loadAssets/useAssets. Use when writing FRD or code for Odoo frontend assets.
---

# Odoo Frontend Assets

## Goal
Giúp agent mô tả assets trong FRD và khai báo đúng trong Odoo 19 manifest. Dùng khi có JS, XML template, SCSS/CSS hoặc thư viện frontend custom.

## Source-first khi viết FRD
Trước khi ghi assets, dùng `odoo-frontend-source-analysis` để đọc manifest `assets` và file thực tế trong `static/src`. FRD phải ghi bundle, file path, operation và lý do load order dựa trên source.

## Khi nào dùng
- OWL component, field widget, client action cần `static/src`.
- QWeb template XML frontend.
- CSS/SCSS cho web client hoặc report.
- Lazy-load thư viện chỉ dùng trong một màn hình.

## Bundle Chính
| Bundle | Khi dùng |
|--------|----------|
| `web.assets_backend` | Web client backend: views/actions/widgets/OWL templates |
| `web.assets_common` | Low-level common assets, dùng thận trọng |
| `web.assets_unit_tests` | JS tests trong `static/tests` |
| `web.reports_assets_common` | CSS/font cho QWeb PDF reports |

## Operations
- Mặc định string path là `append`.
- Dùng `prepend`, `before`, `after` khi cần thứ tự load cụ thể.
- Dùng `include` cho sub-bundle.
- Dùng `remove`/`replace` chỉ khi thật sự cần override asset có sẵn và module phải depend module khai báo asset gốc.

## Manifest Pattern
```python
"assets": {
    "web.assets_backend": [
        "my_module/static/src/components/my_widget/my_widget.js",
        "my_module/static/src/components/my_widget/my_widget.xml",
        "my_module/static/src/components/my_widget/my_widget.scss",
    ],
    "web.assets_unit_tests": [
        "my_module/static/tests/**/*",
    ],
}
```

## Lazy Loading
Khi thư viện nặng hoặc chỉ dùng trong một màn hình, đặc tả `useAssets`/`loadAssets` thay vì đưa tất cả vào bundle chính.

```javascript
import { useAssets } from "@web/core/assets";

setup() {
    useAssets({
        jsLibs: ["/my_module/static/lib/chart/chart.js"],
        cssLibs: ["/my_module/static/lib/chart/chart.css"],
    });
}
```

## FRD Checklist
| Hạng mục | Bắt buộc mô tả |
|----------|----------------|
| Bundle | `web.assets_backend`, `web.reports_assets_common`, ... |
| Files | JS/XML/SCSS path, glob nếu dùng |
| Operation | append/prepend/before/after/include/remove/replace |
| Order | Lý do thứ tự load |
| Lazy loading | Asset nào lazy-load, trigger nào |
| Dependency | Module phụ thuộc để asset target tồn tại |

## References
- Odoo 19 Assets: https://www.odoo.com/documentation/19.0/developer/reference/frontend/assets.html

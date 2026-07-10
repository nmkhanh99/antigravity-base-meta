---
name: odoo-frontend-assets
description: Đặc tả và triển khai frontend assets trong Odoo 19 - __manifest__.py assets bundles, JS/XML/SCSS paths, append/prepend/before/after/include/remove/replace, lazy loading bằng loadAssets/useAssets. Use when writing FRD or code for Odoo frontend assets.
---

# Odoo Frontend Assets

## Goal
Giúp agent mô tả assets trong FRD và khai báo đúng trong Odoo 19 manifest. Dùng khi có JS, XML template, SCSS/CSS hoặc thư viện frontend custom.

## When to use this skill
- Viết FRD/code cho OWL component, field widget, client action cần `static/src`.
- Khai báo QWeb template XML frontend.
- Thêm CSS/SCSS cho web client hoặc QWeb PDF report.
- Lazy-load thư viện chỉ dùng trong một màn hình cụ thể.
- Override hoặc thay thế asset của module core Odoo.
- Cần hiểu thứ tự load, deduplication hoặc ir.asset database-driven.

## Instructions

### Bước 1 — Source analysis trước khi viết FRD
- Đọc `__manifest__.py` của module liên quan, mục `assets`.
- Kiểm tra file thực tế trong `static/src/` để xác nhận path.
- FRD phải ghi rõ: bundle, file path, operation, lý do load order.

### Bước 2 — Chọn Bundle đúng
Tham khảo bảng bundles chính tại GUIDE.md § Bundle Chính.

### Bước 3 — Khai báo trong `__manifest__.py`
- String path đơn = mặc định `append`.
- Dùng tuple cho các operation khác: `prepend`, `before`, `after`, `include`, `remove`, `replace`.
- Glob pattern được hỗ trợ: `'my_module/static/src/xml/**/*'`.
- Key `assets` thay thế hoàn toàn `qweb`, `css`, `js` từ Odoo < 15 — không dùng key cũ.

### Bước 4 — Lazy loading (khi cần)
- Import `loadAssets` hoặc `useAssets` từ `@web/core/assets`.
- `useAssets` dùng trong OWL `setup()` — load trước khi render.
- `loadAssets` dùng cho one-shot async load ngoài OWL lifecycle.
- Xem code pattern tại GUIDE.md § Lazy Loading.

### Bước 5 — ir.asset (database-driven, khi cần động)
- Dùng `ir.asset` khi cần load asset có điều kiện mà không restart server.
- `sequence < 16` → áp dụng TRƯỚC manifest assets.
- `sequence >= 16` (default) → áp dụng SAU manifest assets.
- Xem XML pattern tại GUIDE.md § ir.asset.

### Bước 6 — FRD Checklist
Điền đầy đủ bảng FRD Checklist tại GUIDE.md § FRD Checklist cho mỗi asset khai báo.

## Constraints
- **Không dùng** key `qweb`, `css`, `js` trong manifest — đây là legacy Odoo < 15, không hỗ trợ trong Odoo 19.
- **File deduplication**: chỉ lần xuất hiện đầu tiên của một file trong bundle được giữ lại — tránh khai báo trùng.
- `remove`/`replace` chỉ dùng khi thật sự cần override; module **phải depend** module khai báo asset gốc.
- `web.assets_common` chứa `boot.js` — bắt buộc load trước mọi OWL/JS module; không remove/replace nếu không hiểu rõ hậu quả.
- SCSS được compile server-side bằng libsass — thứ tự include bundle ảnh hưởng đến cascade; khai báo variables/mixins trước khi dùng.
- XML QWeb template được serve qua `/web/webclient/qweb/` từ filesystem — không bundle vào JS như Odoo < 15.
- Compiled bundles lưu dưới dạng `ir.attachment` — bundle tự regenerate khi addon thay đổi hoặc checksum khác.

## References
- Odoo 19 Assets: https://www.odoo.com/documentation/19.0/developer/reference/frontend/assets.html

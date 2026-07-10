---
name: odoo-api-integration
description: Hướng dẫn toàn diện về XML-RPC, JSON-RPC, REST API, và external API integration trong Odoo 19. Use when the user asks to integrate external API, use XML-RPC, JSON-RPC, build REST endpoint, or connect Odoo to external systems.
---

# Odoo 19 API Integration

## Goal
Giúp agent tích hợp Odoo 19 với hệ thống bên ngoài qua XML-RPC, JSON-RPC, REST API và webhooks — đúng chuẩn Odoo 19, bảo mật, có error handling.

## When to use this skill
- "tích hợp API", "API integration"
- "XML-RPC", "JSON-RPC"
- "REST API", "webhook"
- "external system", "gọi API ngoài"
- "kết nối hệ thống bên ngoài", "sync dữ liệu"
- "authentication API", "API key", "OAuth2"

## Instructions

### Bước 1 — Xác định loại tích hợp

| Loại | Dùng khi |
|------|----------|
| XML-RPC | Client bên ngoài gọi vào Odoo (legacy, stable) |
| JSON-RPC | Client JS/web gọi vào Odoo |
| REST Controller | Xây endpoint HTTP tùy chỉnh trong Odoo |
| External HTTP call | Odoo gọi ra API bên ngoài |
| Webhook receiver | Odoo nhận event từ hệ thống khác |

### Bước 2 — Implement theo loại

Xem code examples chi tiết tại `references/GUIDE.md`:
- **XML-RPC**: Section "XML-RPC API" — authenticate + CRUD operations
- **JSON-RPC**: Section "JSON-RPC API" — Python và JavaScript client
- **REST Controller**: Section "REST API" — `http.Controller`, `@http.route`
- **External call**: Section "External API Integration" — `requests` + config params
- **Webhook**: Section "Webhook Receiver" — validate source, sudo()
- **Authentication**: Section "Authentication" — API key decorator, OAuth2, `@api.private`
- **Best Practices**: Section "Best Practices" — rate limiting, error handling, response format

### Bước 3 — Checklist trước khi hoàn thành

- [ ] Không hardcode credentials — dùng `ir.config_parameter`
- [ ] Có `timeout` cho mọi external API call
- [ ] Webhook có validate source (secret/signature)
- [ ] Có error handling (`try/except`) với logging
- [ ] Method nội bộ nhạy cảm dùng `@api.private` (Odoo 19+)
- [ ] ACL/security đầy đủ cho model và controller

## Constraints
- KHÔNG hardcode credentials, API key, password trong code — luôn dùng `ir.config_parameter` hoặc environment variable.
- Luôn set `timeout` (khuyến nghị 30s) cho mọi `requests` call.
- Webhook endpoint PHẢI validate source qua secret header hoặc signature — không để `auth='none'` mà không có validation.
- Dùng `@api.private` (Odoo 19) để ngăn method nội bộ bị gọi qua RPC.
- REST controller type `'json'` → trả về dict; type `'http'` → dùng `request.make_response()`.
- Không dùng `_cr`, `_uid`, `_context` (deprecated < Odoo 17).

## References
- https://www.odoo.com/documentation/19.0/developer/reference/external_api.html
- https://www.odoo.com/documentation/19.0/developer/reference/external_api.html#xml-rpc-library
- https://www.odoo.com/documentation/19.0/developer/reference/external_api.html#json-rpc-library
- https://www.odoo.com/documentation/19.0/developer/tutorials/web_services.html

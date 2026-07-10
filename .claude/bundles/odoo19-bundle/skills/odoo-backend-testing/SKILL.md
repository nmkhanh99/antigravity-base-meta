---
name: odoo-backend-testing
description: Hướng dẫn viết và mở rộng coverage test backend Odoo 19 - TransactionCase, HttpCase, workflow, report XLSX, migration, tagged CI và Sonar coverage. Use when writing tests, increasing coverage, or fixing backend test gaps for Odoo modules.
---

# Odoo Backend Testing

## Goal
Giúp agent viết test backend Odoo 19 kiểm chứng hành vi nghiệp vụ và nâng coverage đo được trong CI/Sonar.

## When to use this skill
- Viết/mở rộng test cho models, wizards, controllers, reports hoặc migration.
- Tăng coverage Sonar/coverage.py cho module Odoo đang thiếu test.
- Thiết kế fixture, tags và lệnh CI cho một chiến dịch test backend.
- Sửa test fail hoặc test gap sau khi nâng cấp module lên Odoo 19.

## Instructions

1. Đọc `__manifest__.py`, `tests/__init__.py`, test hiện có và CI để xác định dependency, test tags và coverage command.
2. Đọc source cần phủ; lập danh sách hành vi, validation/error branch, side effects và action trả về.
3. Ưu tiên workflow làm thay đổi chứng từ/số lượng trước, rồi strategy/report, controller, helper và migration.
4. Tạo fixture tối thiểu dùng chung trong `tests/common.py` khi từ hai file test trở lên cần cùng dữ liệu.
5. Ghi docstring ngắn cho từng `test_*`, nêu invariant nghiệp vụ hoặc nhánh hồi quy mà test bảo vệ; chỉ thêm inline comment khi setup/fixture có lý do khó suy ra từ code.
6. Viết test theo lớp phù hợp: `TransactionCase` cho ORM/wizard/report, `HttpCase` cho route/tour, upgrade test riêng cho migration.
7. Gắn tag mà pipeline thực sự chạy; nếu CI dùng `--test-tags=logic`, mọi test cần coverage phải có `@tagged('logic', 'post_install', '-at_install')`.
8. Chạy test mục tiêu trước, sau đó chạy cùng lệnh coverage của CI và đọc uncovered lines trước khi mở rộng wave tiếp theo.

Xem chi tiết patterns, templates và coverage matrix tại `references/GUIDE.md`.

## Constraints

- **Odoo 19**: Import `Form` từ `odoo.tests`, KHÔNG từ `odoo.tests.common` (đã deprecated từ 18.0).
- **Odoo 19**: `TransactionCase` tự động wrap mỗi test method trong savepoint — không cần `SavepointCase` riêng.
- **Odoo 19**: Commit/rollback/close cursor bên trong `TransactionCase` sẽ raise `AssertionError` — không gọi `self.env.cr.commit()`.
- **Odoo 19**: HTTP request ra ngoài bị block mặc định trong standard/click_all tests — chỉ localhost và file:// được phép.
- **Tags**: Mỗi test phải có `at_install` HOẶC `post_install`, không được có cả hai, không được không có cái nào.
- **Tags**: Không dùng tag mà CI không thu thập; coverage local phải khớp command CI.
- **Assertions**: Luôn assert output quan sát được (field/state/record/action/workbook cells/response) — không chỉ gọi method để phủ dòng.
- **Mock**: Mock tại boundary tốn kém hoặc không ổn định; không mock logic đang cần kiểm chứng.
- **Migration**: Không coi migration được phủ bởi test cài mới — cần database upgrade test hoặc ghi rõ exclusion được chấp thuận.
- **Demo data**: Không phụ thuộc demo data nếu không khai báo rõ; tạo data trong `setUpClass`.
- **Production logic**: Không đổi production logic để test dễ viết trừ khi phát hiện bug riêng và báo rõ.
- **Frontend**: OWL/JS unit test dùng `odoo-frontend-testing-hoot`, không trộn vào workflow backend này.
- **domain**: Dùng `=` thay vì `==` trong domain; dùng `_read_group` thay cho `read_group` (deprecated).

## References
- Odoo 19 Testing: https://www.odoo.com/documentation/19.0/developer/reference/backend/testing.html
- GUIDE.md: `.claude/skills/odoo-backend-testing/references/GUIDE.md`

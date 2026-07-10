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

## Instructions
1. Đọc `__manifest__.py`, `tests/__init__.py`, test hiện có và CI để xác định dependency, test tags và coverage command.
2. Đọc source cần phủ; lập danh sách hành vi, validation/error branch, side effects và action trả về.
3. Ưu tiên workflow làm thay đổi chứng từ/số lượng trước, rồi strategy/report, controller, helper và migration.
4. Tạo fixture tối thiểu dùng chung trong `tests/common.py` khi từ hai file test trở lên cần cùng dữ liệu.
5. Ghi docstring ngắn cho từng `test_*`, nêu invariant nghiệp vụ hoặc nhánh hồi quy mà test bảo vệ; chỉ thêm inline comment khi setup/fixture có lý do khó suy ra từ code.
6. Viết test theo lớp phù hợp: `TransactionCase` cho ORM/wizard/report, `HttpCase` cho route/tour, upgrade test riêng cho migration.
7. Gắn tag mà pipeline thực sự chạy; nếu CI dùng `--test-tags=logic`, mọi test cần coverage phải có `@tagged('logic', 'post_install', '-at_install')`.
8. Chạy test mục tiêu trước, sau đó chạy cùng lệnh coverage của CI và đọc uncovered lines trước khi mở rộng wave tiếp theo.

## Test rules
- Test file đặt trong `tests/`, tên bắt đầu `test_`, import trong `tests/__init__.py`.
- Không phụ thuộc demo data nếu không khai báo rõ; tạo data trong `setUpClass`.
- Kiểm chứng output quan sát được: field/state/record/action/workbook cells hoặc response; không chỉ gọi method để phủ dòng.
- Với workflow, luôn có ít nhất một đường thành công và một validation/error branch quan trọng.
- Mock tại boundary tốn kém hoặc không ổn định (`stock.rule.run`, attachment/download, external file); không mock logic đang cần kiểm chứng.
- Không coi migration được phủ bởi test cài mới: chạy database upgrade test hoặc ghi rõ exclusion được chấp thuận.
- Docstring của test phải giải thích hành vi cần giữ ổn định, không lặp lại thao tác hiển nhiên như create/call/assert.
- Khi test kích hoạt warning Odoo 19 trong module đang sửa, kiểm tra và ưu tiên API hiện hành: domain dùng `=`, phép nhóm backend dùng `_read_group` thay cho `read_group`.
- Với controller `@http.route`, unit test payload qua `original_endpoint` khi mock `request`; dùng `HttpCase` hoặc response thật nếu cần kiểm chứng dispatcher/auth/routing.

## Patterns
```python
from odoo.exceptions import AccessError, ValidationError
from odoo.tests import TransactionCase, HttpCase, tagged


@tagged("logic", "post_install", "-at_install")
class TestMyModel(TransactionCase):
    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        cls.record = cls.env["my.model"].create({"name": "Test"})

    def test_confirm_changes_state(self):
        """Confirm must publish the record as a confirmed business document."""
        self.record.action_confirm()
        self.assertEqual(self.record.state, "confirmed")

    def test_constraint_blocks_negative_qty(self):
        """Negative quantities must be rejected before they enter planning."""
        with self.assertRaises(ValidationError):
            self.record.write({"qty": -1})


@tagged("logic", "post_install", "-at_install")
class TestMyHttp(HttpCase):
    def test_portal_route(self):
        response = self.url_open("/my/records")
        self.assertEqual(response.status_code, 200)
```

## Coverage campaign matrix
| Area | Test focus |
|------|------------|
| Workflow/model | State transitions, quantities, created/linked records, constraint failures |
| Wizard | Defaults, filtering, confirmation side effects, invalid selection |
| Report/XLSX | Aggregation plus workbook sheet/header/value or attachment action |
| Controller | Route auth/input via `HttpCase`; generated payload through endpoint implementation |
| Migration | Upgrade from prior schema/data and post-migration invariant |
| Multi-company/security | Allowed company/user and blocked access path |

## Coverage command template
```bash
python3 -m coverage run --source=/path/to/addons/my_module /path/to/odoo-bin \
    -d test_db --addons-path=/path/to/addons \
    -i my_module --test-enable --test-tags=logic --stop-after-init
python3 -m coverage xml -o coverage.xml
python3 -m coverage report --show-missing --omit='*/tests/*'
```

## Constraints
- Không thêm test chỉ để chạy dòng mà không assertion về behavior.
- Không đổi production logic để test dễ viết trừ khi phát hiện bug riêng và báo rõ.
- Không dùng tag mà CI không thu thập; coverage local phải khớp command CI.
- Frontend OWL/JS unit test dùng `odoo-frontend-testing-hoot`, không trộn vào workflow backend này.

## Best practices
- Chia wave theo business flow, đo coverage lại sau mỗi wave và ưu tiên uncovered lines còn lại.
- Dùng builder/helper fixture nhỏ thay vì một `setUpClass` tạo toàn bộ dữ liệu không cần thiết.
- Giữ report tests ở hai tầng: aggregation thuần ORM và smoke test workbook/download action.
- Cài `coverage` trong cùng interpreter chạy Odoo; ghi report local vào thư mục tạm để tránh artifact trong module.
- Nếu runner local giữ HTTP thread sau khi đã in kết quả test, chỉ shutdown có kiểm soát sau khi xác nhận test xong để coverage ghi dữ liệu.

## References
- Odoo 19 Testing: https://www.odoo.com/documentation/19.0/developer/reference/backend/testing.html

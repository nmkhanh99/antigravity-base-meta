# Odoo Backend Testing — Reference Guide

## Base patterns

### TransactionCase (ORM / Wizard / Report)

```python
from odoo.exceptions import AccessError, ValidationError
from odoo.tests import Form, TransactionCase, tagged


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

    def test_access_blocked_for_portal(self):
        """Portal users must not read internal records."""
        portal = self.env.ref("base.group_portal").users[0]
        with self.assertRaises(AccessError):
            self.record.with_user(portal).read(["name"])
```

### HttpCase (Route / Tour)

```python
from odoo.tests import HttpCase, tagged


@tagged("post_install", "-at_install")
class TestMyHttp(HttpCase):
    def test_portal_route_returns_200(self):
        """Portal /my/records route must be accessible to authenticated portal users."""
        self.authenticate("portal", "portal")
        response = self.url_open("/my/records")
        self.assertEqual(response.status_code, 200)

    def test_my_tour(self):
        """End-to-end tour validates the full create → confirm flow in the browser."""
        self.start_tour("/web", "my_tour_name", login="admin")
```

### Form emulator (Odoo 19)

```python
# QUAN TRỌNG Odoo 19: import Form từ odoo.tests, KHÔNG từ odoo.tests.common
from odoo.tests import Form

def test_onchange_sets_default_pricelist(self):
    """Selecting a customer must auto-fill the correct pricelist via onchange."""
    with Form(self.env["sale.order"]) as f:
        f.partner_id = self.partner
    so = f.record
    self.assertEqual(so.pricelist_id, self.expected_pricelist)
```

### new_test_user helper

```python
from odoo.tests.common import new_test_user

@classmethod
def setUpClass(cls):
    super().setUpClass()
    cls.user_manager = new_test_user(
        cls.env,
        login="test_manager",
        groups="my_module.group_manager,base.group_user",
    )
```

### assertRecordValues

```python
def test_invoice_lines(self):
    """Invoice lines must reflect correct product, qty and price after confirm."""
    self.assertRecordValues(
        self.invoice.invoice_line_ids,
        [
            {"product_id": self.product.id, "quantity": 2.0, "price_unit": 100.0},
            {"product_id": self.product2.id, "quantity": 1.0, "price_unit": 50.0},
        ],
    )
```

### assertQueryCount + warmup

```python
from odoo.tests import tagged, TransactionCase
from odoo.tests.common import warmup


@tagged("perf", "post_install", "-at_install")
class TestPerf(TransactionCase):
    @warmup
    def test_search_query_count(self):
        """Searching partners must complete in at most 3 queries for admin."""
        with self.assertQueryCount(admin=3):
            self.env["res.partner"].search([("is_company", "=", True)])
```

### freeze_time

```python
from odoo.tests import tagged, TransactionCase
from odoo.tests.common import freeze_time


@tagged("logic", "post_install", "-at_install")
@freeze_time("2025-01-15 08:00:00")
class TestScheduledActions(TransactionCase):
    def test_reminder_sent_on_due_date(self):
        """Reminder cron must mark activity as done when run on its due date."""
        self.activity.action_feedback(feedback="done")
        self.assertEqual(self.activity.state, "done")
```

### RecordCapturer

```python
from odoo.tests.common import RecordCapturer

def test_wizard_creates_picking(self):
    """Wizard must create exactly one stock picking upon confirmation."""
    with RecordCapturer(self.env["stock.picking"], []) as cap:
        self.wizard.action_confirm()
    self.assertEqual(len(cap.records), 1)
    self.assertEqual(cap.records.state, "assigned")
```

### standalone (install/uninstall test)

```python
from odoo.tests.common import standalone


@standalone("my_module_install")
def test_post_install_data(env):
    """After install, default configuration record must exist."""
    config = env["my.module.config"].search([("is_default", "=", True)])
    assert len(config) == 1, "Expected exactly one default config after install"
```

### Controller unit test (mock request)

```python
from unittest.mock import patch, MagicMock
from odoo.tests import TransactionCase, tagged


@tagged("logic", "post_install", "-at_install")
class TestMyController(TransactionCase):
    def test_endpoint_payload_processing(self):
        """Controller must return 200 and store the payload when input is valid."""
        from odoo.addons.my_module.controllers.main import MyController

        controller = MyController()
        mock_request = MagicMock()
        mock_request.env = self.env
        mock_request.httprequest.method = "POST"

        with patch("odoo.addons.my_module.controllers.main.request", mock_request):
            result = controller.original_endpoint(payload={"key": "value"})

        self.assertEqual(result["status"], "ok")
```

---

## tests/common.py — shared fixture pattern

```python
# tests/common.py
from odoo.tests import TransactionCase


class MyModuleCommon(TransactionCase):
    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        cls.company = cls.env.ref("base.main_company")
        cls.partner = cls.env["res.partner"].create({
            "name": "Test Customer",
            "company_id": cls.company.id,
        })
        cls.product = cls.env["product.product"].create({
            "name": "Test Product",
            "type": "consu",
            "list_price": 100.0,
        })
```

```python
# tests/test_sale.py
from .common import MyModuleCommon
from odoo.tests import tagged


@tagged("logic", "post_install", "-at_install")
class TestSale(MyModuleCommon):
    def test_sale_order_confirm(self):
        """Sale order must move to 'sale' state after confirmation."""
        so = self.env["sale.order"].create({
            "partner_id": self.partner.id,
            "order_line": [(0, 0, {
                "product_id": self.product.id,
                "product_uom_qty": 1,
                "price_unit": 100,
            })],
        })
        so.action_confirm()
        self.assertEqual(so.state, "sale")
```

---

## tests/__init__.py — registering test files

```python
# tests/__init__.py
from . import (
    test_model,
    test_wizard,
    test_report,
    test_controller,
)
```

---

## XLSX / Report tests

```python
import base64
import io

from odoo.tests import TransactionCase, tagged


@tagged("logic", "post_install", "-at_install")
class TestMyReport(TransactionCase):
    def test_report_aggregation(self):
        """Report must sum quantities correctly per product category."""
        result = self.env["my.report.wizard"].create({
            "date_from": "2025-01-01",
            "date_to": "2025-12-31",
        })._get_data()
        category_totals = {row["category"]: row["qty"] for row in result}
        self.assertAlmostEqual(category_totals.get("All"), 10.0)

    def test_report_workbook_structure(self):
        """XLSX export must produce one sheet with correct headers."""
        import openpyxl

        wizard = self.env["my.report.wizard"].create({
            "date_from": "2025-01-01",
            "date_to": "2025-12-31",
        })
        action = wizard.action_export_xlsx()
        attachment = self.env["ir.attachment"].search(
            [("res_model", "=", "my.report.wizard"), ("res_id", "=", wizard.id)],
            limit=1,
        )
        wb = openpyxl.load_workbook(io.BytesIO(base64.b64decode(attachment.datas)))
        ws = wb.active
        self.assertEqual(ws.cell(1, 1).value, "Category")
        self.assertEqual(ws.cell(1, 2).value, "Quantity")
```

---

## Multi-company / security tests

```python
@tagged("security", "post_install", "-at_install")
class TestMultiCompany(TransactionCase):
    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        cls.company_a = cls.env.ref("base.main_company")
        cls.company_b = cls.env["res.company"].create({"name": "Company B"})
        cls.user_a = new_test_user(
            cls.env,
            login="user_a",
            groups="base.group_user",
            company_id=cls.company_a.id,
            company_ids=[(6, 0, [cls.company_a.id])],
        )

    def test_record_invisible_across_companies(self):
        """A record in company A must not be accessible by a user of company B."""
        record = self.env["my.model"].with_company(self.company_a).create({
            "name": "A-only record",
            "company_id": self.company_a.id,
        })
        result = self.env["my.model"].with_user(self.user_a).search(
            [("company_id", "=", self.company_b.id)]
        )
        self.assertNotIn(record, result)
```

---

## Coverage campaign matrix

| Area | Test focus |
|------|------------|
| Workflow / model | State transitions, quantities, created/linked records, constraint failures |
| Wizard | Defaults, filtering, confirmation side effects, invalid selection |
| Report / XLSX | Aggregation thuần ORM + smoke test workbook sheet/header/value |
| Controller | Route auth/input via `HttpCase`; payload qua `original_endpoint` khi mock `request` |
| Migration | Upgrade from prior schema/data and post-migration invariant |
| Multi-company / security | Allowed company/user và blocked access path |
| Performance | `assertQueryCount` + `@warmup` cho hot paths |

---

## Coverage command template

```bash
# Chạy test với coverage
python3 -m coverage run --source=/path/to/addons/my_module \
    /path/to/odoo-bin \
    -d test_db \
    --addons-path=/path/to/addons \
    -i my_module \
    --test-enable \
    --test-tags=logic \
    --stop-after-init

# Xuất báo cáo
python3 -m coverage xml -o coverage.xml
python3 -m coverage report --show-missing --omit='*/tests/*'
```

> **Lưu ý**: Cài `coverage` trong cùng interpreter chạy Odoo. Ghi report local vào thư mục tạm để tránh artifact trong module.
> Nếu runner local giữ HTTP thread sau khi in kết quả test, chỉ shutdown có kiểm soát sau khi xác nhận test xong để coverage ghi dữ liệu.

---

## CLI test selector syntax

```bash
# Chạy tất cả standard tests
python odoo-bin --test-tags=standard

# Chạy post_install tests của một module
python odoo-bin --test-tags=post_install/my_module

# Chạy một class cụ thể
python odoo-bin --test-tags=/my_module:TestMyModel

# Chạy một method cụ thể
python odoo-bin --test-tags=/my_module:TestMyModel.test_confirm_changes_state

# Kết hợp nhiều tag (comma-separated)
python odoo-bin --test-tags=logic,security
```

---

## Odoo 19 — Thay đổi so với các phiên bản cũ

| Điểm thay đổi | Odoo <= 17 | Odoo 19 |
|---------------|-----------|---------|
| Form import | `from odoo.tests.common import Form` | `from odoo.tests import Form` |
| Savepoint per method | `SavepointCase` riêng biệt | `TransactionCase` mặc định |
| Cursor commit trong test | Cho phép (nguy hiểm) | Raise `AssertionError` |
| HTTP external requests | Có thể thực hiện | Bị block mặc định |
| `read_group` | Dùng trực tiếp | Deprecated → dùng `_read_group` |
| Domain operator | `==` được chấp nhận | Dùng `=` |

---

## Checklist trước khi commit test

- [ ] Import `Form` từ `odoo.tests`, không từ `odoo.tests.common`
- [ ] Mỗi test class có tag `at_install` HOẶC `post_install` (không cả hai)
- [ ] Không gọi `self.env.cr.commit()` trong test
- [ ] Data test tạo trong `setUpClass`, không phụ thuộc demo data
- [ ] Mỗi `test_*` có docstring nêu invariant nghiệp vụ
- [ ] Test file được import trong `tests/__init__.py`
- [ ] Coverage command local khớp với CI pipeline
- [ ] Không có test chỉ gọi method mà không assert behavior

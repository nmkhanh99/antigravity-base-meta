# DOCX Mapping Analysis — Reference Guide

## 1. Ma trận Ánh xạ mẫu (Mapping Matrix Template)

Trình bày bảng này để người dùng phê duyệt trước khi sinh JSON:

| # | Vị trí trong DOCX (Tiêu đề thực tế) | Element Type | Nguồn dữ liệu từ MD | Chiến lược thực thi | Điểm mập mờ / Hành động |
|---|--------------------------------------|--------------|---------------------|---------------------|--------------------------|
| 1 | Bảng "Danh mục tham chiếu" (xuất hiện lần 1) | Table | Section 2.1 - Reference List | Điền đủ tất cả items | — |
| 2 | Bảng "Danh mục tham chiếu" (xuất hiện lần 2) | Table | — | **[DELETE]** Bảng dư thừa + tiêu đề | Hỏi: bạn muốn dồn hay xóa? |
| 3 | Mục "Tree View" (Heading only) | Paragraph | Section 3.2 - OWL Logic | **Đề xuất**: Tạo bảng 2 cột [UI Component / Logic] | Cần xác nhận trước khi tạo |
| 4 | `{project_name}` | Text | doc.title (H1) | Transform: `none` | Required — báo lỗi nếu thiếu |
| 5 | `{table_requirements}` | Table Loop | requirements[] | Transform: `table_loop` | Số rows phụ thuộc data |
| 6 | `{logo}` | Image | company.logo | Transform: `image` | Optional — bỏ qua nếu thiếu |

---

## 2. Quy tắc đặt câu hỏi làm rõ (Human-Readable Questions)

### Nghiêm cấm
- "Table Index 4 có vấn đề"
- "Offset 2 bị trùng"
- "Paragraph 15 thiếu dữ liệu"

### Bắt buộc — dùng tiêu đề thực tế và ngữ cảnh

**Dựa trên Tiêu đề:**
> "Bảng 'Danh mục tham chiếu' xuất hiện 2 lần trong template. MD chỉ có 1 danh sách. Bạn muốn dồn vào bảng đầu và xóa bảng sau, hay giữ lại cả hai?"

**Dựa trên Ý nghĩa:**
> "Tôi thấy mục 'Luồng dữ liệu' trong MD rất dài nhưng template chỉ có 1 paragraph ngắn. Tôi nên cắt bớt hay tạo thêm bảng đối soát?"

**Dựa trên Thiết kế:**
> "Bạn muốn trình bày phần Logic bảo mật (ACL/Record Rules) chi tiết cho từng chức năng hay dồn về bảng tổng ở cuối tài liệu?"

**Khi template thiếu slot:**
> "Template chỉ có heading 'OWL Component Logic' mà không có bảng. Tôi đề xuất tạo bảng 2 cột [UI Component | Logic] để trình bày kỹ thuật. Bạn đồng ý không?"

---

## 3. Odoo 19 Mapping Patterns

### Pattern 1 — Reference Fields (Danh mục tham chiếu)
- Template thường có nhiều slot (Table 4, 5 cùng mục đích).
- Nếu MD chỉ có 1 bảng tổng → Gộp vào bảng đầu tiên, đánh dấu `[DELETE]` cho bảng còn lại.

### Pattern 2 — Data Design (Thiết kế Dữ liệu)
- Template thường có 4 bảng: Header model, Line model, Wizard/Other, Spare.
- Map theo thứ tự: T1=Header, T2=Line, T3=Wizard, T4=`[DELETE]` nếu dư.
- Fields kỹ thuật cần điền đầy đủ: `name`, `field_description`, `ttype`, `comodel_name`, `required`, `readonly`, `index`, `compute`, `inverse`, `store`, `help`, `default`.

### Pattern 3 — UI/OWL Specs
- Nếu template chỉ có Heading (VD: "Tree View") mà không có bảng → Đề xuất tạo bảng Spec 2 cột.
- Cấu trúc bảng đề xuất: `[UI Component | Logic / RPC / Client Action]`.
- Mô tả nút bấm kèm: Actor (ai bấm), State change (trạng thái trước → sau), Method/RPC gọi.

### Pattern 4 — Security Matrix (Bảng phân quyền)
- Mặc định điền chi tiết vào từng khối chức năng (module hóa).
- Mỗi model cần: Group, Read, Write, Create, Delete.
- Odoo 19: dùng `ir.model.access.csv` + `record.rule` XML riêng biệt.

### Pattern 5 — Business Logic Paragraphs
Mỗi đối tượng (Header/Line) cần đủ 3 Paragraph:
1. **Mô tả / Mô hình**: Giải thích đối tượng là gì, liên kết với model nào.
2. **Logic nghiệp vụ**: Workflow, compute, onchange, constraint.
3. **Trigger / Event**: Điều kiện kích hoạt, action tự động.

---

## 4. Fill Instructions JSON Template

```json
{
  "version": "1.0",
  "template_file": "template.docx",
  "output_file": "output.docx",
  "mappings": [
    {
      "type": "text_replace",
      "placeholder": "{project_name}",
      "source": "title",
      "transform": "none",
      "required": true
    },
    {
      "type": "table_loop",
      "placeholder": "{table_requirements}",
      "source": "requirements",
      "transform": "table_loop",
      "columns": ["id", "name", "description", "priority"],
      "required": false
    },
    {
      "type": "image_replace",
      "placeholder": "{logo}",
      "source": "company.logo",
      "transform": "image",
      "required": false,
      "fallback": "skip"
    }
  ],
  "fill_tables": [
    {
      "table_context": "Bảng 'Thiết kế Dữ liệu' - Header Model",
      "table_offset": 0,
      "delete": false,
      "rows": [
        {
          "field": "name",
          "description": "Tên bản ghi",
          "ttype": "Char",
          "required": true,
          "help": "Dùng làm display name"
        }
      ]
    },
    {
      "table_context": "Bảng 'Danh mục tham chiếu' lần 2 (dư thừa)",
      "table_offset": 1,
      "delete": true
    }
  ],
  "replace_paragraphs": [
    {
      "context": "Tiêu đề 'Danh mục tham chiếu' lần 2 (đi kèm bảng bị xóa)",
      "paragraph_index": 42,
      "delete": true
    },
    {
      "context": "Placeholder mặc định '<BA cân nhắc...>'",
      "paragraph_index": 55,
      "new_text": "",
      "delete": true
    }
  ]
}
```

---

## 5. Redundancy Analysis Checklist

Trước khi gửi Ma trận cho người dùng, kiểm tra:

- [ ] Đếm số bảng phục vụ cùng mục đích trong template.
- [ ] So sánh với số lượng data tương ứng trong MD.
- [ ] Đánh dấu `[DELETE]` rõ ràng trong Ma trận cho bảng dư.
- [ ] Kiểm tra còn văn bản placeholder mặc định không: `<BA cân nhắc...>`, `BR-xxx-yy`, `TÊN DANH MỤC`.
- [ ] Xác nhận mọi `delete: true` đều có cả hai: `fill_tables` entry và `replace_paragraphs` entry (cho tiêu đề đi kèm).

---

## 6. Paragraph Mapping Audit Checklist

Tự kiểm tra bắt buộc trước khi gửi JSON cuối:

- [ ] **Kỹ thuật**: Mỗi đối tượng (Header/Line) có đủ 3 Paragraph: Mô tả, Logic, Trigger.
- [ ] **Diễn giải (Elaborate)**: Không sao chép nguyên văn MD nếu quá ngắn — dùng văn phong Odoo 19 (VD: "Sử dụng `Many2one` link tới `res.partner`", "Hệ thống tự động tính qua method `_compute_total`").
- [ ] **UI/Actions**: Nút bấm mô tả kèm Actor, Logic state change, Method/RPC trong từng View.
- [ ] **Cleanup**: Paragraph dư từ bảng bị xóa đã được đánh dấu `delete: true` hoặc ghi đè bằng `""`.

---

## 7. Xử lý tài liệu lớn (Large Docs Strategy)

1. Chia MD thành Functional Units theo H2/H3.
2. Xây dựng mapping cho từng unit độc lập.
3. Đảm bảo `table_offset` nhất quán trong phạm vi mỗi block clone.
4. Sinh JSON theo từng unit, sau đó merge lại.
5. Kiểm tra không có `table_offset` bị trùng hoặc bị nhảy số.

---

## 8. Transform Types

| Transform | Mô tả | Khi dùng |
|-----------|-------|-----------|
| `none` | Điền text thẳng vào placeholder | Text đơn giản: tên, ngày, mã số |
| `table_loop` | Lặp rows theo array data | Danh sách requirements, fields, items |
| `conditional` | Hiển thị/ẩn theo điều kiện | Section chỉ xuất hiện khi có data |
| `image` | Chèn ảnh từ path hoặc base64 | Logo, ảnh minh họa |
| `markdown_to_docx` | Convert MD text sang DOCX format | Mô tả dài có formatting |
| `date_format` | Format ngày theo locale | Ngày tháng năm |

---
name: odoo-docx-mapping
description: Hướng dẫn phân tích chuyên sâu mối quan hệ giữa Template DOCX và Markdown, lập Ma trận Ánh xạ và thực hiện giao tiếp làm rõ với người dùng. Use when the user asks to map DOCX template, analyze document structure, or create mapping matrix.
---

# DOCX Mapping Analysis

## Goal
Giúp agent chuyển đổi từ "điền dữ liệu máy móc" sang "hiểu và thiết kế tài liệu chuyên nghiệp" bằng cách phân tích mối quan hệ giữa Template DOCX và Markdown data, tạo ma trận ánh xạ chính xác, và làm rõ mọi điểm mập mờ trước khi sinh `fill_instructions.json`.

## When to use this skill
- "phân tích DOCX", "mapping DOCX"
- "ánh xạ template", "ma trận mapping"
- "DOCX to Markdown", "template analysis"
- "tạo fill instructions", "điền dữ liệu vào template"
- "phân tích bảng DOCX", "xác định placeholder"

## Instructions

### Bước 1 — Phân tích Template DOCX (Template First)
Luôn phân tích template trước, markdown sau.
- Đọc tiêu đề (Header/Caption) và ngữ cảnh xung quanh mỗi bảng — không chỉ đếm số thứ tự.
- Xác định tất cả placeholders: `{field_name}`, merge fields, text động.
- Phân loại từng element: `text`, `table_loop`, `conditional`, `image`.
- Xác định sections lặp lại (loops) và sections có điều kiện.

### Bước 2 — Trích xuất dữ liệu Markdown (Deep Extraction)
- Quét MD theo từng Functional Unit (H2/H3).
- Trích xuất **tất cả** trường dữ liệu và tham số kỹ thuật — không tóm tắt, không dùng ví dụ sơ sài.
- Lập danh sách sections/data có sẵn để so sánh với template.

### Bước 3 — Lập Ma trận Ánh xạ (Mapping Matrix)
Trình bày Ma trận để người dùng phê duyệt **trước khi** sinh JSON.
Xem mẫu chi tiết tại `references/GUIDE.md` — Mục 1.

| Vị trí trong DOCX (Tiêu đề thực tế) | Nguồn dữ liệu từ MD | Chiến lược / Điểm mập mờ |
|--------------------------------------|---------------------|--------------------------|
| VD: Bảng "Thiết kế dữ liệu" đầu tiên | Section 2.1.4 - Bảng Header | Điền đủ 12 fields kỹ thuật. |
| VD: Mục "Tree View" | Section 2.1.5 - OWL Logic | **Đề xuất**: Tạo bảng 2 cột [UI Component / Logic]. |

### Bước 4 — Phân tích Thừa/Thiếu (Redundancy Analysis)
- So sánh số bảng phục vụ cùng mục đích giữa Template và MD.
- Đánh dấu `[DELETE]` trong Ma trận cho các bảng dư thừa.
- Dùng `delete: true` trong JSON cho cả `replace_paragraphs` (tiêu đề) và `fill_tables` (bảng).
- Không để lại văn bản placeholder như `<BA cân nhắc...>`, `BR-xxx-yy`.

### Bước 5 — Clarification Gate (Cổng kiểm soát)
Người dùng phê duyệt Ma trận Ánh xạ là **điều kiện tiên quyết** để sinh JSON.
Khi đặt câu hỏi làm rõ: dùng tiêu đề thực tế, **KHÔNG** dùng "Table Index 4", "Offset 2", "Paragraph 15".
Xem quy tắc đặt câu hỏi tại `references/GUIDE.md` — Mục 2.

### Bước 6 — Sinh fill_instructions.json
Chỉ thực hiện sau khi Ma trận được phê duyệt.
Tham khảo cấu hình chuẩn tại:
`/Users/khanhnm/Desktop/odoo-19.0/business-specs/templates/fill_instructions_template.json`

### Bước 7 — Paragraph Mapping Audit (Tự kiểm tra bắt buộc)
Trước khi gửi JSON, tự kiểm tra:
- [ ] Mỗi đối tượng Header/Line đã có đủ 3 Paragraph: Mô tả, Logic nghiệp vụ, Trigger.
- [ ] Không sao chép nguyên văn MD nếu quá ngắn — dùng văn phong Odoo 19.
- [ ] Các nút bấm mô tả kèm Actor và Logic state change cụ thể.
- [ ] Paragraph dư từ bảng bị xóa đã được đánh dấu xóa hoặc ghi đè rỗng.

## Constraints
- Mapping PHẢI cover 100% placeholders trong template — không được bỏ sót.
- PHẢI trình bày và nhận phê duyệt Ma trận trước khi sinh JSON.
- CẤM dùng thuật ngữ kỹ thuật ("Table Index 4", "Paragraph 15") khi hỏi người dùng.
- Nếu Template thiếu slot cho dữ liệu quan trọng (OWL Specs, Business Logic) → chủ động đề xuất tạo bảng mới, không bỏ qua.
- Xác định rõ transform type cho mỗi placeholder: `none`, `table_loop`, `conditional`, `image`.
- Không để lại văn bản placeholder mặc định trong output cuối.
- Khi tài liệu lớn: xử lý theo từng Functional Unit (H2/H3), đảm bảo nhất quán `table_offset`.

## References
- `references/GUIDE.md` — Ma trận mẫu, quy tắc hỏi, Odoo patterns, mẫu JSON
- `/Users/khanhnm/Desktop/odoo-19.0/business-specs/templates/fill_instructions_template.json` — Cấu hình JSON chuẩn

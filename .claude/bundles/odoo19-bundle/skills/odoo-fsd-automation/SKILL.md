---
name: odoo-fsd-automation
description: Hệ thống tự động hóa FSD - tạo tài liệu Functional Specification Document từ Markdown sang Corporate DOCX Template. Use when the user asks to automate FSD creation, generate DOCX from markdown, or build document automation pipeline.
---

# FSD Automation

## Goal
Tự động hóa việc tạo tài liệu Functional Specification Document (FSD) từ Markdown sang Corporate DOCX Template, phục vụ dự án Odoo 19.

## When to use this skill
- "tạo FSD", "generate FSD"
- "markdown to DOCX", "document automation"
- "FSD pipeline", "automated documentation"
- "điền template DOCX", "xuất FSD từ markdown"
- "chạy md_to_docx", "fill DOCX từ data"

## Instructions

### 1. Pipeline Overview
```
Markdown Data → Analyze → Map to Template → Fill DOCX → Verify → Output
```

### 2. Chuẩn bị
- Template DOCX: chứa anchor/placeholder dạng `[TÊN_PLACEHOLDER:]`
- File Markdown nguồn: chứa heading, table, frontmatter
- Mapping Plan: file `.md` định nghĩa ánh xạ anchor → nguồn dữ liệu

Chi tiết cú pháp và ví dụ → xem `references/GUIDE.md`.

### 3. Quy trình thực thi
1. **Phân tích template**: Parse DOCX → extract tất cả placeholder
2. **Phân tích data**: Parse Markdown → extract sections, tables, metadata
3. **Mapping**: Tạo mapping matrix (placeholder → data source)
4. **Fill**: Chạy engine điền nội dung vào template
5. **Verify**: Kiểm tra output — 100% placeholder phải được điền

### 4. Lệnh thực thi chuẩn
```bash
python3 business-specs/scripts/md_to_docx.py \
  -i business-specs/KHSX/PLA-00_fsd_khsx_master.md \
  -t business-specs/KHSX/FSD.docx \
  -o business-specs/KHSX/PLA-00_fsd_khsx_master.docx \
  -p business-specs/KHSX/mapping_plan.md
```

### 5. Mapping Matrix (JSON)
```json
{
  "document_title": {"source": "h1", "transform": "text"},
  "table_requirements": {"source": "section_3.table", "transform": "table"},
  "version": {"source": "frontmatter.version", "transform": "text"}
}
```

### 6. Kiểm tra kết quả
- Mở file `.docx` output và xác nhận không còn placeholder nào chưa điền.
- Kiểm tra bảng Traceability Matrix đã được điền đầy đủ mã FRD và trạng thái.
- Verify metadata toàn cục (`[DỰ ÁN:]`, `[PHIÊN BẢN:]`) đã được ánh xạ đúng.

## Constraints
- Template placeholders PHẢI match mapping keys (case-sensitive).
- Output PHẢI pass verification — 100% placeholders được điền.
- Backup template gốc trước khi chạy pipeline.
- Không modify trực tiếp template DOCX khi đang chạy engine.
- Mapping plan phải được cập nhật mỗi khi template thay đổi.
- Version control cho cả template và mapping plan.

## References
- Xem `references/GUIDE.md` để biết chi tiết kỹ thuật, ví dụ mapping, và troubleshooting.

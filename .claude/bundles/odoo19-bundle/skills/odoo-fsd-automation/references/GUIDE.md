# FSD Automation - Technical Guide

Kỹ năng này giúp tự động hóa việc tạo tài liệu Functional Specification Document (FSD) từ Markdown sang Corporate DOCX Template dành cho Odoo 19.

---

## Thành phần (Components)

| Component | Mục đích | Vị trí |
|-----------|---------|--------|
| `md_to_docx.py` | Engine chính: xử lý XML, Paragraph Normalization, Cloning | `business-specs/scripts/` |
| `mapping_plan.md` | Định nghĩa ánh xạ từ MD sang DOCX Anchor | `business-specs/KHSX/` |
| `FSD.docx` | Template DOCX gốc chứa placeholders | `business-specs/KHSX/` |
| `PLA-00_fsd_*.md` | File Markdown nguồn dữ liệu | `business-specs/KHSX/` |

---

## Cú pháp Placeholder trong Template DOCX

Placeholder được đặt trong template dưới dạng text thẳng trong paragraph hoặc cell của bảng:

```
[DỰ ÁN:]
[PHIÊN BẢN:]
[NGÀY:]
[TÊN_PLACEHOLDER:]
```

Lưu ý: Engine sử dụng "Scorched Earth Normalization" — làm sạch XML Word để đảm bảo tìm thấy placeholder dù bị chia cắt bởi formatting runs.

---

## Mapping Plan (mapping_plan.md)

File mapping plan định nghĩa ánh xạ từng placeholder đến nguồn dữ liệu trong Markdown:

```markdown
# Mapping Plan

| Anchor (Placeholder) | Nguồn MD | Loại | Ghi chú |
|----------------------|----------|------|---------|
| [DỰ ÁN:] | frontmatter.project | text | Bảng đầu tiên trong MD |
| [PHIÊN BẢN:] | frontmatter.version | text | |
| [NGÀY:] | frontmatter.date | text | |
| [BẢNG_YÊU_CẦU:] | section_3.table | table | Clone rows per item |
| [PHÂN_HỆ_X:] | section_2.subsystem | block | Sub-rule filling |
```

---

## Lệnh thực thi

```bash
# Lệnh đầy đủ
python3 business-specs/scripts/md_to_docx.py \
  -i business-specs/KHSX/PLA-00_fsd_khsx_master.md \
  -t business-specs/KHSX/FSD.docx \
  -o business-specs/KHSX/PLA-00_fsd_khsx_master.docx \
  -p business-specs/KHSX/mapping_plan.md

# Tham số
# -i / --input    : File Markdown nguồn
# -t / --template : File DOCX template
# -o / --output   : File DOCX output
# -p / --plan     : File mapping plan
```

---

## Mapping Matrix (JSON)

```json
{
  "document_title": {
    "source": "h1",
    "transform": "text"
  },
  "table_requirements": {
    "source": "section_3.table",
    "transform": "table"
  },
  "version": {
    "source": "frontmatter.version",
    "transform": "text"
  },
  "subsystem_block": {
    "source": "section_2.subsystem",
    "transform": "block_clone"
  }
}
```

---

## Tính năng kỹ thuật chính (Key Features)

### 1. Sub-rule Filling (FIXED "Trắng tinh")
Engine hỗ trợ đệ quy để điền nội dung con (Danh mục, Tham số, Giao diện...) cho từng phân hệ được nhân bản:
- Mỗi phân hệ trong Markdown → clone block trong DOCX
- Điền đệ quy các sub-placeholder bên trong block đã clone

### 2. Traceability Matrix
Tự động điền mã FRD-PLA-02-X và trạng thái vào đúng bảng truy vết trong template:
- Quét toàn bộ section Traceability trong Markdown
- Map từng dòng vào bảng tương ứng trong DOCX

### 3. Metadata Mapping
Hỗ trợ Placeholder toàn cục lấy từ bảng đầu tiên trong Markdown:
- `[DỰ ÁN:]` → tên dự án
- `[PHIÊN BẢN:]` → số version
- `[NGÀY:]` → ngày tạo tài liệu
- `[ĐƠN VỊ:]` → đơn vị thực hiện

### 4. Scorched Earth Normalization
Làm sạch XML của Word trước khi search placeholder:
- Merge các `<w:r>` (run) bị tách do formatting
- Đảm bảo tìm thấy mọi placeholder dù bị chia cắt
- Restore formatting sau khi fill xong

---

## Troubleshooting

| Vấn đề | Nguyên nhân | Giải pháp |
|--------|-------------|-----------|
| Placeholder không được điền | Run bị split trong XML | Engine tự xử lý với Normalization; kiểm tra mapping key có đúng không |
| Block clone trắng tinh | Sub-rule chưa được định nghĩa | Thêm sub-rule entries vào mapping_plan.md |
| Output thiếu bảng | Section header trong MD không match | Kiểm tra heading level và tên section trong mapping |
| Lỗi template bị corrupt | Chạy pipeline trên file gốc | Luôn backup; dùng `-t` trỏ đến bản copy |

---

## Quy trình kiểm tra (Verification Checklist)

- [ ] 100% placeholders trong template được điền
- [ ] Traceability Matrix có đầy đủ mã FRD và trạng thái
- [ ] Metadata toàn cục (`[DỰ ÁN:]`, `[PHIÊN BẢN:]`) đúng
- [ ] Các block phân hệ được clone đúng số lượng
- [ ] Sub-content trong từng block không bị trắng tinh
- [ ] Formatting (bold, table borders) được giữ nguyên từ template

---

## Version Control

```bash
# Template và mapping plan cần được commit vào git
git add business-specs/KHSX/FSD.docx
git add business-specs/KHSX/mapping_plan.md
git commit -m "docs: update FSD template and mapping plan vX.Y"

# Output docx KHÔNG commit (add vào .gitignore)
echo "business-specs/**/*_master.docx" >> .gitignore
```

---

Bàn giao bởi Antigravity Orchestrator.

# Antigravity Base Meta Kit

Bộ `.agents/` nền tảng cho Google Antigravity, dùng để khởi tạo một meta team có thể tạo, review và cải tiến Skill, Rule, Agent Team và Workflow theo đúng chuẩn.

## Nội dung

- `@meta-engineer` trong `.agents/agents.md`: agent trung tâm có quyền dùng 4 base meta skills.
- 4 base meta skills được bảo vệ:
  - `antigravity-skill-engineer`
  - `antigravity-rule-engineer`
  - `antigravity-agent-architect`
  - `antigravity-workflow-engineer`
- `karpathy-guidelines`: skill/rule hướng dẫn coding discipline khi viết, review hoặc refactor code.
- Rules bảo vệ:
  - `protect-base-meta-skills`
  - `protect-meta-engineer-agent`
  - `karpathy-guidelines`

## Cài đặt

1. Clone repo này.
2. Copy toàn bộ thư mục `.agents/` vào root project Antigravity của bạn.
3. Mở project trong Antigravity.
4. Gọi `@meta-engineer` hoặc yêu cầu trực tiếp bằng ngôn ngữ tự nhiên để Antigravity tự kích hoạt skill phù hợp.

Ví dụ:

```text
@meta-engineer tạo skill mới tên odoo-model-auditor để review model Odoo custom.
```

## Cách dùng nhanh

| Nhu cầu | Skill nên kích hoạt | Ví dụ prompt |
|---|---|---|
| Tạo, review, cải tiến Skill | `antigravity-skill-engineer` | `@meta-engineer tạo skill mới tên api-contract-reviewer để review OpenAPI spec.` |
| Tạo, review, cải tiến Rule | `antigravity-rule-engineer` | `@meta-engineer tạo rule Always On để cấm sửa file migration đã release.` |
| Thiết kế hoặc review AI team/agents | `antigravity-agent-architect` | `@meta-engineer thiết kế team agents cho dự án FastAPI + React.` |
| Tạo slash command/workflow | `antigravity-workflow-engineer` | `@meta-engineer tạo workflow /new-feature gồm plan, implement, test, review.` |
| Viết, review, refactor code gọn và verify được | `karpathy-guidelines` | `Áp dụng karpathy-guidelines để refactor module này, giữ scope nhỏ và chạy test.` |

## Gọi skill bằng trigger tự nhiên

Antigravity sẽ đọc `description` trong từng `SKILL.md` để tự kích hoạt skill khi prompt khớp ngữ cảnh. Bạn không cần mở file skill thủ công trong lúc làm việc bình thường.

Dùng các cụm từ trigger rõ ràng:

- Skill: `tạo skill`, `review skill`, `improve skill`, `refine skill`
- Rule: `tạo rule`, `review rule`, `cải tiến rule`
- Agent team: `thiết kế agents`, `build multi-agent`, `review team agent`
- Workflow: `tạo workflow`, `build slash command`, `review workflow`
- Coding discipline: `áp dụng karpathy-guidelines`, `review code`, `refactor`, `tránh over-engineering`

## Luồng làm việc khuyến nghị

1. Gọi `@meta-engineer` với yêu cầu cụ thể.
2. Nếu tạo mới, nêu rõ tên artifact và mục đích.
3. Nếu review/cải tiến, truyền đường dẫn file cần review.
4. Yêu cầu agent liệt kê file đã tạo/sửa và cách test sau khi xong.

Ví dụ tạo mới:

```text
@meta-engineer tạo workflow /full-review để chạy review code, security check và test summary.
```

Ví dụ review:

```text
@meta-engineer review .agents/skills/my-skill/SKILL.md và đề xuất sửa theo chuẩn Antigravity.
```

Ví dụ coding:

```text
Áp dụng karpathy-guidelines để sửa bug này: chỉ sửa đúng phần liên quan, verify bằng test hiện có.
```

## Quy tắc bảo vệ lõi

Không chỉnh sửa trực tiếp các thành phần lõi sau:

- `.agents/agents.md`
- `.agents/skills/antigravity-skill-engineer/`
- `.agents/skills/antigravity-rule-engineer/`
- `.agents/skills/antigravity-agent-architect/`
- `.agents/skills/antigravity-workflow-engineer/`

Nếu cần mở rộng, hãy tạo artifact mới bên cạnh core:

- Skill mới: `.agents/skills/<ten-skill>/SKILL.md`
- Rule mới: `.agents/rules/<ten-rule>.md`
- Workflow mới: `.agents/workflows/<ten-workflow>.md`

## Về script nội bộ

Các thư mục base skill có `scripts/`, `resources/` và `examples/` để agent dùng khi cần scaffold hoặc kiểm tra chuẩn. Không xem các script này là public CLI ổn định.

Khuyến nghị vận hành:

1. Dùng `@meta-engineer` cho workflow bình thường.
2. Để agent tự chọn script/resource phù hợp theo instruction trong `SKILL.md`.
3. Không chạy script thủ công nếu lệnh có thể ghi đè `.agents/agents.md` hoặc 4 base meta skills.

<!-- Bundle template: merge this file into the app repository's root AGENTS.md when installing. -->

# App Development Instructions

Các path `.agents/rules/...` dưới đây là vị trí sau khi cài bundle; khi review bundle chưa cài, file tương ứng nằm trong `./rules/`.

## Scope
Các instruction này áp dụng cho việc xây dựng, sửa, review và bàn giao ứng dụng trong cây thư mục thuộc phạm vi của file này. Chúng không tự cấp quyền commit, push, deploy, publish hay gửi dữ liệu ra dịch vụ bên ngoài.

## Required workflow
1. Đọc requirement, convention, tài liệu và test liên quan trước khi sửa.
2. Giữ scope nhỏ; bảo tồn hành vi ngoài phạm vi và không ghi đè thay đổi của user.
3. Codex không dựa vào metadata `Activation` để nạp nội dung. Chủ động mở các rule đã cài theo routing sau:
   - Luôn đọc `.agents/rules/documentation-maintenance.md`.
   - Với thay đổi user-facing phức tạp, đọc `.agents/rules/in-app-guidance.md`.
   - Với app dẫn xuất từ spec/curriculum/nguồn chuẩn, đọc `.agents/rules/source-aligned-development.md`.
   - Khi dùng fact, công thức hoặc cấu trúc từ nguồn ngoài, đọc `.agents/rules/verify-authoritative-source.md`.
   - Khi user đã yêu cầu commit, đọc `.agents/rules/claude-review-before-commit.md`.
4. Chạy kiểm tra phù hợp với rủi ro và báo chính xác command, kết quả cùng phần chưa chạy.
5. Không coi output của model là ground truth; kiểm chứng bằng requirement, code, test và nguồn có thẩm quyền.

## Reusable skills
- Dùng `$discuss-with-claude` chỉ khi user yêu cầu lấy ý kiến Claude.
- Dùng `$verify-engine-with-claude` cho workflow kiểm chứng calculation/business-rule engine; chỉ chạy Claude lane khi user hoặc policy dự án cho phép gửi đúng nội dung ra ngoài.
- Nếu chưa có quyền gọi dịch vụ ngoài, vẫn làm phần phân tích/kiểm chứng local và báo `Claude lane: not run`; không tự suy diễn quyền từ việc skill được trigger.

## Handoff
Báo cáo ngắn gọn thay đổi đã làm, file chính, kiểm tra đã chạy, tài liệu đã cập nhật và rủi ro hoặc việc còn lại. Chỉ báo commit/deploy khi hành động đó thực sự hoàn thành.

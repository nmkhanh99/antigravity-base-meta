---
name: discuss-with-claude
description: "Lấy và tổng hợp second opinion read-only từ Claude cho quyết định kỹ thuật. Use when the user explicitly asks Codex to hỏi/trao đổi/brainstorm/phản biện với Claude or get a Claude second opinion; do not use for ordinary brainstorming, delegated edits, pre-commit review, or authoritative verification."
---

# Discuss With Claude

## Goal
Cho Codex đối thoại với một model khác để nhận góc nhìn mới, sau đó tự kiểm chứng và tổng hợp thay vì chuyển nguyên văn output cho user.

## When to use this skill
- Cần phản biện kiến trúc, API, data model, UX, kế hoạch migration hoặc trade-off.
- User yêu cầu hỏi ý kiến, brainstorm hay thảo luận cùng Claude.
- Không dùng khi mục tiêu là để Claude sửa file, review bắt buộc trước commit, hoặc cung cấp ground truth.

## Instructions
1. **Đóng khung câu hỏi.** Tóm tắt decision cần bàn, constraint, lựa chọn hiện có và tiêu chí thành công. Chỉ đưa context cần thiết.
2. **Kiểm tra runtime.** Chạy `claude --version` và `claude auth status --text`. Nếu CLI/auth chưa sẵn sàng, báo user; chỉ dùng sub-agent Codex thay thế khi nói rõ mức độc lập thấp hơn.
3. **Kiểm tra quyền chia sẻ.** Chỉ gọi Claude khi user hoặc policy dự án cho phép gửi đúng loại nội dung đó ra ngoài. Không gửi secret, dữ liệu cá nhân, code độc quyền hoặc tài liệu có bản quyền ngoài phạm vi được phép.
4. **Chuẩn bị input an toàn.** Dùng UUID v4 mới; lưu prompt trong temp file mode `0600`, truyền qua stdin và cleanup bằng trap/finally. Không đặt prompt trong CLI argument, shell history hoặc process list. Chạy no-tool từ một thư mục tạm trung tính, không phải repository:
   ```bash
   claude -p --safe-mode --session-id "<uuid>" \
     --permission-mode plan --tools "" --strict-mcp-config \
     --disallowedTools "mcp__*" --no-chrome < "<prompt-file>"
   ```
   Với một lượt, bỏ `--session-id`, thêm `--no-session-persistence` và không resume.
5. **Chỉ mở quyền đọc khi cần.** Ưu tiên đưa sanitized context vào prompt và giữ no-tool. Nếu user/policy cho phép Claude đọc repository, chạy từ đúng project và chỉ cấp `Read,Grep,Glob`:
   ```bash
   claude -p --safe-mode --session-id "<uuid>" \
     --permission-mode plan --tools "Read,Grep,Glob" --strict-mcp-config \
     --disallowedTools "mcp__*" --no-chrome < "<prompt-file>"
   ```
   Prompt phải ghi rõ chỉ thảo luận và không sửa file. Quyền đọc vẫn có thể làm nội dung file rời máy; prompt không phải security boundary.
6. **Đọc và phản biện.** Kiểm chứng claim của Claude với code, test, requirement hoặc nguồn chính; ghi điểm đồng ý, không đồng ý và lý do.
7. **Hỏi tiếp khi cần.** Dùng lại `-p --resume "<uuid>"` từ cùng cwd, tuần tự, với toàn bộ safety flag và tool set ban đầu. Session nhiều lượt lưu transcript plaintext cục bộ; báo user và không dùng cho nội dung nhạy cảm.
8. **Tổng hợp.** Trình bày quan điểm Claude, đánh giá của Codex, kết luận chung và bất đồng còn lại. Không forward nguyên văn output thay cho phân tích.

## Constraints
- Không cấp Edit, Write, Bash hoặc cờ bỏ qua permission.
- Không coi Claude là nguồn sự thật; đây là second opinion có thể sai.
- Không tự chọn model/effort khác mặc định nếu user không yêu cầu.
- Không dùng session Claude có sẵn vì có thể lẫn context của task khác.
- Nếu invocation lỗi, không retry bằng cách bỏ safety flag; báo lỗi hoặc dùng fallback đã công bố.

## Best practices
- Giới hạn 1–3 lượt và đặt tiêu chí dừng để cuộc thảo luận hội tụ.
- Yêu cầu Claude nêu giả định, failure mode và bằng chứng cần kiểm tra.
- Khi so sánh phương án, dùng cùng constraint và tiêu chí cho mọi phương án.

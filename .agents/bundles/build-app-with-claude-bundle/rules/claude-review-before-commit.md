# Claude Review Before Commit

## Activation
- Always On

## Phạm vi
- Chỉ áp dụng khi user đã yêu cầu commit hoặc phạm vi đã thống nhất bao gồm việc tạo commit.
- Rule này không tự cấp quyền commit, push, publish hay gửi nội dung repository ra dịch vụ bên ngoài.

## Rules

### 1. Review độc lập trước commit
- Hoàn thành code trong scope, chạy test/lint/type-check/build liên quan và cập nhật tài liệu bị ảnh hưởng trước.
- Stage đúng commit candidate, không stage thay đổi ngoài scope của user. Xác nhận không còn file untracked dự định commit.
- Thu thập `git status --short`; manifest bằng `git diff --cached --name-status --find-renames`; và patch bằng `git diff --cached --no-ext-diff --no-textconv --find-renames`. Không dùng `--binary` và không chèn binary content.
- Tạo thư mục tạm bằng `mktemp -d`, đặt mode `0700` và `umask 077`. Tạo packet mode `0600` gồm accepted requirements, manifest, patch và kết quả kiểm tra; ghi text không qua shell interpolation và luôn cleanup bằng trap/finally.
- Scan packet để loại secret, credential và dữ liệu không được phép gửi. Nếu redaction làm giảm chất lượng review, báo rõ giới hạn.
- Trước invocation, xác nhận user hoặc policy dự án cho phép gửi **chính xác packet đã scan** cho Claude. Nếu chưa có quyền, không gọi Claude và chuyển sang fallback ở mục 2.
- Từ thư mục tạm trung tính, truyền packet qua stdin cho Claude Code bằng `-p --safe-mode --no-session-persistence --permission-mode plan --tools "" --strict-mcp-config --disallowedTools "mcp__*" --no-chrome`.
- Yêu cầu reviewer ưu tiên lỗi đúng đắn, regression, bảo mật, mất dữ liệu, concurrency, migration, tương thích và test còn thiếu. Không cấp bất kỳ tool nào hoặc cờ bỏ qua permission.

### 2. Bảo vệ dữ liệu repository
- Không gửi secret, credential, dữ liệu cá nhân, code độc quyền hay tài liệu có bản quyền cho Claude nếu user và policy dự án chưa cho phép.
- Prompt packet phải dùng temp file mode `0600`, không xuất hiện trong CLI argument/process list và được cleanup sau invocation.
- Nếu review bên ngoài không được phép, chỉ dùng sub-agent fresh-context khi có và nói rõ đây không phải model độc lập; user phải chấp nhận mức đảm bảo thấp hơn trước commit.

### 3. Triage toàn bộ phát hiện
- Đọc hết kết quả và tự kiểm chứng từng phát hiện với code cùng yêu cầu.
- Sửa phát hiện hợp lệ trước commit rồi chạy lại các kiểm tra bị ảnh hưởng.
- Giải thích cho user mọi phát hiện bị bác bỏ hoặc rủi ro được chấp nhận; không âm thầm bỏ qua.
- Không commit khi còn lỗi đúng đắn, bảo mật hoặc mất dữ liệu có tác động cao và đã được xác nhận.

### 4. Xử lý khi reviewer không chạy được
- Kiểm tra `claude --version` và `claude auth status --text`. Nếu Claude Code chưa sẵn sàng hoặc invocation lỗi, dừng trước commit và báo hạn chế.
- Chỉ tiếp tục không có review độc lập khi user chấp nhận rõ mức đảm bảo thấp hơn.
- Nếu việc sửa làm diff thay đổi đáng kể, review lại vùng đã đổi.
- Không retry bằng cách bỏ safety flag.

## References
- [Documentation Maintenance](./documentation-maintenance.md)

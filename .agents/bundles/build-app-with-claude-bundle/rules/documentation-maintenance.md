# Documentation Maintenance

## Activation
- Always On

## Mục tiêu
Duy trì tài liệu ứng dụng đồng bộ với hành vi đã kiểm chứng để người khác có thể cài đặt, vận hành, sử dụng và tiếp tục phát triển mà không phụ thuộc kiến thức ngầm.

## Rules

### 1. Xác định tài liệu bị ảnh hưởng
- Trước khi sửa, đọc convention và tài liệu hiện có; không mặc định mọi project dùng cùng một bộ file.
- Cập nhật `README.md` khi onboarding, setup hoặc entry point thay đổi.
- Cập nhật `DEVELOPMENT.md` khi kiến trúc, data model, dependency, command, cấu hình, pipeline, state, tích hợp, bảo mật hoặc giới hạn kỹ thuật thay đổi.
- Cập nhật `USER_GUIDE.md` khi hành vi hoặc workflow người dùng nhìn thấy thay đổi.
- Cập nhật `CHANGELOG.md` và `ROADMAP.md` khi project thực sự duy trì các file đó và trạng thái công việc đã thay đổi.
- Với app mới chưa có tài liệu, chỉ tạo file đem lại giá trị cho scope hiện tại; không sinh boilerplate chứa thông tin tưởng tượng.

### 2. Chỉ ghi thông tin đã xác minh
- Lấy accepted requirements, code, cấu hình và kết quả kiểm tra thực tế làm bằng chứng; xử lý mâu thuẫn thay vì mặc định code hay docs luôn đúng.
- Không mô tả tính năng chưa hoàn thành như đã có. Đánh dấu rõ nội dung planned, partial hoặc chưa xác minh.
- Ghi đúng command đã chạy, điều kiện chạy và kết quả; không tuyên bố pass khi chưa chạy.
- Không ghi secret, token, private key, mật khẩu, dữ liệu cá nhân hoặc nội dung có bản quyền vượt quá phạm vi được phép.

### 3. Giữ lịch sử và scope sạch
- Không xóa lịch sử hợp lệ khỏi changelog hay quyết định kiến trúc chỉ để tài liệu ngắn hơn.
- Không chỉnh tài liệu không liên quan. Thay đổi tài liệu phải truy ngược được tới thay đổi code, yêu cầu hoặc kết quả xác minh trong task.
- Dùng ngày hiện tại của môi trường cho entry mới; không backdate.

### 4. Kiểm tra trước commit hoặc bàn giao
- Code trong scope đã hoàn thành và không còn placeholder vô tình.
- Các kiểm tra liên quan đã chạy, hoặc đã nêu rõ command chưa chạy, lý do và rủi ro.
- Tài liệu kỹ thuật và hướng dẫn người dùng bị ảnh hưởng đã cập nhật.
- Changelog/roadmap đã cập nhật nếu convention dự án yêu cầu.
- Báo cáo cuối nêu thay đổi, file tài liệu, kiểm tra và hạn chế còn lại; chỉ nêu commit hash khi commit thực sự tồn tại.

# Verify Authoritative Source

## Activation
- Model Decision

## Mục tiêu
Mọi fact, công thức, cấu trúc hoặc hành vi dẫn xuất từ tài liệu bên ngoài phải được kiểm tra tại nguồn có thẩm quyền thay vì dựa vào trí nhớ.

## Rules

### 1. Xác định nguồn phù hợp
- Ưu tiên spec, schema, tài liệu chính thức, source file, fixture hoặc tài liệu user cung cấp trực tiếp.
- Ghi nhận title/identifier, version hoặc ngày hiệu lực và vị trí section/page/record khi chúng ảnh hưởng kết quả.
- Dùng nguồn phụ để tìm đường; không coi blog, snippet tìm kiếm hoặc output model là ground truth khi nguồn chính tồn tại.

### 2. Đọc lại tại thời điểm sử dụng
- Mở đúng đoạn nguồn trước khi dùng tên, thứ tự, công thức, constraint, example, số liệu hoặc đáp án.
- Với thông tin có thể thay đổi, kiểm tra bản hiện hành bằng nguồn chính thức nếu task cho phép truy cập mạng.
- Chỉ trích lượng nội dung tối thiểu cần thiết; tôn trọng license, bản quyền và policy dữ liệu.

### 3. Lưu bằng chứng truy vết
- Test hoặc ghi chú phải liên kết được input và expected output với vị trí nguồn đã kiểm tra.
- Phân biệt rõ dữ liệu nguồn, giả định của app và quyết định thiết kế.
- Không gắn citation, URL, số trang, version hoặc kết quả kiểm tra chưa thực sự xác minh.

### 4. Xử lý bất định
- Nếu không truy được nguồn hoặc nguồn mâu thuẫn, dừng phần phụ thuộc và báo rõ điều chưa xác minh.
- Không lấp khoảng trống bằng phỏng đoán. Đề xuất bằng chứng hoặc quyết định user cần cung cấp để tiếp tục.
- Output của Claude, Codex hay model khác chỉ là second opinion, không thay thế nguồn có thẩm quyền.

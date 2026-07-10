# Source-Aligned Development

## Activation
- Model Decision

## Phạm vi
Áp dụng khi cấu trúc hoặc nội dung app được dẫn xuất từ một nguồn có thẩm quyền như curriculum, đặc tả, tiêu chuẩn, quy định, catalog hoặc bộ dữ liệu chuẩn.

## Rules

### 1. Giữ ánh xạ với nguồn
- Xác định nguồn chính, phiên bản và phạm vi áp dụng trước khi thiết kế cấu trúc app.
- Giữ đúng hierarchy, tên và thứ tự của nguồn khi chúng mang ý nghĩa nghiệp vụ; chỉ chuyển đổi khi user hoặc requirement cho phép rõ.
- Tài liệu bổ trợ không được âm thầm thay thế nguồn chính.
- Dùng một manifest/schema nội bộ làm nơi khai báo ánh xạ duy nhất khi project đã có pattern này; không tạo thêm nguồn sự thật cạnh tranh.

### 2. Không suy đoán cấu trúc
- Đối chiếu trực tiếp TOC, schema, heading, identifier hoặc record gốc trước khi thêm/sắp xếp nội dung.
- Không bịa tên, thứ tự, quan hệ hay version từ trí nhớ.
- Nếu nguồn không truy cập được, đánh dấu phần liên quan là chưa xác minh và báo user.

### 3. Phát triển theo dependency thật
- Ưu tiên thứ tự của nguồn khi các phần phụ thuộc nhau hoặc trải nghiệm cần tiến triển tuần tự.
- Cho phép làm song song phần độc lập; không biến thứ tự trình bày thành ràng buộc kỹ thuật giả.
- Hiển thị trạng thái planned/partial rõ ràng nếu app cần lộ cấu trúc chưa hoàn thành.

### 4. Kiểm chứng hành vi dẫn xuất
- Mỗi công cụ hoặc rule nghiệp vụ phải truy ngược được tới section/field/requirement liên quan.
- Kiểm chứng engine bằng test vector, example hoặc fixture có thẩm quyền khi có; nêu tolerance, đơn vị và quy tắc làm tròn.
- Khi app mâu thuẫn với nguồn, xác định liệu nguồn, requirement hay implementation đã thay đổi trước khi sửa.

## References
- [Verify Authoritative Source](./verify-authoritative-source.md)

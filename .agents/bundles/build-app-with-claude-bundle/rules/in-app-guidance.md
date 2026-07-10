# In-App Guidance

## Activation
- Always On

## Mục tiêu
Các workflow hoặc công cụ user-facing có độ phức tạp đáng kể phải tự giải thích đủ để người dùng hiểu mục đích, thao tác đúng và diễn giải kết quả mà không phải đoán.

## Rules

### 1. Giải thích tại điểm cần thiết
- Với khái niệm, input, output hoặc hệ quả khó hiểu, thêm hướng dẫn ngắn ngay cạnh nơi tương tác.
- Nêu rõ công cụ làm gì, dữ liệu cần nhập, đơn vị/định dạng, kết quả có nghĩa gì và giới hạn quan trọng.
- Không nhồi lý thuyết dài vào màn hình; dùng progressive disclosure cho chi tiết nâng cao.
- Không thêm hướng dẫn dư thừa cho tương tác chuẩn đã tự rõ.

### 2. Hướng dẫn workflow đầy đủ
- Với tác vụ nhiều bước, chỉ rõ thứ tự, điều kiện tiên quyết, trạng thái thành công và cách phục hồi khi lỗi.
- Validation và error message phải nói được vấn đề, vị trí và hành động khắc phục; không chỉ báo “invalid” hoặc “failed”.
- Dữ liệu mẫu phải được gắn nhãn là ví dụ và không được trông như dữ liệu thật của user.

### 3. Trỏ tới nguồn khi cần đào sâu
- Khi tính năng phản ánh spec, quy định, công thức hoặc tài liệu nguồn, cung cấp reference đủ cụ thể và đã xác minh.
- Không bịa tên mục, phiên bản, URL, số trang hoặc ví dụ. Tuân theo `verify-authoritative-source`.

### 4. Nhất quán và tiếp cận được
- Tái sử dụng component, terminology và pattern hướng dẫn hiện có của app.
- Hướng dẫn không được chỉ phụ thuộc màu sắc, hover hoặc icon không có nhãn; giữ keyboard và screen-reader flow khả dụng khi stack hỗ trợ.
- Test trạng thái empty, loading, error, success và dữ liệu biên của workflow được hướng dẫn.

## References
- [Verify Authoritative Source](./verify-authoritative-source.md)

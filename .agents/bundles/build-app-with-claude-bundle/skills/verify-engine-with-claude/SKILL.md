---
name: verify-engine-with-claude
description: "Verify deterministic calculation or business-rule logic with a traceable source/fixture, actual output, an independent reference, and blind read-only Claude. Trigger for formula/rule correctness, units, rounding, tolerance, or edge cases; not for UI/CRUD/workflow code, general diff review, or ML/LLM model quality."
---

# Verify Engine With Claude

## Goal
Phát hiện sai công thức, đơn vị, rounding và edge case bằng nhiều lane bằng chứng độc lập. Claude giải thích phép tính như một cross-check, không phải oracle hay ground truth.

## When to use this skill
- Vừa viết hoặc sửa engine deterministic và có example, test vector hoặc fixture để đối chiếu.
- Output app lệch expected result và cần khoanh vùng nguyên nhân.
- Cần xác nhận tolerance, rounding, unit conversion hoặc thứ tự phép tính trước khi bàn giao.

## Instructions
1. **Thu thập ground truth.** Mở nguồn có thẩm quyền và ghi đúng công thức/rule, input, expected output, đơn vị, rounding cùng vị trí/version. Không lấy từ trí nhớ.
2. **Chạy engine thật.** Dùng đúng input, ghi raw output và formatted output. Lưu command/test case đủ để tái lập.
3. **Tạo phép tính tham chiếu.** Dùng implementation nhỏ độc lập hoặc công cụ deterministic phù hợp; không import lại hàm production đang kiểm tra. Với số thực, đặt tolerance trước khi xem kết quả.
4. **Chuẩn bị blind prompt cho Claude.** Chỉ đưa rule/công thức, input, đơn vị và yêu cầu trình bày từng bước. Không tiết lộ expected output, output engine hay kết quả tham chiếu ở lượt đầu.
5. **Chạy Claude cô lập, không tool.** Sau khi user hoặc policy cho phép gửi đúng nội dung ra ngoài, dùng temp file mode `0600`, cleanup bằng trap/finally và chạy từ thư mục tạm trung tính:
   ```bash
   claude -p --safe-mode --no-session-persistence \
     --permission-mode plan --tools "" --strict-mcp-config \
     --disallowedTools "mcp__*" --no-chrome < "<prompt-file>"
   ```
   Không cho Claude đọc repository hoặc sửa file.
6. **Đối chiếu các lane.** So sánh nguồn ↔ engine ↔ phép tính tham chiếu ↔ Claude theo cùng precision, unit và tolerance. Nếu lệch, xác định bước đầu tiên phân kỳ trước khi sửa.
7. **Sửa và chạy lại.** Sau thay đổi, chạy lại engine, reference calculation và test. Chạy blind Claude lần mới nếu công thức hoặc logic đã đổi đáng kể.
8. **Báo kết luận.** Nêu PASS/FAIL/PROVISIONAL, các giá trị cạnh nhau, tolerance, vị trí nguồn, command đã chạy và nguyên nhân nếu lệch.
   Nếu Claude không chạy được, vẫn hoàn tất các lane deterministic và ghi `Claude lane: unavailable`; không retry bằng cách bỏ safety flag.

## Pass criteria
- Chỉ kết luận **PASS** khi engine khớp expected output có thẩm quyền và phép tính deterministic độc lập trong tolerance.
- Nếu không có expected output có thẩm quyền, kết luận **PROVISIONAL** dù engine, reference và Claude đồng ý.
- Bất đồng của Claude không tự làm engine FAIL; phải kiểm chứng phép tính của Claude.

## Constraints
- Không mồi đáp án cho Claude ở lượt blind đầu tiên.
- Không coi output model là bằng chứng độc lập đủ để phát hành logic quan trọng.
- Không gửi secret, dữ liệu cá nhân, code độc quyền hoặc trích đoạn có bản quyền khi chưa được phép.
- Không kết luận pass nếu chưa thực sự chạy engine và phép tính tham chiếu.

## Best practices
- Bao phủ zero, negative, boundary, overflow/precision, missing và invalid input khi domain cho phép.
- Tách raw numeric correctness khỏi formatting/localization.
- Chuyển example đã xác minh thành regression test có tên và nguồn truy vết.

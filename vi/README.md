# Hướng dẫn triển khai Fomoxa

Tập tài liệu hướng dẫn dựng một transport hoặc một SDK Fomoxa trên bất kỳ ngôn ngữ nào, mô tả ở mức khái niệm, không kèm SDK, API hay mã nguồn.

Tài liệu dùng để tra cứu; không cần đọc hết cả hai tài liệu mới bắt đầu được. Chọn đường đọc theo công việc ở [§ Đọc theo công việc](#đọc-theo-công-việc) rồi quay lại tra cứu khi cần.

## Bản đồ tài liệu

| Tài liệu | Nội dung | Độ dài |
|---|---|---|
| [vi/01-overview.md](vi/01-overview.md) | Mô hình ba tầng, ranh giới, phân chia trách nhiệm core ↔ transport, các điều cấm | ~650 dòng |
| [vi/02-flows.md](vi/02-flows.md) | Định dạng trên dây, từng luồng chi tiết, mã giả cho core và transport, danh mục kiểm thử | ~860 dòng |

Bên trong, hai tài liệu gọi nhau bằng tên khái niệm `01_overview.md` và
`02_flows.md`. Chúng tương ứng với `vi/01-overview.md` và `vi/02-flows.md`.

Thư mục `vi/` là bản tiếng Việt. Các bản dịch khác, nếu có, nằm ở thư mục mã ngôn ngữ
của riêng nó.

## Đọc theo công việc

| Việc cần làm | Các mục cần đọc |
|---|---|
| Hiểu Fomoxa chia tầng thế nào | `01` §1–§3 |
| Viết một transport mới (WebSocket, TLS, QUIC, UDP…) | `01` §2, §3, §11, §12 → `02` §9, và bảng tự kiểm §9.5 |
| Chỉ cần byte trên dây (viết bộ mã hóa/giải mã) | `02` §2 |
| Hiểu bắt tay và kiểm tra lược đồ | `02` §3 |
| Hiểu nhịp tim, phát hiện peer chết | `02` §4 |
| Viết một SDK Fomoxa mới trên ngôn ngữ khác | Cả hai, theo thứ tự `01` → `02` |
| Kiểm chứng bản triển khai đã có | `02` §11 → `01` §14 |
| Sửa lỗi liên thông giữa hai bản triển khai | `02` §2 và §10 |

Người viết transport không cần đọc định dạng trên dây. Đây là điều kiện thiết kế: transport chỉ chở byte, nên một transport cần biết byte mang nghĩa gì là transport có logic đặt sai tầng.

## Tóm tắt một trang

Fomoxa có ba tầng, và phần lớn quy tắc là về ranh giới giữa chúng:

```
  ỨNG DỤNG     message có ý nghĩa với nghiệp vụ
  CORE         bắt tay, nhịp tim, đóng/mở khung, sự kiện
  TRANSPORT    chỉ chở byte  (TCP / UDP / WebSocket / TLS / QUIC)
```

Transport cung cấp đúng bốn chức năng (gửi, nhận, đóng mềm, đóng hẳn) và trả lời bằng đúng sáu tín hiệu:

```
  ✔ xong    ⏸ chưa được    ✖ đã đóng    ⚠ lỗi    ⊘ quá lớn    ⤢ chỗ không đủ
```

Transport không có chức năng thứ năm, không chờ chặn và không tự kết nối lại. Việc nhớ và thử lại thuộc core.

Các con số cố định: khung DỮ LIỆU tối đa 16 MiB + 11 byte; nội dung BẮT TAY tối đa 1 MiB; mặc định hạn bắt tay 5 giây, chu kỳ nhịp tim 5 giây, hạn nhịp tim 15 giây.

Chín bất biến mà mọi bản triển khai phải giữ nằm ở `01` §14. Bất biến áp dụng ở mọi nơi là: không lớp nào được phép chặn.

## Ngoài phạm vi tập tài liệu này

- Cách tính vân tay lược đồ và cách mã hóa dữ liệu của message thuộc đặc tả lược đồ
  của Fomoxa.
- Ràng buộc API của từng ngôn ngữ. Mỗi bản triển khai tự đặt tên và tự quyết hình dạng
  lời gọi. Tài liệu này chỉ quy định *hành vi*, và hành vi phải giống nhau ở mọi bản.
- Bắt tay của chính đường truyền (TCP, TLS, nâng cấp WebSocket). Bước này xảy ra trước,
  bên ngoài Fomoxa, do người viết transport tự lo.

## Giấy phép

CC BY 4.0, xem [LICENSE](LICENSE). Bản triển khai phần mềm là dự án độc lập và chọn giấy phép nào tùy tác giả.

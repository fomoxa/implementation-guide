# Transport UDP của Fomoxa: mã phiên

Tài liệu này định nghĩa cách một transport UDP phân biệt các peer. Nó chỉ áp dụng cho UDP. Core không biết đến nó, còn TCP, TLS, WebSocket và QUIC không bị ảnh hưởng, vì mỗi loại đó đã gắn sẵn với một peer (`01_overview.md` §10).

Người viết transport đọc tài liệu này thêm vào `01_overview.md` §2, §3, §11 và §12. Người đó vẫn không phải đọc định dạng trên dây ở `02_flows.md` §2: mã phiên nằm ngoài frame, và transport không bao giờ nhìn vào các byte phía sau nó.

---

## 1. Vì sao địa chỉ nguồn chưa đủ

Một transport UDP nhận diện peer bằng địa chỉ nguồn sẽ mất session mỗi khi địa chỉ đó đổi:

- Điện thoại chuyển từ wifi sang mạng di động, hoặc NAT cấp cổng mới sau một lúc im lặng. Gói vẫn tới, nhưng từ một địa chỉ transport chưa từng thấy.
- Một relay hoặc bộ cân bằng tải đứng trước nhiều server không có gì trong gói để định tuyến.

Mã phiên mang trong mọi datagram tách việc peer là ai khỏi việc nó đang gửi từ đâu.

---

## 2. Bố cục datagram

```
   ┌────────────────────────┬──────────────────────────────────┐
   │ mã phiên               │ một frame                        │
   │ 8B u64 LE              │ đúng như core đã giao            │
   └────────────────────────┴──────────────────────────────────┘
    ^0-7                     ^8 trở đi

   Tổng kích thước = 8 + kích thước frame
```

- Frame là các byte core đã truyền vào `send`, không đổi. Transport không đọc, không cắt và không ghép chúng.
- Datagram ngắn hơn 9 byte bị bỏ.
- Giá trị `0` nghĩa là "chưa được cấp mã". Server không bao giờ cấp giá trị này.
- Trần của chính transport tính cả 8 byte phần đầu. Một frame vừa với trần C byte mỗi datagram thì tối đa C − 8 byte, và transport trả ⊘ khi vượt, như `01_overview.md` §7. Mức khuyến nghị dưới khoảng 1200 byte cũng tính cả phần đầu.

---

## 3. Mã được cấp và dùng thế nào

### 3.1 Server

```
   khi nhận datagram từ địa chỉ nguồn A với mã phiên S:

       S = 0:
           tìm peer đã đăng ký ở địa chỉ A
               có    → nhận frame cho peer đó
               không → nếu bảng peer đã chạm trần: bỏ im lặng
                       ngược lại tạo peer tại A, cấp cho nó mã mới, nhận frame

       S ≠ 0:
           tìm peer sở hữu S
               có    → nhận frame cho peer đó;
                       nếu A khác địa chỉ đã ghi của peer,
                       ghi A làm địa chỉ mới của peer
               không → bỏ im lặng
```

- Mọi datagram server gửi cho một peer đều mang mã của peer đó.
- Mã mới là một giá trị 64 bit ngẫu nhiên từ bộ sinh an toàn mật mã. Nó khác không và khác mã của mọi peer đang tồn tại.
- Địa chỉ đã ghi chỉ đổi sau khi một datagram được nhận. Datagram bị bỏ không bao giờ làm peer dịch chuyển.
- Bảng peer giữ trần ở `01_overview.md` §10. Trần đó áp dụng cho bước tạo peer ở trên.
- Khi peer được giải phóng bằng hard close, mã cũng được giải phóng theo. Datagram nào mang mã đó về sau bị bỏ như mã lạ.

### 3.2 Client

- Client gửi `0` làm mã phiên cho tới khi nhận được một datagram từ server.
- Nó dùng mã của datagram đầu tiên nhận được, và dùng mã đó cho mọi datagram sau.
- Datagram sau đó mang mã khác không bị bỏ. Mã của một session không bao giờ đổi.

### 3.3 Core thấy gì

Core không thấy thay đổi nào. Nó nhận frame theo cùng thứ tự như trước, từ cùng đối tượng peer. Đổi địa chỉ không phát sự kiện nào, và heartbeat vẫn chạy như `02_flows.md` §4 định nghĩa. Nếu địa chỉ cũ còn liên lạc được mà địa chỉ mới thì không, peer sẽ im lặng và core xử lý như mọi peer im lặng.

---

## 4. Những điều nó không làm

- Nó không xác thực và không mã hóa. Ai thấy được các datagram đều thấy mã phiên, và ai biết mã có thể gửi datagram để chuyển peer sang địa chỉ khác. Bảo mật nội dung và xác thực thuộc về tầng bên dưới, như DTLS, hoặc bên chấp nhận kết nối, và nằm ngoài tài liệu này.
- Nó không kiểm tra địa chỉ mới có liên lạc được hay không trước khi server gửi tới đó. Người nắm được một mã bị chộp có thể lái lưu lượng của server sang một địa chỉ thứ ba. Một bản triển khai nên giới hạn lượng dữ liệu gửi tới địa chỉ mới cho tới khi địa chỉ đó tự gửi một datagram, như QUIC làm với path validation.
- Nó không áp dụng qua lần khởi động lại server. Sau khi khởi động lại, mã cũ là mã lạ, datagram của các peer cũ bị bỏ, và các session đó kết thúc qua hạn chót của heartbeat.

---

## 5. Tương thích

Đây là định dạng datagram duy nhất của transport UDP. Datagram không có phần đầu 8 byte không phải datagram hợp lệ, nên một transport UDP không cài tài liệu này không nói chuyện được với transport có cài.

---

## 6. Tự kiểm tra

- [ ] Mọi datagram gửi đi bắt đầu bằng 8 byte mã phiên, theo sau là đúng một frame
- [ ] Datagram dưới 9 byte bị bỏ mà không báo lỗi
- [ ] Server cấp mã khác không, ngẫu nhiên, từ bộ sinh an toàn mật mã, không trùng với peer đang sống
- [ ] Server không bao giờ gửi datagram mang mã `0`
- [ ] Mã `0` chỉ tạo peer khi bảng peer còn dưới trần
- [ ] Mã khác không mà không biết thì bị bỏ im lặng
- [ ] Địa chỉ đã ghi chỉ đổi sau một datagram được nhận
- [ ] Client giữ mã đầu tiên nhận được và bỏ datagram mang mã khác
- [ ] Trần của một frame là trần datagram trừ 8
- [ ] Core không được báo gì về việc đổi địa chỉ

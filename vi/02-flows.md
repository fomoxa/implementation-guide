# Fomoxa - Định dạng trên dây, các luồng, hướng dẫn triển khai

Tài liệu này mô tả hành vi bắt buộc của mọi bản triển khai Fomoxa. Độc lập với ngôn ngữ lập trình. Không tham chiếu implementation, API hay mã nguồn cụ thể nào.

Đọc tài liệu này cùng `01_overview.md` là đủ để dựng lại Fomoxa. Một implementation mới, hoặc một transport mới, trên bất kỳ ngôn ngữ nào.

- `01_overview.md` - mô hình, ranh giới lớp, phân chia trách nhiệm
- `02_flows.md` (tài liệu này) - định dạng byte, từng luồng, hướng dẫn triển khai

Quy ước dùng trong tài liệu:

- **LE** = little-endian. Mọi số nhiều byte trong Fomoxa đều là little-endian.
- Sáu ký hiệu ✔ ⏸ ✖ ⚠ ⊘ ⤢ là sáu câu trả lời của transport, định nghĩa ở `01_overview.md` §2.
- Thuật ngữ giao thức giữ nguyên tiếng Anh và mỗi khái niệm chỉ có một tên: client, server, peer,
  transport, frame, handshake, greeting, verdict, query, schema, fingerprint, session, message,
  heartbeat, probe, tick. Tên frame/trạng thái/sự kiện viết hoa: DATA, POLL, REPLY, HANDSHAKE,
  HANDSHAKING, READY, CLOSED, CONNECT, DISCONNECT, MESSAGE, HANDSHAKE FAILED.
- Tên trạng thái và sự kiện viết hoa (READY, DISCONNECT) là khái niệm trừu tượng. Không ràng buộc vào ngôn ngữ nào. Mỗi bản triển khai tự đặt tên theo quy ước của mình.

---

## 1. Toàn bộ vòng đời

```
   ┌────────────────────────────────────────────────────────────────┐
   │ GIAI ĐOẠN 1 - XÂY DỰNG ỐNG (ngoài Fomoxa)                      │
   │ Kết nối TCP / Handshake TLS / Nâng cấp WebSocket               │
   └────────────────────────────────────────────────────────────────┘
                              ↓ Đường ống đã được thông quan
   ┌────────────────────────────────────────────────────────────────┐
   │ GIAI ĐOẠN 2 - HANDSHAKE FOMOXA §3                              │
   │ greeting (có fingerprint schema) → verdict 1 byte              │
   │ (đôi khi chèn một vòng query ở giữa - §3.3.2)                  │
   │ trạng thái: HANDSHAKING                                        │
   │ ứng dụng chưa gửi bất cứ điều gì                               │
   └────────────────────────────────────────────────────────────────┘
                              ↓ verdict = chấp nhận
   ┌────────────────────────────────────────────────────────────────┐
   │ GIAI ĐOẠN 3 - HOẠT ĐỘNG §4 §5 §6                               │
   │ trạng thái: READY                                              │
   │ ứng dụng gửi/nhận message · heartbeat chạy nền                 │
   └────────────────────────────────────────────────────────────────┘
                              ↓ một trong ba
   ┌────────────────────────────────────────────────────────────────┐
   │ GIAI ĐOẠN 4 - KẾT THÚC §7                                      │
   │ ứng dụng tự động dừng · bên kia đóng · bên kia im lặng quá lâu │
   │ trạng thái: CLOSED                                             │
   └────────────────────────────────────────────────────────────────┘
```

Ba trạng thái session. Đi một chiều, không quay lại:

```
   HANDSHAKE ──chấp nhận──> READY ──────────> CLOSED
       │                                        ↑
       └──từ chối / hết hạn─────────────────────┘
```

---

## 2. Định dạng trên dây

Phần này cấm thay đổi. Hai bản triển khai bất kỳ phải sinh ra và đọc đúng những byte này.

### 2.1 Frame phong bì

Mọi thứ đi trên dây đều là frame. Mọi frame bắt đầu bằng một byte loại:

```
   ┌──────────────┬───────────────────────────────┐
   │ Loại 1 byte  │ nội dung, tùy thuộc vào loại  │
   └──────────────┴───────────────────────────────┘

   Loại giá trị byte:
     0 DATA → phần thân là bản ghi dữ liệu (§2.2)
     1 POLL → không có nội dung. Một byte là toàn bộ frame.
     2 REPLY → không có nội dung. Một byte là toàn bộ frame.
     3 HANDSHAKE → nội dung có độ dài + nhiều byte mờ (§2.3)

   Bất kỳ loại byte nào khác 0-3 đều là vi phạm.
```

### 2.2 Frame DATA

```
   ┌────┬─────┬─────┬──────────────┬───────────────┬─────────────────────┐
   │ 00 │ 46  │ 4F  │ mã định danh │ độ dài        │ dữ liệu             │
   │    │ 'F' │ 'O' │ message      │ dữ liệu       │                     │
   │ 1B │ 1B  │ 1B  │ 4B u32 LE    │ 4B u32 LE     │ = độ dài byte       │
   └────┴─────┴─────┴──────────────┴───────────────┴─────────────────────┘
    ^0   ^1    ^2    ^3-6           ^7-10           ^11 trở đi

   Tổng kích thước = 11 + độ dài dữ liệu
```

- Hai byte `0x46 0x4F` là định danh cố định. Sai một trong hai → frame bị hỏng.
- Mã định danh message là *loại* message, do schema của ứng dụng chỉ định. Nó không tuần tự, không tăng dần. Cấm dùng để sắp xếp hay lọc trùng.
- Phần dữ liệu là byte mờ với transport và với lớp frame. Diễn giải nó là việc của lớp mã hóa dữ liệu, nằm ngoài tài liệu này.
- Độ dài dữ liệu tối đa: 16 MiB (16 777 216 byte). Vượt quá không được mã hóa, cũng không được chấp nhận khi giải mã.

### 2.3 Frame HANDSHAKE

```
   ┌────┬──────────────┬─────────────────────────────────┐
   │ 03 │ độ dài       │ nội dung handshake              │
   │ 1B │ 4B u32 LE    │ = độ dài byte                   │
   └────┴──────────────┴─────────────────────────────────┘
    ^0   ^1-4           ^5 trở đi

   Tổng kích thước = 5 + độ dài
```

Nội dung bên trong mờ, giải thích ở §3. Độ dài tối đa: 1 MiB. Đây là chốt chặn phân bổ, không phải giới hạn ngữ nghĩa. Nội dung handshake là dữ liệu điều khiển, không bao giờ cần lớn bằng dữ liệu ứng dụng.

### 2.4 Frame POLL và REPLY

Chính xác một byte: `0x01` và `0x02`. Không nội dung, không độ dài, không mã định danh. Chúng là dữ liệu điều khiển của transport. Chúng không bao giờ là message ứng dụng và không mang mã định danh. Toàn bộ không gian mã định danh thuộc về ứng dụng.

### 2.5 Đặt frame trên luồng byte

Trên luồng byte, các frame nối tiếp nhau, không dấu phân cách:

```
   ... │ frame │ frame │ frame │ ...
```

Frame tự mô tả: đọc byte loại là biết cần đọc thêm bao nhiêu. Bộ giải mã phải chạy tăng dần: nhận byte theo đợt tùy ý, giữ lại phần dở, chỉ trả frame khi đã đủ.

Mọi vi phạm frame trên luồng byte đều là chí mạng. Không có cách bắt lại nhịp. Mọi byte hợp lệ đều mở đầu bằng một byte loại hợp lệ. Gặp thứ khác nghĩa là phía kia không còn đáng tin để phân định frame. Đóng session.

Bộ giải mã đã báo lỗi thì nhiễm độc vĩnh viễn: mọi lệnh gọi sau đó đều trả lại đúng lỗi đó. Nhờ vậy không ai vô tình viết được kiểu "cứ đọc rồi xem".

### 2.6 Đặt frame lên gói

Trên transport kiểu gói, một gói chứa chính xác một frame - không hơn, không kém.

- Gói hết trước khi frame nó khai báo kết thúc → hỏng.
- Gói còn dư byte sau khi đọc xong một frame → hỏng.

Vi phạm frame trên gói không chí mạng. Vứt đúng gói đó rồi đi tiếp. Một gói không làm hỏng trạng thái phân tích dùng chung như trên luồng byte. Ở đây không có trạng thái dùng chung.

Bất đối xứng này là có chủ ý. Cả hai phía phải làm đúng.

### 2.7 Bảng trần

| Số lượng | Trần | Ghi chú |
|---|---|---|
| Dữ liệu của message | 16 MiB | Chung cho tất cả các triển khai |
| Một frame DATA | 16 MiB + 11 byte | |
| Nội dung của frame HANDSHAKE | 1 MiB | Dừng phân bổ |
| Số message được khai báo trong greeting | 1 000 000 | Dừng số học, xem §3.3 |
| Số mục trong một query | 1 000 000 | Không vượt quá số message trong greeting, xem §3.3.2 |
| Số vòng query mỗi session | 1 | Handshake không quá hai lần |

---

## 3. Luồng: greeting và kiểm tra fingerprint

### 3.1 Toàn cảnh

```
   CLIENT                                           SERVER
     │                                                │
     │  transport vừa được thông quan                 │ transport vừa được thông quan
     │  tạo greeting ngay lập tức                     │
     │                                                │
     │ ── HANDSHAKE (greeting) ─────────────────────> │
     │                                                │ ① kiểm tra frame
     │                                                │ ② kiểm tra phiên bản
     │                                                │ ③ so sánh fingerprint
     │                                                │
     │      Bây giờ bạn có thể quyết định được không? ┤
     │                                                │
     │ <── HANDSHAKE( verdict, 1 byte ) ───────────── │ Được rồi - xong ở đây
     │                                                │
     │ <── HANDSHAKE( 4 = NEED MORE + danh sách ) ─── │ Chưa chưa - §3.3.2
     │ ── HANDSHAKE (trả lời query) ────────────────> │
     │                                                │ ③′ điền phần còn thiếu
     │ <── HANDSHAKE( verdict, 1 byte ) ───────────── │
     │                                                │
     │ 0 → READY                                      │ 0 → READY peer
     │ ≠0 → HANDSHAKE FAILED, đóng                    │ ≠0 → gửi verdict rồi đóng
```

Bốn điều định hình luồng này:

1. Client chào, server phán. Luôn luôn. Thường xong trong một vòng. Đúng một trường hợp server phải hỏi thêm trước khi phán, xem §3.3.2. Không bao giờ quá hai vòng. Client không bao giờ là bên phán.
2. Server là bên duy nhất so sánh. Client gửi schema của mình và không bao giờ thấy schema của server. Kể cả khi bị hỏi thêm, client cũng chỉ trả lời. Hỏi thì đáp, đừng tự phán.
3. Không có byte phân tách giữa greeting và phản hồi query. Cả hai đều nằm trong frame HANDSHAKE; *vai trò cộng trạng thái* quyết định cách đọc. Server đọc payload đầu là greeting, payload thứ hai (nếu có) là phản hồi query. Chiều ngược lại, client đọc byte đầu: `0-3` là verdict, `4` là query.
4. Verdict là một byte, và chính giá trị byte đó là lý do từ chối. Không có bảng tra thứ hai. Giá trị `4` không phải verdict mà là một yêu cầu, nên nó không kết thúc session.

### 3.2 Nội dung greeting

Đây là nội dung bên trong frame HANDSHAKE (§2.3):

```
   ┌───────────────────────────────────────────────────────┐
   │ phiên bản giao thức     4B u32 LE   luôn = 2          │
   │ fingerprint schema      8B u64 LE   của cả hai schema │
   │ số message              4B u32 LE                     │
   ├───────────────────────────────────────────────────────┤
   │ lặp lại "số message" nhiều lần, mỗi mục 14 byte:      │
   │   mã định danh message  4B u32 LE                     │
   │   số trường n           2B u16 LE                     │
   │   fingerprint h_n       8B u64 LE                     │
   └───────────────────────────────────────────────────────┘

   Độ dài phải chính xác bằng 16 + 14 × số lượng message.
   Lệch một byte = thất bại, không có ngoại lệ.
```

Ba đại lượng, mỗi đại lượng một vai trò:

- fingerprint schema - đại diện cho toàn bộ bộ message. Khớp nghĩa là hai bên dựng từ cùng một schema. Server chấp nhận ngay, không đọc mục nào.
- `n` - số trường. Thiết kế cũ thiếu đúng con số này. Thiếu `n` thì không tính được `k = min(n_client, n_server)`, tức là không kiểm tra được điều kiện của RFC-0002 §9.1.
- `h_n` - fingerprint của cả message, tức fingerprint tiền tố tại `k = n`. Chuỗi fingerprint tiền tố vẫn tồn tại đầy đủ, nhưng nằm trong bộ nhớ tĩnh của mỗi bên. Nó không đi qua dây. Mỗi bên giữ chuỗi của mình; đúng một giá trị bay qua mạng. Vì sao thế là đủ: §3.3.1.

Cách tính fingerprint do đặc tả schema của Fomoxa quy định, nằm ngoài tài liệu này. Ứng dụng cung cấp cho mỗi message một bộ ba: mã định danh, `n`, chuỗi fingerprint tiền tố. Transport chỉ so sánh, không tự tính. Transport đòi cách tính đó đúng hai tính chất:

```
1. fingerprint tiền tố thứ k chỉ phụ thuộc vào k trường đầu tiên,
   theo đúng thứ tự khai báo. Không thể sử dụng trường k+1 trở đi
   ảnh hưởng đến nó.

2. Hai message có k trường đầu tiên giống nhau phải có cùng một fingerprint
   tiền tố thứ k - trên tất cả các ngôn ngữ, tất cả cách triển khai.
```

### 3.3 Server quyết định như thế nào?

Ba cánh cửa, đi theo thứ tự này, dừng ở cửa đầu tiên trượt:

```
   nhận nội dung greeting
        │
        ▼
   ① FRAME có đúng không?
        │ ngắn hơn 16 byte?
        │ hay độ dài ≠ 16 + 14×số lượng?
        │ hoặc số lượng > 1 000 000?
        ├── sai ──> verdict = 3 (BROKEN) ──> đóng
        ▼ vâng
   ② PHIÊN BẢN 2 phải không?
        ├── không ──> verdict = 1 (WRONG VERSION) ──> đóng
        ▼ vâng
   ③ SO SÁNH FINGERPRINT
        │
        ├─ schema fingerprint phù hợp với server?
        │  └── MATCH ──────────────────> verdict = 0, ACCEPTED
        │                                   (chưa đọc đoạn nào)
        │
        └─ KHÁC → duyệt từng mục trong greeting.
                  Đối với mỗi mã định danh mà server cũng có, hãy đặt
                      n_c = số trường được client khai báo
                      n_s = số trường mà server có

                  ⓐ Hai bên có bằng nhau không?
                        └── CÓ ──> message này giống hệt, tiếp tục

                  ⓑ n_c == n_s (nhưng h_n thì khác)?
                        └── CÓ ──> cùng độ dài nhưng nội dung khác nhau
                                   → không phải là tiền tố
                                   → verdict = 2 ──> đóng

                  ⓒ n_c < n_s?
                        └── CÓ ──> so sánh h_n của client với h_{n_c} lấy từ
                                   chuỗi tiền tố địa phương của server
                                     ├── KHÁC ──> verdict = 2 ──> đóng
                                     └── MATCH ─> khớp tiền tố, tiếp tục

                  ⓓ n_c > n_s
                        ├── n_s == 0 ──> tiền tố trống, luôn khớp, tiếp tục
                        └── n_s > 0 ──> chưa quyết định.
                                        Viết vào bảng query: (định danh, n_s)

                  Duyệt tất cả:
                     danh sách query trống ──> verdict = 0, ACCEPTED
                     danh sách hỏi có ──> gửi 4 = NEED MORE (§3.3.2)
```

Cổng ③ là phép thử tiền tố của RFC-0002 §9.1, viết bằng byte. `k = min(n_c, n_s)` là số trường của bên ngắn hơn. Hai fingerprint tiền tố thứ k bằng nhau nghĩa là `k` trường đầu của hai bên khớp cả kiểu lẫn thứ tự. Đó đúng là định nghĩa "chuỗi ngắn hơn là tiền tố chính xác của chuỗi dài hơn".

Ba trong bốn nhánh giải quyết tại chỗ. Hoặc `h_n` client gửi đã đúng là `h_k` cần dùng. Hoặc server tra được `h_k` của mình trong chuỗi cục bộ. Chỉ nhánh ⓓ cần một giá trị mà chỉ client sinh được.

Chặn "số lượng > 1 000 000" ở cửa ① phải kiểm tra trước phép nhân `14 × số lượng`, không phải sau. Trên số nguyên 32 bit, một số khai báo gần 2³² làm phép nhân tràn. Phép kiểm tra độ dài khi đó lọt qua. Đó là cửa cho một greeting độc hại.

Lưu ý: `n` không tham gia phân tách frame. Mỗi mục dài đúng 14 byte, bất kể `n` bằng bao nhiêu. `n` sai chỉ làm so sánh trượt hoặc sinh query rỗng. Nó không bao giờ dẫn tới cấp phát theo con số client khai báo. Điểm bất ngờ nhất: schema khác nhau vẫn được chấp nhận.

Điều kiện chấp nhận không phải "hai bên giống hệt nhau". Nó là: trong mọi message mà cả hai cùng biết, các trường mà cả hai cùng có phải khớp. Cụ thể:

| Tình huống | Kết quả | Tại sao |
|---|---|---|
| Schema so sánh fingerprint | ✅ Chấp nhận | Schema tương tự, không cần cân nhắc thêm |
| Client có thêm một server message không xác định | ✅ Chấp nhận | Server không bao giờ nhận message đó |
| Server có thêm message mà client không biết | ✅ Chấp nhận | Client chưa bao giờ gửi message đó |
| Message chung, một bên có trường bổ sung ở cuối | ✅ Chấp nhận | Khớp tiền tố - khác biệt về phiên bản hợp lệ, RFC-0002 §9.1 |
| Message chung, một bên có `n = 0` | ✅ Chấp nhận | Tiền tố trống là tiền tố của mọi thứ |
| Message chung, lệch tại một chỉ mục trong phần chung | ❌ Từ chối (2) | Trường trong chỉ mục đó đã thay đổi - thay đổi giao thức, RFC-0002 §5.1 |
| Message chung, đổi tên một trường (không thay đổi kiểu và thứ tự) | ❌ Từ chối (2) | Đổi tên = xóa + thêm tại cùng chỉ mục; fingerprint có hash tên trường, §3.3.1 |

Nói cách khác: hai bên dùng schema khác nhau vẫn nói chuyện được, miễn phần giao nhau khớp. Khớp ở cấp bộ message, và khớp ở danh sách trường trong từng message.

Nhờ vậy mỗi bên nâng cấp dần được, không phải dừng cả hệ thống, không phụ thuộc transport. RFC-0002 §9.1 cho phép hai bên đọc được của nhau khi chuỗi kiểu của bên ngắn hơn là tiền tố đúng của bên dài hơn. RFC-0003 §8.6 ghim ca đó thành hai vector V-001 và V-002: "không được phép bị từ chối". Hai vector đó ràng buộc bộ giải mã (RFC-0003 §1.3, phép thử tương thích). Ở tầng handshake, cổng ③ tự nguyện giữ đúng ca nối thêm ở cuối bằng chuỗi tiền tố. Đó là lý do duy nhất chuỗi tiền tố tồn tại (§3.3.1).

Chỉ bị chặn khi hai bên đặt hai trường khác nhau vào cùng một chỉ mục trong phần chung. Lúc đó một vị trí mang hai nghĩa. Vị trí là định danh duy nhất trên dây, nên không còn cách nói chuyện an toàn. Một message lệch là đủ để từ chối cả session, dù mọi message khác đều khớp. Không có "chấp nhận một phần": session mở hoàn toàn, hoặc không mở.

Một lưu ý về bản chất cổng ③: nó tin danh sách do client tự khai. Client khai `số message = 0` sẽ lọt cổng ③. Đây là phép kiểm tra tương thích giữa hai bên hợp tác, không phải cơ chế bảo mật. Đừng dùng nó như bảo mật.

### 3.3.1 Vì sao thế là đủ, và vì sao nhánh ⓓ buộc phải hỏi

Câu hỏi của RFC-0002 §9.1 không phải "giống hay khác", mà là *chuỗi ngắn hơn có phải tiền tố chính xác của chuỗi dài hơn không*. Một fingerprint duy nhất cho cả message không trả lời được:

```
     Mục { id, tên } → 0xA3F1…
     Mục { id, name, hp } → 0x77B0… không liên quan gì đến dòng trên

   Sau khi băm, mối quan hệ tiền tố biến mất.
```

Vì vậy phải có chuỗi fingerprint tiền tố. `h_k` cam kết đúng `k` trường đầu. `k = min(n_c, n_s)` chính là điều kiện §9.1, dịch sang byte. Chỉ cần so một mốc: lệch ở bất kỳ chỉ mục nào nhỏ hơn `k` cũng làm `h_k` khác đi.

Nhưng chuỗi không cần đi qua dây. Mỗi bên giữ chuỗi của mình trong bộ nhớ tĩnh. Hóa ra ba trong bốn nhánh không cần thêm gì từ phía kia:

```
   ⓐ h_n bằng nhau → giống nhau. Không cần K.
   ⓑ n_c == n_s → k = n_c = n_s, và h_n khác → bị từ chối. Không cần gì hơn nữa.
   ⓒ n_c < n_s → k = n_c, mà h^client_{n_c} là h_n client vừa gửi.
                 server tra cứu h_{n_c} của nó trong chuỗi cục bộ. Đủ.
   ⓓ n_c > n_s → k = n_s. Cần h^client_{n_s} - chỉ có client mới có thể xuất hiện.
```

Nhánh ⓓ không tránh được. Đây là giới hạn về thông tin, không phải lỗi thiết kế. Client phải trả lời một chỉ mục `n_s` mà lúc gửi greeting nó chưa biết. Muốn phủ hết mọi chỉ mục trong một vòng thì tốn `O(số trường)`, tức là gửi cả chuỗi. Không có đường tắt:

- hàm băm một chiều: không suy ngược từ `h_{n_c}` ra `h_{n_s}`;
- đổi sang XOR hay tổng tích lũy cũng vậy: muốn suy ngược vẫn phải có đúng các trường chưa gửi;
- cây Merkle cho cam kết nhỏ, nhưng *bằng chứng mở* mới là thứ phải gửi. Phủ mọi `k` thì đắt hơn gửi cả chuỗi.

Còn đúng hai lựa chọn: gửi cả chuỗi mỗi lần, hoặc gửi một giá trị rồi hỏi thêm khi rơi vào ⓓ. Greeting 14 byte mỗi message chọn cách sau.

Khi nào nhánh ⓓ nổ? Không phải chuyện "bên nào nâng cấp trước". Đơn giản là server có ít trường hơn client trong một message đã đổi. Hai đường dẫn tới đó:

| | `n_c` vs `n_s` | Bạn có câu hỏi? |
|---|---|---|
| Client đến trước, chắp thêm trường | `n_c > n_s` | ✅ |
| Server đến trước, xóa một trường | `n_c > n_s` | ✅ |
| Server đến trước, nối thêm | trường `n_c < n_s` | ❌ chi nhánh ⓒ |
| Client đến trước, xóa | trường `n_c < n_s` | ❌ chi nhánh ⓒ |

Đây là yêu cầu chặt hơn §9.1, và nó hợp lệ. Tiêu chí §9.1 chỉ xét kiểu và thứ tự; tên trường không có trên dây. Fingerprint của lớp schema thì băm cả tên trường (`fomoxa-fingerprint/2` §5 - đây chính là cơ chế bắt ca hai trường cùng kiểu bị tráo chỗ). Vì vậy cổng ③ từ chối một cặp peer chỉ khác nhau một lần đổi tên. Dù byte hai bên sinh ra giống hệt.

Đây không phải lỗi. Đổi tên một trường chính là xóa trường cũ và thêm một trường khác vào cùng chỉ mục. Nó cùng hạng với chèn giữa hoặc xóa giữa - thứ mà §9.1 xếp vào thay đổi giao thức, không phải khác biệt phiên bản. §9.1 không yêu cầu chấp nhận ca này. Nó chỉ không nhìn thấy ca này, vì đo lệch bằng kiểu, mà hai trường `f32` thì đọc ra như nhau. Đúng điểm mù đó được mục "Phạm vi của phần này" giao lại cho tầng trên: "Việc phát hiện và chặn các trường hợp đó thuộc về tầng schema cấp cao hơn, nơi có sẵn tên field và khai báo đầy đủ để so sánh." Cổng ③ đang làm đúng việc được giao.

Nhớ kỹ ranh giới này, vì hai tầng có hai luật khác nhau. Bộ giải mã không bao giờ được từ chối một luồng byte mà §9.1 coi là hợp lệ. RFC-0003 §1.3 xếp V-001/V-002 vào phép thử tương thích của bộ giải mã. Handshake thì được phép chặt hơn, vì nó thấy thứ bộ giải mã không thấy: tên trường và khai báo đầy đủ.

Giá phải trả ghi rõ trong `SPEC-FINGERPRINT.md` §5, và là đánh đổi có chủ đích. Một lần đổi tên thuần túy cũng bị chặn: hỏng ngay, ồn ào, vô hại. Đổi lại, ca tráo chỗ hai trường cùng kiểu không bao giờ lọt. Ca đó hỏng im lặng và làm hỏng dữ liệu vĩnh viễn. Từ `fomoxa-fingerprint/2`, tên trường được chuẩn hóa trước khi băm: bỏ gạch dưới, gạch ngang, khoảng trắng, hạ chữ hoa. Nhờ vậy khác biệt quy ước đặt tên giữa các ngôn ngữ - `id`/`ID`/`Id`, `player_id`/`PlayerID` - không đổi fingerprint. Chỉ đổi tên thật (`x` → `position_x`) mới đổi.

### 3.3.2 Query: verdict 4 và vòng lặp phản hồi

Chỉ dùng cho nhánh ⓓ. Nội dung nằm trong frame HANDSHAKE, như mọi thứ khác trong phần này.

Server → client. Byte đầu là `4`, để client phân biệt ngay với verdict:

```
   ┌─────────────────────────────────────────────────────┐
   │ 04                      1B          NEED MORE       │
   │ số lượng mục            4B u32 LE                   │
   ├─────────────────────────────────────────────────────┤
   │ lặp lại, mỗi mục 6 byte:                            │
   │   mã định danh message  4B u32 LE                   │
   │   chỉ số yêu cầu n_s    2B u16 LE   (luôn luôn ≥ 1) │
   └─────────────────────────────────────────────────────┘

   Độ dài phải chính xác bằng 5 + 6 × số lượng mục.
```

Server chỉ được hỏi các mã định danh client đã khai trong greeting. Và chỉ khi `1 ≤ n_s < n_c`. Query vi phạm quy tắc này là lỗi của server.

Client → server. Không có byte phân biệt: server mong đợi đúng một payload và đọc nó theo cách tương tự như khi đọc phản hồi query (§3.1, mục 3).

```
   ┌───────────────────────────────────────────────────────────────┐
   │ số lượng mục            4B u32 LE                             │
   ├───────────────────────────────────────────────────────────────┤
   │ lặp lại, mỗi mục 12 byte:                                     │
   │   mã định danh message  4B u32 LE                             │
   │   tiền tố fingerprint   8B u64 LE   ở chỉ mục query chính xác │
   └───────────────────────────────────────────────────────────────┘

   Độ dài phải chính xác bằng 4 + 12 × số lượng mục.
   Phải trả lời các mục được server yêu cầu theo đầy đủ và đúng.
```

Server so sánh: mỗi mục, fingerprint client gửi phải bằng `h_{n_s}` trong chuỗi cục bộ của server. Thiếu một mục → verdict 2. Khớp hết → verdict 0.

Ba quy tắc chặn mọi ý định sinh thêm trạng thái:

```
1. Tối đa một query mỗi session. Sau khi nhận được phản hồi, server phải đưa ra verdict đã quyết định, không thắc mắc nữa.
2. Client nhận được byte 4 lần thứ hai → coi như HANDSHAKE FAILED.
3. Thời hạn handshake của client (§3.5) được tính cho toàn bộ quá trình, không được đặt lại sau mỗi vòng đấu. Hai vòng vẫn có cùng một thuật ngữ.
```

Trên transport kiểu gói, hai gói query thêm này có thể mất. Mất thì handshake hết hạn và thất bại: session không mở, ứng dụng thử lại. Đây không phải chế độ lỗi mới. Fomoxa không truyền lại (RFC-0001 §2), nên mọi handshake trên transport không tin cậy đều là best-effort. Đổi lại, greeting 14 byte mỗi message thường chỉ tốn hai gói. Không phải xé nhỏ một greeting lớn ở mỗi lần kết nối.

Muốn bỏ lượt khứ hồi thừa thì thử lạc quan trước: handshake thất bại thì lần kết nối sau gửi thẳng chuỗi tiền tố, không chờ query. Đây là lựa chọn triển khai, không phải yêu cầu của tài liệu này.

### 3.4 Client đọc verdict như thế nào?

```
   nhận tải trọng từ server
        │
        ▼
   byte đầu tiên == 4?
        │
        ├── đúng ──> QUERY (§3.3.2)
        │            │ đã nhận được query một lần?
        │            │ └── rồi ──────> coi như HANDSHAKE FAILED
        │            │ frame query hợp lệ?  mã định danh có trong greeting?
        │            │ └── không ──> coi như frame hư hỏng, thất bại
        │            └──> gửi phản hồi query, sau đó chờ tiếp tục
        │                 (thời hạn không được đặt lại)
        │
        └── sai ──> VERDICT
                       │ độ dài chính xác là 1 byte?  và giá trị 3?
                       │ └── không ──> coi như HANDSHAKE FAILED
                       ▼ vâng
                    byte == 0?
                       ├── đúng ──> READY.  Từ đây ứng dụng có thể gửi.
                       └── sai ──> HANDSHAKE FAILED, lý do = chính giá trị byte
                                   → session đóng ngay lập tức, ứng dụng không nhận được sự kiện bổ sung
```

Client không bao giờ tự đánh giá schema, kể cả khi bị hỏi. Nó chỉ tra chuỗi tiền tố cục bộ tại đúng chỉ mục rồi gửi lại. Quyết định cuối vẫn thuộc về server (§3.1, mục 2).

Bảng lý do (giá trị byte = lý do):

| Byte | Ý nghĩa | Nguyên nhân thực tế |
|---|---|---|
| 0 | Chấp nhận | |
| 1 | Phiên bản giao thức sai | Hai bên chạy hai phiên bản khác nhau của frame handshake: bố cục greeting đã đổi, ví dụ một bên còn gửi v1. Không liên quan tới schema, cũng không phải "hai phiên bản Fomoxa" |
| 2 | Schema xung đột | Trong một message mà cả hai bên đều biết, cả hai bên đặt hai trường khác nhau vào cùng một chỉ mục trong phần chung - chèn vào giữa, xóa ở giữa, đảo ngược thứ tự hoặc thay đổi loại hoặc đổi tên một trường (đổi tên = xóa + thêm tại cùng một vị trí, §3.3.1). Các trường nối thêm ở cuối không nằm ở đây (§3.3.1) |
| 3 | Greeting hỏng | Lỗi ở phía gửi, hoặc thứ không phải Fomoxa đang gõ vào cổng này |

Giá trị `4` không có trong bảng: nó không phải verdict mà là một yêu cầu (§3.3.2), và không bao giờ kết thúc session. Mọi giá trị từ `5` trở lên đều là hỏng.

### 3.5 Hạn handshake - hai vai tính khác nhau

Đây là chỗ hay hiểu nhầm, vì hai vai dùng hai cơ chế khác hẳn nhau:

```
   CLIENT                                           SERVER
   ──────                                           ──────
   HARD DEADLINE
   Đồng hồ chạy kể từ thời điểm session được tạo.   Không có thời hạn tuyệt đối.
   Đã quá thời hạn handshake mà vẫn chưa xong.      Chỉ cần đếm "đã im lặng được bao lâu".
   → thất bại, kết thúc.                            Có peer nào vẫn trả lời probe
                                                    không? → vẫn chưa được coi là hết hạn.
```

Hệ quả thực tế: một client cứ trả lời probe mà không bao giờ gửi greeting hợp lệ. Nó giữ một chỗ trên server vô thời hạn. Đây là hành vi có chủ ý. Đặc tả không đặt trần tuyệt đối nào; đặt thêm là lệch khỏi thiết kế.

Hạn của client phủ toàn bộ handshake, kể cả các vòng query (§3.3.2). Đồng hồ không reset sau mỗi vòng; hai vòng dùng chung một hạn. Reset là mở đường cho một server ác ý kéo dài session bằng cách hỏi mãi.

### 3.6 Frame khác đến trong lúc handshake

Frame không phải HANDSHAKE vẫn có thể đến trước khi handshake xong. Không có cái nào được coi là vi phạm:

| Frame đến | Client (HANDSHAKING) | Server (peer HANDSHAKING) |
|---|---|---|
| DATA |*Bỏ đi, không giao ứng dụng | Bỏ, nhưng có tính năng hoạt động |
| POLL | Vẫn trả REPLY, không sinh sự kiện | Trả REPLY, tính là hoạt động |
| REPLY | Bỏ qua | Tính là hoạt động |
| HANDSHAKE hai lần (sau khi đã xong) | Bỏ qua | Bỏ qua |

Đừng nhầm dòng cuối với vòng query. Trong lúc HANDSHAKING, một frame HANDSHAKE thứ hai là bình thường và có nghĩa: với client nó là query, với server nó là phản hồi query (§3.3.2). Chỉ frame HANDSHAKE đến sau khi đã có verdict mới bị vứt. Mỗi session tối đa một vòng query; frame HANDSHAKE thứ ba trong lúc handshake là hỏng.

Vì sao client vẫn phải trả REPLY khi chưa READY? Server được phép probe client trước khi trả lời một greeting hợp lệ. Client im lặng vì "chưa handshake xong" thì server kết luận nó chết và cắt. Đúng lúc nó đang chờ server trả lời. Trả REPLY ở đây là điều kiện để quy trình có cơ hội hoàn tất.

---

## 4. Luồng: heartbeat chạy khi nào

### 4.1 Nguyên tắc: probe khi im lặng, không phải gửi định kỳ

Đây là điểm khác biệt lớn nhất so với cách làm thông thường:

```
   ❌ THÔNG THƯỜNG                 ✅ FOMOXA
   cứ 5 giây gửi 1 probe           chỉ probe khi im lặng 5 giây
   dù đang truyền dữ liệu          đang chạy dữ liệu = đã biết peer còn sống
```

Peer nào đang gửi thì không nhận thêm probe thừa. Trên hệ thống chạy 60 tick/giây, heartbeat gần như không sinh gói nào.

### 4.2 Máy trạng thái heartbeat

Mỗi bên đo chiều nghe của mình: client đo server im bao lâu, server đo client im bao lâu. Không bao giờ dùng chung một đồng hồ cho hai chiều.

```
        ┌────────────────────────────────────────┐
        │ NORMAL                                 │
        │ đếm: im lặng bao lâu rồi               │
        └────────────────────────────────────────┘
             │                            ▲
             │ im lặng ≥ cửa sổ im lặng   │ nhận được bất kỳ
             │ → gửi đúng một probe       │ hợp lệ frame
             ▼                            │ → xóa sạch, quay lại
        ┌────────────────────────────────────────┐
        │ PROBING                                │
        │ đếm: probe bao lâu rồi mà chưa thấy gì │
        └────────────────────────────────────────┘
             │
             │ ≥ giới hạn phản hồi mà vẫn không có gì về
             ▼
        ┌────────────────────────────────────────┐
        │ PEER DEAD                              │
        └────────────────────────────────────────┘
```

Hai điều rút ra từ hình trên:
- Không ai cắt ngay cuối cửa sổ im lặng. Luôn có một probe trước đó. Thời gian tệ nhất để phát hiện chết = `cửa sổ im lặng + giới hạn phản hồi`.
- Mọi frame hợp lệ đều tính là dấu hiệu sống: DATA, POLL, REPLY, HANDSHAKE. Không nhất thiết phải là REPLY.

### 4.3 Heartbeat bật lúc nào - hai vai khác nhau

```
   CLIENT
   ──────
   HANDSHAKE: không có heartbeat. Hoàn toàn không.
                  handshake chỉ có một giới hạn cứng duy nhất (§3.5).
                  Trong giai đoạn này, server là bên giám sát
                  kết nối còn sống hay không.
                     ↓
   READY: heartbeat bắt đầu vào đúng thời điểm này.
                  cửa sổ im lặng = chu kỳ heartbeat
                  giới hạn phản hồi = giới hạn heartbeat


   SERVER (cho mỗi peer)
   ─────────────────────
   HANDSHAKE: heartbeat chạy ngay lập tức kể từ thời điểm peer xuất hiện.
                  cửa sổ im lặng = giới hạn handshake ← rộng hơn
                  giới hạn phản hồi = giới hạn heartbeat
                     ↓
   READY: chỉ thay đổi cửa sổ im lặng, không có cài đặt lại nào khác.
                  cửa sổ im lặng = chu kỳ heartbeat ← thu hẹp
                  giới hạn phản hồi = giới hạn heartbeat ← vẫn giữ nguyên
```

Vì sao handshake phía client không có heartbeat? Client vừa gửi greeting và đang chờ đúng một thứ. Không có gì cần "làm mới định kỳ", và nó đã có hạn cứng. Cơ chế thứ hai chỉ là thừa.

Vì sao server thì có? Server có thể đang giữ nhiều peer nửa mở. Nó cần phân biệt "client chạy chậm" với "client đã bốc hơi". Chỉ probe mới phân biệt được.

Lưu ý cách server chuyển sang READY: nó chỉ đổi cửa sổ im lặng. Không đụng mốc hoạt động cuối, không hủy probe đang bay. Một probe gửi ngay trước khi greeting đến vẫn còn hiệu lực và vẫn đang chờ trả lời.

### 4.4 Tham số ba thời gian

| Thông số | Mặc định | Ý nghĩa |
|---|---|---|
| Giới hạn handshake | 5 giây | Client: thời hạn cho toàn bộ quá trình handshake. Server: cửa sổ im lặng khi peer chưa handshake xong |
| Chu kỳ heartbeat | 5 giây | Im lặng bao lâu rồi probe một lần |
| Giới hạn heartbeat | 15 giây | Sau một probe, lâu ngày không trả lời coi như đã chết |

Thời gian tệ nhất để phát hiện peer chết ở READY: 5 + 15 = 20 giây.

Đây là giá trị mặc định đề xuất; ứng dụng cấu hình lại được. Nhưng quan hệ giữa chúng thì không được đảo. Giới hạn phản hồi phải đủ lớn so với độ trễ khứ hồi. Nếu không, một peer khỏe vẫn bị tuyên bố là chết.

### 4.5 Toàn bộ chu trình probe

```
   A                                               B
   │                                               │
   │ ... hai bên đang trao đổi DATA bình thường    │
   │ mỗi frame nhận được sẽ đặt lại đồng hồ của A  │
   │                                               │
   │ ... B đã ngừng gửi ...                        │
   │                                               │
   │ Đồng hồ im lặng của A chạy: 1s 2s 3s 4s 5s    │
   │                                               │
   │ ── POLL ────────────────────────────────────> │ ① chính xác một lần,
   │ A chuyển sang trạng thái PROBING              │    không lặp lại
   │                                               │
   │ <──────────────────────────── REPLY ───────── │ ② B còn sống
   │ A xóa trạng thái probing, trở lại bình thường │
   │ Ứng dụng của A nhận được sự kiện REPLY        │
   │                                               │
   │ ... hoặc B chết thật rồi ...                  │
   │                                               │
   │ 15 giây trôi qua, không có gì quay lại        │
   │ A tuyên bố B đã chết → sự kiện DISCONNECT     │ ③
```

Ở ① chỉ đúng một probe cho mỗi cửa sổ, không phải mỗi tick một cái. Ở ②, bất kỳ frame nào từ B cũng đủ để xóa probe. Không nhất thiết là REPLY; B gửi DATA cũng được.

---

## 5. Luồng: gửi message

### 5.1 Đường dẫn đầy đủ

```
   ỨNG DỤNG
    │ gửi(mã định danh = 42, dữ liệu = 100 byte)
    ▼
   ① KIỂM TRA TRẠNG THÁI
    │ chưa READY? → trả về lỗi "chưa sẵn sàng" ngay lập tức, không gửi gì cả
    ▼
   ② ĐÓNG GÓI VÀO FRAME (§2.2)
    │ [00] ['F'] ['O'] [42 u32 LE] [100 u32 LE] [100 byte dữ liệu]
    │  1    1     1     4           4            100 = 111 byte
    │ dữ liệu được sao chép tại đây → ứng dụng có thể được phát hành ngay lập tức
    │ sau khi lệnh gửi trở lại
    ▼
   ③ FRAME CŨ CÓ BỊ KÍN?
    │ có → trả về lỗi "tắc nghẽn", không xếp hàng
    ▼ không
   ④ ĐẶT TRANSPORT
    │
    ├── ✔ xong rồi không giữ lại gì cả
    ├── ⏸ chép vào ô chờ, trả về ứng dụng: thành công
    ├── ⊘ trả về lỗi "quá lớn" cho ứng dụng, session vẫn sống
    └── ✖⚠ đánh dấu transport đã chết
```

### 5.2 Vì sao cấm gửi dữ liệu trước khi READY

Không phải chuyện hình thức. Trước khi có verdict, chưa ai xác nhận hai bên đọc cùng một bộ byte. Một message gửi sớm có thể tới một server ngay sau đó từ chối schema. Server đó diễn giải nó theo schema của mình và đọc ra giá trị khác thứ client gửi. Chặn ở đây là chỗ duy nhất chặn được.

### 5.3 Frame bị kẹt được đẩy ra khi nào?

```
   tick N   gửi frame ────────────────────────> transport: ⏸ → vào ô chờ
              ứng dụng vẫn chạy bình thường

   tick N+1 việc đầu tiên: đẩy nốt ô chờ ──────> transport: ⏸ → vẫn kẹt
              (nếu ứng dụng tiếp tục gửi trong tick này → lỗi "tắc nghẽn")

   tick N+2 việc đầu tiên: đẩy nốt ô chờ ──────> transport: ✔ → qua
              từ đây gửi lại bình thường
```

"Việc cũ trước" là bắt buộc, không phải tối ưu. Một frame mới ra dây trước phần còn lại của frame cũ thì bộ giải mã phía kia đọc hai frame dính vào nhau. Hỏng vĩnh viễn (§2.5).

---

## 6. Luồng: nhận message

### 6.1 Đường dẫn đầy đủ

```
   TRANSPORT
    │ hiển thị một frame hoàn chỉnh
    ▼
   ① MỞ GÓI (§2)
    │ đọc byte đầu tiên → loại frame
    │ DATA, sau đó kiểm tra 2 byte nhận dạng 'F' 'O'
    │ sai → frame bị hỏng (xử lý theo §2.5 hoặc §2.6 tùy theo loại transport)
    ▼
   ② ĐƯA VÀO MÁY TRẠNG THÁI SESSION
    │
    ├─ HANDSHAKE → xử lý verdict (§3.4) hoặc greeting (§3.3) tùy theo vai trò
    │
    ├─ POLL → trả lời REPLY ngay lập tức + (nếu READY) tạo ra sự kiện POLL
    │
    ├─ REPLY → (nếu READY) xóa trạng thái probing + tạo sự kiện REPLY
    │
    └─ DATA → chưa READY? bị bỏ rơi, không được gán cho ứng dụng
              READY?      đặt lại đồng hồ im lặng
                          Tạo sự kiện MESSAGE
    ▼
   ③ ĐƯA VÀO DANH SÁCH SỰ KIỆN
    ▼
   ỨNG DỤNG đọc danh sách sau khi tick quay trở lại
```

Lưu ý ở ②: REPLY cho POLL được gửi ngay tại chỗ, không chờ ứng dụng. Ứng dụng có thể không bao giờ đọc sự kiện POLL mà heartbeat vẫn chạy đúng.

### 6.2 Vòng đời dữ liệu sự kiện

```
   tick N
     ├── nhận frame, tham chiếu sự kiện vào bộ đệm bên trong
     ├── ứng dụng đọc sự kiện, sử dụng dữ liệu ← hợp lệ tại đây
     └── đánh dấu sự kết thúc

   tick N+1
     └── bộ đệm bị ghi đè ← tham chiếu cũ đến rác
```

Quy tắc: *dữ liệu trong một sự kiện chỉ sống tới tick kế tiếp của cùng session*. Cần giữ lâu hơn thì phải tự chép ra. Đây là quy tắc cho các ngôn ngữ trao dữ liệu bằng tham chiếu không sở hữu.

Bản triển khai nào chép dữ liệu sang ứng dụng, hoặc trao một đối tượng có sở hữu, thì không bị ràng buộc này. Nhưng phải ghi rõ điều đó trong tài liệu của mình. Lập trình viên chuyển từ bản này sang bản khác luôn mang theo giả định cũ.

---

## 7. Luồng: kết thúc session

Ba con đường về cùng một đích. Khác nhau ở chỗ ai phát hiện trước:

```
   ① TẮT ỨNG DỤNG
      cuộc gọi ứng dụng bị gián đoạn → dấu lõi đã đóng → yêu cầu transport đóng mềm
      → session CLOSED. Không có sự kiện bổ sung nào được tạo ra.

   ② PEER ĐÓNG HOẶC LỖI
      transport trả về ✖ hoặc ⚠ khi lõi yêu cầu
      → dấu vết cốt lõi transport người chết
      → nếu session không CLOSED: tạo DISCONNECT

   ③ PEER im lặng
      transport là hoàn toàn bình thường, không có gì về nó
      → đồng hồ đo heartbeat hết hạn ở bước tick
      → tạo DISCONNECT
```

Đúng một sự kiện kết thúc cho mỗi session. Handshake hỏng và đã sinh HANDSHAKE FAILED thì không có DISCONNECT theo sau, kể cả khi transport chết ngay sau đó. Phải giữ một cờ để bảo đảm điều này. Ứng dụng thường dọn dẹp trong trình xử lý sự kiện kết thúc, và dọn hai lần là lỗi.

---

## 8. Hướng dẫn triển khai: phía core

Hình dạng của một nhịp tick. Thứ tự này là bắt buộc, không phải khuyến nghị.

```
   tick (current_time):

       ── 0. Sự kiện CONNECT, chỉ một lần mỗi session trong đời ──
       Nếu kết nối không được báo cáo:
           đẩy sự kiện CONNECT
           Đánh dấu đã báo cáo

       ── 1. đẩy frame bị kẹt ra ── phải trước khi làm bất cứ việc gì khác
       miễn là chế độ chờ có dữ liệu:
           kết quả = Transport.send(phần còn lại)
           ✔ → dọn sạch khu vực chờ, thoát khỏi vòng đấu
           ⏸ → thoát khỏi vòng chơi, để lại tick sau
           lỗi → Transport_dead = true, thoát vòng lặp

       ── 2. cạn kiệt dữ liệu đến, trong giới hạn ──
       nếu transport không chết:
           ngân sách = GIỚI HẠN
           lặp lại trong khi ngân sách vẫn còn:
               kết quả = Transport.receive (bộ đệm)
               ⏸ → thoát khỏi vòng lặp
               ⤢ → tăng bộ đệm, không trừ ngân sách, lặp lại
               ✖⚠ → Transport_death = đúng, thoát khỏi vòng lặp
               ✔ → mở gói vào frame
                     đưa vào máy trạng thái session
                     áp dụng kết quả (xem bên dưới)
                     trừ ngân sách

       ── 3. chạy đồng hồ giao thức ──
       nếu transport không chết:
           result = session.tick(time) thời hạn handshake · probe ý kiến · tuyên bố đã chết
           Áp dụng kết quả

       ── 4. Transport bị dừng nhưng session không đóng ──
       nếu transport bị chết và session không CLOSED:
           kết quả = session.report_transport_close()
           Áp dụng kết quả

       ── 5. gán danh sách sự kiện cho ứng dụng ──
       danh sách trả lại
```

`áp dụng kết quả` - máy trạng thái session trả về tối đa một frame để gửi và tối đa một sự kiện:

```
   áp dụng (kết quả):
       Nếu có frame để gửi:
           Nếu hàng đợi có dữ liệu:
               đặt frame này sau ← không ghi đè
               nếu khu vực chờ vượt quá trần → Transport_dead = true
           ngược lại:
               viết frame đó để transport
               ⏸ → để ở nơi chờ
               ✖⚠ → transport_death = đúng

       nếu có một sự kiện:
           đẩy vào danh sách
           Nếu HANDSHAKE FAILED:
               transport_dead = đúng
               yêu cầu transport đóng mềm
```
"Tối đa một frame và tối đa một sự kiện" không phải sự đơn giản hóa. Đó đúng là thứ giao thức sinh ra. Ca kịch tính nhất là "trả lời rồi kết thúc": server gửi verdict từ chối rồi đóng session. Vòng query (§3.3.2) không phá bất biến này: cả query lẫn phản hồi query đều là frame không kèm sự kiện. Ứng dụng không thấy gì cho tới verdict cuối. Với nó, handshake hai vòng và handshake một vòng là một.

Ba điểm trong khối trên cần nói rõ. Thiếu điểm nào cũng thành lỗi im lặng.

Ô chờ cấm ghi đè. Frame ở đây - POLL, REPLY, verdict handshake - do core tự sinh. Không qua lệnh gửi của ứng dụng (khác kịch bản ở `01_overview.md` §5). Không có ai để báo "tắc nghẽn". Ghi đè ô chờ là bóp chết verdict từ chối: peer không biết vì sao bị từ chối và phải ngồi chờ tới hết hạn.

Hàng đợi chờ phải có trần, và trần đó phải thấp. Bản thân giao thức đã chặn số frame điều khiển sống cùng lúc: đúng một probe mỗi cửa sổ im lặng (§4.5), một REPLY cho mỗi POLL nhận được, tối đa một vòng query mỗi session (§3.3.2). Vượt quá một con số nhỏ nghĩa là một giả định đã vỡ, không phải tải cao bình thường. Con số cụ thể tùy bản triển khai. Bắt buộc phải có một con số.

Chạm trần thì kết thúc session, không ném lỗi. Bất biến B6 đòi đúng một sự kiện kết thúc mỗi session, và DISCONNECT khớp mẫu đó. Ném ngoại lệ ra ngoài nhịp tick thì không: nó phá cam kết "tick trả về một danh sách sự kiện". Một server đang duyệt nhiều peer sẽ mất những peer còn lại trong cùng nhịp đó.

Bốn luật bất di bất dịch của core:

1. Không bao giờ chờ. Core hỏi transport, nhận câu trả lời, đi tiếp. Không vòng lặp nào ở đây được phép quay lại chỉ để "thử lại ngay". Thử lại là việc của tick sau.
2. Máy trạng thái session không được biết transport tồn tại. Nó nhận frame, trả về frame và sự kiện. Không kết nối, không lỗi hệ điều hành, không tự đọc đồng hồ.
3. Mọi mốc thời gian truyền từ ngoài vào. Đây là điều kiện để chạy hết một chu kỳ hết hạn trong kiểm thử mà không phải chờ thật. Nó phải đúng ở mọi tầng.
4. Một sự kiện kết thúc, không hơn. Cờ "đã báo kết thúc" phải được kiểm tra ở mọi đường dẫn tới sự kiện kết thúc.

### Về đồng hồ

Thời gian phải lấy từ đồng hồ đơn điệu: chỉ tăng, không bị chỉnh. Cấm dùng đồng hồ treo tường. Một lần đồng bộ giờ lùi lại là đủ để làm hết hạn một handshake, hoặc giết một peer đang sống.

---

## 9. Hướng dẫn triển khai: phía transport

### 9.1 Trạng thái tối thiểu cần giữ

```
   tình trạng transport:
       Connection_background đối tượng kết nối của thư viện bạn sử dụng
       queue_to những gì đã được đọc về điều cốt lõi vẫn chưa được thực hiện
       queue_go phần không thể đẩy ra ngoài (nếu có)
       đóng để tất cả các cuộc gọi tiếp theo quay trở lại ✖ một cách nhất quán
```

### 9.2 Chức năng GỬI

```
   gửi (byte, độ dài):
       Nếu đóng cửa: trả ✖
       nếu độ dài > my_ceiling: trả về ⊘ ← không ⏸
       hãy thử đẩy kết nối nền xuống, đừng chờ
           Đẩy tất cả → trả tiền ✔
           Không thể đẩy → trả về ⏸ ← không nhận được bất kỳ byte nào
           peer đóng sạch → trả tiền ✖
           lỗi → trả lại ⚠
```

Điểm chết người: khi trả ⏸, transport không được nhận một byte nào. Nhận một nửa rồi trả ⏸ nghĩa là core gửi lại từ đầu, và phía kia nhận nửa đầu hai lần.

Thư viện nền của bạn có thể nhận một phần (kết nối thô) thì có hai cách. Một: khai báo đó là transport kiểu dòng byte và để core lo. Hai: tự đệm phần thừa vào `queue_to_go` rồi trả ✔. Cấm trả ⏸ ở giữa chừng.

### 9.3 Chức năng NHẬN

```
   nhận (vùng_đệm, sức_chứa):
       nếu đóng cửa và hàng_đợi_đến trống: trả ✖

       nếu Arrival_queue trống:
           đọc từ kết nối nền, không đợi
               không có gì → trả tiền ⏸
               peer đóng sạch → đóng = đúng, trả về ✖
               lỗi → đóng = đúng, trả lại ⚠
               có dữ liệu → được lưu trữ trong Arrival_queue

       (chỉ với transport gói)
       Nếu gói hàng đầu hàng chưa được giao đầy đủ: trả lại ⏸

       nếu gói đầu hàng > dung lượng:
           trả ⤢ với số byte cần thiết
           ← gói hàng vẫn xếp hàng, đừng vứt bỏ

       Sao chép gói đầu hàng vào bộ đệm
       loại bỏ nó khỏi hàng đợi
       trả tiền ✔
```

Điểm chết người: ở nhánh ⤢, cấm làm mất gói. Core sẽ nới bộ đệm và hỏi lại ngay. Vứt gói đi là dữ liệu mất vĩnh viễn và không ai biết.

### 9.4 Chức năng ĐÓNG MỀM và ĐÓNG CỨNG

```
   đóng_mềm():
       nếu đã_đóng: được rồi, không làm gì cả
       Gửi tín hiệu đóng giao thức nền
       KHÔNG đợi phía bên kia phản hồi
       KHÔNG phát hành bất cứ thứ gì - vẫn có thể có dữ liệu đến cần được đọc

   đóng_hẳn():
       đóng kết nối nền
       Giải phóng hàng đợi, bộ đệm, mọi thứ của bạn
       ← gọi chính xác một lần. Sau cuộc gọi này, không có chức năng nào khác được gọi.
```

Phân biệt hai hàm: đóng mềm là *lịch sự với peer*, đóng hẳn là *dọn nhà*. Giữa hai lệnh gọi đó, core vẫn có thể hỏi tiếp dữ liệu còn trên đường.

### 9.5 Tự kiểm trước khi đóng review

- [ ] Không hàm nào chờ, ngủ, hay chờ với hạn > 0
- [ ] Không hàm nào tự sinh luồng ngầm
- [ ] Không hàm nào gọi ngược vào core
- [ ] Gửi: chỉ trả ⏸ khi chưa nhận byte nào
- [ ] Gửi: trả ⊘ (không phải ⏸) khi vượt trần
- [ ] Nhận (kiểu gói): không bao giờ trả gói dang dở
- [ ] Nhận: nhánh ⤢ giữ nguyên gói, không vứt
- [ ] Phân biệt đúng ✖ (đóng sạch) với ⚠ (đứt/lỗi)
- [ ] Đóng hẳn giải phóng mọi thứ, và an toàn cả khi đang dở
- [ ] Gọi đóng hai lần không lỗi (dù core cam kết chỉ gọi một lần)
- [ ] Transport có phân biệt kiểu dữ liệu: dùng nhị phân, không dùng văn bản
- [ ] Không thêm byte nào của riêng mình vào dữ liệu

---

## 10. Bảng tra cứu nhanh: sự kiện nào đến từ đâu

| Ứng dụng nhận sự kiện | Sinh ra khi | Ở bước tick nào |
|---|---|---|
| CONNECT | Ở tick đầu tiên luôn | 0 |
| READY | Verdict = 0 quay lại | 2 |
| HANDSHAKE FAILED | Verdict là 1, 2, 3; hoặc verdict hỏng; hoặc handshake hết hạn | 2 hoặc 3 |
| MESSAGE | Frame DATA trả về khi READY | 2 |
| POLL | Frame POLL trả về khi READY (REPLY đã được gửi tự động) | 2 |
| REPLY | Frame REPLY trả về khi READY | 2 |
| DISCONNECT | Báo cáo transport ✖/⚠, hoặc heartbeat đã hết hạn | 3 hoặc 4 |

CONNECT luôn là sự kiện đầu tiên, kể cả trên UDP - nơi không có "kết nối" nào để báo. Ở đó nó được sinh ra một cách nhân tạo, để ứng dụng chỉ phải học một bộ từ vựng cho mọi transport.

Phía server có bộ sự kiện tương ứng cho từng peer: PEER CONNECTED, PEER READY, PEER HANDSHAKE FAILED, PEER DISCONNECTED. Sinh theo đúng quy tắc trên. Khác biệt duy nhất: mỗi sự kiện kèm danh tính peer.

---

## 11. Danh sách kiểm tra tối thiểu

Một bản triển khai được coi là đúng khi qua hết các phép thử này. Chúng không phụ thuộc ngôn ngữ và phải có ở mọi bản.

### Định dạng trên dây
- Mã hóa và giải mã lại từng loại frame, khớp từng byte với bảng trong §2
- Giải mã luồng byte khi tải mỗi lần một byte - vẫn phải xuất đúng frame
- Nhiều frame dính nhau trong một đợt dữ liệu - phải tách đúng
- Byte loại không hợp lệ: chí mạng trên luồng byte, chỉ vứt gói trên transport kiểu gói
- Gói thiếu byte và gói thừa byte - cả hai đều hỏng
- Dữ liệu đúng bằng trần thì qua; hơn trần một byte thì bị từ chối

### Handshake
- Greeting hợp lệ, schema giống hệt → chấp nhận, không đọc mục nào
- Schema khác nhau nhưng phần giao khớp → chấp nhận

Bốn nhánh của cổng ③, mỗi nhánh ít nhất một ca:

- ⓐ `h_n` hai bên bằng nhau → chấp nhận
- ⓑ `n` bằng nhau nhưng `h_n` khác → từ chối với lý do 2
- ⓒ client ít trường hơn, tiền tố khớp → chấp nhận, không gửi query
- ⓒ client ít trường hơn, tiền tố sai → từ chối với lý do 2, không gửi query
- ⓓ client nhiều trường hơn → server phải gửi verdict 4
- ⓓ với `n_s = 0` → chấp nhận, không gửi query

Hai ca hay bị bỏ quên nhất, và là lý do cả thiết kế này tồn tại:

- Client nối thêm trường ở cuối → vào ⓓ → query → chấp nhận
- Server xóa một trường ở cuối → cũng vào ⓓ → query → chấp nhận

Cả hai đều là nối thêm hoặc cắt bớt ở phần cuối. Đúng hai ca được ghim thành V-001/V-002 trong RFC-0003 §8.6. Cổng ③ giữ đúng hành vi này ở tầng handshake bằng chuỗi tiền tố. Từ chối ca này là sai, không phải "chặt chẽ hơn cho an toàn". Ca đổi tên trường thì ngược lại: bị từ chối có chủ đích (§3.3.1).

Vòng lặp query:

- Client trả đúng fingerprint tại mục được hỏi → verdict 0
- Client trả sai fingerprint → verdict 2
- Client trả thiếu mục, thừa mục, hoặc sai thứ tự → verdict 3
- Server gửi query lần 2 → client coi như HANDSHAKE FAILED
- Query yêu cầu một mã định danh không có trong greeting → client coi như HANDSHAKE FAILED
- Thời hạn handshake của client không được đặt lại khi vượt qua vòng query

Frame và giới hạn:

- Message chung, đảo ngược hai trường cùng kiểu → từ chối với lý do 2. Đây là ca mà fingerprint chỉ tính theo kiểu và thứ tự sẽ bỏ lọt; nó bị bắt vì tên trường nằm trong fingerprint (§3.3.1)
- Message chung, đổi tên một trường, kiểu và thứ tự không đổi → từ chối với lý do 2 (§3.3.1)
- Message chung, chỉ khác quy tắc đặt tên (`player_id` ↔ `PlayerID`) → chấp nhận, fingerprint không đổi (`fomoxa-fingerprint/2` §3.2)
- Sai phiên bản (greeting v1 gặp server v2) → từ chối với lý do 1
- Độ dài lệch 1 byte → từ chối với lý do 3
- Độ dài ≠ `16 + 14 × số message` → từ chối với lý do 3
- Số message khai báo rất lớn → từ chối, không tràn số học, không cấp phát lớn
- Verdict có giá trị ≥ 5 → client coi như HANDSHAKE FAILED
- Client: handshake hết hạn → thất bại
- Server: peer cứ trả lời probe mà không gửi greeting → không hết hạn

### Heartbeat
- Có lưu lượng liên tục → không gửi probe nào
- Im lặng hết cửa sổ → gửi đúng một probe, không phải mỗi tick một cái
- Có trả lời → về lại bình thường
- Không trả lời trong hạn → tuyên bố chết
- Frame bất kỳ, không chỉ REPLY, đều xóa được trạng thái probing
- Chạy hết một chu kỳ hết hạn mà không phải chờ thật - chứng minh thời gian là tham số

### Transport và dòng chảy
- Transport giả luôn trả ⏸ → frame nằm ở ô chờ, gửi lại đúng thứ tự, không trùng
- Gửi tiếp khi đang kẹt → báo "tắc nghẽn", không xếp hàng
- ⤢ → nới bộ đệm và lấy lại đúng gói đó, không mất
- ⊘ → lỗi lên ứng dụng, session vẫn sống, core không thử lại
- Dữ liệu đến dồn dập → dừng ở trần, phần còn lại lấy ở tick sau
- Handshake thất bại và transport chết → chỉ một sự kiện kết thúc

### Liên thông
- Trên hết: hai bản triển khai bằng hai ngôn ngữ khác nhau nói chuyện được với nhau, cả hai chiều
- Trên cả transport kiểu dòng byte lẫn transport kiểu gói
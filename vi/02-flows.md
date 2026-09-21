# Fomoxa: định dạng trên dây, các luồng, hướng dẫn triển khai

Tài liệu này mô tả hành vi bắt buộc của mọi bản triển khai Fomoxa, độc lập với ngôn ngữ lập trình và không tham chiếu implementation, API hay mã nguồn cụ thể nào.

Tài liệu này cùng `01_overview.md` đủ để dựng lại Fomoxa, tức một implementation mới hoặc một transport mới, trên bất kỳ ngôn ngữ nào.

- `01_overview.md`: mô hình, ranh giới lớp, phân chia trách nhiệm
- `02_flows.md` (tài liệu này): định dạng byte, từng luồng, hướng dẫn triển khai

Quy ước dùng trong tài liệu:

- **LE** = little-endian. Mọi số nhiều byte trong Fomoxa đều là little-endian.
- Sáu ký hiệu ✔ ⏸ ✖ ⚠ ⊘ ⤢ là sáu câu trả lời của transport, định nghĩa ở `01_overview.md` §2.
- Thuật ngữ giao thức giữ nguyên tiếng Anh và mỗi khái niệm chỉ có một tên: client, server, peer,
  transport, frame, handshake, greeting, verdict, query, schema, fingerprint, session, message,
  heartbeat, probe, tick. Tên frame/trạng thái/sự kiện viết hoa: DATA, POLL, REPLY, HANDSHAKE,
  HANDSHAKING, READY, CLOSED, CONNECT, DISCONNECT, MESSAGE, HANDSHAKE FAILED.
- Tên trạng thái và sự kiện viết hoa (READY, DISCONNECT) là khái niệm trừu tượng, không ràng buộc vào ngôn ngữ nào. Mỗi bản triển khai tự đặt tên theo quy ước của mình.

---

## 1. Toàn bộ vòng đời

```
   ┌────────────────────────────────────────────────────────────────┐
   │ GIAI ĐOẠN 1 - DỰNG ĐƯỜNG ỐNG (ngoài Fomoxa)                    │
   │ Kết nối TCP / Handshake TLS / Nâng cấp WebSocket               │
   └────────────────────────────────────────────────────────────────┘
                              ↓ đường ống đã thông
   ┌────────────────────────────────────────────────────────────────┐
   │ GIAI ĐOẠN 2 - HANDSHAKE FOMOXA §3                              │
   │ greeting (mang fingerprint schema) → verdict 1 byte            │
   │ (đôi khi chèn một vòng query ở giữa - §3.3.2)                  │
   │ trạng thái: HANDSHAKING                                        │
   │ ứng dụng chưa gửi gì                                           │
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
   │ ứng dụng tự dừng · bên kia đóng · bên kia im lặng quá lâu      │
   │ trạng thái: CLOSED                                             │
   └────────────────────────────────────────────────────────────────┘
```

Session có ba trạng thái, đi theo một chiều và không quay lại:

```
   HANDSHAKE ──chấp nhận──> READY ──────────> CLOSED
       │                                        ↑
       └──từ chối / hết hạn─────────────────────┘
```

---

## 2. Định dạng trên dây

Không được thay đổi nội dung mục này. Hai bản triển khai bất kỳ phải sinh ra và đọc đúng những byte này.

### 2.1 Phong bì frame

Mọi thứ đi trên dây đều là frame, và mọi frame bắt đầu bằng một byte loại:

```
   ┌──────────────┬───────────────────────────────┐
   │ Loại 1 byte  │ nội dung, tùy thuộc vào loại  │
   └──────────────┴───────────────────────────────┘

   Giá trị byte loại:
     0 DATA → phần thân là bản ghi dữ liệu (§2.2)
     1 POLL → không có nội dung. Một byte là toàn bộ frame.
     2 REPLY → không có nội dung. Một byte là toàn bộ frame.
     3 HANDSHAKE → nội dung gồm độ dài + các byte mờ (§2.3)

   Mọi byte loại khác 0-3 đều là vi phạm.
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

- Hai byte `0x46 0x4F` là dấu nhận dạng cố định. Sai một trong hai thì frame bị hỏng.
- Mã định danh message là *loại* message, do schema của ứng dụng chỉ định. Nó không tuần tự và không tăng dần. Cấm dùng nó để sắp xếp hay lọc trùng.
- Phần dữ liệu là byte mờ với transport và với lớp frame. Việc diễn giải nó thuộc lớp mã hóa dữ liệu, nằm ngoài tài liệu này.
- Độ dài dữ liệu tối đa: 16 MiB (16 777 216 byte). Dữ liệu vượt quá không được mã hóa và không được chấp nhận khi giải mã.

### 2.3 Frame HANDSHAKE

```
   ┌────┬──────────────┬─────────────────────────────────┐
   │ 03 │ độ dài       │ nội dung handshake              │
   │ 1B │ 4B u32 LE    │ = độ dài byte                   │
   └────┴──────────────┴─────────────────────────────────┘
    ^0   ^1-4           ^5 trở đi

   Tổng kích thước = 5 + độ dài
```

Ở tầng này nội dung là byte mờ; §3 giải thích nội dung đó. Độ dài tối đa là 1 MiB. Đây là chốt chặn cấp phát, không phải giới hạn ngữ nghĩa: nội dung handshake là dữ liệu điều khiển và không bao giờ cần lớn bằng dữ liệu ứng dụng.

### 2.4 Frame POLL và REPLY

Mỗi frame dài đúng một byte: `0x01` và `0x02`, không có nội dung, độ dài hay mã định danh. Chúng là dữ liệu điều khiển của transport, không bao giờ là message ứng dụng và không mang mã định danh message. Toàn bộ không gian mã định danh thuộc về ứng dụng.

### 2.5 Đặt frame trên luồng byte

Trên luồng byte, các frame nối tiếp nhau, không có dấu phân cách:

```
   ... │ frame │ frame │ frame │ ...
```

Frame tự mô tả: đọc byte loại là biết cần đọc thêm bao nhiêu byte. Bộ giải mã phải chạy tăng dần: nhận byte theo đợt tùy ý, giữ lại phần dở và chỉ trả frame khi đã đủ.

Mọi vi phạm frame trên luồng byte đều là lỗi chí mạng, vì không có cách đồng bộ lại. Mọi frame hợp lệ đều mở đầu bằng một byte loại hợp lệ; gặp giá trị khác nghĩa là không còn tin được phía kia để phân định frame. Khi đó phải đóng session.

Bộ giải mã đã báo lỗi thì bị nhiễm độc vĩnh viễn: mọi lệnh gọi sau đó đều trả lại đúng lỗi đó. Nhờ vậy không thể vô tình viết kiểu "cứ đọc tiếp rồi xem".

### 2.6 Đặt frame lên gói

Trên transport kiểu gói, một gói chứa đúng một frame.

- Gói hết trước khi frame nó khai báo kết thúc → hỏng.
- Gói còn dư byte sau khi đọc xong một frame → hỏng.

Vi phạm frame trên gói không phải lỗi chí mạng: bỏ đúng gói đó rồi đi tiếp. Các gói không dùng chung trạng thái phân tích, nên một gói hỏng không ảnh hưởng tới các gói sau như trên luồng byte.

Sự bất đối xứng này là có chủ ý, và cả hai phía phải cài đặt đúng.

### 2.7 Bảng trần

| Đại lượng | Trần | Ghi chú |
|---|---|---|
| Dữ liệu của message | 16 MiB | Chung cho tất cả các bản triển khai |
| Một frame DATA | 16 MiB + 11 byte | |
| Nội dung của frame HANDSHAKE | 1 MiB | Chốt chặn cấp phát |
| Số message được khai báo trong greeting | 1 000 000 | Chốt chặn số học, xem §3.3 |
| Số mục trong một query | 1 000 000 | Không vượt quá số message trong greeting, xem §3.3.2 |
| Số vòng query mỗi session | 1 | Handshake không chạy quá hai lượt |

---

## 3. Luồng: greeting và kiểm tra fingerprint

### 3.1 Toàn cảnh

```
   CLIENT                                           SERVER
     │                                                │
     │  transport vừa thông                           │ transport vừa thông
     │  tạo greeting ngay lập tức                     │
     │                                                │
     │ ── HANDSHAKE (greeting) ─────────────────────> │
     │                                                │ ① kiểm tra frame
     │                                                │ ② kiểm tra phiên bản
     │                                                │ ③ so sánh fingerprint
     │                                                │
     │                    quyết định được ngay chưa?  ┤
     │                                                │
     │ <── HANDSHAKE( verdict, 1 byte ) ───────────── │ Được - xong tại đây
     │                                                │
     │ <── HANDSHAKE( 4 = NEED MORE + danh sách ) ─── │ Chưa - §3.3.2
     │ ── HANDSHAKE (trả lời query) ────────────────> │
     │                                                │ ③′ điền phần còn thiếu
     │ <── HANDSHAKE( verdict, 1 byte ) ───────────── │
     │                                                │
     │ 0 → READY                                      │ 0 → peer READY
     │ ≠0 → HANDSHAKE FAILED, đóng                    │ ≠0 → gửi verdict rồi đóng
```

Bốn điều định hình luồng này:

1. Client luôn là bên chào, server luôn là bên phán quyết; client không bao giờ phán quyết. Luồng thường xong trong một vòng. Chỉ có đúng một trường hợp server phải hỏi thêm trước khi phán quyết (§3.3.2), và luồng không bao giờ quá hai vòng.
2. Server là bên duy nhất so sánh. Client gửi schema của mình và không bao giờ thấy schema của server. Kể cả khi bị hỏi thêm, client chỉ trả lời câu hỏi và không tự phán quyết.
3. Không có byte phân biệt greeting với phản hồi query. Cả hai đều nằm trong frame HANDSHAKE, và *vai trò cộng trạng thái* quyết định cách đọc. Server đọc payload đầu là greeting, payload thứ hai (nếu có) là phản hồi query. Theo chiều ngược lại, client đọc byte đầu: `0-3` là verdict, `4` là query.
4. Verdict là một byte, và chính giá trị byte đó là lý do từ chối; không có bảng tra thứ hai. Giá trị `4` không phải verdict mà là một yêu cầu, nên nó không kết thúc session.

### 3.2 Nội dung greeting

Đây là nội dung bên trong frame HANDSHAKE (§2.3):

```
   ┌───────────────────────────────────────────────────────┐
   │ phiên bản giao thức     4B u32 LE   luôn = 2          │
   │ fingerprint schema      8B u64 LE   của cả bộ message │
   │ số message              4B u32 LE                     │
   ├───────────────────────────────────────────────────────┤
   │ lặp lại "số message" lần, mỗi mục 14 byte:            │
   │   mã định danh message  4B u32 LE                     │
   │   số trường n           2B u16 LE                     │
   │   fingerprint h_n       8B u64 LE                     │
   └───────────────────────────────────────────────────────┘

   Độ dài phải chính xác bằng 16 + 14 × số lượng message.
   Lệch một byte = thất bại, không có ngoại lệ.
```

Ba đại lượng, mỗi đại lượng một vai trò:

- fingerprint schema: đại diện cho toàn bộ bộ message. Khớp nghĩa là hai bên dựng từ cùng một schema, và server chấp nhận ngay mà không đọc mục nào.
- `n`: số trường. Không có `n` thì không tính được `k = min(n_client, n_server)`, và không kiểm tra được điều kiện của RFC-0002 §9.1.
- `h_n`: fingerprint của cả message, tức fingerprint tiền tố tại `k = n`. Chuỗi fingerprint tiền tố vẫn tồn tại đầy đủ nhưng nằm trong bộ nhớ tĩnh của mỗi bên và không đi qua dây. Mỗi bên giữ chuỗi của mình, và mỗi message chỉ gửi một giá trị. §3.3.1 giải thích vì sao như vậy là đủ.

Cách tính fingerprint do đặc tả schema của Fomoxa quy định, nằm ngoài tài liệu này. Ứng dụng cung cấp cho mỗi message một bộ ba: mã định danh, `n`, chuỗi fingerprint tiền tố. Transport chỉ so sánh, không tự tính, và đòi hỏi cách tính đó có đúng hai tính chất:

```
1. fingerprint tiền tố thứ k chỉ phụ thuộc vào k trường đầu tiên,
   theo đúng thứ tự khai báo. Trường k+1 trở đi không được
   ảnh hưởng đến nó.

2. Hai message có k trường đầu tiên giống nhau phải có cùng một fingerprint
   tiền tố thứ k - trên tất cả các ngôn ngữ, tất cả cách triển khai.
```

### 3.3 Server quyết định như thế nào

Server xét ba cổng theo thứ tự và dừng ở cổng đầu tiên không đạt:

```
   nhận nội dung greeting
        │
        ▼
   ① FRAME có đúng không?
        │ ngắn hơn 16 byte?
        │ hay độ dài ≠ 16 + 14×số lượng?
        │ hoặc số lượng > 1 000 000?
        ├── sai ──> verdict = 3 (BROKEN) ──> đóng
        ▼ đúng
   ② PHIÊN BẢN có phải 2 không?
        ├── không ──> verdict = 1 (WRONG VERSION) ──> đóng
        ▼ đúng
   ③ SO SÁNH FINGERPRINT
        │
        ├─ fingerprint schema có khớp với server?
        │  └── KHỚP ───────────────────> verdict = 0, ACCEPTED
        │                                   (chưa đọc mục nào)
        │
        └─ KHÁC → duyệt từng mục trong greeting.
                  Với mỗi mã định danh mà server cũng có, đặt
                      n_c = số trường client khai báo
                      n_s = số trường server có

                  ⓐ h_n hai bên có bằng nhau không?
                        └── CÓ ──> message này giống hệt, tiếp tục

                  ⓑ n_c == n_s (nhưng h_n khác)?
                        └── CÓ ──> cùng độ dài nhưng nội dung khác nhau
                                   → không phải là tiền tố
                                   → verdict = 2 ──> đóng

                  ⓒ n_c < n_s?
                        └── CÓ ──> so sánh h_n của client với h_{n_c} lấy từ
                                   chuỗi tiền tố cục bộ của server
                                     ├── KHÁC ──> verdict = 2 ──> đóng
                                     └── KHỚP ──> khớp tiền tố, tiếp tục

                  ⓓ n_c > n_s
                        ├── n_s == 0 ──> tiền tố rỗng, luôn khớp, tiếp tục
                        └── n_s > 0 ──> chưa quyết định được.
                                        Ghi vào danh sách query: (định danh, n_s)

                  Sau khi duyệt hết:
                     danh sách query rỗng ──> verdict = 0, ACCEPTED
                     danh sách query có mục ──> gửi 4 = NEED MORE (§3.3.2)
```

Cổng ③ là phép thử tiền tố của RFC-0002 §9.1, viết bằng byte. `k = min(n_c, n_s)` là số trường của bên ngắn hơn. Hai fingerprint tiền tố thứ k bằng nhau nghĩa là `k` trường đầu của hai bên khớp cả kiểu lẫn thứ tự, đúng với định nghĩa "chuỗi ngắn hơn là tiền tố chính xác của chuỗi dài hơn".

Ba trong bốn nhánh được giải quyết tại chỗ, vì hoặc `h_n` client gửi đã đúng là `h_k` cần dùng, hoặc server tra được `h_k` của mình trong chuỗi cục bộ. Chỉ nhánh ⓓ cần một giá trị mà chỉ client sinh được.

Chốt chặn "số lượng > 1 000 000" ở cổng ① phải được kiểm tra trước phép nhân `14 × số lượng`, không phải sau. Trên số nguyên 32 bit, một số lượng khai báo gần 2³² làm phép nhân tràn, phép kiểm tra độ dài khi đó vẫn đạt, và một greeting độc hại lọt qua.

Lưu ý: `n` không tham gia phân tách frame. Mỗi mục dài đúng 14 byte bất kể `n` bằng bao nhiêu. `n` sai chỉ làm phép so sánh trượt hoặc sinh query rỗng, và không bao giờ dẫn tới cấp phát theo con số client khai báo.

Hai schema khác nhau vẫn có thể được chấp nhận. Điều kiện chấp nhận không đòi hai schema giống hệt nhau, mà đòi trong mọi message cả hai cùng biết, các trường cả hai cùng có phải khớp:

| Tình huống | Kết quả | Lý do |
|---|---|---|
| Fingerprint schema khớp | ✅ Chấp nhận | Cùng schema, không cần xét thêm |
| Client có một message mà server không biết | ✅ Chấp nhận | Server không bao giờ nhận message đó |
| Server có một message mà client không biết | ✅ Chấp nhận | Client không bao giờ gửi message đó |
| Message chung, một bên có trường bổ sung ở cuối | ✅ Chấp nhận | Khớp tiền tố, là khác biệt phiên bản hợp lệ (RFC-0002 §9.1) |
| Message chung, một bên có `n = 0` | ✅ Chấp nhận | Tiền tố rỗng là tiền tố của mọi chuỗi |
| Message chung, lệch tại một chỉ mục trong phần chung | ❌ Từ chối (2) | Trường tại chỉ mục đó đã đổi, là thay đổi giao thức (RFC-0002 §5.1) |
| Message chung, đổi tên một trường (không đổi kiểu và thứ tự) | ❌ Từ chối (2) | Đổi tên = xóa + thêm tại cùng chỉ mục; fingerprint có băm tên trường, §3.3.1 |

Như vậy hai bên dùng schema khác nhau vẫn liên thông được, miễn phần giao nhau khớp, cả ở cấp bộ message lẫn ở danh sách trường trong từng message chung.

Nhờ đó mỗi bên nâng cấp dần được, không phải dừng cả hệ thống và không phụ thuộc transport. RFC-0002 §9.1 cho phép hai bên đọc được dữ liệu của nhau khi chuỗi kiểu của bên ngắn hơn là tiền tố chính xác của bên dài hơn. RFC-0003 §8.6 ghim trường hợp đó thành hai vector V-001 và V-002 với yêu cầu "không được từ chối". Hai vector này ràng buộc bộ giải mã (RFC-0003 §1.3, phép thử tương thích). Ở tầng handshake, cổng ③ chủ động giữ trường hợp nối thêm ở cuối bằng chuỗi tiền tố, và đây là lý do duy nhất chuỗi tiền tố tồn tại (§3.3.1).

Liên thông chỉ bị chặn khi hai bên đặt hai trường khác nhau vào cùng một chỉ mục trong phần chung. Khi đó một vị trí mang hai nghĩa, và vì vị trí là định danh duy nhất trên dây nên không còn cách liên thông an toàn. Một message lệch là đủ để từ chối cả session, dù mọi message khác đều khớp. Không có chấp nhận một phần: session hoặc mở hoàn toàn, hoặc không mở.

Về bản chất, cổng ③ tin vào danh sách do client tự khai; client khai `số message = 0` sẽ đi qua cổng ③. Đây là phép kiểm tra tương thích giữa hai bên hợp tác, không phải cơ chế bảo mật, và không được dùng như cơ chế bảo mật.

### 3.3.1 Vì sao như vậy là đủ, và vì sao nhánh ⓓ phải hỏi thêm

RFC-0002 §9.1 hỏi *chuỗi ngắn hơn có phải tiền tố chính xác của chuỗi dài hơn không*, chứ không chỉ hỏi hai message giống hay khác. Một fingerprint duy nhất cho cả message không trả lời được câu hỏi đó:

```
     Item { id, name } → 0xA3F1…
     Item { id, name, hp } → 0x77B0… không liên quan gì đến dòng trên

   Sau khi băm, quan hệ tiền tố biến mất.
```

Vì vậy cần chuỗi fingerprint tiền tố. `h_k` cam kết đúng `k` trường đầu, và `k = min(n_c, n_s)` chính là điều kiện §9.1 dịch sang byte. Chỉ cần so một mốc, vì lệch ở bất kỳ chỉ mục nào nhỏ hơn `k` cũng làm `h_k` khác đi.

Tuy nhiên chuỗi không cần đi qua dây. Mỗi bên giữ chuỗi của mình trong bộ nhớ tĩnh, và ba trong bốn nhánh không cần thêm thông tin gì từ phía kia:

```
   ⓐ h_n bằng nhau → giống nhau. Không cần k.
   ⓑ n_c == n_s → k = n_c = n_s, và h_n khác → từ chối. Không cần gì thêm.
   ⓒ n_c < n_s → k = n_c, mà h^client_{n_c} là h_n client vừa gửi.
                 server tra h_{n_c} của nó trong chuỗi cục bộ. Đủ.
   ⓓ n_c > n_s → k = n_s. Cần h^client_{n_s} - chỉ client mới tạo ra được.
```

Nhánh ⓓ không tránh được do giới hạn về thông tin, không phải do lỗi thiết kế. Client phải trả lời tại một chỉ mục `n_s` mà lúc gửi greeting nó chưa biết. Muốn phủ mọi chỉ mục trong một vòng thì tốn `O(số trường)`, tức là gửi cả chuỗi. Không có cách nào ngắn hơn:

- hàm băm là một chiều: không suy ngược từ `h_{n_c}` ra `h_{n_s}`;
- đổi sang XOR hay tổng tích lũy cũng không giúp gì: muốn suy ngược vẫn cần đúng các trường chưa gửi;
- cây Merkle cho cam kết nhỏ, nhưng *bằng chứng mở* mới là thứ phải gửi, và phủ mọi `k` thì tốn hơn gửi cả chuỗi.

Chỉ còn hai lựa chọn: gửi cả chuỗi mỗi lần, hoặc gửi một giá trị rồi hỏi thêm khi rơi vào ⓓ. Greeting 14 byte mỗi message chọn cách thứ hai.

Nhánh ⓓ xảy ra khi server có ít trường hơn client trong một message đã thay đổi, không phụ thuộc bên nào nâng cấp trước. Có hai đường dẫn tới đó:

| | `n_c` vs `n_s` | Cần query? |
|---|---|---|
| Client nâng cấp trước, nối thêm một trường | `n_c > n_s` | ✅ |
| Server nâng cấp trước, xóa một trường | `n_c > n_s` | ✅ |
| Server nâng cấp trước, nối thêm một trường | `n_c < n_s` | ❌ nhánh ⓒ |
| Client nâng cấp trước, xóa một trường | `n_c < n_s` | ❌ nhánh ⓒ |

Yêu cầu này chặt hơn §9.1, và điều đó hợp lệ. Tiêu chí §9.1 chỉ xét kiểu và thứ tự, vì tên trường không có trên dây. Fingerprint của tầng schema thì băm cả tên trường (`fomoxa-fingerprint/2` §5; đây là cơ chế bắt trường hợp hai trường cùng kiểu bị tráo chỗ). Vì vậy cổng ③ từ chối một cặp peer chỉ khác nhau một lần đổi tên, dù byte hai bên sinh ra giống hệt.

Đây không phải lỗi. Đổi tên một trường tương đương xóa trường cũ và thêm một trường khác vào cùng chỉ mục, cùng loại với chèn giữa hoặc xóa giữa, mà §9.1 xếp vào thay đổi giao thức chứ không phải khác biệt phiên bản. §9.1 không yêu cầu chấp nhận trường hợp này; nó không nhìn thấy trường hợp này, vì nó đo độ lệch bằng kiểu, và hai trường `f32` đọc ra như nhau. Mục "Phạm vi của phần này" giao lại đúng điểm mù đó cho tầng trên: "Việc phát hiện và chặn các trường hợp đó thuộc về tầng schema cấp cao hơn, nơi có sẵn tên field và khai báo đầy đủ để so sánh." Cổng ③ thực hiện đúng việc được giao đó.

Cần giữ rõ ranh giới này, vì hai tầng tuân theo hai quy tắc khác nhau. Bộ giải mã không bao giờ được từ chối một luồng byte mà §9.1 coi là hợp lệ; RFC-0003 §1.3 xếp V-001/V-002 vào phép thử tương thích của bộ giải mã. Handshake được phép chặt hơn, vì nó thấy những thứ bộ giải mã không thấy: tên trường và khai báo đầy đủ.

Cái giá được ghi trong `SPEC-FINGERPRINT.md` §5 và là đánh đổi có chủ đích. Một lần đổi tên thuần túy cũng bị chặn, nhưng lỗi đó xảy ra ngay, dễ thấy và vô hại. Đổi lại, trường hợp tráo chỗ hai trường cùng kiểu không bao giờ lọt qua; nếu lọt, lỗi đó im lặng và làm hỏng dữ liệu vĩnh viễn. Từ `fomoxa-fingerprint/2`, tên trường được chuẩn hóa trước khi băm: bỏ gạch dưới, gạch ngang, khoảng trắng và chuyển chữ hoa thành chữ thường. Nhờ vậy khác biệt quy ước đặt tên giữa các ngôn ngữ (`id`/`ID`/`Id`, `player_id`/`PlayerID`) không làm đổi fingerprint; chỉ đổi tên thật (`x` → `position_x`) mới làm đổi.

### 3.3.2 Query: verdict 4 và vòng trả lời

Chỉ dùng cho nhánh ⓓ. Nội dung nằm trong frame HANDSHAKE, như mọi nội dung khác trong mục này.

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

Server chỉ được hỏi các mã định danh client đã khai trong greeting, và chỉ khi `1 ≤ n_s < n_c`. Query vi phạm quy tắc này là lỗi của server.

Client → server. Không có byte phân biệt: server chờ đúng một payload và đọc nó như phản hồi query (§3.1, mục 3).

```
   ┌───────────────────────────────────────────────────────────────┐
   │ số lượng mục            4B u32 LE                             │
   ├───────────────────────────────────────────────────────────────┤
   │ lặp lại, mỗi mục 12 byte:                                     │
   │   mã định danh message  4B u32 LE                             │
   │   tiền tố fingerprint   8B u64 LE   ở chỉ mục query chính xác │
   └───────────────────────────────────────────────────────────────┘

   Độ dài phải chính xác bằng 4 + 12 × số lượng mục.
   Phải trả lời đầy đủ và đúng thứ tự các mục server yêu cầu.
```

Server so sánh từng mục: fingerprint client gửi phải bằng `h_{n_s}` trong chuỗi cục bộ của server. Thiếu một mục → verdict 2. Khớp hết → verdict 0.

Ba quy tắc ngăn phát sinh thêm trạng thái:

```
1. Tối đa một query mỗi session. Sau khi nhận phản hồi, server phải phán quyết, không hỏi thêm.
2. Client nhận byte 4 lần thứ hai → coi như HANDSHAKE FAILED.
3. Hạn handshake của client (§3.5) tính cho toàn bộ quá trình, không được đặt lại sau mỗi vòng. Hai vòng dùng chung một hạn.
```

Trên transport kiểu gói, hai gói query thêm này có thể bị mất. Khi đó handshake hết hạn và thất bại: session không mở, ứng dụng thử lại. Đây không phải chế độ lỗi mới. Fomoxa không truyền lại (RFC-0001 §2), nên mọi handshake trên transport không tin cậy đều là best-effort. Đổi lại, greeting 14 byte mỗi message thường chỉ tốn hai gói, và không phải chia nhỏ một greeting lớn ở mỗi lần kết nối.

Để bỏ lượt khứ hồi thừa, bản triển khai có thể thử lạc quan: nếu handshake thất bại, lần kết nối sau gửi thẳng chuỗi tiền tố thay vì chờ query. Đây là lựa chọn triển khai, không phải yêu cầu của tài liệu này.

### 3.4 Client đọc verdict như thế nào

```
   nhận payload từ server
        │
        ▼
   byte đầu tiên == 4?
        │
        ├── đúng ──> QUERY (§3.3.2)
        │            │ đã nhận query một lần rồi?
        │            │ └── rồi ──────> coi như HANDSHAKE FAILED
        │            │ frame query hợp lệ?  mã định danh có trong greeting?
        │            │ └── không ──> coi như frame hỏng, thất bại
        │            └──> gửi phản hồi query, rồi tiếp tục chờ
        │                 (hạn không được đặt lại)
        │
        └── sai ──> VERDICT
                       │ độ dài đúng 1 byte?  và giá trị ≤ 3?
                       │ └── không ──> coi như HANDSHAKE FAILED
                       ▼ đúng
                    byte == 0?
                       ├── đúng ──> READY.  Từ đây ứng dụng có thể gửi.
                       └── sai ──> HANDSHAKE FAILED, lý do = chính giá trị byte
                                   → session đóng ngay, ứng dụng không nhận thêm sự kiện nào
```

Client không bao giờ tự đánh giá schema, kể cả khi bị hỏi. Nó chỉ tra chuỗi tiền tố cục bộ tại đúng chỉ mục được hỏi rồi gửi lại giá trị. Quyết định cuối cùng vẫn thuộc về server (§3.1, mục 2).

Bảng lý do (giá trị byte = lý do):

| Byte | Ý nghĩa | Nguyên nhân thực tế |
|---|---|---|
| 0 | Chấp nhận | |
| 1 | Phiên bản giao thức sai | Hai bên chạy hai phiên bản khác nhau của frame handshake: bố cục greeting đã đổi, ví dụ một bên còn gửi v1. Không liên quan tới schema, cũng không phải "hai phiên bản Fomoxa" |
| 2 | Schema xung đột | Trong một message cả hai bên đều biết, hai bên đặt hai trường khác nhau vào cùng một chỉ mục trong phần chung: chèn giữa, xóa giữa, đổi thứ tự, đổi kiểu hoặc đổi tên một trường (đổi tên = xóa + thêm tại cùng một vị trí, §3.3.1). Các trường nối thêm ở cuối không thuộc trường hợp này (§3.3.1) |
| 3 | Greeting hỏng | Lỗi ở phía gửi, hoặc một bên không phải Fomoxa đang kết nối vào cổng này |

Giá trị `4` không có trong bảng: nó không phải verdict mà là một yêu cầu (§3.3.2), và không bao giờ kết thúc session. Mọi giá trị từ `5` trở lên đều là hỏng.

### 3.5 Hạn handshake: hai vai tính khác nhau

Hai vai dùng hai cơ chế khác hẳn nhau:

```
   CLIENT                                           SERVER
   ──────                                           ──────
   HẠN CỨNG
   Đồng hồ chạy từ lúc session được tạo.            Không có hạn tuyệt đối.
   Quá hạn handshake mà vẫn chưa xong               Chỉ đếm "đã im lặng bao lâu".
   → thất bại, kết thúc.                            Peer vẫn trả lời probe?
                                                    → chưa hết hạn.
```

Trên thực tế, một client cứ trả lời probe mà không bao giờ gửi greeting hợp lệ sẽ giữ một chỗ trên server vô thời hạn. Hành vi này là có chủ ý. Đặc tả không đặt trần tuyệt đối nào, và đặt thêm trần là lệch khỏi thiết kế.

Hạn của client phủ toàn bộ handshake, kể cả vòng query (§3.3.2). Đồng hồ không đặt lại sau mỗi vòng; hai vòng dùng chung một hạn. Đặt lại đồng hồ sẽ cho phép một server độc hại kéo dài session bằng cách hỏi mãi.

### 3.6 Frame khác đến trong lúc handshake

Frame không phải HANDSHAKE có thể đến trước khi handshake xong, và không frame nào trong số đó bị coi là vi phạm:

| Frame đến | Client (HANDSHAKING) | Server (peer HANDSHAKING) |
|---|---|---|
| DATA | Bỏ, không giao cho ứng dụng | Bỏ, nhưng tính là hoạt động |
| POLL | Vẫn trả REPLY, không sinh sự kiện | Trả REPLY, tính là hoạt động |
| REPLY | Bỏ qua | Tính là hoạt động |
| HANDSHAKE thứ hai (sau khi đã có verdict) | Bỏ qua | Bỏ qua |

Dòng cuối khác với vòng query. Trong lúc HANDSHAKING, frame HANDSHAKE thứ hai là bình thường và có nghĩa: với client nó là query, với server nó là phản hồi query (§3.3.2). Chỉ frame HANDSHAKE đến sau khi đã có verdict mới bị bỏ. Mỗi session có tối đa một vòng query, nên frame HANDSHAKE thứ ba trong lúc handshake là hỏng.

Client vẫn phải trả REPLY khi chưa READY vì server được phép probe client trước khi trả lời một greeting hợp lệ. Nếu client im lặng với lý do chưa handshake xong, server kết luận client đã chết và cắt kết nối trong lúc client vẫn đang chờ server trả lời. Trả REPLY ở đây là điều kiện để handshake có thể hoàn tất.

---

## 4. Luồng: heartbeat chạy khi nào

### 4.1 Nguyên tắc: probe khi im lặng, không gửi định kỳ

Cách làm này khác với cách làm thông thường:

```
   ❌ THÔNG THƯỜNG                 ✅ FOMOXA
   cứ 5 giây gửi 1 probe           chỉ probe khi im lặng 5 giây
   dù đang truyền dữ liệu          đang chạy dữ liệu = đã biết peer còn sống
```

Peer đang gửi dữ liệu thì không nhận thêm probe. Trên hệ thống chạy 60 tick/giây, heartbeat gần như không sinh gói nào.

### 4.2 Máy trạng thái heartbeat

Mỗi bên đo chiều nghe của mình: client đo server im lặng bao lâu, server đo client im lặng bao lâu. Không bao giờ dùng chung một đồng hồ cho hai chiều.

```
        ┌────────────────────────────────────────┐
        │ NORMAL                                 │
        │ đếm: im lặng bao lâu rồi               │
        └────────────────────────────────────────┘
             │                            ▲
             │ im lặng ≥ cửa sổ im lặng   │ nhận được bất kỳ
             │ → gửi đúng một probe       │ frame hợp lệ nào
             ▼                            │ → xóa, quay lại
        ┌────────────────────────────────────────┐
        │ PROBING                                │
        │ đếm: probe bao lâu rồi mà chưa thấy gì │
        └────────────────────────────────────────┘
             │
             │ ≥ hạn phản hồi mà vẫn không có gì quay về
             ▼
        ┌────────────────────────────────────────┐
        │ PEER DEAD                              │
        └────────────────────────────────────────┘
```

Từ sơ đồ trên:
- Không bên nào cắt kết nối ngay khi hết cửa sổ im lặng; luôn có một probe trước đó. Thời gian phát hiện chết tệ nhất = `cửa sổ im lặng + hạn phản hồi`.
- Mọi frame hợp lệ đều được tính là dấu hiệu còn sống: DATA, POLL, REPLY, HANDSHAKE. Không nhất thiết phải là REPLY.

### 4.3 Heartbeat bật lúc nào: hai vai khác nhau

```
   CLIENT
   ──────
   HANDSHAKE: không có heartbeat.
                  handshake chỉ có một hạn cứng duy nhất (§3.5).
                  Trong giai đoạn này, server là bên giám sát
                  kết nối còn sống hay không.
                     ↓
   READY: heartbeat bắt đầu đúng tại thời điểm này.
                  cửa sổ im lặng = chu kỳ heartbeat
                  hạn phản hồi = hạn heartbeat


   SERVER (cho mỗi peer)
   ─────────────────────
   HANDSHAKE: heartbeat chạy ngay từ lúc peer xuất hiện.
                  cửa sổ im lặng = hạn handshake ← rộng hơn
                  hạn phản hồi = hạn heartbeat
                     ↓
   READY: chỉ đổi cửa sổ im lặng, không đặt lại gì khác.
                  cửa sổ im lặng = chu kỳ heartbeat ← hẹp hơn
                  hạn phản hồi = hạn heartbeat ← giữ nguyên
```

Handshake phía client không có heartbeat vì client vừa gửi greeting và chỉ chờ đúng một phản hồi. Không có gì cần làm mới định kỳ, và client đã có hạn cứng, nên một cơ chế thứ hai là thừa.

Server thì có heartbeat vì server có thể đang giữ nhiều peer nửa mở và cần phân biệt client chạy chậm với client đã biến mất. Chỉ probe mới phân biệt được hai trường hợp này.

Khi server chuyển sang READY, nó chỉ đổi cửa sổ im lặng, không đụng tới mốc hoạt động cuối và không hủy probe đang chờ. Một probe gửi ngay trước khi greeting đến vẫn còn hiệu lực và vẫn đang chờ trả lời.

### 4.4 Ba tham số thời gian

| Tham số | Mặc định | Ý nghĩa |
|---|---|---|
| Hạn handshake | 5 giây | Client: hạn cho toàn bộ quá trình handshake. Server: cửa sổ im lặng khi peer chưa handshake xong |
| Chu kỳ heartbeat | 5 giây | Im lặng bao lâu thì gửi một probe |
| Hạn heartbeat | 15 giây | Sau một probe, không có trả lời trong khoảng này thì coi như đã chết |

Thời gian phát hiện peer chết tệ nhất ở READY: 5 + 15 = 20 giây.

Đây là giá trị mặc định đề xuất và ứng dụng có thể cấu hình lại, nhưng không được đảo quan hệ giữa chúng. Hạn phản hồi phải đủ lớn so với độ trễ khứ hồi; nếu không, một peer bình thường vẫn bị tuyên bố là chết.

### 4.5 Toàn bộ chu trình probe

```
   A                                               B
   │                                               │
   │ ... hai bên đang trao đổi DATA bình thường    │
   │ mỗi frame nhận được sẽ đặt lại đồng hồ của A  │
   │                                               │
   │ ... B ngừng gửi ...                           │
   │                                               │
   │ Đồng hồ im lặng của A chạy: 1s 2s 3s 4s 5s    │
   │                                               │
   │ ── POLL ────────────────────────────────────> │ ① đúng một lần,
   │ A chuyển sang trạng thái PROBING              │    không lặp lại
   │                                               │
   │ <──────────────────────────── REPLY ───────── │ ② B còn sống
   │ A xóa trạng thái probing, trở lại bình thường │
   │ Ứng dụng của A nhận được sự kiện REPLY        │
   │                                               │
   │ ... hoặc B đã chết thật ...                   │
   │                                               │
   │ 15 giây trôi qua, không có gì quay lại        │
   │ A tuyên bố B đã chết → sự kiện DISCONNECT     │ ③
```

Ở ①, mỗi cửa sổ chỉ có đúng một probe, không phải mỗi tick một probe. Ở ②, bất kỳ frame nào từ B cũng đủ để xóa trạng thái probe; không nhất thiết là REPLY, B gửi DATA cũng được.

---

## 5. Luồng: gửi message

### 5.1 Đường đi đầy đủ

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
    │ dữ liệu được chép tại đây → ứng dụng có thể giải phóng nó ngay
    │ khi lệnh gửi trả về
    ▼
   ③ FRAME CŨ CÒN KẸT?
    │ có → trả về lỗi "tắc nghẽn", không xếp hàng
    ▼ không
   ④ GIAO CHO TRANSPORT
    │
    ├── ✔ xong, không giữ lại gì
    ├── ⏸ chép vào ô chờ, trả về ứng dụng: thành công
    ├── ⊘ trả về lỗi "quá lớn" cho ứng dụng, session vẫn sống
    └── ✖⚠ đánh dấu transport đã chết
```

### 5.2 Vì sao cấm gửi dữ liệu trước khi READY

Trước khi có verdict, chưa có gì xác nhận hai bên đọc cùng một bộ byte. Một message gửi sớm có thể tới một server ngay sau đó từ chối schema; server đó diễn giải message theo schema của mình và đọc ra giá trị khác với giá trị client gửi. Phép kiểm tra trạng thái ở bước ① là chỗ duy nhất chặn được trường hợp này.

### 5.3 Frame bị kẹt được đẩy ra khi nào

```
   tick N   gửi frame ────────────────────────> transport: ⏸ → vào ô chờ
              ứng dụng vẫn chạy bình thường

   tick N+1 việc đầu tiên: đẩy nốt ô chờ ──────> transport: ⏸ → vẫn kẹt
              (nếu ứng dụng tiếp tục gửi trong tick này → lỗi "tắc nghẽn")

   tick N+2 việc đầu tiên: đẩy nốt ô chờ ──────> transport: ✔ → qua
              từ đây gửi lại bình thường
```

Xử lý việc cũ trước là bắt buộc, không phải tối ưu. Nếu một frame mới ra dây trước phần còn lại của frame cũ, bộ giải mã phía kia đọc hai frame dính vào nhau và hỏng vĩnh viễn (§2.5).

---

## 6. Luồng: nhận message

### 6.1 Đường đi đầy đủ

```
   TRANSPORT
    │ đưa lên một frame hoàn chỉnh
    ▼
   ① MỞ GÓI (§2)
    │ đọc byte đầu tiên → loại frame
    │ với DATA, kiểm tra thêm 2 byte nhận dạng 'F' 'O'
    │ sai → frame hỏng (xử lý theo §2.5 hoặc §2.6 tùy loại transport)
    ▼
   ② ĐƯA VÀO MÁY TRẠNG THÁI SESSION
    │
    ├─ HANDSHAKE → xử lý verdict (§3.4) hoặc greeting (§3.3) tùy vai trò
    │
    ├─ POLL → trả REPLY ngay lập tức + (nếu READY) sinh sự kiện POLL
    │
    ├─ REPLY → (nếu READY) xóa trạng thái probing + sinh sự kiện REPLY
    │
    └─ DATA → chưa READY? bỏ, không giao cho ứng dụng
              READY?      đặt lại đồng hồ im lặng
                          sinh sự kiện MESSAGE
    ▼
   ③ ĐƯA VÀO DANH SÁCH SỰ KIỆN
    ▼
   ỨNG DỤNG đọc danh sách sau khi tick trả về
```

Ở ②, REPLY cho POLL được gửi ngay tại chỗ, không chờ ứng dụng. Ứng dụng có thể không bao giờ đọc sự kiện POLL mà heartbeat vẫn chạy đúng.

### 6.2 Vòng đời dữ liệu sự kiện

```
   tick N
     ├── nhận frame, sự kiện tham chiếu vào bộ đệm bên trong
     ├── ứng dụng đọc sự kiện, sử dụng dữ liệu ← hợp lệ tại đây
     └── tick kết thúc

   tick N+1
     └── bộ đệm bị ghi đè ← tham chiếu cũ giờ trỏ vào rác
```

Quy tắc: *dữ liệu trong một sự kiện chỉ còn hợp lệ tới tick kế tiếp của cùng session*. Dữ liệu cần giữ lâu hơn phải được chép ra. Quy tắc này áp dụng cho các ngôn ngữ trao dữ liệu bằng tham chiếu không sở hữu.

Bản triển khai chép dữ liệu sang ứng dụng, hoặc trao một đối tượng có sở hữu, thì không bị ràng buộc này, nhưng phải ghi rõ điều đó trong tài liệu của mình, vì lập trình viên chuyển từ bản này sang bản khác thường mang theo giả định cũ.

---

## 7. Luồng: kết thúc session

Session có thể kết thúc theo ba đường, khác nhau ở chỗ bên nào phát hiện trước:

```
   ① ỨNG DỤNG TẮT
      ứng dụng gọi ngắt → core tự đánh dấu đã đóng → yêu cầu transport đóng mềm
      → session CLOSED. Không sinh thêm sự kiện nào.

   ② PEER ĐÓNG HOẶC LỖI
      transport trả về ✖ hoặc ⚠ khi core hỏi
      → core đánh dấu transport đã chết
      → nếu session chưa CLOSED: sinh DISCONNECT

   ③ PEER IM LẶNG
      transport hoàn toàn bình thường, không báo gì
      → đồng hồ heartbeat hết hạn trong bước tick
      → sinh DISCONNECT
```

Mỗi session có đúng một sự kiện kết thúc. Nếu handshake hỏng và đã sinh HANDSHAKE FAILED thì không có DISCONNECT theo sau, kể cả khi transport chết ngay sau đó. Cần một cờ để bảo đảm điều này. Ứng dụng thường dọn dẹp trong trình xử lý sự kiện kết thúc, và dọn dẹp hai lần là lỗi.

---

## 8. Hướng dẫn triển khai: phía core

Hình dạng của một tick. Thứ tự này là bắt buộc, không phải khuyến nghị.

```
   tick(thời_điểm_hiện_tại):

       ── 0. Sự kiện CONNECT, chỉ một lần trong vòng đời session ──
       nếu kết nối chưa được báo:
           đẩy sự kiện CONNECT
           đánh dấu đã báo

       ── 1. đẩy frame bị kẹt ra ── phải chạy trước mọi việc khác
       trong khi ô chờ còn dữ liệu:
           kết quả = transport.gửi(phần còn lại)
           ✔ → dọn ô chờ, thoát vòng lặp
           ⏸ → thoát vòng lặp, để tick sau
           lỗi → transport_chết = đúng, thoát vòng lặp

       ── 2. đọc cạn dữ liệu đến, trong hạn mức ──
       nếu transport chưa chết:
           ngân_sách = GIỚI_HẠN
           lặp trong khi còn ngân sách:
               kết quả = transport.nhận(bộ_đệm)
               ⏸ → thoát vòng lặp
               ⤢ → nới bộ đệm, không trừ ngân sách, thử lại
               ✖⚠ → transport_chết = đúng, thoát vòng lặp
               ✔ → mở gói thành frame
                     đưa vào máy trạng thái session
                     áp dụng kết quả (xem bên dưới)
                     trừ một đơn vị ngân sách

       ── 3. chạy đồng hồ giao thức ──
       nếu transport chưa chết:
           kết quả = session.tick(thời_điểm)   hạn handshake · probe · tuyên bố chết
           áp dụng kết quả

       ── 4. transport đã chết nhưng session chưa đóng ──
       nếu transport đã chết và session chưa CLOSED:
           kết quả = session.báo_transport_đóng()
           áp dụng kết quả

       ── 5. trả danh sách sự kiện cho ứng dụng ──
       trả về danh sách
```

`áp dụng kết quả`: máy trạng thái session trả về tối đa một frame để gửi và tối đa một sự kiện:

```
   áp_dụng(kết_quả):
       nếu có frame cần gửi:
           nếu ô chờ còn dữ liệu:
               xếp frame này sau nó ← không ghi đè
               nếu hàng chờ vượt trần → transport_chết = đúng
           ngược lại:
               ghi frame đó vào transport
               ⏸ → cất vào ô chờ
               ✖⚠ → transport_chết = đúng

       nếu có sự kiện:
           đẩy vào danh sách
           nếu là HANDSHAKE FAILED:
               transport_chết = đúng
               yêu cầu transport đóng mềm
```

"Tối đa một frame và tối đa một sự kiện" phản ánh đúng những gì giao thức sinh ra. Trường hợp phức tạp nhất là trả lời rồi kết thúc: server gửi verdict từ chối rồi đóng session. Vòng query (§3.3.2) không phá bất biến này, vì cả query lẫn phản hồi query đều là frame không kèm sự kiện. Ứng dụng không thấy gì cho tới verdict cuối cùng, nên với ứng dụng, handshake hai vòng và handshake một vòng là như nhau.

Ba điểm trong khối trên cần nêu rõ, vì thiếu điểm nào cũng gây lỗi im lặng.

Cấm ghi đè ô chờ. Các frame ở đây (POLL, REPLY, verdict handshake) do core tự sinh và không đi qua lệnh gửi của ứng dụng (khác với kịch bản ở `01_overview.md` §5), nên không có ai để báo "tắc nghẽn". Ghi đè ô chờ làm mất verdict từ chối: peer không biết vì sao bị từ chối và phải chờ tới hết hạn.

Hàng chờ phải có trần, và trần đó phải thấp. Bản thân giao thức đã giới hạn số frame điều khiển tồn tại cùng lúc: đúng một probe mỗi cửa sổ im lặng (§4.5), một REPLY cho mỗi POLL nhận được, tối đa một vòng query mỗi session (§3.3.2). Vượt quá một con số nhỏ nghĩa là một giả định đã sai, không phải tải cao bình thường. Con số cụ thể tùy bản triển khai, nhưng bắt buộc phải có.

Chạm trần thì kết thúc session, không ném lỗi. Bất biến B6 yêu cầu đúng một sự kiện kết thúc mỗi session, và DISCONNECT đáp ứng yêu cầu đó. Ném ngoại lệ ra khỏi tick thì không: nó phá cam kết "tick trả về một danh sách sự kiện", và một server đang duyệt nhiều peer sẽ bỏ sót các peer còn lại trong cùng tick đó.

Bốn quy tắc cố định của core:

1. Không bao giờ chờ. Core hỏi transport, nhận câu trả lời rồi đi tiếp. Không vòng lặp nào ở đây được quay lại chỉ để thử lại ngay; thử lại là việc của tick sau.
2. Máy trạng thái session không được biết transport tồn tại. Nó nhận frame, trả về frame và sự kiện, không giữ kết nối, không xử lý lỗi hệ điều hành và không tự đọc đồng hồ.
3. Mọi mốc thời gian được truyền từ ngoài vào. Đây là điều kiện để chạy hết một chu kỳ hết hạn trong kiểm thử mà không phải chờ thật, và phải đúng ở mọi tầng.
4. Chỉ một sự kiện kết thúc. Cờ "đã báo kết thúc" phải được kiểm tra trên mọi đường dẫn tới sự kiện kết thúc.

### Đồng hồ

Thời gian phải lấy từ đồng hồ đơn điệu: chỉ tăng, không bị chỉnh. Cấm dùng đồng hồ treo tường, vì chỉ một lần đồng bộ giờ lùi lại là đủ để làm hết hạn một handshake hoặc giết một peer đang sống.

---

## 9. Hướng dẫn triển khai: phía transport

### 9.1 Trạng thái tối thiểu cần giữ

```
   trạng thái transport:
       kết_nối           đối tượng kết nối của thư viện đang dùng
       hàng_đợi_đến      dữ liệu đã đọc nhưng chưa giao cho core
       hàng_đợi_đi       phần chưa đẩy ra được (nếu có)
       đã_đóng           để mọi lệnh gọi sau đó đều trả ✖ nhất quán
```

### 9.2 Chức năng GỬI

```
   gửi(byte, độ_dài):
       nếu đã_đóng: trả ✖
       nếu độ_dài > trần_của_mình: trả ⊘ ← không phải ⏸
       thử đẩy vào kết nối nền, không chờ
           đẩy được hết → trả ✔
           không đẩy được → trả ⏸ ← chưa nhận byte nào
           peer đóng sạch → trả ✖
           lỗi → trả ⚠
```

Khi trả ⏸, transport không được nhận byte nào. Nếu nhận một nửa rồi trả ⏸, core sẽ gửi lại từ đầu và phía kia nhận nửa đầu hai lần.

Nếu thư viện nền có thể nhận một phần (kết nối thô), có hai cách: khai báo transport là kiểu dòng byte và để core xử lý, hoặc tự đệm phần còn lại vào `hàng_đợi_đi` rồi trả ✔. Cấm trả ⏸ khi đã nhận một phần.

### 9.3 Chức năng NHẬN

```
   nhận(bộ_đệm, sức_chứa):
       nếu đã_đóng và hàng_đợi_đến rỗng: trả ✖

       nếu hàng_đợi_đến rỗng:
           đọc từ kết nối nền, không chờ
               không có gì → trả ⏸
               peer đóng sạch → đã_đóng = đúng, trả ✖
               lỗi → đã_đóng = đúng, trả ⚠
               có dữ liệu → cất vào hàng_đợi_đến

       (chỉ với transport kiểu gói)
       nếu gói đầu hàng chưa đến đủ: trả ⏸

       nếu gói đầu hàng > sức_chứa:
           trả ⤢ kèm số byte cần
           ← gói vẫn nằm trong hàng, không được bỏ

       chép gói đầu hàng vào bộ_đệm
       gỡ nó khỏi hàng đợi
       trả ✔
```

Ở nhánh ⤢, cấm làm mất gói. Core sẽ nới bộ đệm và hỏi lại ngay; bỏ gói đi thì dữ liệu mất vĩnh viễn mà không bên nào biết.

### 9.4 Chức năng ĐÓNG MỀM và ĐÓNG HẲN

```
   đóng_mềm():
       nếu đã_đóng: không làm gì
       gửi tín hiệu đóng của giao thức nền
       KHÔNG chờ phía kia phản hồi
       KHÔNG giải phóng gì - vẫn có thể còn dữ liệu đến cần đọc

   đóng_hẳn():
       đóng kết nối nền
       giải phóng hàng đợi, bộ đệm và mọi tài nguyên đang giữ
       ← được gọi đúng một lần. Sau lệnh gọi này không chức năng nào khác được gọi.
```

Hai chức năng khác nhau: đóng mềm là *thông báo lịch sự cho peer*, đóng hẳn là *dọn dẹp tài nguyên*. Giữa hai lệnh gọi này, core vẫn có thể hỏi tiếp dữ liệu còn trên đường truyền.

### 9.5 Tự kiểm trước khi kết thúc review

- [ ] Không hàm nào chờ, ngủ, hay chờ với hạn > 0
- [ ] Không hàm nào tự sinh luồng ngầm
- [ ] Không hàm nào gọi ngược vào core
- [ ] Gửi: chỉ trả ⏸ khi chưa nhận byte nào
- [ ] Gửi: trả ⊘ (không phải ⏸) khi vượt trần
- [ ] Nhận (kiểu gói): không bao giờ trả gói dang dở
- [ ] Nhận: nhánh ⤢ giữ nguyên gói, không bỏ
- [ ] Phân biệt đúng ✖ (đóng sạch) với ⚠ (đứt/lỗi)
- [ ] Đóng hẳn giải phóng mọi thứ, và an toàn cả khi trạng thái đang dở
- [ ] Gọi đóng hai lần không lỗi (dù core cam kết chỉ gọi một lần)
- [ ] Transport có phân biệt kiểu dữ liệu: dùng nhị phân, không dùng văn bản
- [ ] Không thêm byte nào của riêng mình vào dữ liệu

---

## 10. Bảng tra cứu nhanh: sự kiện nào đến từ đâu

| Ứng dụng nhận sự kiện | Sinh ra khi | Ở bước tick nào |
|---|---|---|
| CONNECT | Luôn ở tick đầu tiên | 0 |
| READY | Nhận verdict = 0 | 2 |
| HANDSHAKE FAILED | Verdict là 1, 2, 3; hoặc verdict hỏng; hoặc handshake hết hạn | 2 hoặc 3 |
| MESSAGE | Frame DATA đến khi READY | 2 |
| POLL | Frame POLL đến khi READY (REPLY đã được gửi tự động) | 2 |
| REPLY | Frame REPLY đến khi READY | 2 |
| DISCONNECT | Transport báo ✖/⚠, hoặc heartbeat hết hạn | 3 hoặc 4 |

CONNECT luôn là sự kiện đầu tiên, kể cả trên UDP, nơi không có kết nối nào để báo. Trên UDP, sự kiện này được sinh nhân tạo để ứng dụng chỉ phải dùng một bộ từ vựng cho mọi transport.

Phía server có bộ sự kiện tương ứng cho từng peer: PEER CONNECTED, PEER READY, PEER HANDSHAKE FAILED, PEER DISCONNECTED. Chúng được sinh theo đúng các quy tắc trên, với khác biệt duy nhất là mỗi sự kiện kèm danh tính peer.

---

## 11. Danh sách kiểm tra tối thiểu

Một bản triển khai được coi là đúng khi qua hết các phép thử sau. Chúng không phụ thuộc ngôn ngữ và phải có ở mọi bản triển khai.

### Định dạng trên dây
- Mã hóa và giải mã lại từng loại frame, khớp từng byte với bảng trong §2
- Giải mã luồng byte khi nạp mỗi lần một byte; vẫn phải ra đúng các frame
- Nhiều frame dính nhau trong một đợt dữ liệu phải được tách đúng
- Byte loại không hợp lệ: chí mạng trên luồng byte, chỉ bỏ gói trên transport kiểu gói
- Gói thiếu byte và gói thừa byte đều là hỏng
- Dữ liệu đúng bằng trần thì qua; vượt trần một byte thì bị từ chối

### Handshake
- Greeting hợp lệ, schema giống hệt → chấp nhận, không đọc mục nào
- Schema khác nhau nhưng phần giao khớp → chấp nhận

Bốn nhánh của cổng ③, mỗi nhánh ít nhất một trường hợp:

- ⓐ `h_n` hai bên bằng nhau → chấp nhận
- ⓑ `n` bằng nhau nhưng `h_n` khác → từ chối với lý do 2
- ⓒ client ít trường hơn, tiền tố khớp → chấp nhận, không gửi query
- ⓒ client ít trường hơn, tiền tố sai → từ chối với lý do 2, không gửi query
- ⓓ client nhiều trường hơn → server phải gửi verdict 4
- ⓓ với `n_s = 0` → chấp nhận, không gửi query

Hai trường hợp hay bị bỏ sót nhất, và cũng là lý do của thiết kế này:

- Client nối thêm trường ở cuối → vào ⓓ → query → chấp nhận
- Server xóa một trường ở cuối → cũng vào ⓓ → query → chấp nhận

Cả hai đều là nối thêm hoặc cắt bớt ở phần cuối, đúng hai trường hợp được ghim thành V-001/V-002 trong RFC-0003 §8.6. Cổng ③ giữ hành vi này ở tầng handshake bằng chuỗi tiền tố. Từ chối các trường hợp này là sai, không phải chặt chẽ thêm cho an toàn. Trường hợp đổi tên trường thì ngược lại: bị từ chối có chủ đích (§3.3.1).

Vòng query:

- Client trả đúng fingerprint tại mục được hỏi → verdict 0
- Client trả sai fingerprint → verdict 2
- Client trả thiếu mục, thừa mục, hoặc sai thứ tự → verdict 3
- Server gửi query lần 2 → client coi như HANDSHAKE FAILED
- Query yêu cầu một mã định danh không có trong greeting → client coi như HANDSHAKE FAILED
- Hạn handshake của client không được đặt lại qua vòng query

Frame và giới hạn:

- Message chung, tráo chỗ hai trường cùng kiểu → từ chối với lý do 2. Fingerprint chỉ tính theo kiểu và thứ tự sẽ bỏ lọt trường hợp này; nó bị bắt vì tên trường nằm trong fingerprint (§3.3.1)
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
- Im lặng hết cửa sổ → gửi đúng một probe, không phải mỗi tick một probe
- Có trả lời → trở lại bình thường
- Không có trả lời trong hạn → tuyên bố chết
- Frame bất kỳ, không chỉ REPLY, đều xóa được trạng thái probing
- Chạy hết một chu kỳ hết hạn mà không phải chờ thật, chứng minh thời gian là tham số

### Transport và dòng chảy
- Transport giả luôn trả ⏸ → frame nằm ở ô chờ, được gửi lại đúng thứ tự, không trùng
- Gửi tiếp khi đang kẹt → báo "tắc nghẽn", không xếp hàng
- ⤢ → nới bộ đệm và lấy lại đúng gói đó, không mất
- ⊘ → lỗi lên ứng dụng, session vẫn sống, core không thử lại
- Dữ liệu đến dồn dập → dừng ở trần, phần còn lại lấy ở tick sau
- Handshake thất bại và transport chết → chỉ một sự kiện kết thúc
- Hàng đợi nhận của transport kiểu gói đầy → gói cũ nhất bị bỏ, gói mới nhất được giữ
- Transport giả luôn trả ⏸, cộng một peer gửi POLL mỗi tick → hàng chờ dừng ở trần, đúng một sự kiện kết thúc, không ném lỗi ra khỏi tick
- Nhiều địa chỉ nguồn lạ gửi vào một điểm cuối UDP → bảng peer dừng ở trần, session đang chạy không bị ảnh hưởng

### Liên thông
- Tiêu chí chính: hai bản triển khai bằng hai ngôn ngữ khác nhau liên thông được với nhau theo cả hai chiều
- Trên cả transport kiểu dòng byte lẫn transport kiểu gói

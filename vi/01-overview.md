# Fomoxa Transport: mô hình và trách nhiệm

Tài liệu này mô tả mô hình transport của Fomoxa ở mức khái niệm, độc lập với mọi ngôn ngữ lập trình và không tham chiếu implementation, API hay mã nguồn cụ thể nào.

Tài liệu này cùng `02_flows.md` đủ để dựng lại Fomoxa, tức một implementation mới hoặc một transport mới, trên bất kỳ ngôn ngữ nào mà không cần đọc mã nguồn của bản triển khai đang có.

| Tài liệu | Nội dung | Đọc khi |
|---|---|---|
| `01_overview.md` (này) | Mô hình, ranh giới tầng, phân chia trách nhiệm | Muốn hiểu thiết kế |
| `02_flows.md` | Định dạng trên dây, từng luồng chi tiết, hướng dẫn triển khai | Bắt tay vào viết |

Mỗi ngôn ngữ có thể có tài liệu ràng buộc API riêng. Những tài liệu đó mô tả *cách gọi*, không mô tả *hành vi*. Hành vi được quy định ở hai tài liệu này và phải giống nhau ở mọi bản triển khai.

---

## 1. Ba tầng

```
  ỨNG DỤNG          gửi/nhận message có ý nghĩa với nghiệp vụ
      |
  ─────────────────────────────────────────────
      |
  CORE              handshake, heartbeat, phát hiện peer chết,
                    đóng gói/mở gói frame, quản lý sự kiện
      |
  ─────────────────────────────────────────────
      |
  TRANSPORT         chỉ vận chuyển byte từ máy này sang máy kia
                    TCP / UDP / WebSocket / TLS / QUIC ...
```

Ranh giới cơ bản nhất: core không biết dữ liệu đi bằng đường nào, transport không biết dữ liệu chứa gì. Transport không thấy handshake, heartbeat hay trạng thái peer, và không phân biệt được greeting với dữ liệu vị trí nhân vật. Với transport, mọi dữ liệu chỉ là một khối byte cần đẩy sang phía kia.

Theo chiều ngược lại, core không biết và không được phép biết mình chạy trên TCP hay WebSocket.

---

## 2. Bốn chức năng transport phải cung cấp

Transport cung cấp đúng bốn chức năng:

| Chức năng | Nghĩa | Ai gọi |
|-----------|-------|--------|
| Gửi | Nhận một cục byte, đưa sang phía kia | Core, khi có gì cần gửi |
| Nhận | Trả về dữ liệu đã tới từ phía kia, nếu có | Core, mỗi tick |
| Đóng mềm | Báo phía kia biết ta sắp ngắt | Core, khi ứng dụng yêu cầu ngắt |
| Đóng hẳn | Giải phóng mọi tài nguyên | Core, khi hủy session |

Không có chức năng thứ năm, nghĩa là không có chức năng kết nối lại, chờ tới khi gửi được, hay báo còn bao nhiêu chỗ trống.

### Bốn câu trả lời chung

Với cả lệnh gửi lẫn lệnh nhận, transport trả lời bằng cùng một bộ từ vựng:

```
  ✔  Tôi làm được rồi      → core yên tâm đi tiếp
  ⏸  Bây giờ chưa được     → core giữ lại, tick sau hỏi tiếp
  ✖  Tôi đóng rồi          → core kết thúc session
  ⚠  Tôi bị lỗi            → core kết thúc session
```

### Hai câu trả lời đặc biệt, mỗi chiều một câu

Bốn câu trên không bao gồm hai tình huống không phải thành công, không phải tạm chưa được và cũng không phải session kết thúc:

```
  Chiều GỬI
  ⊘  Cục này quá lớn so với khả năng của tôi
     → không giết session. Core báo lỗi lên ứng dụng, kết nối vẫn sống.
     → Core không thử lại. (§7)

  Chiều NHẬN
  ⤢  Có dữ liệu, nhưng chỗ anh đưa không đủ. Tôi cần bằng này byte.
     → Core nới chỗ rồi hỏi lại ngay. Dữ liệu không mất. (§7)
```

⊘ phải tách khỏi ⏸ vì hai tín hiệu mang nghĩa khác nhau. ⏸ nghĩa là thử lại sau thì sẽ được. Nếu transport trả ⏸ cho một payload quá lớn, core giữ payload lại và thử lại mỗi tick, nhưng payload vẫn quá lớn ở mọi tick sau, nên vòng lặp không kết thúc và kênh gửi tắc vĩnh viễn. ⊘ nghĩa là ngược lại: *không thử lại; payload này không bao giờ gửi được*.

⤢ cũng thuộc nhóm thử lại sẽ được, nhưng kèm một con số: core phải nới bộ đệm trước khi hỏi lại. Vì vậy nó cũng không gộp được vào ⏸.

### Điều cấm số một

Transport không có câu trả lời nào mang nghĩa "để tôi chờ", và đây là điều cấm số một của thiết kế. Fomoxa chạy trong một vòng lặp có định thời, nên transport chờ thì cả vòng lặp dừng. Transport chỉ báo trạng thái *tại thời điểm được hỏi* rồi trả lại quyền điều khiển; việc thử lại thuộc core.

---

## 3. Hai loại transport

Có đúng hai loại, và transport phải khai báo mình thuộc loại nào.

### Kiểu dòng byte (stream)

Byte chảy liên tục, không có ranh giới. Gửi 3 cục 100 byte thì phía kia có thể nhận 1 cục 300 byte hoặc 7 cục nhỏ, và không có thông tin nào cho biết một đơn vị kết thúc ở đâu.

→ TCP, TLS, QUIC stream

### Kiểu gói (packet)

Transport tự giữ ranh giới. Gửi 3 cục thì phía kia nhận đúng 3 cục, đúng ranh giới và không dính vào nhau.

→ UDP, WebSocket, QUIC datagram

### Vì sao phải phân biệt

Dữ liệu từ transport kiểu dòng byte cần được ráp lại, và core làm việc này. Khi transport khai báo kiểu dòng byte, core tự chèn một lớp ráp frame vào giữa, nên người viết transport không cần biết Fomoxa đóng frame thế nào.

```
  Transport kiểu gói             Transport kiểu dòng byte
  ──────────────────             ────────────────────────
        CORE                            CORE
          |                               |
          |                       [ lớp ráp frame ]   ← core tự lo
          |                               |
      WebSocket                          TCP
```

Nhờ vậy người viết transport WebSocket hay TLS không phải đọc định dạng trên dây.

Lớp ráp frame phải nằm giữa core và transport, không nằm trong bên nào. Nếu đặt trong transport, transport phải hiểu định dạng frame, tức là sai tầng. Nếu đặt trong core, core phải kiểm tra loại transport ở mỗi lệnh gọi, cũng là sai tầng. Đặt ở giữa thì việc phân nhánh chỉ xảy ra một lần lúc khởi tạo, và cả core lẫn transport đều không chứa logic của nhau.

### Nghĩa vụ khi trả dữ liệu về: hai loại khác nhau hoàn toàn

Nghĩa vụ trả dữ liệu của hai loại transport dễ bị nhầm lẫn:

| | Kiểu dòng byte | Kiểu gói |
|---|---|---|
| Mỗi lần trả về | Bao nhiêu byte cũng được | Đúng một gói trọn vẹn |
| Trả 1 byte lẻ? | Hợp lệ | Không hợp lệ |
| Trả nửa frame? | Hợp lệ | Không hợp lệ |
| Trả 2,5 frame dính nhau? | Hợp lệ | Không hợp lệ |
| Có gói chưa đủ? | Cứ trả phần đã có | Phải báo ⏸ và tự giữ lại |
| Ai ghép/đệm? | Lớp ráp frame của core | Chính transport |

Transport kiểu dòng byte không được cố trả về đúng một frame, vì nó không có đủ thông tin; muốn làm vậy nó phải đọc định dạng trên dây, tức là sai tầng. Nó trả byte thô ngay khi nhận được.

Transport kiểu gói thì ngược lại: cấm trả về một gói dang dở. Khi mới nhận được một phần, transport trả ⏸, và phần dang dở không được đi lên core.

---

## 4. Luồng: kết nối lần đầu

```
  ỨNG DỤNG               CORE                  TRANSPORT
   |                      |                            |
   |  "kết nối tới X"     |                            |
   |─────────────────────>|                            |
   |                      |                            |
   |                      | (transport đã được tạo     |
   |                      | và kết nối sẵn ở ngoài)    |
   |                      |                            |
   |                      | tạo session, sinh greeting |
   |                      |                            |
   |                      | gửi greeting ─────────────>|
   |                      |<────────── ✔ hoặc ⏸        |
   |<─── xong ────────────|                            |
   |                      |                            |
```

Transport phải kết nối xong trước khi bàn giao cho core. Handshake TCP, handshake TLS và nâng cấp WebSocket đều diễn ra bên ngoài core, trước bước này, và do người viết transport đảm nhận.

### Ai tạo transport

Core không bao giờ tự tạo transport. Việc tạo và kết nối thuộc về thư viện mở rộng, không thuộc ứng dụng hay core:

```
   ỨNG DỤNG        THƯ VIỆN WEBSOCKET                CORE
      |                       |                         |
      | "kết nối tới          |                         |
      |  wss://ví.dụ/ws"      |                         |
      |──────────────────────>|                         |
      |                       | mở TCP                  |
      |                       | nâng cấp giao thức      |
      |                       | chờ xác nhận            |
      |                       | → đường ống đã thông    |
      |                       |                         |
      |                       | gói thành transport ───>|
      |                       |                         | tạo session
      |                       |<──── handle session ────|
      |<──── handle session ──|                         |
      |
      | từ đây ứng dụng dùng nó y hệt một session TCP
```

Thư viện nên bọc phần này trong một hàm tiện dụng, ví dụ "kết nối WebSocket, trả về một session". Tốt nhất là ứng dụng không bao giờ thấy khái niệm transport mà chỉ chọn thư viện để gọi.

### Hai handshake độc lập

Core nhận một đường ống đã thông, rồi mới chạy handshake của riêng Fomoxa trên đường ống đó:

```
  handshake của transport        handshake của Fomoxa
  (TCP, TLS, WebSocket)  →     (greeting / verdict)
  ↑ ngoài phạm vi Fomoxa      ↑ core lo, chạy qua nhiều tick
```

Handshake Fomoxa chưa xong khi `connect` trả về mà tiếp tục chạy qua nhiều tick. Ứng dụng biết handshake đã xong khi nhận sự kiện READY. Trước thời điểm đó mọi lệnh gửi đều bị từ chối, nên không byte ứng dụng nào ra dây trước khi hai bên chốt xong.

---

## 5. Luồng: gửi dữ liệu

### Trường hợp thuận lợi

```
  ỨNG DỤNG               CORE                  TRANSPORT
   |                      |                            |
   |  "gửi message này"   |                            |
   |─────────────────────>|                            |
   |                      | đóng gói thành frame       |
   |                      | ───────── gửi ────────────>|
   |                      | <──────── ✔ ───────────────|
   |<─── xong ────────────|                            |
```

### Khi transport nghẽn

Luồng này khác với cách làm chờ chặn thông thường:

```
  ỨNG DỤNG               CORE                  TRANSPORT
   |                      |                            |
   |  "gửi message này"   |                            |
   |─────────────────────>|                            |
   |                      | ───────── gửi ────────────>|
   |                      | <──────── ⏸ chưa được ─────|
   |                      |                            |
   |                      | cất vào chỗ chờ            |
   |<─── xong ────────────|                            |
   |                      |                            |
   |  ... ứng dụng chạy tiếp, không bị đứng ...        |
   |                      |                            |
   |  tick kế tiếp        |                            |
   |─────────────────────>|                            |
   |                      | ─ gửi lại phần còn thiếu ─>|
   |                      | <──────── ✔ ───────────────|
```

Từ luồng trên:

1. Ứng dụng không bao giờ bị chặn. Lệnh gửi trả về ngay, kể cả khi transport đang đầy.
2. Transport không chờ. Nó trả "chưa được" rồi trả lại quyền điều khiển.
3. Core nhớ và thử lại. Ở mỗi tick, việc đầu tiên là đẩy nốt phần tồn đọng, trước mọi việc khác.

Thứ tự trên dây là tuyệt đối. Frame mới chen vào giữa một frame đang gửi dở làm hỏng vĩnh viễn bộ giải mã phía nhận, nên khi còn frame đang chờ thì cấm gửi frame mới.

### Khi nghẽn kéo dài

Chỗ chờ giữ một frame, cho một session, theo chiều gửi ra. Chi tiết:

- Một session là một kênh gửi logic. Tầng này không có khái niệm nhiều luồng song song.
- Chiều nhận không có khe chờ trong core, nhưng điều đó không có nghĩa là dữ liệu bị bỏ. Xem §6.
- Giới hạn một frame áp cho lệnh gửi của ứng dụng. Frame điều khiển do core tự sinh (POLL, REPLY, verdict handshake) xếp *sau* khe chờ, không ghi đè lên nó, và có trần riêng, xem `02_flows.md` §8.
- Kết nối nhiều luồng con (QUIC): giao diện transport chỉ lộ một cặp kênh gửi/nhận. Transport muốn dùng nhiều luồng con thì tự ghép kênh *bên trong* và tự giữ đúng thứ tự khi ghép. Core chỉ thấy một đường ống, và quy tắc một frame chờ áp cho kênh logic đó, không áp cho từng luồng con.

Nếu frame trước chưa đi mà ứng dụng gửi tiếp, lệnh gửi bị từ chối với lỗi nghẽn. Fomoxa không xếp hàng vô hạn. Khi peer không đọc, ứng dụng phải được biết và tự quyết định bỏ gói, giảm nhịp gửi hay ngắt kết nối, thay vì để bộ nhớ tăng dần mà không ai biết.

---

## 6. Luồng: nhận dữ liệu

Ở mỗi tick, core đọc cạn transport:

```
  CORE                                  TRANSPORT
   |                                        |
   |───── có gì mới không? ────────────────>|
   |<──── ✔ đây, một frame ─────────────────|
   | xử lý frame                            |
   |───── còn nữa không? ──────────────────>|
   |<──── ✔ đây, frame nữa ─────────────────|
   | xử lý frame                            |
   |───── còn nữa không? ──────────────────>|
   |<──── ⏸ hết rồi ────────────────────────|
   | dừng vòng lặp, sang việc khác          |
```

Mỗi lần hỏi, core nhận đúng một frame trọn vẹn hoặc tín hiệu hết dữ liệu. Với transport kiểu gói, transport tự đảm bảo điều này; với transport kiểu dòng byte, lớp ráp frame đảm bảo, còn transport bên dưới chỉ trả byte thô (§3).

Sau khi đọc cạn, core đưa từng frame vào bộ xử lý giao thức. Bộ xử lý nhận dạng frame là handshake, heartbeat hay message ứng dụng, rồi sinh sự kiện.

### Ngân sách mỗi tick

Vòng lặp trên phải có trần. Core giới hạn số frame xử lý trong một tick; khi chạm trần, core dừng và để phần còn lại cho tick sau, nhờ đó ứng dụng lấy lại quyền điều khiển đúng hạn.

Trần này không nhằm phòng transport viết sai, vì transport do chính ứng dụng chọn và nạp vào. Nó tồn tại để chia đều thời gian cho vòng lặp: nếu không có trần, một peer gửi dồn dập sẽ giữ vòng lặp chạy mãi. Dữ liệu chưa xử lý không mất mà nằm lại trong transport và được lấy ở tick sau.

### Dữ liệu ở đâu khi ứng dụng ngừng tick

Dữ liệu vẫn nằm nguyên và không bị core bỏ. Tuy nhiên vị trí của nó tùy transport, và hậu quả khác nhau đáng kể. Core không bỏ dữ liệu cũ, nhưng cũng không tự chép dữ liệu sang một hàng đợi riêng.

Dữ liệu chưa lấy hết nằm trong bộ đệm transport hoặc bộ đệm hệ điều hành. Khi bộ đệm đó đầy:

| Transport | Chuyện gì xảy ra khi ứng dụng ngừng tick lâu |
|-----------|------------------------------------------------|
| TCP / TLS / WebSocket | Cửa sổ nhận của hệ điều hành đầy → ngừng báo nhận → phía gửi bị nghẽn và thấy ⏸ ở đầu bên kia. Không mất byte nào, nhưng peer bị chặn lại. |
| UDP | Bộ đệm nhận đầy → hệ điều hành lặng lẽ vứt gói mới. Mất dữ liệu và không bên nào được báo, đúng bản chất UDP. |
| Transport tự đệm bên trong | Bộ đệm đầy → transport vứt gói cũ nhất để nhận gói mới. Bộ đệm này bắt buộc có trần; để nó phình vô hạn là vi phạm. |

Bộ đệm cuối chuỗi thuộc về transport, không thuộc core, vì core không có hàng đợi đầu vào (§5). Transport quyết định việc bỏ gói, và core không biết có gói nào bị bỏ.

Transport bỏ gói cũ nhất thay vì gói mới nhất vì hai lý do. Transport không được đọc payload (§12), nên không phân biệt được gói handshake với gói dữ liệu để ưu tiên, và cần một quy tắc duy nhất cho mọi trạng thái session. Với giao thức thời gian thực, dữ liệu mới có giá trị hơn dữ liệu cũ. Ứng dụng cần bảo đảm không mất dữ liệu thì dùng transport kiểu dòng byte (§7) thay vì dựa vào hàng đợi này.

Nếu gói bị bỏ là gói handshake, không phát sinh chế độ lỗi mới: mất gói handshake dẫn tới hết hạn rồi thất bại. Đây là fail-closed, khớp với vòng query mô tả ở `02_flows.md` §3.3.2. Fomoxa không truyền lại, nên mọi handshake trên transport kiểu gói đều là best-effort.

Hướng hỏng này an toàn. Session chỉ mở khi nhận đúng byte verdict `0` (`02_flows.md` §3.4), nên mất gói không bao giờ biến một verdict từ chối thành chấp nhận; nó chỉ chặn một session lẽ ra đã mở, và ứng dụng phải kết nối lại. Hàng đợi cũng gần như rỗng lúc handshake vì cả quá trình chỉ có hai đến bốn gói nhỏ; trần hàng đợi chỉ bị chạm khi lưu lượng dồn.

Ngoài ra, khi ứng dụng ngừng tick thì heartbeat cũng ngừng (bước 3, §8). Peer gửi probe, không nhận được trả lời và tuyên bố session chết. Vì vậy ngừng tick lâu không phải một trạng thái tạm dừng an toàn mà dẫn tới mất kết nối.

---

## 7. Giới hạn kích thước, và những gì Fomoxa không bảo đảm

### Kích thước tối đa

Fomoxa giới hạn cứng payload của message ứng dụng ở 16 MiB, tức 16 MiB + 11 byte khi tính cả tiêu đề frame. Con số này chung cho mọi bản triển khai. Giá trị chính xác và bố cục byte nằm ở `02_flows.md` §2.

Trần của transport thường thấp hơn nhiều, và điều này là bình thường:

| Transport | Trần thực tế |
|-----------|--------------|
| TCP / TLS | Không có trần riêng; dòng byte chảy bao nhiêu cũng được |
| WebSocket | Rất lớn về lý thuyết, nhưng thư viện và proxy trung gian thường đặt trần cấu hình được |
| UDP | ~64 KiB tuyệt đối; và nên giữ dưới ~1200 byte để không bị phân mảnh ở tầng IP |

Frame vượt trần transport thì transport trả ⊘ "quá lớn" (§2), không trả ⏸. Lỗi này không kết thúc session: kết nối vẫn sống, chỉ message đó không gửi được. Ứng dụng nhận lỗi và tự xử lý; core không thử lại.

Cấm transport tự cắt nhỏ frame để gửi, vì cắt nhỏ là đổi định dạng trên dây.

Theo chiều nhận, khi gói đến lớn hơn chỗ core cấp, transport trả ⤢ kèm số byte cần. Core nới bộ đệm và hỏi lại ngay trong tick đó. Cấm làm mất gói.

### Những gì Fomoxa không làm

Core không sắp xếp lại, không truyền lại gói mất và không lọc gói trùng, vì định dạng trên dây không có số thứ tự để làm những việc đó. Trường định danh message trong frame là *loại* message, không phải *số thứ tự*.

Hệ quả trực tiếp:

```
  Trên TCP/TLS   →  thứ tự và tính toàn vẹn do chính transport bảo đảm.
                    Fomoxa thừa hưởng, không làm thêm gì.

  Trên UDP       →  gói có thể mất, có thể đến lệch thứ tự, có thể đến hai lần.
                    Fomoxa giao lên ứng dụng đúng thứ tự nó nhận được, không sửa gì.
```

Transport UDP không phải sắp xếp lại thứ tự gói và cũng không được phép làm, vì sắp xếp lại đòi hỏi đệm và trì hoãn gói, tạo ra độ trễ mà core không tính trước được.

Ứng dụng cần đúng thứ tự và không mất mát có hai lựa chọn: dùng transport kiểu dòng byte (TCP, TLS, WebSocket đã bảo đảm sẵn), hoặc tự xử lý thứ tự và độ tin cậy ở tầng ứng dụng. Đây là đánh đổi có chủ đích; trong game, mất một gói vị trí thường tốt hơn chờ truyền lại.

Heartbeat và phát hiện peer chết vẫn chạy trên mọi transport, kể cả UDP, vì chúng dựa vào thời gian chứ không dựa vào số thứ tự gói.

---

## 8. Luồng: tick

Một tick làm đúng bốn việc, theo đúng thứ tự sau:

```
  ┌─ 1. đẩy nốt frame còn tồn đọng     (thứ tự trên dây là bất khả xâm phạm)
  │
  ├─ 2. vắt cạn dữ liệu đến             (§6, trong hạn mức)
  │
  ├─ 3. chạy đồng hồ giao thức          (hết hạn handshake? cần gửi probe?
  │                                       peer im lặng quá lâu → coi như chết?)
  │
  └─ 4. trả danh sách sự kiện cho ứng dụng
```

Transport chỉ tham gia bước 1 và 2. Bước 3 nằm hoàn toàn trong core và transport không can thiệp được, nhờ đó heartbeat và phát hiện peer chết chạy giống hệt nhau trên mọi transport mà không phải viết lại cho từng loại.

---

## 9. Luồng: đóng kết nối

Một session có thể kết thúc theo ba cách, và vai trò của transport khác nhau ở từng cách.

### 9.1 Ứng dụng chủ động ngắt

```
  ỨNG DỤNG ──"ngắt"──> CORE ──đóng mềm──> TRANSPORT ──> báo phía kia (nếu làm được)
```

Với kết nối có cơ chế đóng êm (TCP, WebSocket), transport gửi tín hiệu đóng. Với kết nối không có cơ chế này (UDP), transport không làm gì, và đó là hành vi đúng.

### 9.2 Phía kia biến mất

Có hai tình huống khác nhau về bản chất, và transport phải phân biệt được:

```
  ✖ ĐÓNG CÓ TRẬT TỰ                    ⚠ ĐỨT ĐỘT NGỘT
  ─────────────────                    ────────────────
  TCP kết thúc sạch (FIN)              TCP bị reset (RST)
  WebSocket gửi frame đóng             lỗi đọc ở tầng dưới
  luồng QUIC kết thúc sạch             transport hỏng giữa chừng
                                       dữ liệu đến sai định dạng
  ↓                                    ↓
  "phía kia nói lời tạm biệt"          "phía kia mất tích"
```

Transport phải trả đúng ✖ hoặc ⚠ theo tình huống thực tế. Cấm gộp hai trạng thái này.

Core có thể chưa dùng đến sự phân biệt đó; một bản triển khai đơn giản có thể gộp cả hai thành một sự kiện DISCONNECT, và việc gộp hay không là quyền của core. Khi core bắt đầu phân biệt ✖ với ⚠, transport báo đúng sẽ chạy được ngay mà không phải sửa. Transport báo sai thì phải xem lại toàn bộ logic xử lý lỗi vào lúc hệ thống đã chạy thật. Làm đúng từ đầu gần như không tốn thêm công, vì thư viện mạng nền vốn đã phân biệt hai trường hợp này.

### 9.3 Phía kia im lặng

Trường hợp này không có thủ tục đóng: peer ngừng trả lời. Core tự phát hiện bằng đồng hồ ở bước 3 của tick, bằng cách gửi một probe và tuyên bố peer chết nếu quá hạn mà không nhận được trả lời.

Transport không tham gia vào việc phát hiện này, nhờ đó cơ chế chạy được cả trên UDP, nơi không có kết nối nào để mất.

---

## 10. Ai chịu trách nhiệm gì

| Việc | Core | Transport |
|------|:----:|:---------:|
| Handshake của transport (TCP/TLS/WebSocket) | | ✔ |
| Handshake của Fomoxa (greeting/verdict) | ✔ | |
| Cắt và ráp frame | ✔ | |
| Giữ thứ tự trên dây | ✔ | |
| Thử gửi lại khi nghẽn | ✔ | |
| Heartbeat, phát hiện peer chết | ✔ | |
| Đo thời gian, hạn định | ✔ | |
| Giới hạn số frame xử lý mỗi tick | ✔ | |
| Sinh sự kiện cho ứng dụng | ✔ | |
| Đưa byte qua mạng | | ✔ |
| Báo trạng thái tức thời (✔ / ⏸ / ✖ / ⚠ / ⊘ / ⤢) | | ✔ |
| Bỏ qua dữ liệu không phải của session này | | ✔ |
| Từ chối frame vượt trần của mình | | ✔ |
| Giải phóng tài nguyên của chính nó | | ✔ |

Cột transport được giữ ngắn có chủ ý để việc viết một transport mới là việc nhỏ.

### Chi tiết về "bỏ qua dữ liệu không phải của session này"

Quy định này chỉ áp dụng cho transport không kết nối, trên thực tế là UDP. Một điểm cuối UDP nhận gói từ mọi nguồn, nên transport UDP phải đối chiếu địa chỉ nguồn với peer đang chạy. Gói lạ bị bỏ mà không báo lỗi và không kết thúc session, vì một gói lạ không phản ánh trạng thái session.

Điểm cuối đó cũng phải đặt trần cho bảng peer. Mỗi địa chỉ nguồn mới gửi một byte là tạo một mục trong bảng, không cần handshake hay điều kiện nào khác, và một máy có thể sinh hàng chục nghìn cổng nguồn. Không có trần thì bảng tăng vô hạn, điều mà §6 cấm. Khi chạm trần, gói từ địa chỉ chưa biết bị bỏ như gói lạ ở trên, còn session đang chạy không bị ảnh hưởng. Con số cụ thể tùy bản triển khai, nhưng bắt buộc phải có.

TCP, TLS, WebSocket và QUIC đã gắn sẵn với một peer, nên điều kiện này tự thỏa mãn và transport không phải làm gì thêm.

Bước này không phải kiểm tra bảo mật: nó không xác thực và không chống giả mạo. Việc hai bên có dùng cùng schema hay không do handshake Fomoxa ở core kiểm tra. Các kiểm tra riêng của giao thức thuộc về bên chấp nhận kết nối và nằm ngoài phạm vi tài liệu này.

---

## 11. Thêm một transport mới

Ví dụ: viết một transport WebSocket.

### Bước 1: xác định loại

WebSocket giữ ranh giới gói nên thuộc kiểu gói. (Transport TLS thuộc kiểu dòng byte và dùng lớp ráp frame có sẵn của core.)

### Bước 2: tự xử lý phần kết nối

Transport tự mở kết nối WebSocket: phân tích địa chỉ, mở kết nối nền, nâng cấp giao thức, xử lý phản hồi. Core không tham gia và không cần biết. Kết quả là một đường ống đã thông.

### Bước 3: cung cấp bốn chức năng

- Gửi: nhận một khối byte từ core, gói vào một gói WebSocket nhị phân và gửi đi. Gửi được thì trả ✔; không được thì trả ⏸ *ngay*, cấm chờ. Khối byte vượt trần thì trả ⊘; cấm cắt nhỏ, cấm trả ⏸.
- Nhận: có một gói WebSocket trọn vẹn thì đưa cho core. Chưa trọn thì trả ⏸ và tự giữ trong bộ đệm. Gói lớn hơn chỗ core cấp thì trả ⤢ kèm số byte cần. Đóng sạch trả ✖, đứt đột ngột trả ⚠.
- Đóng mềm: gửi frame đóng của WebSocket.
- Đóng hẳn: giải phóng kết nối, bộ đệm và mọi tài nguyên.

> Phải sử dụng kiểu dữ liệu nhị phân; cấm sử dụng kiểu văn bản.
> Các gói WebSocket dạng văn bản yêu cầu nội dung UTF-8 hợp lệ, và các thư viện ở hai đầu kết nối
> có thể kiểm tra tính hợp lệ hoặc chuẩn hóa nội dung đó. Các frame Fomoxa bao gồm các byte nhị phân thô
> mà gần như chắc chắn không phải là UTF-8 hợp lệ; việc gửi chúng qua các gói văn bản sẽ
> làm hỏng dữ liệu hoặc khiến chính thư viện WebSocket từ chối chúng. Quy tắc này
> áp dụng cho bất kỳ transport nào có phân biệt giữa các "kiểu dữ liệu": luôn chọn định dạng nhị phân thô.

### Bước 4: giao cho core

Gói đường ống đã thông thành một transport kiểu gói, đưa vào core và nhận về một session. Bước này nên được bọc trong một hàm tiện dụng của thư viện (§4) để ứng dụng không phải thấy khái niệm transport.

Từ đây ứng dụng dùng session này giống hệt một session TCP: cùng lệnh gửi, cùng tick, cùng danh sách sự kiện.

### Bước 5: không sửa core

Nếu phải sửa core để transport mới chạy được thì hoặc thiết kế còn thiếu, hoặc logic đang đặt sai tầng. Trường hợp này cần dừng lại để xem xét.

### Những việc người viết transport không phải làm

- Không cần hiểu định dạng trên dây của Fomoxa
- Không cần biết greeting, verdict, POLL hay REPLY là gì
- Không cần cắt hoặc ghép các frame
- Không cần duy trì thứ tự (trừ khi tự gộp nhiều luồng con, §5)
- Không cần xử lý việc truyền lại dữ liệu
- Không cần thực hiện bất kỳ phép đo thời gian nào
- Không cần làm heartbeat
- Không cần sắp xếp lại các gói đến sai thứ tự (§7)

---

## 12. Những điều transport không được làm

| Cấm | Vì sao |
|-----|--------|
| Ngồi chờ dưới mọi hình thức | Treo vòng lặp chính. Đây là điều cấm số một. |
| Tự sinh luồng thực thi ngầm | Fomoxa quy ước một session do một luồng lái; luồng ngầm phá vỡ điều đó ở chỗ không ai kiểm tra được. |
| Gọi ngược vào core | Đang ở giữa một thao tác của core, gọi ngược lại là đệ quy vào trạng thái đang dở dang. |
| Nhận nửa cục byte rồi báo ⏸ (với kiểu gói) | Core sẽ gửi lại từ đầu, phía kia nhận frame hỏng và hỏng vĩnh viễn. |
| Trả về một gói dang dở (với kiểu gói) | Core coi mọi thứ nhận được là một frame trọn vẹn. |
| Tự cắt nhỏ frame để gửi | Thay đổi định dạng trên dây, mất tương thích với các bản triển khai khác. |
| Tự sắp xếp lại hay gửi lại gói | Thêm độ trễ mà core không biết, và phá vỡ đánh đổi có chủ đích của UDP. |
| Tự kết nối lại khi đứt | Core đang tính peer đã chết. Một đường ống sống lại trong im lặng khiến hai bên hiểu khác nhau về trạng thái session. |
| Diễn giải nội dung byte | Sai tầng. Byte đó có thể là handshake hoặc dữ liệu ứng dụng, và transport không được phân biệt. |
| Thêm phần đầu của riêng mình | Định dạng trên dây là cố định để mọi bản triển khai tương thích với nhau; thêm byte là mất tương thích. |
| Dùng kiểu dữ liệu văn bản (nếu transport có phân biệt) | Frame Fomoxa là nhị phân thuần túy, không phải UTF-8 hợp lệ. |

---

## 13. Toàn cảnh: một message đi từ đầu này sang đầu kia

```
   ỨNG DỤNG (máy A)                              ỨNG DỤNG (máy B)
       │                                         ▲
       │ gửi message                             │ sự kiện MESSAGE
       ▼                                         │
   CORE ── đóng thành frame                      CORE ── mở frame,
       │                                         │   nhận ra là dữ liệu
       │ (nếu đã READY)                          │   của ứng dụng
       ▼                                         │
   TRANSPORT ── thành gói WebSocket              TRANSPORT ── nhận gói WebSocket
       │                                         ▲
       └──────────────── mạng ───────────────────┘

   Đổi WebSocket thành TCP, UDP hay QUIC:
   hai hộp CORE và hai hộp ỨNG DỤNG không đổi một chút nào.
```

Thiết kế này nhằm đạt đúng tính chất trên: đổi transport không làm thay đổi core hay ứng dụng.

---

## 14. Bất biến mà mọi bản triển khai phải giữ

Khi dựng lại Fomoxa trên ngôn ngữ khác, các điều sau không được thay đổi. Vi phạm một điều là mất khả năng liên thông với các bản triển khai khác.

| # | Bất biến |
|---|----------|
| B1 | Định dạng trên dây là cố định. Không thêm, bớt, hay đổi thứ tự byte nào. Không thêm lớp đóng gói riêng. (`02_flows.md` §2) |
| B2 | Không có lớp nào được phép chặn. Mọi thao tác trả quyền điều khiển lại ngay. |
| B3 | Handshake xong mới được gửi dữ liệu ứng dụng. Không byte ứng dụng nào ra dây trước verdict chấp nhận. |
| B4 | Server là bên duy nhất so sánh schema. Client gửi greeting và chỉ đọc một byte verdict. |
| B5 | Probe khi im lặng, không phải gửi định kỳ. Và luôn probe trước khi tuyên bố chết. |
| B6 | Đúng một sự kiện kết thúc cho mỗi session. |
| B7 | Thứ tự trên dây tuyệt đối. Không frame nào chen vào giữa một frame đang gửi dở. |
| B8 | Mốc thời gian là tham số truyền vào, không phải đồng hồ đọc bên trong. Đây là điều kiện để kiểm thử được toàn bộ chu kỳ hết hạn mà không cần chờ thật. |
| B9 | Đồng hồ phải là đồng hồ đơn điệu, không bao giờ là đồng hồ treo tường: một lần đồng bộ giờ hệ thống không được phép làm hết hạn một cuộc handshake hay giết một peer đang sống. |

Tiêu chí kiểm chứng: hai bản triển khai bất kỳ, viết bằng hai ngôn ngữ bất kỳ, phải liên thông được với nhau mà không bên nào cần biết bên kia viết bằng gì.

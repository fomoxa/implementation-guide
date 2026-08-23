# Fomoxa Transport  -  Mô hình và trách nhiệm

Tài liệu này mô tả mô hình transport của Fomoxa ở mức khái niệm. Độc lập với mọi ngôn ngữ lập trình. Không tham chiếu implementation, API hay mã nguồn cụ thể nào.

Đọc tài liệu này cùng `02_flows.md` là đủ để dựng lại Fomoxa: một implementation mới, hoặc một transport mới, trên bất kỳ ngôn ngữ nào. Không cần đọc mã nguồn của bản triển khai nào đang có.

| Tài liệu | Nội dung | Đọc khi |
|---|---|---|
| `01_overview.md` (này) | Mô hình, ranh giới tầng, phân chia trách nhiệm | Muốn hiểu thiết kế |
| `02_flows.md` | Định dạng trên dây, từng luồng chi tiết, hướng dẫn triển khai | Bắt tay vào viết |

Mỗi ngôn ngữ có thể có tài liệu ràng buộc API riêng. Những tài liệu đó mô tả *cách gọi*, không mô tả *hành vi*. Hành vi nằm ở hai tài liệu này. Mọi bản triển khai phải giống nhau.

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

Ranh giới quan trọng nhất: core không biết dữ liệu đi bằng đường nào, transport không biết dữ liệu chứa gì. Transport không thấy handshake, không thấy heartbeat, không thấy trạng thái peer. Nó không phân biệt được greeting với dữ liệu vị trí nhân vật. Với transport, tất cả chỉ là một khối byte cần đẩy sang phía kia.

Chiều ngược lại: core không biết mình chạy trên TCP hay WebSocket. Nó cũng không được phép biết.

---

## 2. Bốn chức năng transport phải cung cấp

Đúng bốn, không hơn:

| Chức năng | Nghĩa | Ai gọi |
|-----------|-------|--------|
| Gửi | Nhận một cục byte, đưa sang phía kia | Core, khi có gì cần gửi |
| Nhận | Trả về dữ liệu đã tới từ phía kia, nếu có | Core, mỗi tick |
| Đóng mềm | Báo phía kia biết ta sắp ngắt | Core, khi ứng dụng yêu cầu ngắt |
| Đóng hẳn | Giải phóng mọi tài nguyên | Core, khi hủy session |

Không có chức năng thứ năm. Không có "kết nối lại". Không có "đợi cho tới khi gửi được". Không có "cho tôi biết còn bao nhiêu chỗ trống".

### Bốn câu trả lời chung

Hỏi gửi hay hỏi nhận, transport đều trả lời bằng một bộ từ vựng:

```
  ✔  Tôi làm được rồi      → core yên tâm đi tiếp
  ⏸  Bây giờ chưa được     → core giữ lại, tick sau hỏi tiếp
  ✖  Tôi đóng rồi          → core kết thúc session
  ⚠  Tôi bị lỗi            → core kết thúc session
```

### Và hai câu đặc biệt, mỗi chiều một câu

Bốn câu trên chưa phủ hết. Còn hai tình huống: không phải thành công, không phải "chờ tí", cũng không phải chết:

```
  Chiều GỬI
  ⊘  Cục này quá lớn so với khả năng của tôi
     → không giết session. Core báo lỗi lên ứng dụng, kết nối vẫn sống.
     → Core không thử lại. (§7)

  Chiều NHẬN
  ⤢  Có dữ liệu, nhưng chỗ anh đưa không đủ. Tôi cần bằng này byte.
     → Core nới chỗ rồi hỏi lại ngay. Dữ liệu không mất. (§7)
```

⊘ phải tách khỏi ⏸ vì hai câu này nói hai điều khác nhau. ⏸ nghĩa là "thử lại sau, rồi sẽ được". Transport trả ⏸ cho một payload quá lớn thì core giữ lại và thử lại mỗi tick. Payload vẫn quá lớn ở tick sau. Vòng lặp không bao giờ dứt, kênh gửi tắc vĩnh viễn. ⊘ nói điều ngược lại: *đừng thử lại; payload này không bao giờ gửi được*.

⤢ cũng thuộc nhóm "thử lại rồi sẽ được". Nó kèm một con số: core phải nới bộ đệm trước khi hỏi lại. Nên nó cũng không gộp được vào ⏸.

### Điều cấm quan trọng nhất

Không có câu trả lời kiểu "để tôi chờ". Đây là điều cấm số một của thiết kế. Fomoxa chạy trong một vòng lặp có định thời. Transport ngồi chờ là cả vòng lặp đứng. Transport chỉ báo trạng thái *ngay lúc được hỏi* rồi trả lại quyền điều khiển. Thử lại là việc của core.

---

## 3. Hai loại transport

Đúng hai kiểu. Transport phải khai báo mình thuộc kiểu nào:

### Kiểu dòng byte (stream)

Byte chảy liên tục, không vách ngăn. Gửi 3 cục 100 byte, phía kia có thể nhận 1 cục 300 byte. Hoặc 7 cục lặt vặt. Không ai biết chỗ nào là hết một đơn vị.

→ TCP, TLS, QUIC stream

### Kiểu gói (packet)

Transport tự giữ vách ngăn. Gửi 3 cục, phía kia nhận đúng 3 cục. Đúng ranh giới, không dính vào nhau.

→ UDP, WebSocket, QUIC datagram

### Vì sao phải phân biệt

Transport kiểu dòng byte cần ráp lại dữ liệu. Core làm việc đó. Transport khai báo kiểu dòng byte thì core tự chèn một lớp ráp frame vào giữa. Người viết transport không cần biết Fomoxa đóng frame thế nào.

```
  Transport kiểu gói             Transport kiểu dòng byte
  ──────────────────             ────────────────────────
        CORE                            CORE
          |                               |
          |                       [ lớp ráp frame ]   ← core tự lo
          |                               |
      WebSocket                          TCP
```

Nhờ vậy người viết WebSocket hay TLS không phải đọc định dạng trên dây.

Lớp ráp frame phải nằm giữa core và transport, không nằm trong bên nào. Đặt trong transport thì transport phải hiểu định dạng frame - sai tầng. Đặt trong core thì core phải kiểm tra kiểu transport ở mỗi lệnh gọi. Cũng sai tầng. Đặt ở giữa thì việc phân nhánh chỉ xảy ra một lần, lúc khởi tạo. Hai đầu đều sạch.

### Nghĩa vụ khi trả dữ liệu về  -  hai kiểu khác hẳn nhau

Đây là chỗ dễ hiểu nhầm nhất. Nói thật rõ:

| | Kiểu dòng byte | Kiểu gói |
|---|---|---|
| Mỗi lần trả về | Bao nhiêu byte cũng được | Đúng một gói trọn vẹn |
| Trả 1 byte lẻ? | Hợp lệ | Không hợp lệ |
| Trả nửa frame? | Hợp lệ | Không hợp lệ |
| Trả 2,5 frame dính nhau? | Hợp lệ | Không hợp lệ |
| Có gói chưa đủ? | Cứ trả phần đã có | Phải báo ⏸ và tự giữ lại |
| Ai ghép/đệm? | Lớp ráp frame của core | Chính transport |

Transport kiểu dòng byte không được cố trả về "đúng một frame". Nó không đủ thông tin để làm. Muốn làm thì phải đọc định dạng trên dây - sai tầng. Cứ trả byte thô ngay khi nhận được.

Transport kiểu gói thì ngược lại. Cấm trả về một gói dang dở. Mới nhận được một phần thì trả ⏸. Phần dang dở không được đi lên core.

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

Điểm mấu chốt: transport phải thông trước khi bàn giao cho core. Handshake TCP, handshake TLS, nâng cấp WebSocket - làm hết ở ngoài, trước đó. Người viết transport lo phần này.

### Ai tạo transport

Core không bao giờ tự tạo transport. Việc tạo và kết nối thuộc về thư viện mở rộng. Không phải việc của ứng dụng, cũng không phải của core:

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

Thư viện nên bọc phần đó trong một hàm tiện dụng. Ví dụ: "kết nối WebSocket, trả về một session". Lý tưởng nhất là ứng dụng không bao giờ thấy khái niệm transport. Nó chỉ chọn thư viện nào để gọi.

### Hai cái handshake, hoàn toàn độc lập

Core nhận một đường ống đã thông. Rồi mới chạy handshake của riêng Fomoxa trên đường ống đó:

```
  handshake của transport        handshake của Fomoxa
  (TCP, TLS, WebSocket)  →     (greeting / verdict)
  ↑ ngoài phạm vi Fomoxa      ↑ core lo, chạy qua nhiều tick
```

Handshake Fomoxa chưa xong khi `connect` trả về. Nó chạy tiếp qua nhiều tick. Ứng dụng biết nó xong khi nhận sự kiện READY. Trước đó mọi lệnh gửi đều bị từ chối. Không byte ứng dụng nào ra dây trước khi hai bên chốt xong.

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

Đây là luồng quan trọng. Chỗ này khác hẳn cách làm ngây thơ:

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

Ba điều rút ra:

1. Ứng dụng không bao giờ bị chặn. Lệnh gửi trả về ngay, kể cả khi transport đang đầy.
2. Transport không phải ngồi chờ. Nó nói "chưa được" rồi thôi.
3. Core nhớ và thử lại. Mỗi tick, việc đầu tiên là đẩy nốt phần tồn đọng - trước mọi việc khác.

Thứ tự trên dây là tuyệt đối. Frame mới chen vào giữa frame đang gửi dở làm hỏng bộ giải mã phía nhận. Hỏng vĩnh viễn. Còn frame đang chờ thì cấm gửi frame mới.

### Khi nghẽn kéo dài

Chỗ chờ giữ một frame, cho một session, theo chiều gửi ra. Nói cho hết ý:

- Một session là một kênh gửi logic. Tầng này không có khái niệm nhiều luồng song song.
- Chiều nhận không có khe chờ trong core. Không có nghĩa là dữ liệu bị vứt. Xem §6.
- Giới hạn "một frame" áp cho lệnh gửi của ứng dụng. Frame điều khiển do core tự sinh - POLL, REPLY, verdict handshake - xếp *sau* khe chờ, không ghi đè lên nó. Chúng có trần riêng, xem `02_flows.md` §8.
- Kết nối nhiều luồng con (QUIC): giao diện transport chỉ lộ một cặp kênh gửi/nhận. Transport muốn dùng nhiều luồng con thì tự ghép kênh *bên trong*, và tự giữ đúng thứ tự khi ghép. Core chỉ thấy một đường ống. Quy tắc "một frame chờ" áp cho kênh logic đó, không áp cho từng luồng con.

Frame trước chưa đi mà ứng dụng gửi tiếp thì lệnh gửi bị từ chối, báo "nghẽn". Fomoxa không xếp hàng vô hạn. Peer không đọc thì ứng dụng phải biết và tự quyết: vứt gói, giảm nhịp gửi, hoặc ngắt. Không để bộ nhớ âm thầm phình lên.

---

## 6. Luồng: nhận dữ liệu

Mỗi tick, core vắt cạn transport:

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

Mỗi lần hỏi, core nhận đúng một frame trọn vẹn, hoặc một tín hiệu "hết rồi". Transport kiểu gói tự lo việc này. Transport kiểu dòng byte thì lớp ráp frame lo. Transport bên dưới chỉ trả byte thô (§3).

Vắt cạn xong core mới đưa từng frame vào bộ xử lý giao thức. Ở đó frame được nhận dạng: handshake, heartbeat hay message ứng dụng. Rồi sinh sự kiện.

### Ngân sách mỗi tick

Vòng lặp trên phải có trần. Core giới hạn số frame xử lý trong một tick. Chạm trần thì dừng, phần còn lại để tick sau. Ứng dụng lấy lại quyền điều khiển đúng hạn.

Trần này không phải để phòng transport viết ẩu. Transport do chính ứng dụng chọn và nạp vào, không phải thành phần lạ. Nó tồn tại để chia đều thời gian cho vòng lặp. Thiếu nó, một peer gửi dồn dập sẽ giữ vòng lặp chạy mãi. Dữ liệu chưa xử lý không mất. Nó nằm lại trong transport, tick sau lấy tiếp.

### Nếu ứng dụng ngừng tick thì dữ liệu đi đâu

Dữ liệu vẫn nằm nguyên đó, không ai vứt nó. Nhưng "đó" là chỗ nào thì tùy transport, và hậu quả khác nhau hẳn. Core không vứt dữ liệu cũ. Nó chỉ không tự chép dữ liệu sang một hàng đợi riêng.

Dữ liệu chưa lấy hết nằm trong bộ đệm transport hoặc bộ đệm hệ điều hành. Khi chỗ đó đầy:

| Transport | Chuyện gì xảy ra khi ứng dụng ngừng tick lâu |
|-----------|------------------------------------------------|
| TCP / TLS / WebSocket | Cửa sổ nhận của hệ điều hành đầy → ngừng báo nhận → phía gửi bị nghẽn và thấy ⏸ ở đầu bên kia. Không mất byte nào, nhưng peer bị chặn lại. |
| UDP | Bộ đệm nhận đầy → hệ điều hành lặng lẽ vứt gói mới. Mất dữ liệu, và không ai được báo  -  đúng bản chất UDP. |
| Transport tự đệm bên trong | Bộ đệm đầy → transport vứt gói cũ nhất để nhận gói mới. Bộ đệm này bắt buộc có trần; để nó phình vô hạn là vi phạm. |

Bộ đệm cuối chuỗi thuộc về transport, không thuộc core. Core không có hàng đợi đầu vào (§5). Việc vứt gói do transport quyết. Core không biết có gói nào bị vứt.

Vì sao vứt gói cũ nhất, không vứt gói mới nhất? Transport không được đọc payload (§12). Nó không phân biệt được gói handshake với gói dữ liệu để ưu tiên. Một quy tắc duy nhất cho mọi trạng thái session. Với giao thức thời gian thực, dữ liệu mới đáng giá hơn dữ liệu cũ. Cần bảo đảm không mất dữ liệu thì dùng transport kiểu dòng byte (§7). Đừng trông vào hàng đợi này.

Gói bị vứt lại đúng là gói handshake thì sao? Không có chế độ lỗi mới nào. Mất gói handshake dẫn tới hết hạn rồi thất bại. Đây là fail-closed, khớp với vòng query mô tả ở `02_flows.md` §3.3.2. Fomoxa không truyền lại. Mọi handshake trên transport kiểu gói đều là best-effort.

Cái an toàn nằm ở hướng hỏng. Session chỉ mở khi nhận đúng byte verdict `0` (`02_flows.md` §3.4). Mất gói không bao giờ biến một verdict từ chối thành chấp nhận. Nó chỉ chặn một session lẽ ra đã mở, và ứng dụng phải kết nối lại. Hàng đợi cũng gần như rỗng lúc handshake: cả quá trình chỉ hai đến bốn gói nhỏ. Trần hàng đợi chỉ chạm khi lưu lượng dồn.

Còn một hệ quả nữa. Ứng dụng ngừng tick thì heartbeat cũng ngừng (bước 3, §8). Peer probe, không thấy trả lời, tuyên bố session chết. Ngừng tick lâu không phải "tạm dừng an toàn". Đó là cách mất kết nối.

---

## 7. Giới hạn kích thước, và những gì Fomoxa không bảo đảm

### Kích thước tối đa

Fomoxa chặn cứng payload của message ứng dụng: 16 MiB. Tính cả tiêu đề frame là 16 MiB + 11 byte. Con số này chung cho mọi bản triển khai. Giá trị chính xác và bố cục byte: `02_flows.md` §2.

Trần của transport thường thấp hơn nhiều. Đó là chuyện bình thường:

| Transport | Trần thực tế |
|-----------|--------------|
| TCP / TLS | Không có trần riêng  -  dòng byte chảy bao nhiêu cũng được |
| WebSocket | Rất lớn về lý thuyết, nhưng thư viện và proxy trung gian thường đặt trần cấu hình được |
| UDP | ~64 KiB tuyệt đối; và nên giữ dưới ~1200 byte để không bị phân mảnh ở tầng IP |

Frame vượt trần transport thì transport trả ⊘ "quá lớn" (§2), không phải ⏸. Đây là lỗi không giết session: kết nối vẫn sống, chỉ message đó không đi được. Ứng dụng nhận lỗi và tự xử. Core không thử lại.

Cấm transport tự cắt nhỏ frame để gửi. Cắt nhỏ là đổi định dạng trên dây.

Chiều ngược lại: gói đến lớn hơn chỗ core đưa thì transport trả ⤢ kèm số byte cần. Core nới bộ đệm và hỏi lại ngay trong tick đó. Cấm làm mất gói.

### Những gì Fomoxa không làm

Core không sắp xếp lại, không truyền lại gói mất, không lọc gói trùng. Định dạng trên dây không có số thứ tự để làm những việc đó. Trường định danh message trong frame là *loại* message, không phải *số thứ tự*.

Hệ quả trực tiếp:

```
  Trên TCP/TLS   →  thứ tự và tính toàn vẹn do chính transport bảo đảm.
                    Fomoxa thừa hưởng, không làm thêm gì.

  Trên UDP       →  gói có thể mất, có thể đến lệch thứ tự, có thể đến hai lần.
                    Fomoxa giao lên ứng dụng đúng thứ tự nó nhận được, không sửa gì.
```

Transport UDP không phải sắp xếp lại thứ tự gói. Nó cũng không được phép làm. Sắp xếp lại là phải đệm và trì hoãn gói. Độ trễ đó core không lường được.

Ứng dụng cần đúng thứ tự và không mất mát thì có hai lựa chọn. Một: dùng transport kiểu dòng byte - TCP, TLS, WebSocket đã bảo đảm sẵn. Hai: tự lo thứ tự và độ tin cậy ở tầng ứng dụng. Đây là đánh đổi có chủ đích. Trong game, mất một gói vị trí thường tốt hơn chờ truyền lại.

Heartbeat và phát hiện peer chết vẫn chạy trên mọi transport, kể cả UDP. Chúng dựa vào thời gian, không dựa vào số thứ tự gói.

---

## 8. Luồng: tick

Một tick làm đúng bốn việc, theo đúng thứ tự này:

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

Transport chỉ dính vào bước 1 và 2. Bước 3 nằm trọn trong core, không transport nào can thiệp được. Nhờ vậy heartbeat và phát hiện peer chết chạy y hệt nhau trên mọi transport. Không phải viết lại cho từng loại.

---

## 9. Luồng: đóng kết nối

Có ba cách một session kết thúc. Transport tham gia khác nhau ở từng cách:

### 9.1 Ứng dụng chủ động ngắt

```
  ỨNG DỤNG ──"ngắt"──> CORE ──đóng mềm──> TRANSPORT ──> báo phía kia (nếu làm được)
```

Kết nối có cơ chế đóng êm - TCP, WebSocket - thì transport gửi tín hiệu đóng. Kết nối không có - UDP - thì không làm gì cả. Đó là hành vi đúng, không phải thiếu sót.

### 9.2 Phía kia biến mất

Hai tình huống khác nhau về bản chất. Transport phải phân biệt được:

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

Transport phải trả đúng ✖ hoặc ⚠ theo tình huống thật. Cấm gộp hai trạng thái này.

Core có thể chưa dùng đến sự phân biệt đó. Một bản triển khai đơn giản có thể gộp cả hai thành một sự kiện DISCONNECT. Gộp hay không là quyền của core. Core bắt đầu phân biệt ✖ với ⚠ thì transport báo đúng chạy ngay. Không phải sửa. Transport báo sai thì phải xem lại toàn bộ logic lỗi, vào đúng lúc khó nhất: khi hệ thống đã chạy thật. Làm đúng từ đầu gần như không tốn gì. Thư viện mạng nền vốn đã phân biệt hai ca này.

### 9.3 Phía kia im lặng

Không có thủ tục đóng nào cả. Peer ngừng trả lời. Core tự phát hiện bằng đồng hồ ở bước 3 của tick: gửi một probe, quá hạn không thấy trả lời thì tuyên bố peer chết.

Transport không tham gia gì vào việc phát hiện này. Nhờ vậy nó chạy cả trên UDP - nơi không có "kết nối" nào để mất.

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

Cột phải rất ngắn. Đó là chủ ý. Viết một transport mới phải là việc nhỏ.

### Nói rõ về "bỏ qua dữ liệu không phải của session này"

Chỉ áp dụng cho transport không kết nối, thực tế là chỉ UDP. Một điểm cuối UDP nhận gói từ mọi nguồn. Transport UDP phải đối chiếu địa chỉ nguồn với peer đang chạy. Gói lạ thì vứt im lặng. Không báo lỗi, không kết thúc session. Một gói lạ không nói lên điều gì về trạng thái session.

TCP, TLS, WebSocket và QUIC đã gắn sẵn với một peer. Điều kiện này tự thỏa mãn, transport không phải làm gì thêm.

Đây không phải bước kiểm tra bảo mật. Nó không xác thực, không chống giả mạo. Việc hai bên có nói cùng một thứ tiếng hay không do handshake Fomoxa lo, ở core. Kiểm tra riêng của giao thức thuộc về bên chấp nhận kết nối. Ngoài phạm vi tài liệu này.

---

## 11. Muốn thêm một transport mới thì làm gì

Ví dụ: làm một transport WebSocket.

### Bước 1  -  Xác định loại

WebSocket giữ ranh giới gói → kiểu gói. (Làm TLS thì là kiểu dòng byte, và bạn được lớp ráp frame miễn phí.)

### Bước 2  -  Tự lo phần kết nối

Tự mở kết nối WebSocket: phân tích địa chỉ, mở kết nối nền, nâng cấp giao thức, xử lý phản hồi. Core không giúp gì và cũng không cần biết. Kết quả là một đường ống đã thông.

### Bước 3  -  Cung cấp bốn chức năng

- Gửi: nhận một khối byte từ core, gói vào một gói WebSocket nhị phân, đẩy đi. Được thì trả ✔. Không được thì trả ⏸ *ngay*, cấm chờ. Khối byte vượt trần thì trả ⊘; cấm cắt nhỏ, cấm trả ⏸.
- Nhận: có một gói WebSocket trọn vẹn thì đưa cho core. Chưa trọn thì trả ⏸ và tự giữ trong bộ đệm. Gói lớn hơn chỗ core đưa thì trả ⤢ kèm số byte cần. Đóng sạch trả ✖, đứt đột ngột trả ⚠.
- Đóng mềm: gửi frame đóng của WebSocket.
- Đóng hẳn: giải phóng kết nối, bộ đệm, mọi tài nguyên.

> Phải sử dụng kiểu dữ liệu nhị phân; cấm sử dụng kiểu văn bản.
> Các gói WebSocket dạng văn bản yêu cầu nội dung UTF-8 hợp lệ, và các thư viện ở hai đầu kết nối
> có thể kiểm tra tính hợp lệ hoặc chuẩn hóa nội dung đó. Các frame Fomoxa bao gồm các byte nhị phân thô
> mà gần như chắc chắn không phải là UTF-8 hợp lệ; việc gửi chúng qua các gói văn bản sẽ
> làm hỏng dữ liệu hoặc khiến chính thư viện WebSocket từ chối chúng. Quy tắc này
> áp dụng cho bất kỳ transport nào có phân biệt giữa các "kiểu dữ liệu": hãy luôn chọn định dạng nhị phân thô.

### Bước 4  -  Giao cho core

Gói đường ống đã thông thành một transport kiểu gói, đưa vào core, nhận về một session. Nên bọc bước này trong một hàm tiện dụng của thư viện (§4). Ứng dụng không phải thấy khái niệm transport.

Từ đây ứng dụng dùng nó y hệt một session TCP: cùng lệnh gửi, cùng tick, cùng danh sách sự kiện.

### Bước 5  -  Không sửa gì trong core

Phải sửa core để transport mới chạy được là dấu hiệu xấu. Hoặc thiết kế này thiếu, hoặc bạn đặt logic sai tầng. Dừng lại xem lại.

### Việc bạn không phải làm

- Không cần hiểu định dạng trên dây của Fomoxa
- Không cần biết greeting, verdict, POLL hay REPLY là gì
- Không cần cắt hoặc ghép các frame
- Không cần duy trì thứ tự (trừ khi bạn tự gộp nhiều luồng con – §5)
- Không cần xử lý việc truyền lại dữ liệu
- Không cần thực hiện bất kỳ phép đo thời gian nào
- Không cần làm heartbeat
- Không cần sắp xếp lại các gói đến sai thứ tự (§7)

---

## 12. Những điều transport tuyệt đối không được làm

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
| Diễn giải nội dung byte | Sai tầng. Byte đó có thể là handshake, có thể là dữ liệu ứng dụng  -  transport không được phép quan tâm. |
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

Đó là toàn bộ mục đích của thiết kế này.

---

## 14. Bất biến mà mọi bản triển khai phải giữ

Dựng lại Fomoxa trên ngôn ngữ khác thì đây là những điều không được đổi. Vi phạm một điều là mất khả năng nói chuyện với các bản triển khai khác.

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

Kiểm chứng: hai bản triển khai bất kỳ, hai ngôn ngữ bất kỳ, phải nói chuyện được với nhau. Không bên nào cần biết bên kia viết bằng gì.

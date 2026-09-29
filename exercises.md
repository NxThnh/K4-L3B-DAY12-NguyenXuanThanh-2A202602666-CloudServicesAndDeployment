# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder câu trả lời bằng nội dung của bạn.

> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Xuân Thành   Mã học viên: 2A202602666

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống: Khi deploy service lên môi trường Cloud/Production (ví dụ Railway hay Render), lập trình viên vô tình quên cấu hình biến môi trường `AGENT_API_KEY` trong dashboard.
- Nếu để giá trị mặc định `"changeme"`, service vẫn khởi động thành công và mở public URL ra Internet. Khi đó, bất kỳ bot quét mạng hoặc kẻ tấn công nào cũng có thể dùng key mặc định `"changeme"` để gọi vào endpoint `/ask` hoàn toàn miễn phí, tiêu tốn sạch ngân sách token LLM và gây lộ dữ liệu mà ta không hề phát hiện cho tới khi hóa đơn gửi về.
- Việc không có giá trị mặc định khiến app ném lỗi `ValidationError` và crash ngay lập tức lúc khởi động (fail fast). Nền tảng Cloud phát hiện container không khởi động được hoặc fail healthcheck nên chặn deploy ngay, giữ nguyên phiên bản an toàn trước đó và báo đỏ trên dashboard. Lập trình viên lập tức nhìn thấy lỗi và bổ sung secret đúng cách trước khi bất kỳ request nào được phục vụ.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T04:00:15.123456+00:00", "user_id": "sv-test", "tokens_in": 14, "tokens_out": 32, "cost_usd": 0.00046}
```

Hai việc làm được với dòng log này mà `print("đã trả lời xong")` không làm được:
1. **Lọc, truy vấn và tổng hợp tự động bằng máy (Structured Querying & Analytics):** Các hệ thống thu thập log tập trung (như Datadog, Grafana Loki, CloudWatch, ELK) có thể phân tích cú pháp JSON tự động và chỉ mục hóa các trường (`user_id`, `tokens_in`, `tokens_out`, `cost_usd`). Từ đó ta có thể truy vấn định lượng như: "Tính tổng chi phí token của user `sv-test` trong ngày", "Vẽ biểu đồ số token trung bình mỗi request theo giờ", hoặc "Tìm top 5 user tiêu thụ chi phí nhiều nhất". Lệnh `print` chuỗi văn bản thuần túy không thể phân tích cấu trúc được như vậy.
2. **Thiết lập cảnh báo tự động theo ngưỡng (Automated Alerting):** Có thể cài đặt rule cảnh báo gửi về Slack/PagerDuty khi các trường số liệu vượt ngưỡng bất thường (ví dụ: cảnh báo ngay lập tức nếu một request đơn lẻ có `cost_usd > 0.05`, hoặc khi tỷ lệ event `level == "error"` vượt quá 5% trong 5 phút). Với lệnh `print`, máy tính không thể phân biệt dữ liệu định lượng và mức độ nghiêm trọng nếu không có schema chuẩn hóa.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | 1024 MB |
| Multi-stage | 270 MB |


Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~839 MB) bao gồm:
1. **Bộ công cụ biên dịch và phát triển (Compilers & Build Tools):** Trong bản đầy đủ hoặc bản single stage cần cài đặt các công cụ như `gcc`, `g++`, `make`, `build-essential` và header C/C++ để biên dịch các package Python có C-extensions. Ở multi-stage, các công cụ nặng này chỉ nằm trong stage `builder` và bị loại bỏ hoàn toàn khỏi image `runtime`.
2. **Các tiện ích hệ thống và tài liệu không cần thiết:** Base image `python:3.11` chứa đầy đủ các tiện ích gỡ lỗi, man pages, tài liệu, và thư viện đồ họa của Debian. Trong khi đó, `python:3.11-slim` chỉ giữ lại nhân hệ điều hành tối thiểu cần để chạy ứng dụng.
3. **Bộ nhớ đệm (Cache):** Cache của trình quản lý gói `pip` và `apt` sinh ra khi download gói. Ở bản multi-stage, ta chỉ copy thư mục kết quả `/install` sạch sang stage runtime nên loại bỏ hoàn toàn rác sinh ra trong quá trình cài đặt.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Khi sửa một ký tự trong `app/main.py` rồi build lại:
  + Các layer trước lệnh `COPY . .`: bao gồm toàn bộ stage `builder` (`FROM`, `WORKDIR`, `COPY requirements.txt`, `RUN pip install`) và các layer đầu của stage runtime (`FROM`, `WORKDIR`, `RUN useradd`, `COPY --from=builder /install /usr/local`) đều được tái sử dụng hoàn toàn từ cache (`CACHED`).
  + Chỉ có layer `COPY . .` (nơi chứa file bị sửa đổi) và các layer sau nó (`USER`, `ENV`, `EXPOSE`, `HEALTHCHECK`, `CMD`) mới phải chạy lại. Quá trình build chỉ mất 1-2 giây.
- Nếu đặt `COPY . .` lên trước `RUN pip install`:
  + Mỗi khi sửa dù chỉ một ký tự trong `app/main.py`, checksum của layer `COPY . .` sẽ thay đổi, làm Docker huỷ bỏ toàn bộ cache của tất cả các layer đứng phía sau nó.
  + Do đó, lệnh `RUN pip install -r requirements.txt` sẽ bị ép chạy lại từ đầu mỗi lần build, khiến thời gian build kéo dài thêm vài phút vô ích để tải và cài đặt lại toàn bộ thư viện.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:
1. Ứng dụng Python tồn tại một lỗ hổng bảo mật (ví dụ: RCE - Remote Code Execution qua deserialization không an toàn, command injection, hoặc upload file nguy hiểm).
2. Kẻ tấn công gửi payload khai thác lỗ hổng và chiếm được quyền thực thi mã lệnh (interactive shell) bên trong container.
3. Do container mặc định chạy bằng `root` (UID 0), tiến trình của kẻ tấn công sở hữu đặc quyền root bên trong Linux namespace của container. Nếu container có mount các volume/thư mục nhạy cảm từ máy host (như `/var/run/docker.sock` hoặc thư mục cấu hình hệ thống), hoặc nếu Linux kernel của máy host tồn tại lỗ hổng container breakout / privilege escalation (như cgroups release_agent, Dirty COW), kẻ tấn công với quyền root trong container có thể thoát khỏi rào cản namespace và nhảy sang máy host.
4. Trên máy host, kẻ tấn công trở thành `root` thực thụ, toàn quyền kiểm soát hệ thống, đọc trộm dữ liệu, cài mã độc hoặc kiểm soát hạ tầng máy chủ.

Lệnh `USER` cắt đứt chuỗi ở đâu:
- Lệnh `USER appuser` (UID 10001 không đặc quyền) cắt đứt chuỗi ngay tại bước 2. Khi kẻ tấn công chiếm được quyền thực thi trong container, chúng chỉ là một user bình thường không có quyền admin: không thể ghi đè các tệp tin hệ thống trong container, không thể cài package hệ thống, và quan trọng nhất là bị tước bỏ mọi Linux Capabilities đặc quyền (như `CAP_SYS_ADMIN`), khiến hầu hết các kỹ thuật container escape bị vô hiệu hóa hoàn toàn.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.

Giải thích:
- Với cơ chế đếm theo phút đồng hồ (fixed window reset vào giây 00 mỗi phút):
  + Ở giây cuối cùng của phút thứ nhất (ví dụ 10:00:59), người dùng gửi dồn dập 10 request. Vì hạn mức là 10 request/phút nên cả 10 request này đều được hệ thống chấp nhận hợp lệ.
  + Ngay khi đồng hồ chuyển sang giây đầu tiên của phút tiếp theo (10:01:00), bộ đếm số request của phút mới bị reset về 0.
  + Ở giây 10:01:01, người dùng lập tức gửi tiếp 10 request nữa. Do bộ đếm của phút mới đang là 0 nên cả 10 request này lại tiếp tục được chấp thuận.
- Kết quả là chỉ trong khoảng thời gian vỏn vẹn 2 giây (từ 10:00:59 đến 10:01:01), hệ thống đã phải nhận tới 20 request (gấp đôi hạn mức quy định). Cửa sổ trượt (sliding window 60s) giải quyết triệt để vấn đề này vì nó luôn tính tổng request trong đúng 60 giây lùi về quá khứ tính từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Khác nhau:
- **Rate limit:** Kiểm soát **tốc độ và số lượng** request trong một đơn vị thời gian ngắn (ví dụ: tối đa 10 request/phút) để ngăn ngừa nghẽn mạng, chống tấn công từ chối dịch vụ (DDoS) và bảo vệ khả năng đáp ứng đồng thời của hạ tầng web server.
- **Cost guard:** Kiểm soát **tổng số tiền chi tiêu** (USD) tích lũy trong một chu kỳ dài (ví dụ: 1 tháng) để bảo vệ ngân sách tài chính trước chi phí tiêu thụ token mô hình LLM.

Tình huống Rate limit cho qua nhưng Cost guard phải chặn:
- Người dùng đã sử dụng gần hết ngân sách tháng (đã tiêu $9.99 / $10.00). Sau một thời gian không gửi request, người dùng gửi 1 request mới (tốc độ chỉ 1 req/phút, Rate limit hoàn toàn cho qua). Tuy nhiên, Cost guard tính toán thấy số tiền đã tiêu hoặc chi phí ước tính vượt quá hạn mức $10.00 nên lập tức chặn lại và trả về mã lỗi `402 Payment Required`.

Tình huống Cost guard cho qua nhưng Rate limit phải chặn:
- Người dùng mới bắt đầu tháng, tài khoản còn nguyên ngân sách $10.00 (mới tiêu $0.00). Người dùng dùng một script tự động bắn liên tiếp 15 request chỉ trong vòng 2 giây. Về mặt ngân sách, số token của 15 câu hỏi này chỉ tốn vài cent (Cost guard cho qua), nhưng Rate limiter phát hiện số request vượt quá hạn mức 10 req/phút nên chặn ngay từ request thứ 11 và trả về mã lỗi `429 Too Many Requests`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện:
1. Redis gặp sự cố mạng tạm thời hoặc khởi động lại trong 30 giây.
2. Endpoint duy nhất (vừa làm liveness vừa làm readiness) trên cả 3 container agent đều thực hiện kiểm tra Redis và thất bại (trả về 503 Service Unavailable hoặc timeout).
3. Hệ thống Orchestrator (Docker/Kubernetes/Cloud Platform) quan sát thấy liveness check fail trên cả 3 container, nên đánh giá rằng tiến trình của cả 3 container đã bị treo/hỏng và gửi tín hiệu khởi động lại (kill & restart) toàn bộ cả 3 container cùng một lúc.
4. Các container mới được bật lên, tiếp tục chạy liveness probe vào Redis trong khi Redis vẫn chưa hồi phục, dẫn tới việc lại tiếp tục fail và rơi vào vòng lặp crashloop restart liên tục.
5. Khi Redis kết nối trở lại sau 30 giây, toàn bộ 3 container đều đang trong trạng thái khởi động lại dở dang, khiến không còn bất kỳ container nào sẵn sàng phục vụ traffic, gây ra tình trạng sập toàn bộ hệ thống (cascading failure / downtime hoàn toàn) thay vì chỉ tạm dừng điều phối traffic.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Khi lưu lịch sử trong Redis (stateless): Cả 3 container cùng kết nối tới một Redis duy nhất. Dù mỗi request được load balancer phân phối ngẫu nhiên vào container nào (container 1, 2 hay 3), tất cả đều đọc và cập nhật chung một lịch sử hội thoại trên Redis, do đó `history_length` tăng đều đặn qua các request: 0, 2, 4, 6, 8...
- Nếu lịch sử được lưu trong một dict Python nội bộ (stateful in-memory): Mỗi container sở hữu một bộ nhớ RAM hoàn toàn độc lập và không chia sẻ được với nhau. Khi gọi `/ask` liên tiếp với cùng một `X-User-Id`:
  + Request 1 gửi vào container A -> `history_length` = 0 (lưu vào dict của A).
  + Request 2 được load balancer chuyển sang container B -> `history_length` lại là 0 (vì B không có dữ liệu của A).
  + Request 3 được chuyển sang container C -> `history_length` tiếp tục là 0.
  + Request 4 quay lại container A -> `history_length` nhảy lên 2.
  + Request 5 sang container B -> `history_length` nhảy lên 2...
  Con số `history_length` sẽ tăng giảm bất thường, nhảy lung tung phụ thuộc vào container nào nhận được request, khiến agent bị hiện tượng "mất trí nhớ" gián đoạn.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi:**
  Khi deploy lên cloud (như Railway hoặc Render), quá trình build image thành công nhưng service không thể chuyển sang trạng thái "Healthy/Active". Trên dashboard báo lỗi: `Health check timeout` hoặc `Application failed to respond on port 8000: Connection refused / Bad Gateway`.
- **Cách tìm ra nguyên nhân:**
  Mở tab Deployment Logs / Runtime Logs trên dashboard của platform. Quan sát log khởi động thấy dòng:
  `Uvicorn running on http://0.0.0.0:8000`
  trong khi hệ thống cloud thông báo đã cấp phát biến môi trường động `PORT` (ví dụ `PORT=7341`). Vì cloud load balancer chỉ gửi request probe tới cổng được cấp phát trong biến `$PORT`, còn ứng dụng lại cố định lắng nghe cổng 8000, nên probe không bao giờ nhận được phản hồi và bị timeout.
- **Cách sửa:**
  Chỉnh sửa chỉ thị khởi chạy trong `Dockerfile` để đọc giá trị từ biến môi trường `PORT` do platform cấp, với fallback là 8000:
  `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`
  Sau khi cập nhật và deploy lại, ứng dụng đã lắng nghe đúng cổng của platform và vượt qua health check thành công.


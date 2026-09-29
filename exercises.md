# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đỗ Hoàng Quân .............  Mã học viên: 2A202603016................

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> *Khi triển khai lên môi trường Cloud/Staging, lập trình viên có thể vô tình quên thiết lập biến `AGENT_API_KEY` trong trang quản trị (Dashboard). Nếu có giá trị mặc định `"changeme"`, ứng dụng vẫn khởi động âm thầm và kẻ xấu hoặc bot quét có thể dùng khóa mặc định `"changeme"` để gọi API LLM thoải mái, làm cạn kiệt ngân sách hoặc gây hóa đơn hàng nghìn USD mà ta không hề hay biết. Ngược lại, cơ chế fail fast làm container crash ngay lập tức lúc khởi động, orchestrator gửi cảnh báo deploy failed, giúp ta nhận ra và bổ sung secret ngay lập tức trước khi bất kỳ traffic công khai nào chạm tới service.*

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T06:38:11.123456+00:00", "user_id": "sv-test", "tokens_in": 439, "tokens_out": 47, "cost_usd": 9.405e-05}`
>
> Hai việc làm được với log JSON mà print text thường không làm được:
> 1. **Lọc và cảnh báo tự động trên hệ thống quản lý log (Datadog/CloudWatch/ELK):** Máy có thể tự động parse các trường có cấu trúc để thiết lập bộ lọc (ví dụ: truy vấn toàn bộ sự kiện có `level == "error"` hoặc tìm tất cả request của một `user_id` nhất định) mà không cần viết regex phức tạp.
> 2. **Tổng hợp số liệu định lượng (Metric aggregation):** Hệ thống có thể tự động đọc và tính tổng chi phí (`cost_usd`), tổng số token (`tokens_in`, `tokens_out`) theo từng user hoặc theo khung giờ để vẽ biểu đồ giám sát chi tiêu theo thời gian thực.

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
| 1 stage (bản đầu) | ... MB |
| Multi-stage | ... MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần chênh lệch ~800 MB bao gồm:
> 1. Base image đầy đủ (`python:3.11`) chứa cả hệ điều hành Debian với các công cụ build C/C++, compiler (`gcc`, `g++`, `make`), header dev không cần thiết ở runtime.
> 2. Cache của package manager (`pip cache`, `apt cache`) và các artifact trung gian sinh ra trong quá trình biên dịch thư viện ở stage `builder`. Với multi-stage, stage runtime chỉ copy thư mục virtualenv đã biên dịch xong sang base `python:3.11-slim`, loại bỏ hoàn toàn các công cụ thừa.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> - Các layer được dùng lại từ cache: Base image, cài đặt `curl`, copy `requirements.txt`, cài đặt virtualenv (`pip install`), copy `/opt/venv`, và lệnh tạo user `RUN useradd`.
> - Layer phải chạy lại: `COPY --chown=appuser:appuser . .` (và các layer sau đó nếu có).
> - Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi khi sửa bất kỳ ký tự nào trong code, cache của layer copy code sẽ bị vô hiệu hóa (invalidated). Docker sẽ bị ép phải chạy lại toàn bộ bước tải và cài đặt thư viện (`pip install`), làm thời gian build tốn thêm vài phút thay vì chỉ mất 1-2 giây.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện:
> 1. Ứng dụng Python có lỗ hổng (như Remote Code Execution qua deserialize, command injection, hoặc thư viện bên thứ 3).
> 2. Kẻ tấn công kích hoạt lỗ hổng để chạy shell/lệnh tùy ý bên trong container.
> 3. Vì container chạy mặc định bằng root (UID 0), kẻ tấn công chiếm toàn quyền root trong container.
> 4. Kẻ tấn công sử dụng các kỹ thuật container escape (lợi dụng lỗ hổng nhân Linux, mount Docker socket hoặc cấu hình sai cgroups/capabilities) để thoát ra ngoài máy host. Do UID 0 trong container map thẳng tới UID 0 (root) trên máy host, kẻ tấn công chiếm luôn quyền điều khiển cao nhất của máy chủ host.
>
> Lệnh `USER appuser` cắt đứt chuỗi ở **bước 3**: Khi tiến trình chạy dưới quyền non-root (UID 1000), kẻ tấn công khi đột nhập vào container chỉ có quyền hạn tối thiểu, không thể chỉnh sửa file hệ thống cốt lõi và không thể thực hiện các quyền hạn đặc biệt (capabilities) cần thiết để khai thác container escape ra máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.
> Cách đạt được:
> - Tại giây `10:00:59` (giây cuối cùng của phút thứ nhất), người dùng gửi 10 request. Hệ thống tính là 10/10 request của phút đó nên cho qua toàn bộ.
> - Ngay tại giây tiếp theo `10:01:00` (giây đầu tiên của phút thứ hai), bộ đếm phút đồng hồ được reset về 0. Người dùng lập tức gửi tiếp 10 request và hệ thống tiếp tục cho qua.
> Kết quả là từ `10:00:59` đến `10:01:01` (chỉ trong 2 giây), hệ thống phải hứng chịu 20 request (gấp đôi hạn mức), gây nguy cơ sập server. Thuật toán sliding window 60 giây giải quyết được vấn đề này vì nó luôn tính tổng request trong 60 giây trượt lùi từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Khác nhau:
> - **Rate limit:** Kiểm soát **tần suất** (số lượng request trong một đơn vị thời gian ngắn, ví dụ: 10 req/phút) nhằm bảo vệ tính sẵn sàng của hạ tầng, chống nghẽn và DoS.
> - **Cost guard:** Kiểm soát **chi phí tài chính** (tổng số tiền/token tiêu thụ trong một chu kỳ dài, ví dụ: $10.0/tháng) nhằm bảo vệ ngân sách tài chính của chủ sở hữu.
>
> Tình huống minh họa:
> - **Rate limit cho qua nhưng Cost guard chặn:** Một người dùng cả tháng không gọi API, hôm nay chỉ gửi đúng 1 request trong phút (hoàn toàn hợp lệ theo rate limit 10 req/phút). Tuy nhiên, request này chứa context cực lớn tiêu tốn token vượt quá ngân sách tháng còn lại của họ -> Cost guard chặn lại với mã lỗi 402 Payment Required.
> - **Cost guard cho qua nhưng Rate limit chặn:** Một người dùng mới còn nguyên hạn mức $10.0 trong tháng, nhưng dùng script gửi dồn dập 20 request chỉ trong vòng 5 giây -> Cost guard thấy chưa hết tiền nhưng Rate limit sẽ chặn từ request thứ 11 với mã lỗi 429 Too Many Requests để tránh nghẽn server.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện:
> 1. Redis gặp sự cố mạng hoặc khởi động lại, không phản hồi trong 30 giây.
> 2. Bộ kiểm tra liveness probe của Orchestrator (K8s/Docker) gọi định kỳ vào endpoint gộp chung của cả 3 container.
> 3. Vì endpoint này kiểm tra Redis và Redis đang tạch, cả 3 container đều đồng loạt trả về lỗi 503 / thất bại.
> 4. Orchestrator nhận định sai rằng mã nguồn của toàn bộ 3 container bị treo/chết, và lập tức gửi tín hiệu SIGKILL/restart toàn bộ 3 container cùng lúc.
> 5. Khi 3 container mới khởi động lại, Redis vẫn chưa online kịp, container lại tiếp tục fail healthcheck và tiếp tục bị restart vòng lặp vô tận (restart cascade / crash loop backoff).
> 6. Toàn bộ hệ thống rơi vào trạng thái sập hoàn toàn thay vì chỉ tạm ngưng nhận request từ Load Balancer để chờ Redis kết nối lại.


---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Nếu lịch sử lưu trong dict Python (in-memory):
Mỗi container sở hữu một vùng nhớ RAM độc lập. Khi gọi `/ask`, Load Balancer sẽ phân phối các request luân phiên ngẫu nhiên giữa 3 container (A, B, C). Do đó, `history_length` sẽ **nhảy lung tung không theo thứ tự** (ví dụ câu 1 vào A thì A có độ dài 1; câu 2 vào B thì B chưa có gì nên độ dài là 0; câu 3 vào C lại là 0; câu 4 quay lại A mới tăng lên 2...). Agent sẽ bị "mất trí nhớ", không thể hiểu ngữ cảnh câu hỏi trước đó của cùng một user.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> - **Thông báo lỗi:** Khi container khởi động bị crash ngay lập tức với lỗi:
  `starlette.routing.Lifespan: NotImplementedError: TODO (CP4): cài đặt install`
> - **Cách tìm ra nguyên nhân:** Chạy lệnh `docker compose logs agent` (hoặc kiểm tra tab Logs trên Render), phát hiện luồng khởi động FastAPI gọi hook `lifespan` và kích hoạt hàm `lifecycle.install()`, nhưng lúc đó code trong `app/lifecycle.py` vẫn giữ `raise NotImplementedError`.
> - **Cách sửa:** Cài đặt đầy đủ logic đăng ký signal `SIGTERM`/`SIGINT` và lưu handler cũ trong hàm `install()`, sau đó chạy lại lệnh `docker compose up -d --build` (hoặc push commit lên GitHub để Render tự build lại image mới).
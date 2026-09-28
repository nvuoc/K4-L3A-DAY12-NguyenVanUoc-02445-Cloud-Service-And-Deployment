# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời vào từng câu hỏi bên dưới.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Văn Ước  Mã học viên: 2A202602445

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu để giá trị mặc định `agent_api_key = "changeme"`, khi deploy service lên môi trường cloud (như Railway hoặc Render) mà kỹ sư quên khai báo biến `AGENT_API_KEY` trong dashboard, ứng dụng vẫn khởi động thành công và báo trạng thái healthy. Lúc này, API `/ask` mở ra cho toàn bộ Internet và bất kỳ ai hoặc bot tự động nào dò quét với khóa mặc định `"changeme"` đều có thể gọi vào agent. Hậu quả là tài nguyên LLM bị khai thác trái phép và tài khoản phát sinh hàng nghìn USD chi phí mà đội ngũ phát triển không hề hay biết cho tới khi nhận hóa đơn cuối tháng.
Ngược lại, khi không có giá trị mặc định (`agent_api_key: str`), theo nguyên tắc Fail-Fast của 12-Factor App, thư viện Pydantic sẽ ném ngay lỗi `ValidationError` lúc container vừa khởi chạy. Quá trình deploy lập tức thất bại và hiển thị cảnh báo đỏ trên dashboard của nền tảng khi kỹ sư vẫn đang theo dõi quá trình triển khai, giúp phát hiện và bổ sung biến môi trường ngay lập tức trước khi bất kỳ request nào từ bên ngoài có thể lọt vào.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T08:45:12.345678+00:00", "user_id": "sv-test", "tokens_in": 12, "tokens_out": 45, "cost_usd": 0.0000288}
```

Hai việc làm được với log có cấu trúc JSON:
1. **Lọc, tổng hợp và thống kê định lượng theo trường dữ liệu (Metrics & Aggregation):** Các công cụ thu thập log tập trung (như Datadog, Grafana Loki, CloudWatch) có thể tự động parse các key như `user_id`, `tokens_in`, `tokens_out`, `cost_usd` để tính toán tổng chi phí tiêu thụ theo giờ/ngày, thống kê user nào dùng nhiều token nhất, hoặc vẽ biểu đồ chi phí thời gian thực. Lệnh `print("đã trả lời xong")` chỉ là chuỗi văn bản thuần túy, máy móc không thể tự động tổng hợp hay tính toán số liệu nếu không viết regex phức tạp và dễ vỡ.
2. **Cảnh báo và giám sát sự cố tự động (Alerting & Anomaly Detection):** Dựa vào trường `level` và các metadata chuẩn hóa, hệ thống giám sát có thể kích hoạt cảnh báo tức thời gửi tới Slack/PagerDuty (ví dụ: khi tỷ lệ `level == "error"` vượt quá 5% trong vòng 5 phút, hoặc khi `cost_usd` của một request đơn lẻ vượt ngưỡng $0.10). Chuỗi log dạng print không hỗ trợ phân loại mức độ nghiêm trọng hay lọc theo ngữ cảnh sự cố.

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
| Multi-stage | 185 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~839 MB) bao gồm:
1. **Trình biên dịch và thư viện phát triển hệ điều hành:** Base image `python:3.11` đầy đủ dựa trên Debian bản chuẩn, chứa rất nhiều công cụ biên dịch (`gcc`, `g++`, `make`), header C/C++ (`build-essential`), man pages và các gói tiện ích không cần thiết cho môi trường chạy production; trong khi `python:3.11-slim` đã được lược bỏ toàn bộ các thành phần này.
2. **Cache của trình quản lý gói:** Khi chạy `pip install` ở bản 1 stage thông thường, pip sẽ lưu lại các file bánh xe (.whl) và gói tải về trong thư mục cache `~/.cache/pip`. Ở bản multi-stage, ta dùng cờ `--no-cache-dir`.
3. **Artifacts trung gian của giai đoạn build:** Ở mô hình multi-stage, các file tạm, dependency phục vụ biên dịch đều được giữ lại ở stage `builder` và bị loại bỏ hoàn toàn, chỉ có các package đã cài đặt trong `/install` được copy sang stage `runtime`, giúp image cuối cùng cực kỳ tinh gọn.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- **Với Dockerfile hiện tại của bài lab:**
  - Các layer dùng lại từ cache: `FROM python:3.11-slim AS builder`, `WORKDIR /app`, `COPY requirements.txt .`, `RUN pip install ...`, `FROM python:3.11-slim AS runtime`, `COPY --from=builder /install /usr/local`, và `USER appuser`. Do `requirements.txt` không thay đổi, Docker giữ nguyên cache của bước cài đặt thư viện nặng nề nhất.
  - Các layer phải chạy lại: Bắt đầu từ `COPY app ./app` (vì thư mục `app/` có file `main.py` bị sửa ký tự), kéo theo các layer bên dưới như `COPY utils ./utils`, cấu hình `HEALTHCHECK` và `CMD`. Quá trình build lại diễn ra gần như tức thì (dưới 1 giây).
- **Nếu đặt `COPY . .` lên trước `RUN pip install`:**
  - Docker tuân thủ cơ chế cache theo từng layer từ trên xuống dưới. Nếu `COPY . .` nằm trước, khi sửa 1 ký tự trong `main.py`, checksum của layer `COPY . .` sẽ bị thay đổi và làm mất cache (cache invalidated).
  - Toàn bộ các layer phía sau nó, bao gồm cả `RUN pip install -r requirements.txt`, sẽ bắt buộc phải chạy lại từ đầu. Kết quả là mỗi lần sửa code nhỏ, Docker lại phải tải và cài đặt lại toàn bộ các thư viện Python qua mạng, khiến thời gian build kéo dài thêm vài phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- **Chuỗi sự kiện dẫn tới chiếm quyền máy host:**
  1. Ứng dụng Python tồn tại lỗ hổng bảo mật (ví dụ: Command Injection qua `subprocess`, Remote Code Execution qua `eval`, hoặc thư viện bên thứ ba chưa vá lỗ hổng). Kẻ tấn công gửi payload độc hại để thực thi lệnh tùy ý.
  2. Vì container chạy mặc định với user `root` (UID 0), shell được tạo ra bởi payload có toàn quyền root bên trong container (có thể sửa đổi file hệ thống, cài thêm công cụ tấn công).
  3. Do container chia sẻ chung nhân Linux Kernel với máy host, nếu máy chủ có lỗ hổng kernel (như Dirty COW, cgroup release_agent escape), hoặc socket Docker (`/var/run/docker.sock`) vô tình bị mount vào container, tiến trình root (UID 0) từ container có thể vượt rào (container escape) ra máy host. Vì UID 0 trong container tương ứng trực tiếp với UID 0 trên host (nếu không bật user namespaces), kẻ tấn công chính thức trở thành root của máy chủ vật lý và kiểm soát toàn bộ hạ tầng.
- **Lệnh `USER appuser` cắt đứt chuỗi ở đâu:**
  Lệnh `USER appuser` chuyển tiến trình ứng dụng sang một người dùng thông thường không có đặc quyền (UID 10001). Khi kẻ tấn công khai thác được RCE trong app Python, shell chúng nhận được chỉ có quyền của `appuser`:
  - Không thể chỉnh sửa các file hệ điều hành của container (`/bin`, `/usr`, `/etc`).
  - Bị tước bỏ hầu hết Linux Capabilities (`CAP_SYS_ADMIN`, `CAP_NET_ADMIN`,...).
  - Không thể thực hiện các cuộc tấn công vượt rào container (escape) vốn đòi hỏi quyền root của UID 0, từ đó bảo vệ an toàn cho máy chủ host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- **Số request tối đa trong 2 giây liên tiếp:** **20 request**.
- **Giải thích:**
  Với cơ chế đếm theo phút đồng hồ (Fixed Window Counter), bộ đếm tự động reset về 0 tại mốc giây 00 của mỗi phút (ví dụ 10:00:00, 10:01:00):
  - Người dùng gửi dồn 10 request vào giây cuối cùng của phút thứ nhất: **10:00:59**. Hệ thống kiểm tra thấy trong phút 10:00 người này mới gọi 10/10 request nên cho qua toàn bộ.
  - Ngay 1 giây sau đó, tại thời điểm **10:01:00**, hệ thống bước sang phút mới và reset bộ đếm về 0. Người dùng lập tức gửi tiếp 10 request. Hệ thống tính đây là 10 request của phút 10:01 nên tiếp tục cho qua.
  - Như vậy, trong khoảng thời gian chỉ vỏn vẹn **2 giây** (từ 10:00:59 đến 10:01:00), hệ thống đã phải xử lý tới **20 request** liên tiếp (gấp 2 lần hạn mức cho phép), có thể gây nghẽn dịch vụ.
  - Cơ chế Sliding Window (cửa sổ trượt) giải quyết triệt để lỗi này bằng cách luôn tính chính xác số request trong khoảng thời gian `[now - 60, now]`. Tại giây 10:01:00, cửa sổ trượt nhìn lại 60 giây trước và thấy 10 request của giây 10:00:59 vẫn nằm trong cửa sổ, do đó sẽ lập tức từ chối và trả về HTTP 429.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- **Khác biệt cốt lõi:**
  - **Rate Limit:** Giới hạn theo **tần suất / số lượng request trong một đơn vị thời gian ngắn** (ví dụ: tối đa 10 request/phút) để bảo vệ hạ tầng máy chủ, chống cạn kiệt tài nguyên mạng và tấn công từ chối dịch vụ (DoS).
  - **Cost Guard:** Giới hạn theo **tổng chi phí tài chính tích lũy trong một chu kỳ dài** (ví dụ: tối đa 10.0 USD/tháng) dựa trên số lượng token in/out của mô hình AI, nhằm bảo vệ ngân sách doanh nghiệp khỏi việc bị tiêu xài ngoài tầm kiểm soát.
- **Tình huống Rate Limit cho qua nhưng Cost Guard chặn:**
  - Người dùng gửi 1 request duy nhất trong cả ngày (hoàn toàn hợp lệ theo hạn mức 10 request/phút của Rate Limit). Tuy nhiên, người này đã tích lũy chi tiêu đạt $9.98 / $10.00 ngân sách tháng, và câu hỏi mới kèm prompt quá dài dự tính tốn thêm $0.05. Cost Guard phát hiện tổng chi phí sẽ vượt quá $10.00 nên lập tức chặn và trả về **HTTP 402 Payment Required**, dù Rate Limit không bị vi phạm.
- **Tình huống Cost Guard cho qua nhưng Rate Limit chặn:**
  - Vào ngày đầu tiên của tháng mới, người dùng mới tiêu $0.00 (còn nguyên ngân sách $10.00). Người này viết script gửi 15 câu hỏi liên tiếp "Xin chào" trong 5 giây. Vì mỗi câu hỏi cực ngắn chỉ tốn ~$0.00002 (tổng 15 câu chưa tới $0.001, hoàn toàn nằm trong ngân sách của Cost Guard), nhưng do tần suất 15 request/5 giây đã vượt quá ngưỡng 10 request/phút, Rate Limit sẽ lập tức chặn từ request thứ 11 và trả về **HTTP 429 Too Many Requests**.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự các sự kiện diễn ra:
1. **Redis mất kết nối trong 30 giây** (do sự cố mạng, bảo trì, hoặc Redis khởi động lại).
2. **Endpoint probe của cả 3 container agent đều trả về 503 hoặc timeout** vì nó phụ thuộc trực tiếp vào trạng thái kết nối của Redis.
3. **Orchestrator hiểu nhầm đây là sự cố Liveness**: Hệ thống điều phối (Docker Compose, Kubernetes, Railway) cho rằng process của các container đã chết hoặc bị treo, do đó lập tức **kill và restart đồng loạt cả 3 container**.
4. **Vòng lặp khởi động thất bại liên hoàn (Cascading Failure / CrashLoopBackOff)**: Các container sau khi restart khởi động lại, tiếp tục gọi health check kiểm tra Redis. Vì Redis vẫn chưa kết nối lại được, health check tiếp tục fail, khiến orchestrator tiếp tục kill và restart chúng nhiều lần.
5. **Toàn hệ thống ngừng trệ hoàn toàn (Total Outage)**: Cụm service không còn container nào ở trạng thái sẵn sàng phục vụ. Khi Redis phục hồi xong sau 30 giây, hệ thống vẫn phải mất thêm thời gian chờ các container hoàn tất chu kỳ restart, biến một sự cố gián đoạn phụ thuộc nhỏ thành thảm họa sập toàn bộ dịch vụ.
- **Quy tắc đúng:** `/health` (liveness) chỉ kiểm tra process bản thân container có còn chạy không (để restart nếu bị deadlock). `/ready` (readiness) mới kiểm tra phụ thuộc Redis; khi Redis chết, Load Balancer chỉ tạm thời ngưng chuyển traffic tới container mà không hề restart container.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- **Khi lưu trong Redis (Stateless - Thiết kế đúng):**
  Cả 3 container agent đều truy cập vào cùng một kho dữ liệu Redis tập trung. Bất kể request được Load Balancer điều phối vào container nào, dữ liệu lịch sử vẫn được lấy ra đầy đủ. Kết quả `history_length` trong response sẽ tăng đều đặn và tuyến tính sau mỗi lượt hỏi đáp: **0, 2, 4, 6, 8...**
- **Nếu lưu trong biến dict Python trong bộ nhớ RAM của process (Stateful):**
  Mỗi container chạy trên một tiến trình độc lập và sở hữu một vùng nhớ RAM riêng. Khi Load Balancer phân phối các request theo thuật toán Round-Robin hoặc Least-Connections luân phiên qua 3 container (A, B, C):
  - Request 1 vào A: A lưu vào RAM của A → trả về `history_length = 0`.
  - Request 2 vào B: B kiểm tra RAM của B thấy trống → trả về `history_length = 0` (B không biết gì về câu hỏi ở A).
  - Request 3 vào C: C kiểm tra RAM của C thấy trống → trả về `history_length = 0`.
  - Request 4 vào A: A thấy lịch sử câu 1 của mình → trả về `history_length = 2`.
  - Request 5 vào B: B thấy lịch sử câu 2 của mình → trả về `history_length = 2`.
  Kết quả là `history_length` sẽ nhảy lộn xộn, ngắt quãng (ví dụ: 0, 0, 0, 2, 2, 2, 4...) và AI agent sẽ bị "mất trí nhớ" ngẫu nhiên tùy thuộc vào việc request rơi trúng container nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi gặp phải:**
  `Health check failed: connection refused or timeout on port 8000. Container failed to start.`
- **Cách tìm ra nguyên nhân:**
  Mở mục Deployment Logs trên trang quản lý của nền tảng Cloud (Render/Railway), nhận thấy nền tảng tự động cấp phát một cổng động ngẫu nhiên thông qua biến môi trường `$PORT` (ví dụ `PORT=10000` trên Render), đồng thời Reverse Proxy của cloud chỉ định tuyến lưu lượng kiểm tra tới cổng `$PORT` này. Trong khi đó, file cấu hình ban đầu chạy lệnh Uvicorn cố định cứng cờ `--port 8000`, khiến container lắng nghe ở cổng 8000 trong khi proxy của cloud lại gọi vào cổng khác, dẫn tới health check bị timeout và nền tảng hủy bỏ lần deploy.
- **Cách khắc phục:**
  1. Trong `Dockerfile`, sửa lệnh `CMD` để ưu tiên đọc cổng từ biến môi trường `$PORT` do nền tảng cung cấp, đồng thời giữ fallback 8000 cho môi trường local:
     `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`
  2. Trong `app/config.py`, đảm bảo trường `port: int = 8000` kế thừa từ `BaseSettings` để tự động map với biến môi trường `PORT`.
  3. Đảm bảo cấu hình Uvicorn lắng nghe trên host `0.0.0.0` (thay vì `127.0.0.1` hay `localhost`) để các request từ proxy bên ngoài container có thể đi vào được.

# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
Cách trả lời: thay các dòng câu trả lời mẫu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đinh Kim Thái  Mã học viên: 2A202602417

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống: Khi deploy dịch vụ lên môi trường Cloud, nếu nhà phát triển quên cài đặt biến môi trường `AGENT_API_KEY` trên Dashboard Variables:
- Nếu để mặc định `"changeme"`: Ứng dụng vẫn khởi động bình thường. Nhà phát triển nhầm tưởng hệ thống đã hoạt động an toàn, nhưng thực tế kẻ tấn công hoặc bot quét tự động có thể dùng key mặc định `"changeme"` để gọi endpoint `/ask`, liên tục làm hao phí tiền token LLM và tài nguyên máy chủ mà không bị phát hiện cho tới khi nhận hóa đơn.
- Nếu "chết sớm" (Fail Fast): App sẽ lập tức ngắt tiến trình ngay lúc khởi động với lỗi `ValidationError`. Dashboard báo deploy thất bại ngay lập tức, ngăn không cho service nhận traffic sai và buộc nhà phát triển phải bổ sung `AGENT_API_KEY` hợp lệ trước khi đưa vào hoạt động.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T12:15:30.123456+00:00", "user_id": "sv-test", "tokens_in": 15, "tokens_out": 42, "cost_usd": 0.00015}`

Hai việc làm được với log JSON mà print thông thường không làm được:
1. Lọc và truy vấn tự động bằng các công cụ quản lý log (như Datadog, ELK, Grafana Loki): Máy tính có thể parse các trường JSON để lọc chính xác lịch sử hoạt động của từng `user_id` hoặc tính tổng số token/chi phí `cost_usd` theo ngày mà không cần viết script cắt chuỗi thủ công.
2. Tự động kích hoạt cảnh báo (Alerting): Các hệ thống giám sát có thể tạo rule cảnh báo tự động khi phát hiện các trường có `"level": "error"` hoặc `"cost_usd"` vượt ngưỡng an toàn để tự động gửi thông báo qua Slack/PagerDuty cho đội ngũ kỹ thuật.

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
| 1 stage (bản đầu) | 1020 MB |
| Multi-stage | 210 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~810 MB) bao gồm các trình biên dịch (GCC, g++), các thư viện C/C++ header, công cụ build (build-essentials, git, documentation) và các gói phụ trợ hệ điều hành đi kèm trong base image `python:3.11` đầy đủ. Bản Multi-stage đã loại bỏ toàn bộ các công cụ biên dịch này ở stage 1 và chỉ copy các file bytecode/thư viện đã build xong sang image runtime `python:3.11-slim` tối giản.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại: Khi chỉ sửa 1 ký tự trong `app/main.py`, các layer từ `FROM`, `WORKDIR`, `COPY requirements.txt .` đến `RUN pip install` đều được dùng lại từ Docker Cache (do file `requirements.txt` không đổi). Chỉ có layer `COPY app /app/app` và các lệnh phía sau phải chạy lại.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi khi thay đổi dù chỉ 1 ký tự code, layer `COPY . .` sẽ làm mất hiệu lực cache, khiến Docker buộc phải chạy lại toàn bộ lệnh `RUN pip install` từ đầu, tốn rất nhiều thời gian tải lại thư viện mỗi lần build.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện leo quyền:
  1. Code Python có lỗ hổng RCE (Remote Code Execution).
  2. Kẻ tấn công lợi dụng lỗ hổng để thực thi lệnh shell bên trong container.
  3. Vì container chạy mặc định bằng `root`, tiến trình shell của kẻ tấn công có toàn quyền root bên trong container.
  4. Kẻ tấn công khai thác lỗ hổng thoát khỏi container (container escape) để leo lên chiếm quyền `root` của chính máy host.
- Lệnh `USER appuser` cắt đứt chuỗi ở bước 3: Bằng cách chuyển sang một user thường không có đặc quyền admin, khi kẻ tấn công thực thi được lệnh shell ở bước 2, chúng chỉ có quyền hạn bị giới hạn của `appuser`, không thể sửa file hệ thống container và bị chặn hoàn toàn khi cố gắng thực hiện hành vi leo quyền lên máy host ở bước 4.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- Số request tối đa trong 2 giây liên tiếp: 20 request.
- Cách đạt được: Người dùng gửi 10 request vào 2 giây cuối của phút thứ 1 (từ giây 58 đến giây 59). Lúc này hạn mức 10 req/phút của phút 1 đã vừa hết. Ngay ở giây 00 của phút thứ 2, bộ đếm theo phút đồng hồ reset về 0, người dùng gửi tiếp liền 10 request ở 2 giây đầu của phút thứ 2 (từ giây 00 đến giây 01). Kết quả là trong khoảng thời gian chỉ 4 giây liên tiếp, hệ thống cho phép tổng cộng 20 request đi qua. Thuật toán Sliding Window 60s khắc phục điều này bằng cách tính tổng request trong đúng 60 giây gần nhất và chặn ngay request thứ 11.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Khác nhau: Rate Limit quản lý tần suất/số lượng request trong thời gian ngắn (ví dụ: 10 req/phút) để chống DDoS và quá tải server. Cost Guard quản lý tổng ngân sách tài chính (ví dụ: $10.0/tháng) để bảo vệ ví tiền của chủ ứng dụng.
- Rate Limit cho qua nhưng Cost Guard chặn: User chỉ gửi 1 request trong phút (Rate limit 1/10 -> cho qua), nhưng request đó yêu cầu prompt dài 200,000 token làm tổng chi phí tháng của user vượt $10.0 -> Cost Guard chặn và trả về HTTP 402.
- Cost Guard cho qua nhưng Rate Limit chặn: User mới tiêu $0.01 / $10.0 ngân sách (Cost Guard -> cho qua), nhưng gửi 15 request liên tục chỉ trong 3 giây -> Rate Limit lập tức chặn và trả về HTTP 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

1. Giây 0: Mạng tới Redis bị chập chờn / ngắt kết nối trong 30 giây.
2. Giây 10: Endpoint chung gọi kiểm tra Redis thất bại và trả về lỗi 503.
3. Orchestrator hiểu lầm tiến trình Python của container bị treo/chết (Liveness failure) nên tiến hành tiêu diệt (kill) và restart lại toàn bộ 3 container.
4. Giây 15-25: 3 container khởi động lại nhưng Redis vẫn đang ngắt kết nối -> Probe tiếp tục fail -> Orchestrator lại restart container lần nữa (tạo thành vòng lặp crash loop).
5. Kết quả: Việc restart ứng dụng không sửa được lỗi Redis nhưng gây lãng phí CPU, mất RAM cache và gián đoạn dịch vụ. Việc tách riêng giúp `/health` giữ container không bị restart lãng phí, còn `/ready` báo 503 để Load Balancer tạm dừng gửi request.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lưu trong dict Python (Stateful trong RAM), con số `history_length` sẽ trồi sụt thất thường và nhảy loạn lên (ví dụ: 0 -> 0 -> 2 -> 0 -> 2...) tùy thuộc vào việc Load Balancer điều hướng request đến container A, B hay C, vì mỗi container giữ một dict riêng trong RAM của nó và không thể nhìn thấy lịch sử của container khác. Khi lưu vào Redis (Stateless), con số tăng đều đặn `0 -> 2 -> 4 -> 6...` trên mọi container.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- Thông báo lỗi: `HTTP/1.1 502 Bad Gateway - Application failed to respond` khi gọi URL công khai trên Cloud.
- Nguyên nhân: Vào tab Deployments / View logs trên Dashboard của Cloud xem log khởi động và phát hiện lỗi `pydantic_core._pydantic_core.ValidationError: 1 validation error for Settings agent_api_key Field required`. Nguyên nhân do biến `AGENT_API_KEY` chưa được thêm vào phần Variables của Cloud Dashboard, kích hoạt cơ chế Fail Fast làm ứng dụng tự dừng ngay khi start up.
- Cách sửa: Vào tab Variables của dịch vụ trên Dashboard Cloud, thêm biến `AGENT_API_KEY` với giá trị secret key hợp lệ, sau đó bấm Deploy/Save. App khởi động lại thành công và `/health` trả về 200 OK.

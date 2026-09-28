# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời chi tiết vào bên dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Đức Phát  Mã học viên: 2A202602753

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu ta đặt giá trị mặc định là `"changeme"` hoặc một chuỗi placeholder, khi deploy lên Cloud (hoặc môi trường staging/production), nếu ta quên cấu hình biến môi trường `AGENT_API_KEY`, ứng dụng vẫn sẽ khởi động thành công và báo trạng thái "Running/Healthy". Khi đó, bất kỳ kẻ tấn công hoặc bot quét Internet nào sử dụng khóa mặc định `"changeme"` đều có thể thoải mái gọi endpoint `/ask` của ta, tiêu tốn hạn ngạch và làm phát sinh chi phí LLM khổng lồ mà ta không hề hay biết cho đến khi nhận hóa đơn. Ngược lại, việc không đặt giá trị mặc định (Fail Fast) khiến Pydantic ném `ValidationError` và crash ngay lúc khởi động (deploy fail), buộc lập trình viên phải nhận ra ngay lập tức trên dashboard/log và cấu hình biến môi trường trước khi bất kỳ request nào được tiếp nhận.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T10:23:27.623402+00:00", "user_id": "sv01", "tokens_in": 3, "tokens_out": 35, "cost_usd": 2.145e-05}`

Hai việc làm được với dòng log JSON cấu trúc này:
1. **Lọc, truy vấn và tổng hợp số liệu định lượng (structured query/aggregation)**: Các hệ thống thu thập log tập trung (như CloudWatch, Datadog, Grafana Loki, ELK) có thể phân tích cú pháp các trường số để tính tổng chi phí `SUM(cost_usd)`, đếm lượng token trung bình tiêu thụ, hoặc nhóm theo `user_id` để thống kê người dùng tiêu tốn nhiều tài nguyên nhất mà không cần viết các biểu thức chính quy (regex) phức tạp và dễ vỡ.
2. **Thiết lập hệ thống cảnh báo tự động theo thời gian thực (real-time metric alerting)**: Có thể dễ dàng cấu hình cảnh báo tự động khi trường `level == "error"` vượt ngưỡng cho phép (ví dụ: > 5% trong 5 phút), hoặc khi `cost_usd` của một truy vấn vượt ngưỡng ngân sách đột biến để tự động tạm khóa API key nhằm ngăn chặn sự cố tràn chi phí.

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
| 1 stage (bản đầu) | 1050 MB |
| Multi-stage | 297 MB (content size: 64 MB) |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~750 MB) bao gồm:
1. Toàn bộ chuỗi công cụ biên dịch C/C++ và các gói phát triển hệ điều hành (`build-essential`, `gcc`, `g++`, `make`, `python3-dev`, các thư viện liên kết tĩnh `.a`, header `.h`) cần thiết khi biên dịch thư viện C-extension nhưng hoàn toàn dư thừa trong quá trình chạy mã Python ở môi trường production.
2. Các gói phần mềm, tài liệu man pages, cache của trình quản lý gói hệ điều hành `apt` của Debian đầy đủ (so với `python:3.11-slim`).
3. Bộ nhớ đệm (cache) khi tải các wheel của `pip` (`~/.cache/pip`) và các tệp build trung gian. Trong Dockerfile multi-stage, ta dùng cờ `--no-cache-dir` và chỉ copy đúng thư mục kết quả `/install` sang stage runtime tối giản.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại: Docker cache lại toàn bộ các layer phía trước bao gồm base image, thiết lập thư mục làm việc `WORKDIR`, lệnh `COPY requirements.txt .`, và lệnh tốn thời gian nhất là `RUN pip install ...` (toàn bộ các layer này hiển thị `CACHED`). Chỉ từ layer `COPY app ./app` trở đi mới bị thay đổi và phải thực thi lại. Do đó, quá trình build lại diễn ra tức thì (chưa tới 1 giây).
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi khi chỉnh sửa dù chỉ một ký tự trong `app/main.py`, mã băm checksum của ngữ cảnh thư mục sẽ thay đổi, làm vô hiệu hóa (cache bust) layer `COPY . .` và toàn bộ các layer tiếp theo phía sau nó. Hậu quả là Docker buộc phải chạy lại toàn bộ bước tải và cài đặt dependencies `RUN pip install` từ đầu qua mạng, khiến mỗi lần sửa code nhỏ phải mất từ 1 đến 3 phút chờ đợi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện leo thang quyền hạn:
1. Ứng dụng Python xuất hiện lỗ hổng nghiêm trọng (ví dụ: Remote Code Execution qua deserialization `pickle`, thực thi lệnh hệ thống `subprocess`/`os.system` không được lọc đầu vào, hoặc tràn bộ đệm trong thư viện C).
2. Kẻ tấn công kích hoạt lỗ hổng để chiếm quyền thực thi shell bên trong container. Do container chạy mặc định bằng `root`, tiến trình shell của kẻ tấn công có quyền UID 0 (root) bên trong container namespace.
3. Kẻ tấn công khai thác tiếp một lỗ hổng container breakout (ví dụ: khai thác lỗ hổng kernel Linux của máy host thông qua system call, lạm dụng Docker socket `/var/run/docker.sock` nếu bị mount nhầm, hoặc các Linux capabilities nguy hiểm chưa được drop).
4. Do tiến trình bên trong container vốn chạy bằng UID 0, khi thoát thành công khỏi ranh giới container (breakout), nó sẽ ánh xạ trực tiếp thành quyền root (UID 0) trên máy chủ vật lý host, giúp kẻ tấn công chiếm toàn quyền kiểm soát hạ tầng máy chủ.
Vị trí lệnh `USER` cắt đứt:
Lệnh `USER appuser` chuyển quyền thực thi của container sang một người dùng phi đặc quyền (UID 10001). Khi đó, nếu kẻ tấn công chiếm được shell trong container, chúng chỉ có quyền hạn hạn chế của `appuser`, không thể sửa file hệ thống của container, không có các Linux capabilities đặc quyền, và bị chặn hoàn toàn các kỹ thuật container breakout yêu cầu quyền root, dập tắt chuỗi tấn công ngay từ bước 2.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.
Cách đạt được: Bộ đếm cố định theo phút sẽ tự động xóa về 0 vào mỗi đầu phút (giây :00). Kẻ tấn công sẽ gửi dồn dập 10 request ở giây cuối cùng của phút thứ nhất (từ 10:00:59 đến 10:00:59.999). Ngay sau đó 1 giây khi đồng hồ nhảy sang 10:01:00, bộ đếm bị reset về 0, kẻ tấn công lập tức gửi thêm 10 request nữa trong giây đầu tiên của phút thứ hai (từ 10:01:00 đến 10:01:01). Cả hai đợt đều hợp lệ theo góc nhìn của từng phút độc lập (mỗi phút chỉ có 10 request), nhưng trên thực tế chỉ trong 2 giây liên tiếp (10:00:59 - 10:01:01), hệ thống phải hứng chịu tới 20 request, vượt gấp đôi giới hạn cho phép. Thuật toán Sliding Window (dùng Redis ZSET) tính chính xác khoảng trôi 60 giây gần nhất tính từ thời điểm hiện tại, loại bỏ hoàn toàn kẽ hở này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Khác biệt cốt lõi:
- Rate limit kiểm soát **tần suất và số lượng request trong một khoảng thời gian ngắn** (ví dụ: 10 request / 60 giây) nhằm bảo vệ tính sẵn sàng của hạ tầng máy chủ, chống nghẽn mạng và tấn công DoS/Spam.
- Cost guard kiểm soát **tổng số tiền chi tiêu / hạn mức tài chính trong một chu kỳ dài** (ví dụ: 10.0 USD / tháng) nhằm bảo vệ ngân sách tài chính của người dùng và dịch vụ.

Tình huống 1 (Rate limit cho qua nhưng Cost guard chặn):
Người dùng chỉ gửi 1 request duy nhất trong ngày (hoàn toàn thỏa mãn < 10 request/phút của rate limit), nhưng request đó đính kèm một tập tài liệu RAG khổng lồ chứa hàng trăm nghìn tokens, khiến chi phí ước tính vượt quá hạn mức 10 USD của tháng $\to$ Cost guard lập tức chặn và trả về lỗi 402 (Payment Required), trong khi Rate limit vẫn cho qua.

Tình huống 2 (Cost guard cho qua nhưng Rate limit chặn):
Đầu tháng khi ngân sách 10 USD vẫn còn nguyên vẹn, một script tự động gửi liên tục 20 request ngắn trong vòng 5 giây (mỗi request chỉ tốn 0.00002 USD, tổng cộng chỉ tốn 0.0004 USD, rất nhỏ so với 10 USD) $\to$ Ngân sách hoàn toàn đủ nên Cost guard cho qua, nhưng Rate limit sẽ chặn từ request thứ 11 và trả về lỗi 429 (Too Many Requests) kèm header `Retry-After: 60`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện xảy ra:
1. Redis gặp sự cố kết nối mạng tạm thời hoặc đang restart trong vòng 30 giây.
2. Endpoint gộp kiểm tra thấy Redis không phản hồi, lập tức trả về mã lỗi 503 cho bộ kiểm tra liveness probe của orchestrator (Kubernetes / Docker Swarm / Cloud Platform).
3. Do liveness probe thất bại, orchestrator cho rằng tiến trình container đã bị treo/hỏng và đồng loạt ra lệnh restart (kill và khởi động lại) cả 3 container agent cùng một lúc.
4. Toàn bộ các request đang được xử lý dở dang của khách hàng bị ngắt quãng đột ngột, người dùng nhận về lỗi 502 Bad Gateway.
5. Khi Redis kết nối trở lại sau 30 giây, cả 3 container agent vẫn đang trong giai đoạn khởi động lại (bootstrapping / import thư viện), không có bất kỳ instance nào sẵn sàng nhận request. Sự cố mất mạng tạm thời của Redis biến thành sự cố sập toàn bộ dịch vụ (cascading failure).
(Khi tách riêng: `/health` độc lập chỉ kiểm tra tiến trình app còn sống để không bị restart oan; `/ready` trả 503 để Load Balancer tạm thời không điều phối traffic mới tới cho đến khi Redis phục hồi).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Khi lưu trong Redis (Stateless): Toàn bộ 3 container đều chia sẻ chung một kho dữ liệu Redis tập trung. Bất kể Load Balancer chuyển tiếp request đến container nào, lịch sử hội thoại của người dùng vẫn được đọc và ghi nhất quán, `history_length` tăng tuần tự và đều đặn: 0 $\to$ 2 $\to$ 4 $\to$ 6 $\to$ 8...
- Nếu lưu trong dict Python trong bộ nhớ RAM (Stateful): Vì Load Balancer phân phối các request theo cơ chế round-robin ngẫu nhiên qua 3 container độc lập, mỗi container chỉ lưu giữ mẩu lịch sử của các request rơi vào chính nó. Khi đó `history_length` sẽ nhảy lộn xộn (ví dụ: request 1 vào container A thấy độ dài 0, request 2 vào container B thấy độ dài 0, request 3 vào container C thấy độ dài 0, request 4 rơi lại vào container A mới thấy độ dài 2...). Trải nghiệm người dùng sẽ bị hỏng do AI có vẻ bị "mất trí nhớ" ngẫu nhiên giữa các lượt hội thoại.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi**: `Application failed to respond / Healthcheck timeout after 300s` và `Connection refused on port 8000` trên dashboard deployment.
- **Cách tìm ra nguyên nhân**: Mở tab `Deployment Logs` trên dashboard của cloud platform (Railway/Render), nhận thấy platform tự động phân bổ một cổng động ngẫu nhiên qua biến môi trường `PORT=6543`, trong khi dòng lệnh khởi chạy Uvicorn trong Dockerfile lại bị cấu hình cứng là `--port 8000`. Vì vậy, ứng dụng lắng nghe ở cổng 8000 bên trong container, trong khi bộ định tuyến ingress của Cloud lại chờ đợi kết nối ở cổng 6543, dẫn đến healthcheck probe bị timeout và deploy thất bại.
- **Cách sửa**: Sửa lại lệnh `CMD` trong Dockerfile thành `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]` để tận dụng shell expansion đọc biến `$PORT` từ môi trường cloud, đồng thời gán giá trị mặc định là 8000 nếu chạy local. Sau khi cập nhật và commit lại, container đã lắng nghe chính xác cổng được cấp phát và healthcheck chuyển sang trạng thái xanh ngay lập tức.

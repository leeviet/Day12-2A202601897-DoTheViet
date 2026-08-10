# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> (câu trả lời của bạn)` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đỗ Thế Việt  Mã học viên: 2A202601897

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu để mặc định 'changeme', khi đưa lên production mà quên set biến, hacker phát hiện ra khóa này có thể dùng nó để gọi API gây tốn tiền hoặc phá hoại hệ thống. Chết sớm (fail fast) giúp ta phát hiện ra cấu hình thiếu sót ngay lúc khởi động app, ngăn chặn việc vô tình expose API bằng khoá yếu.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log: `{"timestamp": "2026-08-10T12:00:00Z", "level": "INFO", "event": "ask_request", "user_id": "sv-test", "tokens": 25, "cost": 0.05}`
> Hai việc làm được: 1. Đưa log vào ELK/CloudWatch để dễ dàng tìm kiếm (filter `user_id == "sv-test"` hoặc tính tổng `cost`). 2. Dễ dàng vẽ biểu đồ thống kê token usage hoặc chi phí tự động mà không phải dùng regex tách chuỗi thủ công như khi in ra dạng text thường.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

> | 1 stage (bản đầu) | ~ 1100 MB |
> | Multi-stage | ~ 150 MB |
> 
> Giải thích: Phần dung lượng giảm đi chính là do ta đã bỏ đi bộ trình biên dịch (compiler gcc), các thư viện dev header, cache của pip và các công cụ build hệ điều hành vốn chỉ cần trong lúc cài đặt thư viện chứ không cần lúc chạy app.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Sửa 1 ký tự trong code: Layer `COPY requirements.txt` và `RUN pip install` được dùng lại (CACHE). Layer `COPY . .` và các layer sau nó phải chạy lại.
> Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi khi code thay đổi (dù 1 ký tự), Docker sẽ đánh rớt toàn bộ cache từ dòng `COPY . .` trở xuống. Điều này khiến `RUN pip install` bị gọi chạy lại từ đầu vô cùng mất thời gian cho mỗi lần sửa code.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Hacker tìm được lỗ hổng trong code -> Lợi dụng để thực thi mã độc/lệnh shell -> Vì app chạy bằng quyền root, hacker có quyền root trong container -> Hacker thực hiện thủ thuật "container breakout" để thoát ra và chiếm quyền root trên máy host (máy chủ thật).
> Lệnh `USER` cắt đứt chuỗi ở chỗ: App chỉ chạy bằng user thường, nên dù hacker có chạy được mã độc hay chiếm được shell thì cũng không có đặc quyền để phá hoại thêm hoặc breakout ra ngoài host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Họ gửi được tối đa 20 request.
> Giải thích: Do đếm theo phút (Fixed window) sẽ bị reset ở giây thứ 00. Nếu user gửi 10 request lúc 11:00:59, cửa sổ sẽ reset ở 11:01:00. Ngay lập tức lúc 11:01:01 họ gửi tiếp 10 request nữa. Tổng cộng trong khoảng 2 giây, họ đã gửi thành công 20 request (gấp đôi hạn mức).

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Khác nhau: Rate limit chặn tần suất (spam liên tục trong 1 phút), Cost guard chặn tổng dung lượng/chi tiêu trong một thời gian dài (vượt ngân sách 1 tháng).
> - Cho qua Rate Limit nhưng chặn Cost Guard: Gọi rất từ tốn (1 request/phút) nên qua được rate limit, nhưng đã dùng nhiều ngày tích luỹ làm cạn sạch 10$ trong tháng, nên sẽ bị chặn.
> - Chặn Rate Limit nhưng qua Cost Guard: Tài khoản mới tinh (chưa tốn đồng nào), nhưng user dùng công cụ bấm F5 liên tục gửi 15 request/giây -> Bị rate limit chặn ngay lập tức do spam.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện:
> 1. Redis chết.
> 2. Load Balancer gọi /health kiểm tra app và nhận kết quả fail (do check Redis fail).
> 3. Vì /health fail, Orchestrator (Docker/K8s) coi container đã "đột tử" nên lập tức Kill container và khởi động container mới.
> 4. Container mới lên, nhưng Redis vẫn chết -> lại báo /health fail -> lại bị Kill -> Gây hiện tượng CrashLoop liên tục.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Nếu dùng Redis (Stateless): Dù load balancer điều hướng request sang agent nào thì chúng cũng đọc chung từ Redis -> `history_length` tăng dần đều và nhất quán (1, 2, 3...).
> Nếu dùng Python Dict (Stateful): Do có 3 agent độc lập tự ôm bộ nhớ riêng, request nhảy ngẫu nhiên vào agent nào thì chỉ tăng bộ đếm của agent đó -> `history_length` sẽ tăng lộn xộn (ví dụ: Agent 1 trả về 1, Agent 2 trả về 1, Agent 1 trả về 2, Agent 3 trả về 1...), gây mất đồng bộ lịch sử hội thoại.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi gặp phải: Báo lỗi "Error: Invalid value for '--port': '$PORT' is not a valid integer." khi deploy Railway.
> Cách tìm nguyên nhân: Xem Deploy Logs trên trang dashboard của Railway.
> Cách sửa: Sửa file `railway.toml` dòng `startCommand = "sh -c 'uvicorn app.main:app --host 0.0.0.0 --port $PORT'"` (dùng lệnh sh -c) để shell tự động dịch biến $PORT do Railway cấp thành con số thật thay vì để nguyên chữ "$PORT".

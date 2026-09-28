# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng gợi ý bên dưới mỗi câu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Đình Khang  Mã học viên: 2A202602584

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Railway, nếu quên đặt `AGENT_API_KEY` mà app có mặc định `"changeme"`, service vẫn chạy nhưng bất kỳ ai đoán được khóa mặc định đều có thể gọi `/ask` và làm phát sinh chi phí. Với cấu hình fail fast, app dừng ngay khi thiếu biến môi trường; em phát hiện lỗi ngay trong log deploy và không mở một API có xác thực yếu.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> > `{"event":"ask_completed","level":"info","timestamp":"2026-09-28T09:24:04.145061+00:00","user_id":"sv01","tokens_out":47,"cost_usd":0.0000423}`
>
> Từ log JSON này em có thể lọc các request theo `user_id` để kiểm tra hoạt động của một người dùng cụ thể. Em cũng có thể tổng hợp trường `cost_usd` theo ngày hoặc tháng để theo dõi chi phí sử dụng LLM. `print("đã trả lời xong")` chỉ là văn bản tự do, không có các trường dữ liệu cố định để cloud có thể lọc và thống kê tự động.

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
| 1 stage (bản đầu) | 1.7GB |
| Multi-stage | 297 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Bản 1-stage là 1.7 GB, còn bản multi-stage là 297 MB, giảm khoảng 1.4 GB. Phần chênh lệch chủ yếu đến từ base image Python đầy đủ và các công cụ/phụ thuộc chỉ cần trong lúc build. Multi-stage chỉ copy dependency đã cài cùng source code cần chạy sang runtime, nên không mang theo các thành phần build không cần thiết.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Sau khi sửa một ký tự trong `app/main.py`, các layer `FROM`, `WORKDIR`, `COPY requirements.txt` và `RUN pip install` được lấy lại từ cache vì `requirements.txt` chưa thay đổi. Layer copy source code và các layer sau nó phải chạy lại. Nếu đặt `COPY . .` trước `RUN pip install`, bất kỳ thay đổi nào của source cũng làm Docker phải cài lại toàn bộ dependency dù requirements không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Một lỗ hổng như thực thi lệnh tùy ý có thể cho kẻ tấn công shell bên trong container. Nếu process chạy root, họ có quyền root trong container, có thể đọc hoặc sửa file nhạy cảm và tiếp tục khai thác cấu hình Docker hoặc lỗ hổng escape để ảnh hưởng host. `USER app` chuyển process sang UID 10001, nên shell có được chỉ có quyền user thường; nó giảm đáng kể tác hại khi app bị xâm nhập.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa là 20 request trong 2 giây. Người dùng gửi 10 request ở giây 59 của một phút, rồi gửi thêm 10 request ở giây 00 của phút kế tiếp. Bộ đếm theo phút đồng hồ reset tại mốc 00 nên cho qua cả hai nhóm. Sliding window luôn tính 60 giây gần nhất nên không có lỗ hổng này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn số lần gọi trong một khoảng thời gian để chống spam hoặc tải đột biến. Cost guard giới hạn tổng tiền đã dùng của từng user trong tháng để kiểm soát ngân sách. Ví dụ, một user chỉ gửi vài câu hỏi dài nhưng đã dùng hết ngân sách tháng: rate limit cho qua nhưng cost guard trả 402. Ngược lại, user còn ngân sách nhưng gửi request thứ 11 trong 60 giây: cost guard cho qua nhưng rate limit trả 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu `/health` cũng ping Redis, Redis mất kết nối thì cả ba container vẫn đang chạy Python nhưng đều trả health check lỗi. Load balancer hoặc orchestrator lần lượt coi chúng không khỏe, rút khỏi traffic hoặc khởi động lại. Các container khởi động lại tiếp tục kiểm tra Redis đang lỗi và tiếp tục fail, tạo vòng lặp restart. Tách `/health` chỉ kiểm tra process còn sống, còn `/ready` trả 503 khi Redis lỗi, giúp app không nhận request mới nhưng không bị coi là đã chết.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Khi history nằm trong Redis, ba instance cùng đọc và ghi một key theo `X-User-Id`, nên `history_length` tăng nhất quán theo các message mới, bất kể request được route vào instance nào. Nếu dùng dict Python, mỗi container có một dict riêng. Khi load balancer đổi instance, `history_length` có thể thấp hoặc quay về 0 rồi tăng riêng trên từng instance, làm cuộc hội thoại không liên tục.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Khi chạy GitHub Actions để deploy Railway, job báo lỗi `Unauthorized. Please check that your RAILWAY_API_TOKEN is valid and has access to the resource you're trying to use.` Em xem log của bước `railway up` và kiểm tra lại secret trên GitHub cũng như token trên Railway. Nguyên nhân là workflow không nhận được token Railway hợp lệ. Em tạo Project Token cho environment `production`, lưu nó vào GitHub Repository Secret với tên `RAILWAY_TOKEN`, rồi chạy lại workflow. Sau đó bước deploy và smoke test `/health` thành công.
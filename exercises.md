# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng trích dẫn placeholder bên dưới mỗi câu hỏi bằng nội dung trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Vũ Viết Hoàng  Mã học viên: 2A202602398

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Lúc deploy lên Render, mình quên set `AGENT_API_KEY` trong dashboard ở một
> lần thử đầu. Vì không có default, service crash ngay và log ghi rõ
> `ValidationError: agent_api_key Field required` — mình biết ngay lỗi ở đâu.
> Nếu để mặc định `"changeme"`, app vẫn chạy bình thường, `/health` vẫn 200,
> nhưng bất kỳ ai cũng gọi được `/ask` bằng khóa `"changeme"` mà không bị
> phát hiện — chỉ lộ ra khi nhìn hóa đơn LLM tăng bất thường, lúc đó đã tốn
> tiền và có khi đã bị lộ dữ liệu.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> ```
> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T08:05:47.778208+00:00", "user_id": "sv01", "tokens_in": 469, "tokens_out": 52, "cost_usd": 0.00010155}
> ```
> Hai việc làm được: (1) lọc/tổng hợp theo trường — ví dụ đưa vào công cụ log
> và query `SUM(cost_usd) WHERE user_id="sv01"` để biết user nào tiêu nhiều
> tiền nhất, mà không cần regex vào chuỗi text; (2) đặt cảnh báo tự động, ví
> dụ "báo khi `cost_usd` của một request vượt 0.01" — máy đọc được field
> `cost_usd` trực tiếp. Với `print("đã trả lời xong")` thì không có trường
> nào để lọc hay so sánh, chỉ đọc được bằng mắt người.

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
| 1 stage (bản đầu) | 1730 MB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Bản 1 stage dùng `python:3.11` đầy đủ (có compiler, header dev, nhiều gói
> hệ thống) và giữ luôn toàn bộ cache pip, source `.git`, `.venv`... vì
> `COPY . .` chép hết. Bản multi-stage chỉ giữ lại kết quả `pip install`
> (`/install`) copy sang stage `runtime` dựa trên `python:3.11-slim`, không
> mang theo compiler hay file build trung gian. Chênh lệch ~1.46GB chủ yếu là
> compiler/toolchain và các gói hệ thống không cần lúc chạy, chỉ cần lúc cài.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Dockerfile của mình copy `requirements.txt` → `pip install` (stage
> `builder`) → rồi mới `COPY app ./app` (stage `runtime`). Sửa 1 ký tự trong
> `app/main.py` chỉ làm layer `COPY app ./app` và các layer sau nó (USER,
> HEALTHCHECK, CMD) chạy lại; layer `pip install` (mất ~90 giây khi mình
> build) vẫn được lấy từ cache vì `requirements.txt` không đổi. Nếu đặt
> `COPY . .` lên trước `pip install`, Docker sẽ thấy nội dung thư mục đổi
> (vì file code đổi) nên hủy cache ngay từ đó — kéo theo `pip install` phải
> chạy lại toàn bộ mỗi lần sửa 1 dòng code, dù `requirements.txt` không đổi
> gì cả.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: (1) code Python có lỗ hổng (ví dụ dependency cũ có RCE) →
> (2) kẻ tấn công chạy được lệnh tùy ý bên trong container → (3) nếu process
> đang chạy bằng root, lệnh đó có toàn quyền trong container → (4) nếu có
> thêm lỗ hổng thoát container (container breakout, khá phổ biến với
> Docker cấu hình sai) thì kẻ tấn công mang theo quyền root đó ra ngoài máy
> host thật. Lệnh `USER appuser` (mình dùng uid 10001) cắt đứt chuỗi này ở
> bước (3): dù có RCE, lệnh chạy được cũng chỉ có quyền của `appuser` — user
> thường, không ghi được file hệ thống, không cài package, nên dù thoát được
> container thì quyền mang theo cũng không phải root.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa 20 request trong 2 giây. Cách đạt được: gửi 10 request lúc 10:00:59
> (vẫn tính vào "phút 10:00", chưa reset), rồi gửi tiếp 10 request lúc
> 10:01:01 (bộ đếm đã reset về 0 vì sang "phút 10:01" mới). Cả 20 request đều
> "đúng luật" theo cách đếm phút đồng hồ dù chỉ cách nhau 2 giây thực tế.
> Sliding window của mình (`zremrangebyscore` theo `now - 60`) không có kẽ hở
> này vì nó luôn nhìn đúng 60 giây gần nhất tính từ thời điểm hiện tại, không
> phụ thuộc mốc đồng hồ tròn phút.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn *số lượng* request/phút, cost guard giới hạn *số tiền*
> tích lũy/tháng — hai trục độc lập. Tình huống 1 (rate limit cho qua, cost
> guard chặn): user chỉ gửi 2 request/phút (dưới hạn mức 10), nhưng mỗi
> request hỏi câu rất dài (nhiều token) nên chi phí cộng dồn vượt
> `MONTHLY_BUDGET_USD` giữa tháng — `/ask` trả 402 dù tần suất gọi thấp.
> Tình huống 2 (cost guard cho qua, rate limit chặn): user gửi 15 request/giây
> toàn câu hỏi ngắn "hi" — mỗi request rẻ nên ngân sách tháng còn dư rất
> nhiều, nhưng vượt `RATE_LIMIT_PER_MINUTE=10` nên bị 429 ngay ở request thứ
> 11, trước khi kịp tốn thêm tiền.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> (1) Redis mất kết nối → cả 3 container gọi `store.ping()` đều trả `False` →
> endpoint gộp trả 503 ở cả liveness lẫn readiness. (2) Vì đây là liveness
> (orchestrator dùng để quyết định restart), cả 3 container bị đánh dấu
> "unhealthy" và bị **restart** đồng loạt — không phải chỉ rút khỏi vòng xoay
> load balancer như readiness thật sự nên làm. (3) Trong lúc cả 3 container
> đang restart, không còn instance nào phục vụ được request — downtime toàn
> bộ, dù bản thân code service không hề lỗi. (4) Khi Redis sống lại sau 30
> giây, nếu 3 container restart không đồng bộ (ví dụ có cooldown khác nhau),
> có thể còn chưa có container nào kịp lên lại để nhận traffic — biến sự cố
> Redis 30 giây (nhỏ) thành gián đoạn dịch vụ dài hơn (lớn). Vì vậy `/health`
> (liveness) phải tách khỏi Redis, chỉ `/ready` (readiness) mới được kiểm tra.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Mình scale 3 container (port 8000/8001/8002), gọi lần lượt xoay vòng qua cả
> 3 với cùng `X-User-Id: sv-scale-demo`, kết quả `history_length` thực tế:
> `0 → 2 → 4 → 6 → 8` — tăng đều đặn dù mỗi request rơi vào container khác
> nhau, vì cả 3 cùng đọc/ghi chung một Redis. Nếu lịch sử lưu trong dict
> Python (trong RAM từng process) thay vì Redis, con số sẽ **không tăng đều**
> mà nhảy lung tung theo kiểu 0, 0, 0, 2, 0... — mỗi container chỉ thấy phần
> lịch sử do chính nó ghi, không thấy phần 2 container kia ghi, tạo cảm giác
> "agent bị mất trí nhớ" ngẫu nhiên tùy request rơi vào container nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi gặp: sau khi push code CP1–CP4 lên GitHub và tạo Blueprint trên Render,
> dashboard hiện "Sync: 306b897" — nhưng đó lại là commit **cũ** (trước khi
> mình sửa code), không phải commit mới nhất (`c8f1f21`) vừa push. Mình phát
> hiện ra bằng cách so `git log --oneline -3` ở máy với hash hiển thị trên
> dashboard Render — không khớp. Nguyên nhân: Render bắt đầu sync ngay khi
> tạo Blueprint, trước khi lệnh `git push` của mình kịp chạy xong, nên nó
> build với snapshot code cũ (vẫn còn `NotImplementedError`). Cách sửa: đợi
> lần sync đầu chạy xong, sau đó Render tự phát hiện commit mới trên GitHub
> và tự kích hoạt một lần deploy mới đúng với `c8f1f21` — dashboard sau đó
> hiển thị đúng commit và service chạy ổn định.

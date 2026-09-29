# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Ngô Lê Thuỷ Tiên  Mã học viên: 2A202602614

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu mặc định là `"changeme"` và mình quên đặt `AGENT_API_KEY` trên Render, app vẫn khởi động bình thường, `/health` trả 200, dashboard báo Live. Mình không biết gì cho tới khi có người thử khóa `changeme` và gọi `/ask` miễn phí, lúc đó hóa đơn LLM mới lộ ra. Còn khi không có mặc định, thiếu biến thì Pydantic ném `ValidationError` ngay lúc khởi động, deploy fail và mình thấy lỗi trong log build. Khi chạy test `test_thieu_api_key_thi_fail_fast`, bỏ biến đi thì app chết ngay đúng như vậy.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Một dòng log mình lấy được từ container khi gọi `/ask`:

```
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T05:04:38.473934+00:00", "user_id": "sv-scale-5625", "tokens_in": 145, "tokens_out": 47, "cost_usd": 4.995e-05}
```

Hai việc làm được với dòng này mà `print("đã trả lời xong")` không làm được: (1) lọc và đếm theo field, ví dụ đếm số request của riêng `user_id` này hoặc cộng `cost_usd` để biết user tiêu bao nhiêu tiền; (2) đặt cảnh báo trên hệ thống log (Render, Datadog...) khi `level` là error hoặc khi `cost_usd` vượt ngưỡng, vì máy đọc được field chứ không phải đoán từ câu chữ.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

Giải thích: phần dung lượng chênh lệch đó là những gì?

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu, `python:3.11`) | 1.73 GB |
| Multi-stage (`python:3.11-slim`) | 297 MB |

Mình build bản 1 stage từ Dockerfile gốc của lab (lấy từ commit đầu), còn bản multi-stage là Dockerfile hiện tại. Phần chênh khoảng 1.4 GB chủ yếu là base image: `python:3.11` bản đầy đủ mang theo gcc, make, header, git và nhiều công cụ build, trong khi `slim` không có. Multi-stage còn giúp runtime chỉ copy thư mục `/install` chứa thư viện đã cài, không kéo theo cache pip hay công cụ ở stage builder. Ngoài ra bản đầu `COPY . .` nên có thể mang cả `.git`, `.venv`, test vào image.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Mình thêm một dòng comment vào `app/main.py` rồi build lại. Kết quả với Dockerfile hiện tại: các layer `builder` (WORKDIR, `COPY requirements.txt`, `RUN pip install`), `WORKDIR /app`, tạo user và `COPY --from=builder /install` đều **CACHED**. Chỉ hai layer `COPY app ./app` và `COPY utils ./utils` phải chạy lại. Vì `pip install` nằm trước và chỉ phụ thuộc `requirements.txt`, nên sửa code không phải cài lại thư viện.

Khi mình thử đặt `COPY . .` trước `RUN pip install` (Dockerfile mẫu ban đầu), cùng thay đổi đó làm layer `COPY . .` bị invalidate, và mọi layer phía sau, gồm cả `RUN pip install`, phải chạy lại. Mỗi lần sửa một dòng code là phải cài lại toàn bộ thư viện, build chậm hơn nhiều (bản đó mất khoảng 27 giây dù mạng đã ấm).

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện: một lỗ hổng trong code Python (ví dụ nhận input không kiểm tra rồi chạy lệnh shell hoặc cho phép tải file tùy ý) cho kẻ tấn công thực thi lệnh bên trong container. Nếu container chạy bằng root thì các lệnh đó có quyền root trong container. Từ đó kẻ tấn công có thể ghi vào mọi file, cài công cụ, đọc secret trong biến môi trường, và nếu có thêm một lỗi thoát container hoặc volume/socket mount thì root trong container dễ trở thành root trên host, vì UID 0 ở hai bên là một.

Dòng `USER app` cắt ở chỗ đầu tiên: dù kẻ tấn công vào được process, họ chỉ là user `app` không có đặc quyền, không cài được gói, không ghi được ra ngoài thư mục được phép. Mình kiểm tra bằng `docker compose exec agent id` và thấy `uid=999(app)`.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Tối đa **20 request** trong 2 giây. Cách đạt được: đếm theo phút đồng hồ reset lúc giây 00. Người dùng gửi 10 request vào giây 59 của phút trước (đủ hạn mức 10 của phút đó) rồi gửi thêm 10 request vào giây 00 của phút sau, khi bộ đếm vừa reset về 0. Cả hai lô đều "đúng luật", nhưng cộng lại là 20 request trong khoảng 2 giây, gấp đôi hạn mức. Sliding window 60 giây thì không bị vậy: lúc gửi lô thứ hai, 10 request vừa rồi vẫn còn nằm trong 60 giây gần nhất, nên bị chặn 429. Trên Render mình gọi 15 lần liên tiếp và nhận `200 ×10` rồi `429 ×5`.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn **số lượng** request theo thời gian (10 request/60 giây), còn cost guard giới hạn **số tiền** đã tiêu trong tháng (mặc định 10 USD). Hai thứ độc lập vì một request rẻ hay đắt không liên quan tới việc nó nhiều hay ít.

- Rate limit cho qua nhưng cost guard chặn: user chỉ gửi 1 request mỗi phút (không bao giờ chạm 10/phút) nhưng mỗi request có lịch sử và prompt rất dài, vài chục nghìn token. Sau vài trăm request tổng chi phí trong tháng vượt 10 USD, `guard.check` trả 402 dù rate limit chưa từng cản.
- Ngược lại: user gửi 15 request ngắn liên tiếp, mỗi cái chỉ tốn vài phần nghìn cent. Request thứ 11 bị rate limit trả 429 dù tổng chi phí còn rất xa ngân sách. Trong lần thử của mình trên Render, 5 request cuối bị 429 trong khi `cost_usd` mỗi lần chỉ cỡ 1e-5.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Giả sử `/health` cũng kiểm tra Redis, và Redis mất kết nối 30 giây. Thứ tự sự kiện: (1) Redis mất kết nối. (2) `/health` của cả 3 container đều bắt đầu trả 503 vì cùng phụ thuộc vào một Redis. (3) Orchestrator coi cả 3 container là hỏng, sau vài lần healthcheck thất bại sẽ **restart cả 3**. (4) Trong lúc restart, không còn container nào nhận request nên toàn bộ service ngừng, dù chỉ Redis có vấn đề còn code vẫn sống. (5) Redis quay lại sau 30 giây nhưng 3 container còn đang khởi động lại, service phải chờ thêm mới hoạt động. Restart không giúp Redis sống lại nên chỉ làm tình hình tệ hơn.

Vì vậy `/health` không gọi Redis (container còn sống thì không restart), còn `/ready` mới kiểm tra Redis: khi Redis mất, `/ready` trả 503, load balancer ngừng gửi request tới đó, và khi Redis về thì `/ready` tự trả 200 lại mà không cần restart gì.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Compose không scale được ngay vì map cổng cố định `8000:8000` (chỉ một container chiếm được cổng, hai container kia báo `port is already allocated`), nên mình tự chạy hai container agent riêng, cùng nối vào một Redis, một cái ở cổng 8000 và một ở cổng 18001, rồi gọi luân phiên với cùng một `X-User-Id`. Kết quả `history_length` là `0, 2, 4, 6, 8`: tăng đều dù request đi vào hai container khác nhau, vì lịch sử nằm trong Redis chung.

Nếu lịch sử lưu trong dict Python của từng process thì mỗi container có bản riêng. Gọi luân phiên sẽ thấy `0, 0, 2, 2, 4` (mỗi container đếm riêng, container kia không biết gì), agent như bị "mất trí nhớ" tùy request rơi vào container nào. Nếu container restart thì con số cũng về 0.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi mình gặp khi kiểm tra service đã deploy trên Render: gọi `/ask` có `-H "X-API-Key: $AGENT_API_KEY"` mà server trả `HTTP/2 401` với `{"detail":"invalid or missing API key"}`, trong khi `/health` và `/ready` vẫn trả 200. Mình tìm nguyên nhân bằng cách để ý `/health` và `/ready` (không cần key) chạy tốt, chỉ `/ask` lỗi, tức app và Redis đều bình thường mà chỉ bước xác thực từ chối. Sau đó kiểm tra biến thì `$AGENT_API_KEY` trong shell là key dùng ở máy local hoặc trống, không phải giá trị mình nhập trên Render lúc tạo Blueprint (hai nơi có thể khác nhau). Cách sửa: đặt `DEPLOY_API_KEY` trong `.env` đúng bằng key đã nhập trên Render và gọi bằng key đó, sau đó `/ask` trả 200 và test rate limit ra `429` đúng như mong đợi.

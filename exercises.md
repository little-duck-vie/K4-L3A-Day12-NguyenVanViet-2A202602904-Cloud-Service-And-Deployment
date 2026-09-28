# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder dưới mỗi câu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: .......Nguyễn Văn Việt...................  Mã học viên: ........2A202602904..................

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu deploy lên Render mà tôi quên set `AGENT_API_KEY`, app chết ngay bằng lỗi
> `ValidationError: agent_api_key Field required`. Nhờ vậy tôi biết thiếu secret
> trước khi public URL hoạt động. Nếu để mặc định `"changeme"`, service vẫn chạy
> và người khác có thể đoán key đó để gọi `/ask`, làm tốn quota/chi phí mà tôi
> không phát hiện ngay.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log tôi thu được có dạng:
> `{"event":"ask_completed","level":"info","timestamp":"2026-09-28T09:56:00+00:00","user_id":"sv-test","tokens_in":3,"tokens_out":37,"cost_usd":0.00002265}`.
> Với log JSON này tôi có thể lọc theo `event` hoặc `user_id` để biết user nào
> gọi API nhiều, và có thể cộng `cost_usd`/token để theo dõi chi phí. Nếu chỉ
> `print("đã trả lời xong")` thì máy không biết trường nào là user, trường nào
> là chi phí, rất khó thống kê và cảnh báo.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản                 | Dung lượng |
| -------------------- | ------------ |
| 1 stage (bản đầu) | khoảng 1GB nếu dùng `python:3.11` đầy đủ |
| Multi-stage          | 271-310 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần chênh lệch chủ yếu là base image đầy đủ, cache build, compiler/tool hệ
> thống và các file không cần ở runtime. Multi-stage chỉ copy virtualenv đã cài
> dependency và source cần chạy sang image `python:3.11-slim`, nên image cuối
> không mang theo toàn bộ môi trường build.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile hiện tại, khi sửa một ký tự trong `app/main.py`, các layer
> `FROM`, `WORKDIR`, `COPY requirements.txt`, tạo venv và `pip install` được dùng
> lại từ cache. Layer phải chạy lại là `COPY app ./app` và các layer sau nó.
> Nếu đặt `COPY . .` trước `RUN pip install`, chỉ cần sửa code là Docker thấy
> context đổi, làm cache của layer install dependency bị mất và phải cài lại thư
> viện từ đầu, build chậm hơn nhiều.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu app Python có lỗ hổng cho phép chạy lệnh trong container, kẻ tấn công sẽ
> chạy lệnh với quyền user hiện tại. Nếu container chạy root, các file/process
> trong container cũng thuộc root, và khi có mount hoặc lỗi escape container thì
> rủi ro leo lên quyền cao trên host lớn hơn. Lệnh `USER appuser` cắt chuỗi này
> ở bước thực thi lệnh: kể cả chiếm được app, attacker chỉ có quyền user thường
> trong container, giảm thiệt hại.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Nếu đếm theo phút đồng hồ, user có thể gửi tối đa 20 request trong khoảng 2
> giây: gửi 10 request lúc 10:00:59, rồi khi đồng hồ sang 10:01:00 bộ đếm reset,
> gửi tiếp 10 request lúc 10:01:01. Sliding window 60 giây tránh lỗ hổng này vì
> nó luôn nhìn lại 60 giây gần nhất, không phụ thuộc ranh giới phút.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tốc độ/số request, còn cost guard giới hạn tổng tiền đã
> tiêu. Ví dụ rate limit cho qua 1 request lớn vì user chưa vượt 10 request/phút,
> nhưng request đó ước tính quá đắt hoặc user đã gần hết ngân sách tháng nên cost
> guard phải chặn. Ngược lại, user còn rất nhiều ngân sách nhưng spam 11 request
> trong 60 giây thì cost guard vẫn cho về mặt tiền, còn rate limit phải chặn bằng
> 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp `/health` và `/ready` rồi cho endpoint đó kiểm tra Redis, khi Redis mất
> kết nối 30 giây thì cả 3 container đều trả health check lỗi. Orchestrator tưởng
> cả 3 process bị chết nên restart lần lượt hoặc đồng thời. Trong lúc restart,
> load balancer vẫn không có instance khỏe để nhận request, user thấy lỗi 502/503.
> Thực tế app vẫn còn sống, chỉ là dependency Redis tạm thời chưa ready, nên phải
> tách `/health` để kiểm tra process và `/ready` để kiểm tra dependency.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Khi history nằm trong Redis, dù request vào instance nào thì cùng `X-User-Id`
> vẫn đọc được lịch sử chung, nên `history_length` tăng ổn định: lần đầu là 0,
> lần sau thấy 2 message trước đó, rồi tăng tiếp theo các lượt hỏi. Nếu lưu bằng
> dict Python trong RAM, mỗi container có bộ nhớ riêng; request bị load balance
> sang instance khác sẽ thấy history rỗng hoặc số lúc tăng lúc tụt, giống như
> agent bị mất trí nhớ.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi tôi gặp khi deploy là `/ready` trả 500 và log Render báo
> `pydantic_core.ValidationError: agent_api_key Field required`. Tôi tìm nguyên
> nhân bằng cách mở tab Logs của service trên Render và thấy `Settings` thiếu
> `agent_api_key`. Cách sửa là vào Environment Variables của Render, thêm đúng
> biến `AGENT_API_KEY` bằng key tạo từ `secrets.token_urlsafe(32)`, kiểm tra lại
> `REDIS_URL`, rồi redeploy. Sau đó `/health` trả 200, `/ready` trả
> `{"status":"ready","redis":true}` và `/ask` không key trả 401.

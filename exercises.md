# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng trả lời mẫu dưới mỗi câu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Hoàng Bảo Minh  Mã học viên: 2A202601626

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu quên đặt `API_TOKEN` khi deploy, app sẽ dừng ngay lúc khởi động thay vì chạy với token `changeme`. Nhờ vậy health check báo lỗi sớm và mình biết secret đang thiếu trước khi service nhận traffic.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Một dòng log có các field như `event`, `severity`, `ts`, `client_id` và `usd_cost`. Máy có thể lọc event để thống kê request theo client và tính chi phí; `print("đã trả lời xong")` không có cấu trúc để máy lọc và tổng hợp.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | khoảng 1.8 GB theo ghi chú lab |
| Multi-stage | 183 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Multi-stage không đưa layer build và dữ liệu thừa vào runtime. Image chỉ copy package đã cài và source cần thiết nên nhỏ hơn, ít tool thừa hơn và giảm attack surface.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Khi sửa một ký tự trong `app/main.py`, Docker dùng lại base image, `WORKDIR`, `COPY requirements.txt` và `pip install`; chỉ các layer copy source chạy lại. Nếu `COPY . .` đặt trước `pip install`, mọi thay đổi source làm mất cache của layer cài dependency.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Nếu lỗ hổng cho phép thực thi lệnh, chạy bằng root khiến attacker có quyền cao trong container và có thể khai thác thêm misconfiguration hoặc container escape. `USER appuser` giới hạn process ở quyền user thường, cắt chuỗi leo thang quyền ngay trong container.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

`WWW-Authenticate: Bearer` báo cho client biết server yêu cầu cơ chế Bearer và cách gửi token. Dùng cùng thông báo cho thiếu header, sai scheme và sai token giúp không làm lộ thông tin cho người đang dò token; chi tiết có thể xem ở log nội bộ.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

Bucket ban đầu có 10 token và sau 10 phút vẫn tối đa 10 nhờ `min(capacity, tokens)`, nên gửi được 10 request rồi request thứ 11 bị `429`. Nếu bỏ giới hạn, refill 10 token/phút trong 10 phút tạo khoảng 100 token, cho phép burst khoảng 100 request.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

Hạn mức 30 USD/tháng có thể cho sự cố tiêu gần 30 USD trước khi hết tháng và chỉ hồi phục khi sang tháng mới. Hạn mức 1 USD/ngày giới hạn thiệt hại khoảng 1 USD trong ngày và tự hồi phục khi sang ngày UTC tiếp theo.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Nếu `/healthz` cũng kiểm tra Redis, Redis mất 30 giây có thể làm cả 3 container bị đánh dấu unhealthy và bị loại hoặc restart. Tách `/healthz` và `/readyz` giúp process vẫn sống, còn readiness trả `503` để ngừng traffic mới.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi thực tế là chạy `--scale agent=3` nhưng Compose báo `no such service: agent` vì service tên `chat`. Đổi thành `--scale chat=3` thì các container tranh port 8000. Mình đổi `chat` sang `expose: 8000`, thêm Nginx map `8000:80`, rồi scale lại thành công.

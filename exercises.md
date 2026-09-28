# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: viết câu trả lời của mình ngay dưới từng câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Hoàng Sơn  Mã học viên: 2A202602457

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Ví dụ lúc đưa service lên Railway, nếu mình quên đặt `AGENT_API_KEY`, Pydantic báo thiếu biến ngay khi khởi động. Mình thấy lỗi trong log và sửa cấu hình trước khi để người khác gọi API. Nếu code tự dùng `"changeme"`, service vẫn mở URL công khai với một khóa ai cũng đoán được; người lạ có thể gọi `/ask` và dùng hết ngân sách của mình.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Mình gọi `/ask` trên Railway và lấy được một dòng JSON từ log (không chứa API key):
>
> `{"cost_usd":0.00004785,"level":"info","tokens_in":139,"message":"","timestamp":"2026-09-28T09:54:11.414110+00:00","tokens_out":45,"event":"ask_completed","user_id":"cp5-test"}`
>
> Từ các trường có tên rõ ràng, mình có thể (1) lọc và đếm các sự kiện `ask_completed` theo `user_id` hoặc thời điểm để điều tra lỗi; (2) cộng `cost_usd` hoặc `tokens_in` để theo dõi chi phí và đặt cảnh báo. Dòng `print("đã trả lời xong")` không mang những dữ liệu đó để máy truy vấn.

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
| 1 stage (bản dựng đối chiếu) | 287 MB |
| Multi-stage (Dockerfile hiện tại) | 310 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Mình dựng bản 1 stage tạm từ cùng `python:3.11-slim` và `requirements.txt`, cài pip trực tiếp vào Python hệ thống; sau đó build Dockerfile hiện tại thành `agent:multi`. `docker image ls` cho kết quả 287 MB và 310 MB: bản multi lớn hơn 23 MB. `docker history` cho thấy layer cài thư viện trực tiếp của bản 1 stage khoảng 78 MB, còn layer chép `/opt/venv` của bản multi khoảng 96 MB. Builder stage không có trong image cuối; ở bài này virtualenv và các layer runtime làm bản multi lớn hơn, vì cả hai đều dùng cùng base slim và thư viện đã có wheel. Multi-stage giúp tách bước build và chỉ mang phần cần chạy sang runtime, nhưng không bảo đảm image luôn nhỏ hơn.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Mình thử build hai lần trong một bản sao tạm, chỉ thêm một comment vào `app/main.py` ở lần sau. Log build cho thấy `COPY requirements.txt`, bước tạo venv/cài pip, `RUN useradd` và `COPY --from=builder /opt/venv` đều `CACHED`; hai bước `COPY app` rồi `COPY utils` chạy lại. `COPY utils` cũng phải làm lại vì nó nằm sau layer `COPY app` đã đổi. Nếu đặt `COPY . .` trước `RUN pip install`, mỗi lần sửa code sẽ làm mất cache của bước cài thư viện, khiến build chậm hơn nhiều.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu lỗ hổng Python cho phép chạy lệnh tùy ý, kẻ tấn công trước hết có quyền của process trong container. Khi process chạy bằng root, kết hợp thêm Docker socket/bind mount nhạy cảm hoặc lỗ hổng thoát container, họ có thể tiến tới quyền cao trên host. `USER appuser` trong Dockerfile hạ quyền process xuống UID 10001 ngay từ bước đầu: mã bị khai thác không mặc nhiên đọc/ghi được các file chỉ root trong container. Biện pháp này giảm thiệt hại, dù không thay thế việc khóa mount và vá lỗ hổng kernel.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa **20 request**: gửi 10 lần lúc 10:00:59 rồi 10 lần lúc 10:01:00. Bộ đếm theo phút vừa reset nên cả hai nhóm đều hợp lệ dù chỉ cách nhau khoảng 2 giây. Sliding window của mình đếm 60 giây gần nhất trong Redis, nên nhóm thứ hai bị 429. Khi kiểm tra bản Railway bằng 15 request cùng user, mình nhận 10 lần 200 rồi 5 lần 429.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit chặn số request trong 60 giây; cost guard chặn tổng chi phí của một user trong tháng. Nếu hôm nay user mới gửi một request nhưng đã tiêu hết ngân sách tháng trước đó, rate limit cho qua còn cost guard trả 402. Ngược lại, 10 câu hỏi rất ngắn trong một phút có thể chưa tốn đáng kể so với ngân sách 10 USD, nhưng request thứ 11 vẫn bị rate limit trả 429. Trong `/ask`, mình kiểm tra cả hai trước khi gọi mock LLM để request bị chặn không phát sinh thêm chi phí.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp và để liveness kiểm tra Redis: (1) Redis ngắt; (2) cả 3 container vẫn chạy nhưng endpoint chung trả lỗi; (3) bộ giám sát coi cả 3 là hỏng và lần lượt restart chúng; (4) Redis vẫn chưa trở lại nên container mới cũng tiếp tục báo lỗi, làm cụm gián đoạn và tốn thời gian khởi động lại. Trong code hiện tại, `/health` chỉ báo process còn sống, còn `/ready` ping Redis. Khi Redis mất, `/ready` trả 503; nếu bộ điều phối dùng readiness probe, nó ngừng gửi request vào container mà không restart hàng loạt. Các container có thể sẵn sàng lại khi Redis hồi phục.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Compose hiện map cố định cổng host `8000`, nên mình giữ container `agent` đang chạy và tạo thêm 2 container `agent` tạm trong cùng mạng Compose để kiểm tra tương đương, rồi xóa hai container tạm. Gọi `/ask` một lần vào từng container với cùng `X-User-Id`, mình thấy `history_length` là **0, 2, 4** vì mỗi câu hỏi thêm một message `user` và một message `assistant` vào Redis chung. Nếu dùng dict Python riêng trong từng process, cả ba lần gọi đầu vào ba container khác nhau sẽ đều thấy 0; các lần sau phụ thuộc container nào nhận request và lịch sử sẽ mất khi container đó restart.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lúc tạo Railway project, mình dùng tên dài theo tên repository và CLI báo đúng dòng: `Project names must be between 1 and 32 characters.` Mình kiểm tra lại độ dài tên ngay từ thông báo này, rút thành `k4-day12-son` rồi chạy `railway init` lại. Một lần thử quá nhanh còn bị giới hạn tạo project trong 30 giây; chờ hết khoảng đó thì project được tạo, sau đó mình thêm Redis, gắn repo và kiểm tra `/health` cùng `/ready` đều trả 200

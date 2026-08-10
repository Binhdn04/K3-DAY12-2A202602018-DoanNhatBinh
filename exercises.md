# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: ..........................  Mã học viên: ..........................

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Ví dụ khi deploy lên Railway, tôi quên khai báo biến `AGENT_API_KEY`. Nếu để mặc định `"changeme"`, service vẫn khởi động và bất kỳ ai đoán được khóa mặc định đều có thể gọi `/ask`, gây tốn chi phí. Khi không có giá trị mặc định, Settings báo lỗi ngay lúc khởi động, deploy thất bại để tôi bổ sung secret trước khi service nhận request.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Ví dụ log JSON: `{"event":"ask_completed","level":"info","timestamp":"2026-08-10T03:15:00+00:00","user_id":"sv01","tokens_in":12,"tokens_out":24,"cost_usd":0.0001}`. Từ log này tôi có thể lọc và cộng `cost_usd` theo `user_id` để tìm người dùng tiêu nhiều nhất hoặc cảnh báo các event theo `level` trong một khoảng thời gian. Ngược lại, `print("đã trả lời xong")` không có trường dữ liệu để máy lọc, tổng hợp hay tạo cảnh báo đáng tin cậy.

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
| 1 stage (bản đầu) | 1190 MB |
| Multi-stage | 205 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Bản multi-stage nhỏ hơn vì image runtime không mang theo toàn bộ môi trường build. Stage builder dùng để cài dependency, sau đó runtime chỉ copy virtual environment /opt/venv và source code cần thiết. Trong bản 1-stage, tất cả những gì xuất hiện trong image build đều nằm trong image cuối, bao gồm base image lớn hơn và các file/cache/phụ thuộc phục vụ quá trình cài đặt. Với multi-stage, stage builder bị loại khỏi image runtime nên các thành phần chỉ cần lúc build không được đưa vào production image. Ngoài ra, bản multi-stage sử dụng python:3.11-slim thay vì python:3.11, nên ngay từ base image đã nhỏ hơn đáng kể.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi tôi chỉ sửa app/main.py, các layer liên quan đến requirements.txt vẫn được lấy từ cache. Layer COPY . . phải được tạo lại vì source code đã thay đổi. Các layer phía sau nó cũng không còn sử dụng cache theo chuỗi layer cũ. Nếu COPY . . được đặt trước RUN pip install, chỉ cần sửa một file Python cũng làm layer COPY thay đổi. Vì cache của Docker phụ thuộc vào các layer trước đó, layer pip install phía sau cũng phải chạy lại.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu ứng dụng Python có lỗ hổng cho phép Remote Code Execution, kẻ tấn công có thể thực thi command với cùng quyền của process chạy ứng dụng. Nếu container chạy mặc định bằng root thì process có UID 0, vì vậy attacker sau khi chiếm được ứng dụng cũng có quyền root bên trong container. Root trong container không đồng nghĩa ngay lập tức với root trên host vì Docker vẫn có namespace và các cơ chế isolation. Tuy nhiên, quyền root làm hậu quả nghiêm trọng hơn và tạo điều kiện cho các bước tiếp theo như khai thác kernel/container runtime vulnerability, truy cập volume nhạy cảm, Docker socket nếu bị mount sai, hoặc các cấu hình container không an toàn để thoát ra host. Lệnh USER appuser cắt chuỗi tấn công ngay sau bước chiếm quyền thực thi ứng dụng. Khi đó code của attacker chỉ chạy với quyền của user thường thay vì UID 0. Attacker vẫn có thể chiếm ứng dụng, nhưng phạm vi quyền hạn bị giảm đáng kể, làm container escape và thay đổi tài nguyên hệ thống khó hơn.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa là 20 request trong 2 giây. Người dùng gửi 10 request tại 10:00:59, ngay trước khi bộ đếm của phút 10:00 bị reset; sau đó gửi tiếp 10 request tại 10:01:01. Hai đợt thuộc hai phút đồng hồ khác nhau nên bộ đếm cố định cho phép cả 20 request. Sliding window 60 giây sẽ tính cả hai đợt trong cùng một cửa sổ và chỉ cho tối đa 10 request.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn số request trong một khoảng thời gian ngắn, còn cost guard giới hạn tổng chi phí của một người dùng trong tháng. Ví dụ user chỉ gửi 1 request trong phút nên chưa vượt rate limit, nhưng đã dùng 9,9 USD trong ngân sách 10 USD và request tiếp theo ước tính tốn 0,2 USD thì cost guard phải chặn. Ngược lại, user vẫn còn gần đủ ngân sách tháng nhưng gửi request thứ 11 trong cùng 60 giây khi giới hạn là 10/phút thì rate limit chặn, còn cost guard vẫn có thể cho qua.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> *Câu trả lời của bạn*

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> *Câu trả lời của bạn*

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> *Câu trả lời của bạn*

# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng mẫu "Câu trả lời của bạn" bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đoàn Tuấn Long  Mã học viên: 2A202602609

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống: lần đầu tạo service trên Railway, tôi kết nối repo GitHub nhưng
chưa vào tab Variables đặt `AGENT_API_KEY`.

- **Có mặc định `"changeme"`:** app vẫn khởi động, `/health` vẫn 200, dashboard
  báo xanh. Service lúc này có URL công khai với khóa là `"changeme"` — chuỗi
  nằm ngay trong source code trên GitHub public. Bot hoặc bất kỳ ai đọc repo
  đều gọi được `/ask`, và tôi chỉ phát hiện khi nhìn hóa đơn LLM cuối tháng.
- **Không có mặc định:** `Settings()` ném `ValidationError: agent_api_key Field
  required` ngay lúc import, container crash, Railway báo deploy thất bại. Lỗi
  hiện ra đúng lúc tôi đang nhìn màn hình deploy, sửa mất 1 phút (thêm biến,
  redeploy), và không có giây nào service chạy với khóa ai cũng biết.

Test `test_thieu_api_key_thi_fail_fast` kiểm tra đúng hành vi này. Các trường
không phải secret (`port`, `rate_limit_per_minute`...) thì vẫn có mặc định vì
thiếu chúng không gây rủi ro bảo mật.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log thật lấy từ `docker compose logs agent`:

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T09:08:42.309334+00:00", "user_id": "demo-1342830203", "tokens_in": 88, "tokens_out": 44, "cost_usd": 3.96e-05}
```

Hai việc làm được mà `print` không làm được:

1. **Tổng hợp chi phí theo user.** Lọc `event == "ask_completed"`, group by
   `user_id`, sum `cost_usd` → biết ngay user nào tiêu nhiều tiền nhất hôm nay.
   Với `print("đã trả lời xong")` không có user, không có con số nào để cộng.
2. **Đặt cảnh báo tự động theo trường.** Ví dụ: cảnh báo khi `level == "error"`
   vượt 5% số dòng trong 5 phút, hoặc khi `tokens_in` của một request vượt
   ngưỡng (dấu hiệu prompt bị nhồi / lịch sử phình to). Công cụ log trên cloud
   hiểu từng trường JSON; chuỗi tự do thì chỉ tìm kiếm text được.

Ngoài ra `timestamp` theo ISO-8601 UTC giúp sắp xếp đúng thứ tự khi log đến từ
nhiều container, và mỗi event nằm trên **một dòng** nên không bị cắt vụn.

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
| 1 stage (bản đầu) | 1730 MB (1.73GB) |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Chênh lệch khoảng 1.46GB (image nhỏ đi ~6.4 lần). Bản 1 stage lấy từ Dockerfile
gốc (`git show 306b897:Dockerfile`), phần chênh gồm:

- **Base image đầy đủ `python:3.11`** (Debian đầy đủ): có sẵn `gcc`, `make`,
  header phát triển, `git`, các thư viện `lib*-dev`, man page... — thứ chỉ cần
  khi *biên dịch*, không cần khi *chạy*. Đây là phần lớn nhất. Bản multi-stage
  dùng `python:3.11-slim` cho cả hai stage.
- **Cache của pip**: bản gốc chạy `pip install` không có `--no-cache-dir`, nên
  file wheel đã tải về nằm lại trong image.
- **`COPY . .` trước khi có `.dockerignore` đầy đủ**: kéo theo `.git`, `.venv`
  (nếu có), `__pycache__`, `tests/`, tài liệu... Bản mới chỉ copy `app/` và
  `utils/`.

Stage `builder` cài dependency vào `/install`, stage `runtime` chỉ
`COPY --from=builder /install /usr/local` — mọi thứ phát sinh trong lúc build
bị bỏ lại cùng stage builder.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Tôi thêm một dòng comment vào cuối `app/main.py` rồi chạy
`docker build --progress=plain -t agent:multi .`. Kết quả quan sát được:

- **CACHED:** `WORKDIR /build`, `COPY requirements.txt .`,
  `RUN pip install ...`, `WORKDIR /app`, `COPY --from=builder /install`,
  `RUN useradd ...`.
- **Chạy lại:** chỉ `COPY app ./app` và `COPY utils ./utils` (mỗi bước 0.1s).
  Tổng thời gian build **0.5 giây**.

Docker tính cache theo nội dung đầu vào của từng layer, và khi một layer đổi
thì mọi layer *phía sau* nó đều phải build lại. `requirements.txt` không đổi
nên layer `pip install` giữ nguyên cache.

Nếu đặt `COPY . .` trước `RUN pip install`: sửa một ký tự trong `main.py` làm
layer `COPY . .` đổi → layer `pip install` đứng sau cũng mất cache → cài lại
toàn bộ FastAPI, uvicorn, redis, pydantic... mỗi lần build (khoảng 1–2 phút
thay vì 0.5 giây). Trên CI/CD, chênh lệch đó nhân với mỗi lần push.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện khi container chạy root:

1. App có lỗ hổng cho phép thực thi lệnh (ví dụ một thư viện phụ thuộc bị lỗi
   deserialize, hoặc code lỡ đưa input người dùng vào `subprocess`/`eval`).
2. Kẻ tấn công chạy được shell **với quyền của process Python** — ở đây là
   **root (UID 0)** trong container.
3. Là root trong container, họ đọc/ghi được mọi file: biến môi trường chứa
   `AGENT_API_KEY`, cài công cụ, sửa code app để cài backdoor.
4. Container chia sẻ kernel với host. UID 0 trong container cũng là UID 0 trên
   host nếu không bật user namespace. Chỉ cần thêm một điểm yếu — volume mount
   thư mục host, mount `/var/run/docker.sock`, container chạy `--privileged`,
   hoặc một lỗ hổng kernel — là họ thoát ra và **thành root trên host**, điều
   khiển cả các container khác.

`USER appuser` (UID 10001, tôi kiểm tra bằng `docker run --rm --entrypoint
whoami` ra `appuser`) cắt chuỗi ở **bước 2–3**: shell kẻ tấn công chỉ có quyền
user thường, không ghi được vào thư mục hệ thống, không cài được gói, và nếu có
thoát khỏi container thì cũng chỉ là UID 10001 không đặc quyền trên host. Lỗ
hổng vẫn là lỗ hổng, nhưng thiệt hại bị giới hạn.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

**20 request trong 2 giây** — gấp đôi hạn mức.

Cách làm: gửi 10 request lúc `10:00:59` (bộ đếm của phút 10:00 lên 10/10, vẫn
hợp lệ). Sang `10:01:00` bộ đếm reset về 0. Gửi tiếp 10 request lúc
`10:01:00`–`10:01:01` (phút 10:01 cũng 10/10). Mỗi "phút đồng hồ" đều đúng
luật, nhưng trong khoảng 2 giây thực tế server nhận 20 request.

Với sliding window, request lúc `10:01:01` nhìn lại 60 giây trước đó (từ
`10:00:01`) và thấy đã có 10 request → trả 429. Không có ranh giới nào để lợi
dụng. Tôi thấy đúng hành vi này trên bản deploy Railway: gọi liên tiếp với cùng
user thì sau lượt thứ 10 là `429 429 429...`.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

**Khác nhau:** rate limit giới hạn **số lượng request theo thời gian ngắn**
(10 request / 60 giây, trả 429, tự hết sau 1 phút). Cost guard giới hạn **tổng
số tiền theo tháng** (10 USD / user / tháng, trả 402, chỉ reset khi sang tháng
mới vì key có dạng `cost:<user>:<YYYY-MM>`). Một cái chống spam tốc độ, một cái
chống cháy ngân sách.

- **Rate limit cho qua, cost guard chặn:** user gửi đều đặn 8 request/phút,
  mỗi request kèm câu hỏi rất dài và lịch sử 20 message (hàng chục nghìn
  token). Không lúc nào vượt 10/phút nên rate limit luôn cho qua, nhưng sau vài
  giờ tổng `cost_usd` chạm 10 USD → cost guard trả 402.
- **Cost guard cho qua, rate limit chặn:** một script lỗi gọi `/ask` 50 lần
  trong 5 giây với câu hỏi ngắn `"hi"`. Mỗi request chỉ tốn khoảng 0.00002 USD
  nên ngân sách gần như chưa bị đụng tới, nhưng từ request thứ 11 rate limit
  trả 429 để bảo vệ server và các user khác.

Cả hai đều chạy **trước** `ask_llm`, vì tiền mất ở bước gọi LLM.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

1. **Giây 0:** Redis mất kết nối. Cả 3 container app vẫn khỏe (process sống,
   CPU/RAM bình thường).
2. **Vài giây sau:** health check của orchestrator gọi endpoint gộp → endpoint
   ping Redis thất bại → cả 3 container đồng loạt trả 503.
3. **Sau N lần thất bại liên tiếp** (ví dụ `retries: 3`, `interval: 10s` →
   khoảng 30 giây): orchestrator kết luận cả 3 container "chết" và **restart cả
   3 cùng lúc**.
4. **Trong lúc restart:** không còn container nào nhận request — người dùng
   thấy 502/503 cho *mọi* endpoint, kể cả những endpoint không cần Redis.
5. **Redis quay lại ở giây 30** nhưng các container vẫn đang khởi động lại;
   nếu Redis còn chập chờn, chúng lại fail health check và bị restart tiếp —
   vòng lặp restart.

Kết quả: sự cố 30 giây của Redis thành sự cố toàn hệ thống kéo dài hơn.

Khi tách riêng như bài làm: `/health` (liveness) không đụng Redis nên vẫn 200
→ **không container nào bị restart**. `/ready` (readiness) trả 503 → load
balancer chỉ tạm **ngừng gửi traffic**. Redis sống lại → `/ready` về 200 →
traffic quay lại ngay, không mất thời gian khởi động lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Tôi chạy 3 container `agent` sau nginx (`nginx/nginx.conf`, round-robin) và gọi
`/ask` 6 lần với cùng user `scale-1301262430`. Kết quả thật:

- `history_length`: **0, 2, 4, 6, 8, 10** — tăng đều 2 sau mỗi lượt.
- Đếm trong `docker logs` từng container: agent-1 xử lý 2 request, agent-2 xử
  lý 2, agent-3 xử lý 2.

Nghĩa là các request liên tiếp rơi vào 3 container khác nhau, nhưng container
nào cũng đọc được lịch sử do container khác ghi, vì lịch sử nằm trong Redis
(`history:<user_id>`). Tôi cũng thử `docker compose restart agent`: lượt hỏi
tiếp theo vẫn thấy `history_length=6`, lịch sử không mất khi container chết.

Nếu lưu trong dict Python, mỗi container có RAM riêng nên mỗi container chỉ
thấy phần lịch sử nó tự ghi. Với round-robin 3 container, dãy số sẽ kiểu
**0, 0, 0, 2, 2, 2** thay vì 0, 2, 4, 6, 8, 10: agent "quên" hội thoại ngẫu
nhiên tùy request rơi vào đâu. Restart container thì về 0 hết.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

**Lỗi:** lần deploy đầu tiên trên Railway (kết nối thẳng repo GitHub), service
báo **CRASHED** ngay sau khi build xong, dashboard ghi "Crashed 26 seconds ago".

**Tìm nguyên nhân:**

1. Trong tab Deployments, bản deploy ghi commit **"Correct submission naming to
   L3A"** — đây là commit gốc của đề, không phải code tôi đã làm.
2. Chạy `git status` trên máy: 13 file (`app/*.py`, `Dockerfile`,
   `railway.toml`...) đang ở trạng thái đã sửa nhưng **chưa commit**.
   `git ls-remote` xác nhận nhánh main trên GitHub vẫn ở commit `306b897`.
3. Vậy Railway đang chạy code template còn `raise NotImplementedError`. Lỗi
   này tôi đã thấy khi chạy thử container local ở CP2:
   `NotImplementedError: TODO (CP4): cài đặt install` từ `lifecycle.install()`
   trong `lifespan` → `Application startup failed. Exiting.`

**Sửa:**

- Commit và push code CP1–CP4 lên GitHub (commit `a7736a7`), Railway tự build
  lại từ commit mới.
- Thêm Redis vào project và đặt biến ở tab Variables: `AGENT_API_KEY`,
  `REDIS_URL=${{Redis.REDIS_URL}}`, không đặt `PORT` để Railway tự gán.
- Bỏ `startCommand` trong `railway.toml` để dùng `CMD` của Dockerfile
  (`exec uvicorn ... --port ${PORT:-8000}`), giữ uvicorn là PID 1 nhận SIGTERM.
- Generate Domain với port 8080.

Kết quả: `/health` 200, `/ready` 200 `{"redis": true}`, `/ask` không key 401,
gọi liên tục thì 429 sau lượt thứ 10; `pytest tests/test_cp5.py` 9/9 pass.

Bài học: "deploy từ GitHub" nghĩa là deploy **cái đã push**, không phải cái
đang có trên máy. Đây cũng là lý do CI/CD nên chạy test trên chính commit sẽ
được deploy.

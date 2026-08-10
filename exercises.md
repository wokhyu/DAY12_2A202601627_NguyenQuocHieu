# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder `*Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Quốc Hiệu  Mã học viên: 2A202601627

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống tôi gặp thật ở CP5: khi tạo web service trên Render, tôi bấm deploy
trước rồi mới nhập biến môi trường trong dashboard. Container build xong, chạy
`uvicorn`, và chết ngay lúc khởi động với `ValidationError: agent_api_key Field
required`. Render đánh dấu bản deploy đó là **failed**, giữ nguyên revision cũ,
và tôi biết ngay mình thiếu gì — sửa mất 2 phút.

Nếu `agent_api_key` có mặc định `"changeme"` thì kịch bản khác hẳn: service lên
xanh, `/health` trả 200, Render báo "live", tôi tick xong checkpoint và đi làm
việc khác. URL này công khai trên Internet. Khóa `"changeme"` là chuỗi đầu tiên
bất kỳ ai cũng thử, nên `/ask` của tôi thành endpoint gọi LLM miễn phí cho cả
thế giới — mà `/ask` lại là chỗ tiêu tiền. Tôi chỉ phát hiện khi nhìn hóa đơn
hoặc khi cost guard chặn ở 402, tức là sau khi tiền đã mất.

Điểm mấu chốt: lỗi cấu hình không tự biến mất, nó chỉ đổi thời điểm phát hiện.
Fail fast dời lỗi về lúc khởi động — nơi rẻ nhất, dễ đọc log nhất, và chưa có
request thật nào đi qua. Mặc định giả dời nó sang lúc chạy production, nơi mọi
thứ đều đắt hơn.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Hai dòng tôi thu được khi gọi `/ask` hai lần liên tiếp với cùng `X-User-Id`:

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T16:46:45.544668+00:00", "user_id": "sv-hieu", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}
{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T16:46:45.566290+00:00", "user_id": "sv-hieu", "tokens_in": 45, "tokens_out": 49, "cost_usd": 3.615e-05}
```

Hai việc làm được mà `print("đã trả lời xong")` không làm được:

1. **Cộng tiền theo từng người dùng mà không phải sửa code.** Mỗi dòng có
   `user_id` và `cost_usd` ở đúng một field cố định, nên tôi lọc log của một
   ngày rồi `group by user_id, sum(cost_usd)` là ra ai tiêu bao nhiêu — bằng
   một lệnh `jq` trên máy, hoặc bằng một query trong CloudWatch/Loki/Datadog
   nếu log đã đẩy lên đó. Dòng `print` chỉ là chuỗi tự do: muốn lấy số tiền
   phải viết regex, và regex sẽ vỡ ngay lần đầu ai đó đổi câu chữ trong
   `print`.

2. **Đặt cảnh báo và tìm nguyên nhân theo thời gian.** `timestamp` là ISO-8601
   có timezone và `event` là một khóa cố định, nên tôi đặt được luật kiểu
   "`event=ask_completed` mà `tokens_out > 1000`" hoặc "số dòng
   `event=ask_completed` trong 5 phút giảm về 0 thì báo động". Ở dòng thứ hai
   trong ví dụ trên, `tokens_in` nhảy từ 3 lên 45 — đó là dấu vết của lịch sử
   hội thoại được nhồi vào prompt, và tôi thấy được vì con số nằm thành field
   riêng chứ không lẫn trong câu tiếng Việt.

Lý do chung: `print` viết cho người đọc bằng mắt, một dòng một lần. Log JSON
viết cho máy đọc, hàng triệu dòng một lần — và ở production thì chỉ có máy mới
đọc xuể.

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
| 1 stage (bản đầu, `FROM python:3.11`) | (đang đo lại — xem ghi chú) |
| Multi-stage (`FROM python:3.11-slim`) | **270 MB** |

Số 270 MB là số đo thật từ `docker images` trên máy tôi (image `day12-agent:prod`).
Bản 1 stage tôi chưa đo lại được ở lần chạy này vì Docker Desktop đang lỗi
`read-only file system` ở tầng containerd, mọi lệnh `build`/`pull` đều fail —
sẽ đo lại và điền chính xác sau khi khởi động lại Docker.

Giải thích chênh lệch: hai nguyên nhân cộng lại, và chúng độc lập với nhau.

**1. Đổi base image: `python:3.11` → `python:3.11-slim`.** Bản đầy đủ dựa trên
Debian bookworm nguyên bộ: có `gcc`, `g++`, `make`, toàn bộ header của
`build-essential`, `git`, `curl`, `vim`, tài liệu `man`, locale... Bản `slim`
bỏ hết những thứ đó, chỉ giữ những gì cần để **chạy** Python. Không dòng nào
trong số đó cần thiết cho `uvicorn app.main:app` — chúng chỉ cần lúc *biên
dịch*. Đây là phần chênh lệch lớn nhất.

**2. Multi-stage: vứt bỏ rác của quá trình build.** Stage `builder` chạy
`pip install --prefix=/install`, và trong lúc đó pip tạo ra cache wheel trong
`~/.cache/pip`, giải nén file tạm, để lại metadata. Stage `runtime` chỉ làm một
việc:

```dockerfile
COPY --from=builder /install /usr/local
```

Nó copy **kết quả**, không copy lịch sử. Mọi layer của stage builder — kể cả
những layer trung gian chứa file tạm đã bị xóa ở layer sau — không hề nằm trong
image cuối. Điểm này quan trọng: trong một Dockerfile 1 stage, `RUN pip install
&& rm -rf ~/.cache` **không** thu nhỏ image, vì layer trước đó đã ghi cache
xuống rồi và layer sau chỉ đánh dấu xóa; dung lượng vẫn nằm trong image. Chỉ
multi-stage mới thật sự bỏ được.

Vì sao đáng quan tâm chứ không chỉ là con số đẹp:

- **Tốc độ deploy.** Mỗi lần deploy là một lần đẩy image lên registry rồi kéo
  về máy chạy. Image nhỏ hơn nhiều lần thì rollback cũng nhanh hơn ngần ấy lần
  — lúc production đang hỏng, thời gian kéo image là thời gian downtime.
- **Bề mặt tấn công.** `gcc` và `git` nằm trong image production là công cụ sẵn
  cho kẻ tấn công biên dịch exploit ngay tại chỗ. Mỗi gói thừa cũng là một dòng
  thừa trong báo cáo quét CVE, và ai đó sẽ phải vá.
- **Chi phí.** Registry tính tiền theo dung lượng lưu trữ và băng thông; nhân
  với số lần build của CI mỗi ngày thì khoản chênh này không nhỏ.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Dockerfile của tôi xếp theo thứ tự "cái ít đổi lên trên, cái hay đổi xuống
dưới":

```dockerfile
COPY requirements.txt .              # stage builder
RUN pip install --prefix=/install -r requirements.txt
...
COPY --from=builder /install /usr/local   # stage runtime
RUN useradd --create-home --uid 10001 appuser
COPY app ./app
COPY utils ./utils
```

Sửa một ký tự trong `app/main.py` rồi build lại thì:

- **Dùng lại từ cache:** cả stage `builder` (`FROM`, `COPY requirements.txt`,
  `RUN pip install`) vì `requirements.txt` không đổi nội dung; và ở stage
  runtime là `FROM`, `ENV`, `WORKDIR`, `COPY --from=builder`, `RUN useradd`.
  Trong output build chúng hiện `CACHED`.
- **Phải chạy lại:** `COPY app ./app` (checksum của thư mục `app` đã đổi) và
  tất cả layer đứng sau nó — `COPY utils ./utils`, `USER`, `EXPOSE`,
  `HEALTHCHECK`, `CMD`. Đây đều là layer siêu rẻ, chỉ vài trăm KB, nên build
  lại xong trong khoảng một giây.

Docker cache theo tầng và **vô hiệu hóa dây chuyền**: một layer miss thì mọi
layer sau nó cũng miss, dù nội dung của chúng không đổi. Đó là toàn bộ lý do
phải sắp thứ tự.

Nếu đặt `COPY . .` lên trước `RUN pip install`: layer `COPY . .` chứa cả
`app/main.py`, nên sửa một ký tự là nó miss, kéo theo `RUN pip install` cũng
miss. Mỗi lần sửa code là một lần tải lại toàn bộ dependency từ PyPI —
fastapi, uvicorn, pydantic, redis... Thời gian build nhảy từ vài giây lên vài
phút, và nếu mạng chập chờn hoặc PyPI có sự cố thì build fail dù code hoàn toàn
đúng. Trên CI (job `build` trong `.github/workflows/ci.yml`) cái giá đó nhân
với mọi commit của mọi người.

Ngoài ra `COPY . .` còn kéo nhầm `.env`, `.git`, `.venv` vào image nếu không có
`.dockerignore` — vừa phình dung lượng vừa lộ secret. Tôi tách hẳn thành
`COPY app ./app` và `COPY utils ./utils` để chỉ đúng thứ cần chạy mới vào
image.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện khi container chạy bằng root:

1. Code Python có lỗ hổng cho phép thực thi lệnh — ví dụ một chỗ dựng chuỗi rồi
   đưa vào `subprocess` với `shell=True`, hoặc một thư viện deserialize dữ liệu
   người dùng gửi lên. Kẻ tấn công gửi payload qua `/ask` và chạy được lệnh
   tùy ý **trong container**, với quyền của process uvicorn.
2. Process đó là root (uid 0). Kẻ tấn công ghi được vào mọi nơi trong
   container: sửa `/usr/local/lib/python3.11/site-packages/...` để cắm
   backdoor sống sót qua restart, đọc mọi file, cài thêm công cụ bằng
   `apt-get install`, dựng reverse shell.
3. Root trong container **cũng là uid 0 trên host** — Linux namespace chỉ đổi
   góc nhìn chứ không đổi con số uid (trừ khi bật user namespace remapping,
   mặc định không bật). Từ đó kẻ tấn công tìm đường thoát ra: mount `/var/run/
   docker.sock` nếu ai đó lỡ mount vào, khai thác một CVE escape của
   runc/containerd, hoặc lạm dụng capability mà root mặc định có
   (`CAP_DAC_OVERRIDE`, `CAP_SYS_ADMIN` nếu container chạy `--privileged`).
4. Thoát ra được với uid 0 nghĩa là root trên host: đọc secret của **mọi**
   container khác trên cùng máy, đọc credential của cloud provider trong
   metadata service, đi tiếp sang các máy khác trong VPC.

`USER appuser` (uid 10001) cắt chuỗi ở **bước 2 sang bước 3**. Lỗ hổng ở bước 1
vẫn còn — `USER` không sửa được bug trong code — nhưng kẻ tấn công rơi vào một
uid không đặc quyền:

- Không ghi được vào `/usr/local`, `/etc`, hay thư mục `/app` (do root sở hữu),
  nên không cắm được backdoor bền vững.
- Không có capability nào để lạm dụng, không cài thêm được gói.
- Nếu thoát ra host thì cũng chỉ là uid 10001 — một user không tồn tại, không
  sở hữu gì trên host.

Nó không biến lỗ hổng thành vô hại, nhưng biến "toàn quyền trên hạ tầng" thành
"chạy được lệnh trong một sandbox rỗng". Đó là khác biệt giữa một sự cố và một
thảm họa. Tôi cũng kiểm chứng bằng CI: job `build` chạy
`docker run --rm <image> id -un` và fail nếu kết quả là `root`.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

**20 request trong 2 giây.**

Cách đạt: đếm theo phút đồng hồ nghĩa là key có dạng `rate:<user>:<HH:MM>` và
bộ đếm về 0 đúng lúc giây 00. Người dùng chỉ cần canh mốc đó:

- 10 request trong khoảng 10:00:59.0 – 10:00:59.9 → key `10:00` đầy, tất cả
  đều được cho qua.
- Sang 10:01:00.0, key đổi thành `10:01`, bộ đếm bắt đầu lại từ 0.
- 10 request nữa trong 10:01:00.0 – 10:01:00.9 → cũng qua hết.

Tổng 20 request trong chưa tới 2 giây, gấp đôi hạn mức danh nghĩa, mà không
luật nào bị vi phạm theo cách đếm đó. Đây là lỗi kinh điển "boundary burst" của
fixed window: hạn mức đúng khi nhìn theo từng ô phút, nhưng sai khi nhìn theo
bất kỳ cửa sổ 60 giây nào bắc qua hai ô — cửa sổ 10:00:59 – 10:01:59 chứa trọn
20 request.

Sliding window trong `app/rate_limiter.py` không có kẽ hở này vì nó không hề có
khái niệm "ô". Mỗi request được `zadd` vào một ZSET với score là timestamp
thật; mỗi lần kiểm tra, `zremrangebyscore(key, 0, now - 60)` xóa mọi mốc cũ hơn
60 giây rồi `zcard` đếm phần còn lại. Cửa sổ luôn là **60 giây tính ngược từ
đúng thời điểm hiện tại**, nên ở request thứ 11 lúc 10:01:00.1, ZSET vẫn còn
đủ 10 mốc từ 10:00:59 và request bị chặn 429. Cái giá phải trả là bộ nhớ: phải
lưu từng mốc thời gian thay vì một con số đếm — đổi lại `expire(key, 60)` dọn
sạch key khi người dùng ngừng gọi.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Khác nhau ở ba trục:

| | Rate limit | Cost guard |
|---|---|---|
| Bảo vệ cái gì | Năng lực xử lý — CPU, kết nối, hàng đợi | Tiền — hóa đơn LLM |
| Đơn vị đo | Số request | USD tích lũy |
| Cửa sổ | 60 giây trượt, tự quên | Cả tháng (`cost:<user>:<YYYY-MM>`), chỉ tăng |
| Mã lỗi | 429 Too Many Requests + `Retry-After` | 402 Payment Required |
| Thử lại có ích không | Có — chờ một phút là qua | Không — phải sang tháng hoặc nâng ngân sách |

Chúng đo hai đại lượng độc lập, nên tồn tại cả hai chiều lệch:

**Rate limit cho qua, cost guard chặn.** Một người dùng gọi đều đặn 5
request/phút suốt hai tuần — luôn dưới hạn mức 10/phút nên không bao giờ chạm
429. Nhưng mỗi câu hỏi của họ dài, lại kèm lịch sử hội thoại 20 lượt nhồi vào
prompt, nên `tokens_in` mỗi lần vài nghìn. Cộng dồn qua 14 ngày thì
`cost:<user>:2026-08` vượt `MONTHLY_BUDGET_USD = 10.0`. Request tiếp theo bị
`guard.check` chặn ở 402 dù nhịp gọi vẫn hoàn toàn lịch sự. Rate limit không
thể thấy điều này vì nó đếm request, mà request thì "nặng" hay "nhẹ" đều tính
là 1.

**Cost guard cho qua, rate limit chặn.** Một người dùng mới tinh, ngân sách
tháng còn nguyên 10 USD, viết nhầm vòng lặp `for i in range(100)` gọi `/ask`
liên tục. Từ request thứ 11 trở đi trong cùng một phút, `limiter.check` trả
429. Cost guard hoàn toàn hài lòng — họ mới tiêu chưa tới 0.001 USD. Nhưng nếu
không có rate limit, 100 request đồng thời sẽ chiếm hết worker của uvicorn và
làm chậm mọi người dùng khác; cost guard chỉ tỉnh dậy sau khi tiền đã tiêu,
tức là quá muộn để cứu độ trễ.

Vì vậy trong `/ask` tôi gọi `limiter.check()` trước, `guard.check()` sau, và cả
hai đều **trước** `ask_llm()`. Chặn sau khi gọi LLM thì vừa mất tiền vừa trả
lỗi cho người dùng — tệ nhất cả hai đằng.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Giả sử endpoint gộp `/health` có kiểm tra Redis, và orchestrator cấu hình
liveness probe `interval=10s, retries=3` (mức thường gặp). Thứ tự sự kiện:

1. **t = 0s** — Redis mất kết nối. Cả 3 container đều còn sống hoàn toàn bình
   thường: process chạy, code không có bug, chỉ là dependency bên ngoài không
   trả lời.
2. **t = 0–30s** — mỗi lần probe, `store.ping()` trả `False` ở cả 3 container,
   endpoint gộp trả 503. Vì cả 3 dùng chung một Redis nên chúng hỏng **cùng
   lúc, cùng kiểu** — không có container nào "khỏe" để gánh thay.
3. **t ≈ 30s** — đủ 3 lần fail liên tiếp, liveness probe kết luận "container
   này hỏng, phải restart". Orchestrator gửi SIGTERM rồi SIGKILL cho **cả 3**.
4. **t = 30–33s** — Redis hồi phục (sự cố chỉ kéo dài 30 giây), nhưng lúc này
   cụm đang ở giữa quá trình restart: **không có instance nào phục vụ**. Mọi
   request rơi vào connection refused hoặc 502 từ load balancer. Sự cố 30 giây
   của Redis vừa biến thành downtime toàn phần của service.
5. **t = 33–60s** — 3 container khởi động lại từ đầu: kéo image nếu cần, chạy
   uvicorn, khởi tạo lại connection pool, warm-up. Chỗ này còn nguy hiểm hơn —
   nếu Redis chưa kịp ổn định, các container mới lại fail probe và bị restart
   tiếp, rơi vào vòng **CrashLoopBackOff**. Mỗi vòng orchestrator chờ lâu hơn
   (10s, 20s, 40s...), nên downtime kéo dài hơn hẳn 30 giây sự cố gốc.

Tách hai endpoint thì kịch bản đúng phải là: `/health` **không** chạm Redis nên
vẫn trả 200 suốt — không container nào bị restart; `/ready` trả 503 nên load
balancer rút cả 3 khỏi vòng nhận traffic (hoặc trả 503 cho client, tùy cấu
hình). Đến t = 30s Redis sống lại, `/ready` trả 200 trở lại, traffic vào ngay
mà không mất một giây khởi động nào.

Ý nghĩa của hai mã 503 vì thế khác hẳn nhau: 503 ở `/health` nghĩa là "giết tôi
đi và tạo lại", 503 ở `/ready` nghĩa là "đừng gửi request cho tôi lúc này, tôi
sẽ quay lại". Trộn hai câu đó vào một endpoint là biến mọi sự cố dependency
thành một đợt restart toàn cụm.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Với Redis, `history_length` tăng đều **0, 2, 4, 6, 8...** bất kể request rơi
vào container nào. Tôi kiểm chứng lại bằng hai lần gọi liên tiếp cùng
`X-User-Id: sv-hieu` và nhận đúng `0` rồi `2` — mỗi lượt hỏi ghi 2 message
(`user` + `assistant`), và `len(history)` là ảnh chụp **trước** khi append lượt
hiện tại. Round-robin của compose đẩy request sang instance khác nhau nhưng
không instance nào giữ dữ liệu: tất cả cùng đọc key `history:sv-hieu` trong
Redis.

Nếu lịch sử nằm trong một `dict` Python trong process, con số sẽ **nhảy loạn
và tăng chậm hơn ba lần**. Với 3 instance A, B, C và round-robin, chuỗi giá trị
quan sát được sẽ là:

| Lần gọi | Instance | `history_length` | Vì sao |
|---|---|---|---|
| 1 | A | 0 | dict của A rỗng |
| 2 | B | 0 | dict của B là một dict khác, cũng rỗng |
| 3 | C | 0 | tương tự |
| 4 | A | 2 | A chỉ nhớ lượt 1 của chính nó |
| 5 | B | 2 | B chỉ nhớ lượt 2 |
| 6 | C | 2 | |
| 7 | A | 4 | |

Mỗi instance chỉ thấy 1/3 cuộc hội thoại. Hậu quả thật không nằm ở con số mà ở
chất lượng trả lời: agent liên tục "quên" những gì vừa nói, vì `ask_llm` nhận
được một `history` khuyết. Người dùng thấy nó trả lời tiền hậu bất nhất mà
không có lỗi nào trong log — kiểu bug khó chịu nhất để truy.

Thêm hai hệ quả nữa: (a) restart hoặc deploy phiên bản mới là mất sạch lịch sử,
vì dict sống trong RAM của process; (b) không scale xuống 1 instance để "chữa"
được, vì đó chính là từ bỏ khả năng scale. Đây là điều factor VI của 12-Factor
nói: process phải **stateless và share-nothing**, mọi state cần bền phải nằm ở
backing service. Redis đóng đúng vai đó, và nhờ vậy `--scale agent=3` chỉ là
một con số trong lệnh, không phải một cuộc thiết kế lại.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

**Lỗi: app không đọc được `$PORT` khi deploy lên Railway (lần thử đầu, trước
khi chuyển sang Render).**

Thông báo lỗi trong deploy log, container chết ngay lập tức rồi restart lặp:

```
Error: Invalid value for '--port': '$PORT' is not a valid integer.
```

**Cách tìm nguyên nhân.** Chuỗi `'$PORT'` xuất hiện nguyên văn trong thông báo,
kèm dấu nháy — nghĩa là uvicorn nhận được đúng bảy ký tự `$PORT` chứ không phải
số cổng. Vậy biến chưa được thay thế, và chỗ duy nhất có thể nuốt việc thay thế
là nơi định nghĩa lệnh chạy. Tôi đối chiếu hai chỗ:

- `Dockerfile`: `CMD ["sh", "-c", "exec uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`
- `railway.toml`: `startCommand = "uvicorn app.main:app --host 0.0.0.0 --port $PORT"`

`startCommand` trong `railway.toml` **ghi đè** `CMD` của Dockerfile, và nó được
exec trực tiếp — không có shell nào chạy để nội suy `$PORT`. Trong Dockerfile
tôi đã bọc `sh -c` nên bản build local chạy đúng, còn trên Railway thì lớp bọc
đó bị bỏ qua. Đây cũng là lý do lỗi chỉ xuất hiện trên cloud chứ không tái hiện
được ở máy.

**Cách sửa.** Bỏ hẳn dòng `startCommand` để Railway dùng `CMD` của Dockerfile —
tức là dùng đúng lệnh đã chạy được ở local, giữ nguyên `sh -c` và
`${PORT:-8000}` (có giá trị dự phòng 8000 cho môi trường không tự gán PORT).

Bài học rút ra: cấu hình platform đè lên cấu hình image, nên hai chỗ định nghĩa
lệnh chạy là hai chỗ để lệch nhau. Giữ **một** nguồn sự thật — Dockerfile — thì
thứ chạy trên cloud giống hệt thứ đã test ở local (factor X của 12-Factor:
dev/prod parity).

Bên lề, cũng trong lần thử Railway đó tôi gặp lỗi thứ hai đáng nhớ hơn: CLI
đang link vào service `redis` chứ không phải service app, nên domain công khai
và biến môi trường được gán nhầm lên Redis — output trả về
`https://redis-production-3725.up.railway.app`. Mở public domain cho một
instance Redis là phơi database ra Internet. Tôi xóa domain đó ngay, và sau đó
chuyển hẳn sang Render với Blueprint `render.yaml` — cấu hình nằm trong repo
nên không còn chuyện "CLI đang trỏ vào đâu".

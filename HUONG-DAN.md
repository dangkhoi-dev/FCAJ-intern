# Hướng dẫn: ba task AI-01, AI-02, AI-09

Đọc file này trước. Nó nói mỗi task giao cái gì, chạy lệnh nào, kiểm tra ra sao,
và nộp cái gì lên task board.

Cả gói có 50 file nhưng chỉ **4 file là tài liệu để đọc**. Phần còn lại là một
script, sáu Dockerfile, hai file compose, và các file `__init__.py` rỗng đánh dấu
khung thư mục.

```
HUONG-DAN.md                                    ← đang đọc
AI-01-repo-ai/
├── setup-repo-ai.sh                            chạy một lệnh, dựng xong nhánh
└── payload/                                    script tự copy vào repo
    ├── README.md                               README gốc repo
    ├── docs/ARCHITECTURE.md                    ★ tài liệu chính, 8 mục
    ├── pyproject.toml  .importlinter  .dockerignore
    ├── deploy/Dockerfile.*  deploy/requirements/api.txt
    ├── tools/check_layout.py  tools/migrate.py
    └── src/catspeak_ai/…                       khung core, read, write, api, shared
                                                + core/migrations/0001_init_ai.sql
AI-02-repo-devops/catspeak-devops/              đẩy nguyên thư mục này lên GitHub
├── README.md                                   ★ tài liệu của repo devops
├── docker-compose.yml  docker-compose.dev.yml
├── .env.example  env/*.env.example
├── infra/postgres/01-extensions.sql  up.sh
AI-09-chi-phi/
└── CHI_PHI_DICH_VU_NGOAI.md                    ★ STT + LLM + TTS
```

Bốn tài liệu đánh dấu ★. Ba tài liệu cũ (`REPO_STRUCTURE.md`, `CONTRACTS.md`,
`AGENT_REPO_RULES.md`, `moderation-data.md`) đã gộp hết vào `docs/ARCHITECTURE.md`;
bỏ gói cũ đi.

---

## Nếu chỉ có 15 phút

1. `AI-09-chi-phi/CHI_PHI_DICH_VU_NGOAI.md` — đọc phần "Kết luận trước" và mục 5.
2. `AI-01-repo-ai/payload/docs/ARCHITECTURE.md` — đọc mục 1 (bốn tầng) và mục 3
   (đặt file mới vào đâu).
3. `AI-02-repo-devops/catspeak-devops/README.md` — đọc mục 5 (còn thiếu gì).

---

# AI-01 — Setup repository AI

## Giao cái gì

Một script gom ba nhánh đang lệch nhau của `catspeak-ai` rồi dựng lại cây thư mục
theo kiến trúc bốn tầng, cộng khung code cho `core`, `read`, `write`, `api`, cộng
migration của database AI riêng, cộng hai công cụ kiểm tra ranh giới tầng.

Cây thư mục đích:

```
src/catspeak_ai/
├── core/        TẦNG DATA — chỗ duy nhất biết về database
│   ├── interfaces/    port: Protocol và dataclass, không SQL
│   ├── repositories/  adapter: SQL, pgvector, Redis, MinIO
│   ├── ingest/        consumer đón dữ liệu vào
│   ├── jobs/          job nền, gồm tổng hợp báo cáo buổi học
│   └── migrations/    migration DB AI (0001_init_ai.sql)
├── read/        chỉ gọi core.interfaces
├── write/       chỉ gọi core.interfaces
│   └── suggestion/    worker gợi ý và tóm tắt (từ assistant-llm)
├── api/         FastAPI, gom endpoint đọc và ghi
├── moderation/  kiểm duyệt (từ moderation)
├── agents/      ai_tutor, room_stt, assistant_stt
└── shared/      config, log, client nhà cung cấp ngoài
```

Sáu container ra từ một repo: `ai-api`, `ai-core-worker`, `ai-moderation`,
`ai-suggestion`, `ai-tutor`, `room-stt`, `assistant-stt`.

## Chạy

Cần Git Bash và git từ 2.36 trở lên. Script clone sạch vào `/tmp`, không đụng working
copy của anh, không push nếu không thêm `--push`. Không cần dọn CRLF hay stash gì
trước.

```bash
cd AI-01-repo-ai
bash setup-repo-ai.sh
```

Script tạo ba commit:

| Commit | Nội dung |
|---|---|
| `refactor: dựng lại cây thư mục…` | chỉ `git mv`, không sửa một dòng nội dung |
| `chore: sửa import và đường dẫn…` | 18 dòng import, đường dẫn model, bỏ đường dẫn máy cá nhân |
| `feat: khung bốn tầng…` | khung core/read/write/api, tài liệu, công cụ, Dockerfile mới |

## Kiểm tra

Script tự chạy bốn bước đối chiếu ở cuối. Kết quả khi tôi chạy thử:

```
[1] Commit di chuyển phải là đổi tên thuần
      đổi tên, nội dung y hệt (R100): 53
      file bị đổi nội dung          : 0
      file bị xóa                   : 0
      OK — không mất dòng code nào

[2] Commit sửa đường dẫn chỉ đụng 14 file, 39 dòng thêm 32 dòng bớt

[3] Cấu trúc và ranh giới tầng
      Quet 43 file .py trong src/catspeak_ai
      OK: 0 loi, 0 canh bao

[4] Lịch sử file có còn không
      moderation/app.py: 3 commit
      ai_tutor/session.py: 3 commit
      write/suggestion/worker.py: 10 commit
```

Tự kiểm thêm bốn thứ này:

```bash
cd /tmp/catspeak-ai-setup

# lịch sử một file bất kỳ có còn không
git log --follow --oneline src/catspeak_ai/moderation/app.py

# commit di chuyển có sửa nội dung gì không (phải rỗng)
git diff --numstat -M100% HEAD~3 HEAD~2 | awk '$1!="0"||$2!="0"'

# mọi file .py còn biên dịch được không
python -m compileall -q src tools && echo OK

# ranh giới tầng
python tools/check_layout.py
pip install import-linter && lint-imports
```

Hai lệnh cuối là thứ giữ cho ranh giới không tan sau sáu tháng. Tôi đã thử bằng cách
cố tình thêm một dòng `from catspeak_ai.core.repositories import ...` vào
`read/__init__.py`: cả hai đều bắt được và trả exit code khác 0. Cắm chúng vào CI.

**Việc phải làm bằng tay sau khi pull nhánh về máy** (script không làm được vì đây là
file không được git theo dõi):

```bash
mv moderation/models  models      # thư mục model 558MB
```

**Việc chưa ai làm và phải làm đầu tiên:** build thử sáu image. Máy tôi không chạy
được docker daemon nên sáu Dockerfile trong `deploy/` chưa build thử lần nào.

```bash
docker build -f deploy/Dockerfile.moderation    -t catspeak/ai-moderation .
docker build -f deploy/Dockerfile.ai-tutor      -t catspeak/ai-tutor .
docker build -f deploy/Dockerfile.room-stt      -t catspeak/room-stt .
docker build -f deploy/Dockerfile.assistant-stt -t catspeak/assistant-stt .
docker build -f deploy/Dockerfile.suggestion    -t catspeak/ai-suggestion .
docker build -f deploy/Dockerfile.api           -t catspeak/ai-api .
```

Build context là gốc repo, không phải thư mục service. Xong thì chạy thử moderation
và gọi `GET /healthz`.

## Nộp gì

Pull request từ `refactor/structure` vào `main`, mô tả gồm: kết quả bốn bước đối
chiếu ở trên, kết quả `docker build`, và link tới `docs/ARCHITECTURE.md`.

Note trong task: repo cần golive là `catspeak-ai`. Chưa golive được cho tới khi có
CI đẩy image (task AI-02 cần nó).

## Nói gì với người khác

Thái và người viết service mới chỉ cần đọc **`docs/ARCHITECTURE.md` mục 3**: một
bảng năm câu hỏi trả lời "file này đặt vào đâu". Không cần đọc bảy mục còn lại.

Ai đang mở nhánh `feature/*` thì `git merge origin/refactor/structure` vào nhánh của
mình. Git nhận diện đổi tên nên phần lớn conflict tự giải quyết; chỗ nào không thì
đối chiếu bảng ở `ARCHITECTURE.md` mục 7.

---

# AI-02 — Setup repository Devops

## Giao cái gì

Một thư mục đẩy thẳng lên GitHub được. Chưa chạy được vì còn thiếu image, nhưng
khung và luật đã đủ để nhóm khác cắm service của họ vào.

Compose gồm 12 service: bốn hạ tầng (Postgres có pgvector, hai Redis, RabbitMQ,
MinIO) và bảy service AI, chia ba profile `agents`, `core`, `all`.

Hai Redis là cố ý. Tài liệu kiến trúc mục 1.3 nói Redis có hai vai phải tách: vai
cache mất được dựng lại từ Postgres, vai giữ transcript phiên đang chạy mất là mất
thật. `redis-cache` tắt ghi đĩa, `redis-stream` bật `appendonly`.

## Chạy

```bash
cd AI-02-repo-devops
# tạo repo rỗng catspeak-devops trên GitHub, KHÔNG tick "Add a README file"
cd catspeak-devops
git init && git add -A && git commit -m "chore: khởi tạo repo devops"
git remote add origin https://github.com/CatSpeak/catspeak-devops.git
git push -u origin main
```

## Kiểm tra

```bash
python -c "import yaml;[yaml.safe_load(open(f)) for f in ['docker-compose.yml','docker-compose.dev.yml']];print('YAML OK')"
docker compose config >/dev/null && echo "compose hợp lệ"
bash -n up.sh && echo "up.sh cú pháp OK"
```

`docker compose config` cần file `.env` mới chạy được, vì compose dùng cú pháp
`${POSTGRES_PASSWORD:?...}` để bắt lỗi thiếu biến. Đó là cố ý: thiếu mật khẩu thì
báo lỗi ngay chứ không dựng lên với mật khẩu rỗng.

Sau khi có image thì thử thật:

```bash
cp .env.example .env
for f in env/*.env.example; do cp "$f" "${f%.example}"; done
# điền khóa
bash up.sh agents        # tiêu chí nghiệm thu của anh Đạt: agent phải lên
docker compose ps
docker compose logs -f room-stt
```

## Còn thiếu gì

Nằm ở mục 5 của `catspeak-devops/README.md`. Cái chặn tất cả là **CI đẩy image**:
repo `catspeak-ai` chưa có workflow build và push lên `ghcr.io/catspeak/*`, nên mọi
dòng `image:` trong compose đều kéo không ra. Trong lúc chờ thì dùng
`docker-compose.dev.yml` để build từ source.

Nếu muốn tôi viết luôn workflow GitHub Actions đẩy image thì nói, nó nằm ở repo
`catspeak-ai` chứ không nằm ở đây.

## Nộp gì

Link repo `catspeak-devops`, cộng một dòng trong task: "chạy được sau khi có CI đẩy
image, xem README mục 5".

---

# AI-09 — Ước tính chi phí dịch vụ ngoài

## Giao cái gì

Một tài liệu: `AI-09-chi-phi/CHI_PHI_DICH_VU_NGOAI.md`. Đủ cả ba loại STT, LLM, TTS.

## Con số để nói trong buổi họp

Một phòng tutor và một phòng hai người chạy liên tục (khoảng 5 request LLM mỗi phút,
đúng mốc được giao):

| Loại | USD/tháng | Tỷ trọng |
|---|---|---|
| STT | 1.037 | 93% |
| LLM | 62 | 6% |
| TTS | 11 | 1% |

Ba câu để nói:

Tranh luận đổi LLM đang nhắm vào 6% hóa đơn. Chênh DeepSeek với Gemini là 22 USD một
tháng; chênh gộp với không gộp phiên âm là 622 USD một tháng.

Phòng hai người đang trả tiền STT gấp bốn lần cần thiết, vì `room-stt` và
`assistant-stt` mỗi bên mở một phiên Deepgram riêng cho từng người trong cùng một
phòng.

Giữ TTS chạy tại chỗ. Kokoro và Supertonic tốn 11 USD tiền CPU một tháng; cùng lượng
đó mua Aura-2 tốn 486 USD, mua ElevenLabs Flash tốn 810 USD.

## Kiểm tra

Mục 1 của tài liệu là bảng đơn giá, mỗi dòng có link nguồn ở cuối bài. Mục 2 là
lượng tiêu thụ, mỗi dòng ghi rõ file và dòng code làm căn cứ. Mục 6 liệt kê ba con
số là giả định và chưa ai đo. Kiểm bằng cách mở đúng file code đó ra đối chiếu.

Muốn tính lại với số khác thì công thức nằm cuối mục 6.

## Việc phải làm ngay, không đợi quyết định chọn giá

`llama-3.1-8b-instant` bị Groq tắt ngày 16/08/2026, tức bốn ngày trước.
`agents/ai_tutor/session.py` dòng 303 và 307 dùng nó làm nhà cung cấp **chính** cho
mọi phiên không phải tiếng Việt. `write/suggestion/utils/groq.py` dòng 11 cũng gọi
nó và không có fallback, nhiều khả năng gợi ý đang chết hẳn.

Soi log staging xác nhận trước. Nếu đúng thì đây là hotfix riêng, dòng thay thế
`openai/gpt-oss-20b` đã có sẵn dạng comment ngay bên dưới trong `utils/groq.py`.

## Nộp gì

Đính tài liệu vào task, cộng ba câu ở phần "con số để nói trong buổi họp".

---

# Việc còn lại sau ba task này

Xếp theo thứ tự nên làm, ghi rõ cái nào chặn cái nào.

| Việc | Chặn cái gì |
|---|---|
| Sửa model LLM đã bị tắt | Không chặn gì, nhưng đang hỏng ở production |
| Build thử sáu image | Chặn CI, chặn repo devops |
| CI đẩy image lên ghcr.io | Chặn toàn bộ repo devops |
| Chạy `0001_init_ai.sql`, gỡ ghi `ModerationLogs` khỏi catspeak-api | Chặn việc tách hẳn DB AI |
| Gộp `ai-stt` | Chặn transcript dùng chung, và là khoản tiết kiệm lớn nhất |
| Dựng bảng ghi nhận chi phí | Chặn việc đo lại hóa đơn sau khi đổi nhà cung cấp |
| Viết `core/repositories` và `core/interfaces` thật | Chặn task báo cáo buổi học (AI-05, AI-06, AI-07) |

Ba việc đầu là của task AI-01 và AI-02. Bốn việc sau là task riêng, nên estimate
tách ra.

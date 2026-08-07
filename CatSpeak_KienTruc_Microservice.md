# CatSpeak — Kiến trúc Microservice cho hệ Cá nhân hóa
### Tài liệu bổ sung cho `CatSpeak_DeXuat_CaNhanHoa.md`
**Người viết:** PM · **Phiên bản:** 1.0 · **Trả lời trực tiếp 3 câu hỏi của sếp**

---

## TRẢ LỜI NGẮN CHO SẾP (đọc 2 phút là đủ họp)

| Câu hỏi của sếp | Trả lời một dòng |
|---|---|
| **Làm microservice thì lưu ở tầng nào tối ưu nhất?** | **Tầng Application (backend phát event sau khi commit) là nguồn chính ~80%; tầng Client chỉ bắt phần server không nhìn thấy được (watch time, impression) và luôn bị coi là "không đáng tin"; tầng Network (gateway/nginx log) KHÔNG dùng làm nguồn dữ liệu.** Còn *lưu* thì lưu trong **DB riêng của service** (database-per-service) — đây mới chính là thứ tạo ra "tách biệt" thật sự. |
| **Làm sạch như thế nào?** | Làm sạch **ngay tại cửa vào (ingest), đồng bộ, trước khi ghi** — không làm sạch lúc đọc. Đi qua **7 chốt chặn**: hợp đồng schema → xác thực danh tính → validate cú pháp → chống trùng (idempotency) → chuẩn hóa → lọc logic/bot → tối thiểu hóa PII. Cái gì rớt thì **rơi vào bảng chờ (dead-letter), không im lặng vứt đi**. |
| **Thêm 1 bảng cho việc test đầu vào?** | Đúng, nhưng cần **4 bảng chứ không phải 1** vì "test đầu vào" gồm 4 việc khác nhau: `EventSchemaRegistry` (hợp đồng — luật để test), `EventFixtures` (bộ ca kiểm thử — test tự động cho CI + sandbox cho team khác), `EventIngestRejects` (cái gì đã rớt và vì sao), `IngestHealthDaily` (sức khỏe cửa vào theo ngày). Kèm 1 endpoint `POST /v1/events:validate` cho phép team khác **bắn thử payload mà không ghi DB**. |
| **Microservice để "call qua call lại"?** | Được, nhưng phải thiết kế **API nói bằng danh từ chung** (`subject / object / action`), không nói "reel", "room". Khi có app/feature mới, họ chỉ cần **đăng ký 1 loại object mới trong Schema Registry** rồi bắn event — **không phải sửa một dòng code nào trong Personalization Service**. Đó mới là decoupled tối đa. |

> ⚠️ **Một khuyến nghị PM đi kèm (nói thẳng với sếp):** làm microservice là đúng hướng, nhưng nên **tách logic + tách DB trước, tách deploy sau** (chi tiết mục 6.4). Tách deploy quá sớm khi mới 1 team và chưa có dữ liệu sẽ tự chuốc thêm: distributed tracing, retry, eventual consistency, 2 pipeline CI/CD — trả giá vận hành mà chưa thu được lợi ích scale. Lộ trình an toàn ở mục 6.4 vẫn cho ra đúng cái sếp muốn (gọi qua API, tách biệt), chỉ khác ở thời điểm bấm nút tách container.

---

## PHẦN 1 — Vì sao phải tách thành microservice (và tách tới đâu)

### 1.1. Vấn đề nếu nhét vào `cath-api` như hiện tại

Hệ cá nhân hóa có 3 đặc tính **xung khắc** với API nghiệp vụ:

| Đặc tính | Hệ cá nhân hóa | `cath-api` (nghiệp vụ) | Hậu quả nếu chung |
|---|---|---|---|
| Khối lượng ghi | Rất lớn, liên tục (mỗi user lướt 100 video = 100 dòng/phút) | Nhỏ, theo hành động chủ đích | Event log làm phình DB nghiệp vụ, backup/restore chậm, vacuum nặng |
| Tính chất dữ liệu | Được phép mất vài %, ưu tiên throughput | Không được mất 1 dòng (payment, friendship) | Cùng một chính sách DB → hoặc quá chặt (chậm) hoặc quá lỏng (mất tiền) |
| Nhịp thay đổi | Đổi công thức/trọng số hàng tuần | Ổn định, đổi là phải regression test | Deploy thuật toán phải deploy cả API thanh toán — rủi ro vô lý |
| Ngôn ngữ tối ưu | Python (ML, pandas, LightGBM) | C#/.NET | Không thể dùng đúng công cụ |

> 💡 **Insight:** lý do thật sự để tách microservice ở đây **không phải là scale** (CatSpeak chưa cần), mà là **tách nhịp deploy và tách rủi ro**. Đội thuật toán phải được đổi trọng số feed lúc 3h chiều thứ Ba mà không cần ai duyệt release của API thanh toán. Nói với sếp bằng lý do này thuyết phục hơn nói "để scale".

### 1.2. Ranh giới của service (Bounded Context)

**Personalization Service SỞ HỮU:**
`ReelViewEvents`, `ReelImpressions`, `ConversationSessionFacts`, `NotificationLog`, `PresenceSnapshots`, `UserInterestProfiles`, `ItemFeatures` (tên chung của `ReelFeatures`), `UserAffinityEdges`, `ExperimentAssignments`, và 4 bảng ingest ở Phần 4.

**Personalization Service KHÔNG sở hữu và KHÔNG được đọc trực tiếp:**
`Account`, `Reel`, `Message`, `Friendship`, `Payment`… — những bảng này thuộc `cath-api`. Service chỉ biết chúng **qua event được bắn sang**, và chỉ giữ lại **ID + vài thuộc tính tối thiểu** (level, ngôn ngữ, quốc gia).

> 💡 **Insight:** đây là chỗ 90% dự án "microservice" thất bại — tách code nhưng vẫn cho service mới `SELECT` thẳng vào DB cũ. Lúc đó đổi 1 cột ở `cath-api` là gãy service kia, mà không ai biết trước. **Quy tắc bất di bất dịch: hai service không bao giờ chung một connection string.** Nếu chưa tách DB được ngay thì tách **schema** trong cùng Postgres (`personalization.*` vs `public.*`) và cấp **user DB riêng chỉ có quyền trên schema của mình** — DB tự chặn giúp mình.

### 1.3. Sơ đồ tổng thể

```
   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
   │ catspeak-    │   │ catspeak-    │   │ catspeak-ai  │   │ APP MỚI      │
   │ client (web) │   │ admin        │   │ (LiveKit/STT)│   │ (chưa có)    │
   └──────┬───────┘   └──────┬───────┘   └──────┬───────┘   └──────┬───────┘
          │ (chỉ UI event)   │                  │                  │
          │                  ▼                  ▼                  ▼
          │           ┌────────────────────────────────────────────────┐
          │           │        cath-api (.NET) — NGHIỆP VỤ             │
          │           │  ghi DB nghiệp vụ + Outbox → bắn event đi      │
          │           └───────────────────────┬────────────────────────┘
          │                                   │
          ▼                                   ▼
   ╔══════════════════════════════════════════════════════════════════════╗
   ║  API GATEWAY  (auth, rate-limit, tracing — KHÔNG phải nơi lưu data)  ║
   ╚══════════════════════════════════════╤═══════════════════════════════╝
                                          ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │  PERSONALIZATION SERVICE  (microservice — "call qua call lại")       │
   │                                                                      │
   │  ┌── WRITE ─────────────────┐        ┌── READ ─────────────────────┐ │
   │  │ POST /v1/events          │        │ POST /v1/rank  (generic)    │ │
   │  │ POST /v1/events:validate │        │ GET  /v1/feed/reels         │ │
   │  │ POST /v1/identify        │        │ GET  /v1/match/candidates   │ │
   │  └───────────┬──────────────┘        │ GET  /v1/friends/suggest    │ │
   │              │                       │ GET  /v1/notify/best-time   │ │
   │   ┌──────────▼──────────┐            │ GET  /v1/profile/{id}       │ │
   │   │ 7 CHỐT LÀM SẠCH     │            └────────────▲────────────────┘ │
   │   │ (Phần 3)            │                         │                  │
   │   └──────────┬──────────┘                    đọc từ cache/Tầng 3     │
   │        hợp lệ│   ↘ rớt → EventIngestRejects        │                 │
   │              ▼                                     │                 │
   │   ┌──────────────────┐  job nền  ┌──────────────┐  │                 │
   │   │ T1: EVENT LOG    │──────────▶│ T2: PROFILE  │──┘                 │
   │   │ (append-only)    │  EMA decay│ (tổng hợp)   │                    │
   │   └──────────────────┘           └──────────────┘                    │
   │              DB RIÊNG (personalization.*) + Redis (T3 serving)       │
   └──────────────────────────────────────────────────────────────────────┘
```

---

## PHẦN 2 — CÂU HỎI 1: Lưu ở tầng nào là tối ưu?

Câu hỏi của sếp thực chất là **hai câu hỏi bị gộp làm một**, phải tách ra mới trả lời đúng:

- **(2a) BẮT sự kiện (capture) ở tầng nào?** → client / network / application / database
- **(2b) LƯU (store) ở đâu?** → trong service nào, tầng lưu trữ nào

### 2.1. (2a) So sánh 4 tầng bắt sự kiện

| Tầng | Bắt được gì | Ưu điểm | Nhược điểm chí mạng | Kết luận cho CatSpeak |
|---|---|---|---|---|
| **Client** (SDK trong React/mobile) | Watch time, impression (video đã hiện ra chưa), scroll depth, thời gian dừng, thứ tự lướt, tab ẩn/hiện | **Là nơi DUY NHẤT biết được hành vi UI** — server không đời nào biết user xem video 12 giây rồi lướt | Không tin được: sửa được payload, adblock chặn, mất mạng thì mất event, đồng hồ máy lệch giờ, user dùng app version cũ 6 tháng vẫn bắn schema cũ | ✅ **Dùng — nhưng CHỈ cho tín hiệu UI, và coi mọi payload là dữ liệu thù địch** (mục 3.2) |
| **Network** (nginx/gateway access log, service mesh) | Log HTTP: IP nào gọi endpoint nào lúc mấy giờ | Không phải sửa 1 dòng code nào | **Không có ngữ nghĩa nghiệp vụ.** `POST /api/reels/12` — là view? là share? là report? Không biết. Không có AccountId nội bộ (chỉ có token). Ghép lại thành hành vi tốn công gấp 10 lần mà vẫn sai | ❌ **KHÔNG dùng làm nguồn dữ liệu.** Gateway chỉ để auth + rate-limit + trace-id |
| **Application** (backend service phát event *sau khi* commit transaction) | Mọi sự kiện nghiệp vụ đã xác thực: match thành công, kết bạn, gửi tin nhắn, session kết thúc, thanh toán, đăng reel | **Đáng tin tuyệt đối** (đã qua auth + business rule), đúng schema, đúng AccountId, nằm trong transaction | Mù hoàn toàn với hành vi UI: không biết user lướt qua bao nhiêu video mà không bấm gì | ✅ **NGUỒN CHÍNH — source of truth, ~80% giá trị dữ liệu** |
| **Database** (CDC — Debezium đọc WAL của Postgres) | Mọi thay đổi bảng | Không cần sửa code app, không sót | Chỉ biết **trạng thái thay đổi**, không biết **ý định**: thấy `Friendship` thêm 1 dòng nhưng không biết user bấm từ màn "gợi ý bạn bè" hay từ tìm kiếm — mà đúng cái đó mới cần để đo thuật toán. Couple chặt vào schema DB | 🔶 **Chỉ dùng để backfill dữ liệu lịch sử một lần**, không dùng chạy thường xuyên |

### 2.2. Kết luận: kiến trúc lai 80/20 — một cửa vào duy nhất

```
Application layer (cath-api, catspeak-ai)  ──┐
   → sự kiện nghiệp vụ, tin cậy, ~80%       │
                                             ├──▶  POST /v1/events  ──▶ 7 chốt làm sạch ──▶ DB
Client layer (SDK mỏng trong web/mobile)  ──┘
   → view/impression/scroll, ~20%, untrusted
```

**Nguyên tắc: mọi nguồn đều đi qua đúng MỘT endpoint `POST /v1/events`.** Không mở đường tắt cho service nội bộ ghi thẳng DB.

> 💡 **Insight — vì sao "một cửa" quan trọng hơn nó có vẻ:** nếu cho `cath-api` ghi thẳng vào bảng event (vì "nó là service nhà, tin được mà"), thì 6 tháng sau logic làm sạch tồn tại ở 2 nơi, và mỗi lần đổi luật validate phải nhớ sửa cả 2. Một cửa vào = **một nơi duy nhất chứa định nghĩa "thế nào là dữ liệu sạch"**. Đây cũng là điều kiện để tính được con số "tỷ lệ event bị từ chối" — nếu có đường tắt thì con số đó vô nghĩa.

### 2.3. Client bắt gì, Application bắt gì — bảng phân công cụ thể cho CatSpeak

| Sự kiện | Bắt ở đâu | Vì sao |
|---|---|---|
| `reel.viewed` (WatchMs, CompletionPct, IsSkip, FeedPosition) | **Client** | Server không nhìn thấy hành vi lướt |
| `reel.impressed` (đã hiển thị trong viewport ≥1s) | **Client** | Chỉ trình duyệt biết viewport |
| `reel.liked` / `reel.commented` / `reel.shared` | **Application** (`cath-api`) | Đã có API, đã ghi DB → phát event sau khi commit. Bắt ở client sẽ đếm trùng khi user bấm rồi undo |
| `queue.joined` (Topic, Level, LanguageType) | **Application** (`QueueService`) | Dữ liệu đã nằm sẵn trong `QueueEntry` |
| `conversation.ended` (Duration, EndedEarly, PartnerIds) | **Application** (LiveKit webhook) | Webhook là nguồn chuẩn, client có thể đóng tab đột ngột |
| `conversation.topics_detected` | **Application** (`catspeak-ai`, sau khi Gemini trích) | Chỉ service AI có transcript |
| `notification.sent` | **Application** (notification worker) | |
| `notification.opened` / `clicked` | **Client** | Chỉ client biết user có mở hay không |
| `friend.requested` / `accepted` | **Application** | Có transaction |
| `session.online` / `offline` | **Application** (SignalR hub) | Đã có sẵn `AccountOnlineSession` |
| `app.foregrounded` / `screen.viewed` | **Client** | Hành vi phiên |

> 💡 **Insight từ bảng này:** khoảng 8/12 loại sự kiện quan trọng nhất bắt được ở **application layer mà không cần client làm gì**. Nghĩa là **có thể bật 80% hệ thu thập dữ liệu mà không chờ team frontend** — mở khóa ngay việc chạy song song hai team, và giai đoạn 1 của roadmap không bị chặn bởi lịch release app.

### 2.4. (2b) Lưu ở đâu — 3 tầng lưu bên trong service

| Tầng lưu | Công nghệ | Giữ bao lâu | Dùng để |
|---|---|---|---|
| **T1 — Raw event** (hot) | Postgres, bảng partition theo tháng | 90 ngày | Tính lại profile, debug, train model |
| **T1b — Raw event** (cold) | Cloudflare R2 (đã có sẵn!), file Parquet theo ngày | 2 năm | Train model, phân tích lịch sử. Rẻ hơn Postgres ~20 lần |
| **T2 — Profile/Feature** | Postgres, bảng nhỏ, 1 dòng/user | Vĩnh viễn | Job nền cập nhật, API đọc |
| **T3 — Serving** | Redis (TTL 5–15 phút) | Tạm | Trả API < 50ms |

> 💡 **Insight:** đừng lưu event thô vĩnh viễn trong Postgres. Với 1.000 DAU × 100 view/ngày = 100k dòng/ngày = **36 triệu dòng/năm** chỉ riêng `ReelViewEvents`. Postgres chịu được, nhưng backup, migration và vacuum sẽ khổ. Đổ xuống R2 dạng Parquet theo ngày là **tận dụng hạ tầng CatSpeak đã trả tiền rồi** — không thêm chi phí mới.

### 2.5. Những thứ TUYỆT ĐỐI không lưu ở đâu

| Nơi | Vì sao không |
|---|---|
| `localStorage` của client | Chỉ được dùng làm **buffer offline tối đa ~50 event / 24h**, gửi xong xóa. Không phải nơi lưu trữ |
| Gateway / nginx log | Không có ngữ nghĩa, xoay vòng mất dữ liệu, không query được |
| DB nghiệp vụ của `cath-api` | Phá vỡ ranh giới service — xem mục 1.2 |
| Chung bảng với `Notification` cũ | Bảng `Notification` hiện tại là **state hiển thị cho user**, khác hoàn toàn với **log để phân tích**. Trộn hai mục đích vào một bảng là lý do hiện giờ không đo được noti nào hiệu quả |

---

## PHẦN 3 — CÂU HỎI 2: Làm sạch như thế nào?

**Nguyên tắc gốc: làm sạch NGAY LÚC GHI (schema-on-write), không làm sạch lúc đọc.**

> 💡 **Insight — vì sao không để "lưu hết rồi lọc sau":** nghe thì linh hoạt, nhưng thực tế 6 tháng sau sẽ có 5 job khác nhau, mỗi job tự lọc theo một cách, và hai dashboard cho ra hai con số DAU khác nhau — mất hết niềm tin vào dữ liệu. Làm sạch ở cửa vào thì **định nghĩa "sạch" chỉ tồn tại ở một chỗ**, ai đọc cũng ra cùng một số.

### 3.1. Đường đi của 1 event qua 7 chốt

```
payload vào
   │
   ├─▶ Chốt 0: HỢP ĐỒNG   — EventType có trong EventSchemaRegistry & còn active?
   ├─▶ Chốt 1: DANH TÍNH  — token hợp lệ? service có quyền bắn loại event này?
   ├─▶ Chốt 2: CÚ PHÁP    — JSON Schema validation (đủ field, đúng kiểu, đúng range)
   ├─▶ Chốt 3: CHỐNG TRÙNG— EventId (UUID) đã tồn tại chưa?
   ├─▶ Chốt 4: CHUẨN HÓA  — UTC, enum lowercase, đơn vị ms, map topic về TopicType
   ├─▶ Chốt 5: LOGIC/BOT  — hợp lý về nghiệp vụ? tần suất người thật?
   ├─▶ Chốt 6: PII        — cắt bỏ dữ liệu cá nhân không cần
   └─▶ Chốt 7: THỜI GIAN  — event đến muộn/lệch thứ tự thì xử sao
   │
   ├── PASS ──▶ ghi vào T1 (event log)          → trả 202 Accepted
   └── FAIL ──▶ ghi vào EventIngestRejects      → trả 200 kèm danh sách lỗi từng dòng
               (KHÔNG bao giờ im lặng vứt đi)
```

### 3.2. Chi tiết từng chốt + insight nếu bỏ qua

| Chốt | Làm gì cụ thể | 💡 Nếu bỏ qua thì thuật toán hỏng ra sao |
|---|---|---|
| **0. Hợp đồng schema** | Đối chiếu `EventType` + `SchemaVersion` với `EventSchemaRegistry`. Không có trong bảng → từ chối ngay | Không có bảng này thì mỗi team tự bịa tên event: `reel_view`, `reelView`, `view_reel` — 3 tháng sau đếm view ra 3 con số khác nhau, không ai dám tin |
| **1. Danh tính** | Service-to-service: mTLS hoặc bearer token có scope. Client: **lấy `AccountId` từ JWT, TUYỆT ĐỐI không lấy từ body** | Nếu tin `AccountId` trong body, ai cũng bơm được view giả cho video của mình → thuật toán bị thao túng, creator gian lận thắng creator tử tế |
| **2. Cú pháp** | JSON Schema: đủ field bắt buộc, đúng kiểu, `WatchMs ≥ 0`, `CompletionPct ∈ [0, 500]`, `EventType` trong enum | Một field null lọt vào là job tính EMA đêm đó chết → sáng hôm sau feed của toàn bộ user về mặc định. Lỗi kiểu này rất khó truy vì nó xảy ra lúc 3h sáng |
| **3. Chống trùng** | `EventId` UUID sinh ở client + `UNIQUE INDEX`. Trùng → bỏ qua, trả `duplicate` (không phải lỗi) | Mạng chập chờn → client retry → 1 lượt xem thành 5 dòng. Không dedupe thì user nào mạng yếu bị hệ thống hiểu là "cực kỳ thích video này" → feed của họ bị bóp méo. **Đây là lỗi âm thầm và phổ biến nhất** |
| **4. Chuẩn hóa** | Mọi `OccurredAt` → UTC **và lưu kèm `TzOffsetMinutes`**; enum về lowercase; đơn vị luôn là ms; topic tự do map về `TopicType` (20 giá trị) qua bảng alias | Không lưu offset thì mất luôn khả năng trả lời "user này hay dùng app lúc mấy giờ **theo giờ địa phương của họ**" — mà đó chính là insight (D) để chọn giờ gửi noti. Đây là lỗi **không sửa lại được về sau** vì thông tin đã mất hẳn |
| **5. Logic & bot** | • `WatchMs > VideoDuration × 5` → clamp (tab để nền)<br>• `WatchMs < 300ms` → không tính là view<br>• `OccurredAt` ở tương lai > 5 phút, hoặc quá khứ > 7 ngày → quarantine<br>• > 300 event/phút/user → gắn cờ `IsSuspect`, vẫn lưu nhưng loại khỏi job tính profile<br>• Event view từ tab ẩn → loại | Không clamp thì 1 user để tab chạy qua đêm sẽ tạo `WatchMs = 8 tiếng` → video đó nhảy lên top toàn hệ thống. **Một dòng rác đủ làm hỏng bảng xếp hạng vì các công thức đều dùng trung bình** |
| **6. PII** | Không nhận email/họ tên/số điện thoại/transcript thô. IP → hash + chỉ giữ quốc gia. Ghi rõ trong hợp đồng schema: field lạ ngoài schema bị **loại bỏ**, không lưu | Đúng nguyên tắc "DB lưu kết luận, không lưu nguyên liệu" đã chốt ở tài liệu trước. Ngoài privacy, nó còn khiến DB nhẹ đi hàng chục lần |
| **7. Thời gian** | Lưu cả `OccurredAt` (lúc xảy ra) và `IngestedAt` (lúc nhận). Job tính profile dùng `OccurredAt`, watermark trễ 15 phút. Event đến muộn > 24h → vẫn lưu nhưng gắn `IsLate` | Client offline gửi bù event của 3 ngày trước. Nếu job tính theo `IngestedAt` thì biểu đồ "giờ vàng" bị đổ dồn về lúc user mở lại app → chọn sai giờ gửi noti cho cả một nhóm user |

### 3.3. Chính sách trả lời khi có event bẩn

```
POST /v1/events   { events: [ 50 dòng ] }

→ 200 OK
{
  "accepted": 47,
  "duplicated": 2,
  "rejected": [
    { "index": 12, "eventId": "...", "code": "SCHEMA_MISSING_FIELD", "field": "watchMs" }
  ],
  "traceId": "..."
}
```

**Quy tắc:** 1 event xấu **không** làm hỏng cả batch (partial success). Người gọi nhận đủ thông tin để sửa. Không bao giờ trả 500 vì lý do dữ liệu.

> 💡 **Insight:** phần lớn hệ thu thập dữ liệu chết vì **im lặng**. Client bắn sai field suốt 2 tháng, server lặng lẽ bỏ qua, đến khi làm dashboard mới phát hiện mất 40% dữ liệu — và **không lấy lại được**. Chốt chặn phải luôn đi kèm **tiếng động**: trả lỗi cho người gọi + ghi vào `EventIngestRejects` + alert khi tỷ lệ reject vượt 2%.

---

## PHẦN 4 — CÂU HỎI 3: Bảng test đầu vào

Sếp nói "thêm 1 bảng cho việc test đầu vào" — đúng hướng, nhưng "test đầu vào" thực ra gồm **4 việc riêng biệt**, gộp vào 1 bảng sẽ không dùng được cái nào cho ra hồn:

| Việc | Bảng | Câu hỏi nó trả lời |
|---|---|---|
| Định nghĩa **luật** để test | `EventSchemaRegistry` | Payload thế nào là hợp lệ? |
| **Bộ ca kiểm thử** tự động | `EventFixtures` | Code mới có còn validate đúng không? Team mới bắn thử ở đâu? |
| **Nhật ký cái đã rớt** | `EventIngestRejects` | Hôm qua rớt cái gì, vì sao, cứu lại được không? |
| **Sức khỏe cửa vào** | `IngestHealthDaily` | Nguồn nào đang bắn rác? Có đang mất dữ liệu ngầm không? |

### 4.1. `EventSchemaRegistry` — hợp đồng giữa các service

```sql
CREATE TABLE EventSchemaRegistry (
    Id              SERIAL PRIMARY KEY,
    EventType       VARCHAR(60)  NOT NULL,   -- 'reel.viewed'
    SchemaVersion   INT          NOT NULL DEFAULT 1,
    ObjectType      VARCHAR(30)  NOT NULL,   -- 'reel' | 'room' | 'post' | 'course'...
    JsonSchema      JSONB        NOT NULL,   -- JSON Schema đầy đủ
    RequiredFields  TEXT[]       NOT NULL,
    OwnerService    VARCHAR(50)  NOT NULL,   -- ai được phép bắn: 'cath-api' | 'client-web'
    AllowedSources  TEXT[]       NOT NULL,
    Status          VARCHAR(15)  NOT NULL DEFAULT 'active', -- active|deprecated|disabled
    SampleRate      REAL         DEFAULT 1.0, -- lấy mẫu nếu event quá nhiều
    PiiPolicy       VARCHAR(20)  DEFAULT 'strip_unknown',
    Description     TEXT,
    CreatedAt       TIMESTAMPTZ DEFAULT now(),
    UNIQUE (EventType, SchemaVersion)
);
```

> 💡 **INSIGHT — đây mới là "trái tim" của tính decoupled, không phải cái endpoint:**
> - **App mới không cần service sửa code.** Team làm feature "Khóa học" chỉ cần INSERT 1 dòng `course.completed` vào bảng này rồi bắn event. Personalization Service không biết "course" là gì nhưng vẫn cộng điểm sở thích và xếp hạng được — vì nó chỉ làm việc với `subject/object/action`. **Đúng cái sếp muốn: mở rộng không giới hạn mà không đụng vào service.**
> - **Trả lời được "ai đang bắn cái gì" bằng một câu SQL.** Không có bảng này thì kiến thức đó nằm trong đầu vài người và trong code rải rác.
> - **Tắt được một luồng dữ liệu bằng cách đổi 1 dòng** (`Status = 'disabled'`), không cần deploy — cực kỳ quý khi có client version cũ bắn rác lúc 2h sáng.
> - **`SampleRate` = van xả áp.** Khi `reel.impressed` quá nhiều, hạ xuống 0.1 để giữ 10% — thống kê vẫn đúng, chi phí giảm 90%.

### 4.2. `EventFixtures` — đúng "bảng test đầu vào" sếp nói

```sql
CREATE TABLE EventFixtures (
    Id              SERIAL PRIMARY KEY,
    EventType       VARCHAR(60) NOT NULL,
    SchemaVersion   INT         NOT NULL,
    CaseName        VARCHAR(120) NOT NULL,   -- 'watchMs âm', 'thiếu accountId', 'hợp lệ tối thiểu'
    Payload         JSONB       NOT NULL,    -- payload mẫu
    ExpectedResult  VARCHAR(15) NOT NULL,    -- 'accept' | 'reject' | 'clamp' | 'duplicate'
    ExpectedCode    VARCHAR(40),             -- 'SCHEMA_TYPE_MISMATCH'...
    ExpectedNormalized JSONB,                -- kết quả sau chuẩn hóa (kiểm tra chốt 4-5)
    IsRegression    BOOLEAN DEFAULT FALSE,   -- sinh ra từ 1 sự cố thật
    SourceRejectId  BIGINT,                  -- link về dòng rác thật đã gây sự cố
    CreatedAt       TIMESTAMPTZ DEFAULT now(),
    UNIQUE (EventType, SchemaVersion, CaseName)
);
```

Cách dùng:
1. **CI/CD**: pipeline chạy toàn bộ fixture qua pipeline làm sạch, so với `ExpectedResult`. Sai một ca → chặn merge. Đây là lưới an toàn khi đổi luật validate.
2. **Sandbox cho team khác**: `POST /v1/events:validate` + `GET /v1/fixtures?eventType=reel.viewed` → team mới **tự copy payload mẫu, tự test, không cần hỏi ai**.
3. **Vòng lặp tự cải thiện**: mỗi lần có sự cố dữ liệu thật → lấy dòng trong `EventIngestRejects` → thêm thành fixture `IsRegression = true`. **Lỗi đã xảy ra một lần thì vĩnh viễn không xảy ra lại.**

> 💡 **INSIGHT:** thứ này biến việc "tích hợp với Personalization Service" từ **một cuộc họp** thành **một trang tài liệu tự phục vụ**. Với sếp, đó là lợi ích rõ nhất: mỗi app/feature mới muốn dùng hệ cá nhân hóa thì tốn ~0 giờ của đội thuật toán. Chi phí tích hợp gần bằng 0 chính là định nghĩa thực tế của "decoupled".

### 4.3. `EventIngestRejects` — thùng chờ (dead-letter)

```sql
CREATE TABLE EventIngestRejects (
    Id              BIGSERIAL PRIMARY KEY,
    ReceivedAt      TIMESTAMPTZ NOT NULL DEFAULT now(),
    SourceService   VARCHAR(50),
    ClientVersion   VARCHAR(30),
    EventType       VARCHAR(60),
    SchemaVersion   INT,
    RawPayload      JSONB       NOT NULL,    -- giữ nguyên bản để replay
    RejectStage     VARCHAR(20) NOT NULL,    -- schema|auth|dedupe|logic|pii|time
    RejectCode      VARCHAR(40) NOT NULL,
    RejectDetail    TEXT,
    AccountIdGuess  INT,
    IsReplayable    BOOLEAN DEFAULT TRUE,
    ReplayedAt      TIMESTAMPTZ NULL,
    TraceId         VARCHAR(60)
);
CREATE INDEX ix_rej_time ON EventIngestRejects(ReceivedAt DESC);
CREATE INDEX ix_rej_code ON EventIngestRejects(RejectCode, ReceivedAt DESC);
```

> 💡 **INSIGHT:**
> - **Sửa xong là cứu lại được dữ liệu.** Client bắn sai 3 ngày → sửa → chạy replay từ bảng này → dữ liệu 3 ngày đó không mất. Nếu chỉ trả lỗi rồi vứt thì mất vĩnh viễn.
> - **`ClientVersion` + `RejectCode` = tìm ra thủ phạm trong 30 giây.** "95% reject đến từ client-web 2.3.1" — biết ngay phải hotfix bản nào thay vì mò mẫm.
> - **Bảng này là nguồn sinh fixture.** Rác thật ngoài đời luôn sáng tạo hơn trí tưởng tượng của người viết test.
> - **Cảnh báo sớm.** Tỷ lệ reject nhảy vọt = có ai đó vừa deploy thứ gì đó sai, biết trước khi user kêu.

### 4.4. `IngestHealthDaily` — sức khỏe cửa vào

```sql
CREATE TABLE IngestHealthDaily (
    Day             DATE        NOT NULL,
    SourceService   VARCHAR(50) NOT NULL,
    EventType       VARCHAR(60) NOT NULL,
    Received        BIGINT DEFAULT 0,
    Accepted        BIGINT DEFAULT 0,
    Duplicated      BIGINT DEFAULT 0,
    Rejected        BIGINT DEFAULT 0,
    Clamped         BIGINT DEFAULT 0,
    SuspectBot      BIGINT DEFAULT 0,
    AvgLagSeconds   REAL,          -- IngestedAt - OccurredAt
    P99LagSeconds   REAL,
    PRIMARY KEY (Day, SourceService, EventType)
);
```

> 💡 **INSIGHT:** đây là **bảng đầu tiên nên nhìn mỗi sáng**, trước cả dashboard sản phẩm — vì mọi con số sản phẩm đều đứng trên nó.
> - `Received` tụt đột ngột = **client hỏng, không phải user giảm.** Không có bảng này thì sẽ ngồi họp phân tích "vì sao engagement giảm" trong khi thật ra chỉ là một cái deploy làm gãy SDK.
> - `Duplicated` cao = client retry sai → sửa được ngay.
> - `AvgLagSeconds` cao = user vào app từ vùng mạng yếu / dùng offline nhiều → ảnh hưởng cách chọn watermark cho job.
> - `Clamped` cao ở `reel.viewed` = nhiều người để tab chạy nền → cân nhắc đổi cách client đo watch time.

### 4.5. Endpoint dry-run đi kèm

```
POST /v1/events:validate      # y hệt /v1/events nhưng KHÔNG ghi DB
→ { "valid": false, "errors": [...], "normalized": {...} }
```

Trả về cả **payload sau chuẩn hóa** để team tích hợp thấy đúng thứ sẽ được lưu. Đây là "test đầu vào" theo nghĩa runtime, còn `EventFixtures` là "test đầu vào" theo nghĩa CI — cần cả hai.

---

## PHẦN 5 — Thiết kế API "call qua call lại"

### 5.1. Nguyên tắc: nói bằng danh từ chung

Nếu API là `GET /v1/reels/feed` thì service **biết** khái niệm "reel" → có feature mới là phải sửa service. Thay vào đó:

```
subject  = ai (user)
object   = cái gì (type + id):  reel | room | post | course | event | user
action   = làm gì:              view | like | share | complete | skip | join
context  = ở đâu, lúc nào, vị trí nào trong danh sách
```

**Endpoint tổng quát quan trọng nhất:**

```http
POST /v1/rank
{
  "userId": 42,
  "objectType": "reel",
  "candidateIds": [101, 205, 309, ...],   // tối đa 500
  "context": { "surface": "home_feed", "locale": "vi", "limit": 20 },
  "experiment": "feed_v2"
}
→ 200
{
  "ranked": [ {"id": 205, "score": 0.87, "reasons": ["topic:travel", "creator_affinity"]}, ... ],
  "modelVersion": "feed-heuristic-1.3",
  "experimentVariant": "B",
  "fallback": false
}
```

> 💡 **INSIGHT:** với đúng một endpoint này, **mọi feature tương lai đều xài được ngay**: gợi ý khóa học, xếp thứ tự sự kiện, chọn bài tập, sắp danh sách phòng. Team gọi chỉ cần đưa danh sách ứng viên của họ — service trả về thứ tự. **Không có endpoint riêng cho từng feature = không có việc phải làm cho từng feature mới.** Trường `reasons` còn cho phép hiển thị "Vì bạn hay xem về Du lịch" ngoài UI và giúp debug khi feed ra kết quả lạ.

### 5.2. Danh mục API

**Ghi (write)**

| Endpoint | Dùng khi | Ghi chú |
|---|---|---|
| `POST /v1/events` | Bắn event (batch ≤ 100) | Async, trả `202`, p99 < 30ms |
| `POST /v1/events:validate` | Test payload, không ghi | Dành cho team tích hợp |
| `POST /v1/identify` | Cập nhật thuộc tính user (level, ngôn ngữ, quốc gia) | Idempotent |

**Đọc (read) — mọi endpoint p99 < 80ms, có fallback**

| Endpoint | Trả về | Người gọi hiện tại |
|---|---|---|
| `POST /v1/rank` | Xếp hạng danh sách ứng viên bất kỳ | Mọi feature |
| `GET /v1/feed/reels?userId=&limit=&exclude=` | Feed đã cá nhân hóa (service tự lấy ứng viên) | `ReelService` |
| `GET /v1/match/candidates?userId=&roomType=&topN=` | Danh sách người nên ghép + điểm | `QueueService` |
| `GET /v1/friends/suggest?userId=&limit=` | Gợi ý kết bạn + lý do | `FriendshipController` |
| `GET /v1/notify/best-time?userId=&type=` | Giờ nên gửi + loại noti nên gửi + có nên gửi không | Notification worker |
| `GET /v1/profile/{userId}` | TopicScores, LevelEstimate, ActiveHours, segment | Admin, AI agent |
| `GET /v1/similar?objectType=&objectId=` | Item tương tự | Màn "video liên quan" |

**Vận hành**

| Endpoint | Dùng để |
|---|---|
| `GET /v1/health`, `GET /v1/ready` | K8s probe |
| `GET /v1/schemas`, `GET /v1/fixtures` | Team khác tự phục vụ |
| `GET /v1/ingest/health?day=` | Dashboard admin |

### 5.3. Hợp đồng vận hành (SLA) — điều kiện bắt buộc để tách service

| Hạng mục | Cam kết | Cách đạt |
|---|---|---|
| Độ trễ đọc | p99 < 80ms | Redis cache T3, không tính toán nặng lúc request |
| Độ trễ ghi | p99 < 30ms | Nhận → đẩy queue → trả `202`; làm sạch ở worker (nhưng chốt 0–2 chạy đồng bộ để trả lỗi ngay) |
| **Timeout phía gọi** | **200ms, rồi fallback** | `cath-api` phải có sẵn công thức feed cũ làm dự phòng |
| Availability | 99.5% | |
| Versioning | `/v1` giữ tối thiểu 6 tháng, chỉ thêm field, không xóa | Field mới phải optional |

> 💡 **INSIGHT — điều khoản quan trọng nhất trong cả tài liệu này:** **`cath-api` phải chạy được bình thường khi Personalization Service chết.** Feed rơi về công thức cũ, matchmaking rơi về FIFO, noti rơi về giờ mặc định — user thấy trải nghiệm nhạt hơn chứ **không thấy màn hình lỗi**. Nếu không có điều này thì tách microservice là **tự nhân đôi rủi ro sập app** để đổi lấy một tính năng "làm cho hay hơn". Đây là câu phải nói rõ với sếp: **microservice cá nhân hóa phải là tầng tăng cường, không bao giờ là tầng phụ thuộc cứng.**

### 5.4. Bảo mật giữa các service

- **Service → Service**: mTLS hoặc bearer token có scope (`events:write`, `rank:read`). Mỗi service một token riêng → truy vết được ai bắn rác.
- **Client → Service**: **không cho client gọi thẳng.** Đi qua Gateway, Gateway đổi JWT user thành service token nội bộ và **ép `AccountId` lấy từ JWT**.
- **Rate limit**: theo user (chống bơm view) và theo service (chống bug vòng lặp vô hạn).
- Personalization Service **không có** quyền ghi vào bất kỳ bảng nghiệp vụ nào.

---

## PHẦN 6 — Triển khai

### 6.1. Bắn event từ `cath-api` mà không mất — Outbox Pattern

Không được gọi HTTP sang Personalization Service **bên trong** transaction nghiệp vụ (service kia chậm/chết là hỏng luôn nghiệp vụ chính).

```
BEGIN;
  INSERT INTO Friendship (...);            -- nghiệp vụ
  INSERT INTO OutboxEvents (Type, Payload, Status='pending');  -- cùng transaction
COMMIT;

-- Worker riêng: đọc OutboxEvents pending → POST /v1/events → đánh dấu sent
-- Lỗi → retry với exponential backoff. Không mất event, không chặn nghiệp vụ.
```

```sql
CREATE TABLE OutboxEvents (
    Id          BIGSERIAL PRIMARY KEY,
    EventType   VARCHAR(60) NOT NULL,
    Payload     JSONB       NOT NULL,
    Status      VARCHAR(15) DEFAULT 'pending',  -- pending|sent|failed
    Attempts    SMALLINT DEFAULT 0,
    CreatedAt   TIMESTAMPTZ DEFAULT now(),
    SentAt      TIMESTAMPTZ NULL
);
CREATE INDEX ix_outbox_pending ON OutboxEvents(Status, CreatedAt) WHERE Status = 'pending';
```

> 💡 **Insight:** đây là chỗ dễ làm ẩu nhất. Gọi HTTP trực tiếp trong transaction thì mỗi lần service cá nhân hóa chậm 2 giây, **API kết bạn cũng chậm 2 giây** — user đổ lỗi cho tính năng kết bạn. Outbox tách hoàn toàn: nghiệp vụ commit xong là xong, event gửi sau, chậm bao lâu cũng không ai thấy.

### 6.2. Công nghệ đề xuất

| Thành phần | Chọn | Vì sao |
|---|---|---|
| Ingest + Serving API | **.NET 8** (project mới `catspeak-personalization`) | Team đang mạnh .NET; tái dùng EF Core, Npgsql, pattern có sẵn |
| Job tính profile / ML | **Python** (mở rộng repo `catspeak-ai`) | pandas, implicit, LightGBM. Đã có sẵn hạ tầng Python |
| Queue | **Giai đoạn 1: bảng Postgres làm queue** → **Giai đoạn 2: Redis Stream** → **Giai đoạn 3: Kafka** | Đừng dựng Kafka khi chưa tới 10k event/giây. Postgres làm queue thừa sức ở quy mô hiện tại |
| Cache serving | **Redis** | |
| Cold storage | **Cloudflare R2** (đã có) | Không thêm nhà cung cấp mới |
| Schema validation | `JsonSchema.Net` (C#) | Đọc schema từ `EventSchemaRegistry` lúc runtime, cache 60s |

### 6.3. So sánh 3 mức "tách" — chọn mức nào

| Mức | Mô tả | Được gì | Mất gì |
|---|---|---|---|
| **M1 — Module trong monolith** | Project riêng trong solution `catspeak-api`, **schema DB riêng**, gọi nhau qua interface | Rẻ nhất, làm được trong 2 tuần | Vẫn deploy chung |
| **M2 — Service riêng, DB riêng, deploy riêng** ⭐ | Container riêng, gọi qua HTTP nội bộ | **Đúng cái sếp muốn**: tách deploy, tách rủi ro, app mới call API | Thêm CI/CD, monitoring, network hop |
| **M3 — Event-driven đầy đủ** | Kafka, schema registry ngoài, nhiều consumer | Scale rất lớn | Quá sức đội hiện tại |

### 6.4. Lộ trình đề xuất (khớp với roadmap 3 giai đoạn ở tài liệu cũ)

| Giai đoạn | Thời gian | Làm gì | Mức tách |
|---|---|---|---|
| **0 — Chuẩn bị** | Tuần 1–2 | Tạo `EventSchemaRegistry`, `EventFixtures`, `EventIngestRejects`, `IngestHealthDaily`. Viết pipeline 7 chốt. Dựng `POST /v1/events` + `:validate`. Outbox trong `cath-api`. | M1 (schema DB riêng ngay từ đầu) |
| **1 — Thu thập & phục vụ** | Tuần 3–6 | Client SDK bắn `reel.viewed`/`reel.impressed`. Job nền build `UserInterestProfiles`. Mở `POST /v1/rank` + `GET /v1/feed/reels`. `ReelService` gọi sang, có fallback. | M1 → tách thành container riêng cuối giai đoạn |
| **2 — Mở rộng** | Tháng 2–3 | `/v1/match/candidates`, `/v1/friends/suggest`, `/v1/notify/best-time`. Job Python cho CF. A/B qua `ExperimentAssignments`. | **M2 ⭐** |
| **3 — Tối ưu** | Tháng 4–6 | Two-tower + GBDT ranking, cold storage R2, cân nhắc Redis Stream | M2 (+queue) |

> 💡 **Insight PM:** điểm mấu chốt là **tách DB (schema riêng + user DB riêng) NGAY TỪ TUẦN ĐẦU**, còn tách container thì để cuối giai đoạn 1. Vì tách DB muộn là việc **cực khổ** (phải gỡ hàng chục chỗ JOIN chéo đã lỡ viết), còn tách container muộn chỉ là **một buổi chiều** viết Dockerfile. Làm đúng thứ tự này là được cả hai: có "microservice" như sếp muốn, mà không phải trả giá vận hành sớm.

### 6.5. Chỉ số vận hành cần theo dõi

| Nhóm | Chỉ số | Ngưỡng cảnh báo |
|---|---|---|
| Cửa vào | Tỷ lệ reject | > 2% |
| Cửa vào | Event nhận/giờ so với trung bình 7 ngày | lệch > 30% |
| Cửa vào | Độ trễ p99 (`IngestedAt − OccurredAt`) | > 5 phút |
| Phục vụ | p99 latency `/v1/rank` | > 80ms |
| Phục vụ | **Tỷ lệ fallback** (bao nhiêu % request phải dùng công thức cũ) | > 1% |
| Dữ liệu | % user có `UserInterestProfiles` cập nhật trong 24h | < 90% |
| Chất lượng | Job nền chạy quá hạn | > 2× thời gian trung bình |

---

## PHẦN 7 — Bảng tổng hợp: hỏi gì → trả lời ở đâu

| Câu hỏi của sếp | Mục trong tài liệu |
|---|---|
| Tầng nào tối ưu nhất? | Phần 2 (2.1 bảng so sánh 4 tầng, 2.2 kết luận lai 80/20, 2.3 phân công từng event) |
| Vì sao không phải tầng network? | 2.1 dòng "Network" |
| Lưu ở đâu bên trong service? | 2.4 (3 tầng lưu) + 2.5 (nơi tuyệt đối không lưu) |
| Làm sạch thế nào? | Phần 3 (7 chốt + insight nếu bỏ qua từng chốt) |
| Bảng test đầu vào? | Phần 4 (4 bảng + endpoint dry-run) |
| Call qua call lại thế nào? | Phần 5 (API generic `/v1/rank` + danh mục + SLA) |
| Tách biệt tới mức nào, khi nào? | Phần 6 (6.3 so sánh M1/M2/M3, 6.4 lộ trình) |

---

## PHỤ LỤC — Ví dụ payload chuẩn

```jsonc
// POST /v1/events
{
  "source": "client-web",
  "clientVersion": "2.4.0",
  "sentAt": "2026-07-26T09:12:03.221Z",
  "events": [
    {
      "eventId": "9f1c2b7e-...",          // UUID sinh ở client → chống trùng
      "eventType": "reel.viewed",
      "schemaVersion": 1,
      "occurredAt": "2026-07-26T09:11:58.004Z",
      "tzOffsetMinutes": 420,             // BẮT BUỘC — mất là không lấy lại được
      "subject": { "type": "user", "id": 42 },     // server vẫn ghi đè bằng JWT
      "object":  { "type": "reel", "id": 1088 },
      "action":  "view",
      "props": {
        "watchMs": 12400,
        "videoDurationMs": 18000,
        "completionPct": 68.9,
        "replayCount": 0,
        "isSkip": false
      },
      "context": {
        "surface": "home_feed",
        "feedPosition": 7,
        "sessionUuid": "1a2b...",
        "experiment": "feed_v2:B"
      }
    }
  ]
}
```

Cấu trúc `subject / object / action / props / context` này là thứ khiến service **không cần biết "reel" là gì** — và cũng là lý do feature mới chỉ tốn một dòng INSERT vào `EventSchemaRegistry`.

---

*Ghi chú cuối từ góc nhìn PM: câu hỏi của sếp "lưu tầng nào tối ưu" thực ra có một câu trả lời sâu hơn cả bảng so sánh — **tầng tối ưu là tầng gần nhất với nơi phát sinh Ý ĐỊNH của người dùng, không phải nơi thuận tiện nhất để đặt code.** Ý định "tôi thích video này" phát sinh ở ngón tay người dùng (client), ý định "tôi muốn kết bạn với người này" phát sinh ở logic nghiệp vụ (application). Bắt đúng chỗ ý định phát sinh thì dữ liệu tự nhiên sạch và giàu nghĩa; bắt ở tầng network là đi nhặt dấu chân sau khi người ta đã đi qua — luôn thiếu, luôn phải đoán.*

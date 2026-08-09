# CatSpeak — Đề xuất chiến lược Cá nhân hóa & Kích thích hành vi người dùng

*Vai trò: Project Manager | Cập nhật: 26/07/2026 (v3 — bổ sung dẫn chiếu sang tài liệu kiến trúc Microservice)*
*Căn cứ: đọc trực tiếp source code `catspeak-api` (cath-data, cath-service, cath-api, cath-analytics), `catspeak-ai` (agent, room-stt, moderation), `catspeak-client`*
*Tài liệu kèm theo:*
- *`CatSpeak_EERD.html` — sơ đồ EERD toàn bộ bảng cũ + mới*
- *`CatSpeak_KienTruc_Microservice.md` — **MỚI (v3)**: tách hệ cá nhân hóa thành microservice; trả lời 3 câu hỏi "lưu ở tầng nào / làm sạch ra sao / bảng test đầu vào"*

> 📌 **Lịch sử phiên bản:** v1 = đề xuất bảng DB + thuật toán · v2 = thêm mục 💡 INSIGHT cho từng loại dữ liệu · **v3 = thêm 3 hộp dẫn chiếu 🔗 (ở mục 1.1, mục "Việc cần làm ngay" của Phần 3, và bảng chỉ số) sang tài liệu Microservice. Nội dung cũ KHÔNG bị sửa hay xóa dòng nào.**

---

## PHẦN 0 — Hiện trạng từ source code (những gì ta ĐÃ có)

### 0.1. Chức năng & dữ liệu đang lưu

| Nhóm chức năng | Bảng/Entity hiện có | Dữ liệu đang lưu | Mức độ dùng được cho cá nhân hóa |
|---|---|---|---|
| **Tài khoản** | `Account` | Level, PreferredLanguage, Country, Tier, LoginCount, FirstLoginAt, LastSeen, IsOnline | ✅ Profile tĩnh — dùng làm feature nền |
| **Reels** | `Reel`, `ReelLike`, `ReelComment`, `ReelChallenge`, `HashtagRegistry` | Like/comment per-user, **ViewCount chỉ là số tổng** (không biết AI xem), hashtag + UseCount, LanguageCommunity | ⚠️ Thiếu dữ liệu quan trọng nhất: **ai xem gì, xem bao lâu** |
| **Đàm thoại (matching)** | `QueueEntry`, `Room`, `VideoChatSession`, `VideoChatParticipant` | Topic, RequiredLevel, LanguageType, RoomType người dùng chọn khi join queue; DurationSeconds mỗi người trong call | ✅ Rất quý — đây là "declared interest" + "engagement thật" |
| **Chat & bạn bè** | `Conversation`, `Message`, `Friendship` | Tin nhắn 1-1, trạng thái bạn bè | ✅ Đồ thị xã hội — nhưng **chưa có gợi ý bạn** (chỉ search tên) |
| **Posts/Cộng đồng** | `Post`, `PostView`, `Reaction`, `Comment` | Post **có** PostView per-user (Reel thì không!) | ✅ Mẫu tốt, cần nhân rộng sang Reel |
| **Stories** | `UserStory`, `UserStoryInteraction` | Accept/Decline per story | ✅ Tín hiệu implicit feedback |
| **Sự kiện** | `Event`, `EventRegistration`, `EventRegistrationCancellation` | Đăng ký/hủy, CountryId, CityId | ✅ Interest theo địa lý + chủ đề |
| **AI đàm thoại** | `catspeak-ai/agent` (proficiency_level, language), `room-stt` (Deepgram STT), `Recording`, `SubtitleService` | Prompt theo level; **transcript hội thoại có thể lấy được nhưng chưa lưu/khai thác** | ⚠️ Mỏ vàng chưa đào |
| **Hoạt động** | `UserActivityEvent` (login, join_room, create_post, send_message, ask_ai, react_post…, có cột `Metadata` JSON) | Event log tổng quát | ✅ Nền móng event tracking đã có sẵn — chỉ cần mở rộng |
| **Online time** | `AccountOnlineSession` | Start/End/DurationSeconds mỗi phiên online | ✅ Suy ra khung giờ hoạt động |
| **Presence** | `PresenceTrackerService` | Heartbeat 5s/lần nhưng **chỉ giữ in-memory, không persist** | ⚠️ Đang vứt đi dữ liệu khung giờ sử dụng |
| **Notification** | `Notification` | Chỉ có Content + Status, **không có loại, không track mở/click, không track giờ gửi** | ❌ Chưa đo được hiệu quả thông báo |

### 0.2. Thuật toán hiện tại trong code

1. **Reels feed** (`ReelService.GetReelsFeedAsync`): điểm suy giảm theo thời gian, **giống nhau cho mọi người**:
   ```
   Score = (Likes + 0.1 × Views) / (HoursOld + 2)^1.5 + random(0..0.5)
   ```
   → Đây là Hacker News–style decay, KHÔNG cá nhân hóa. Ai mở app cũng thấy gần như cùng một feed.

2. **Matchmaking** (`QueueService`): nhóm theo LanguageType → xếp FIFO theo `JoinedAt` → với Group room có chấm điểm **topic overlap** (số topic trùng). Chưa dùng lịch sử (đã từng gặp ai, cuộc gọi trước kéo dài bao lâu, có kết bạn sau call không).

3. **Search user** (`ScoreUserMatch`): chấm điểm khớp chuỗi username/nickname + bonus nếu là bạn. Không phải gợi ý bạn bè chủ động.

**Kết luận hiện trạng:** Nền móng rất tốt (event log, per-user interaction ở Post/Story, matching criteria do người dùng khai báo). Lỗ hổng lớn nhất: **(a)** không log lượt xem/thời lượng xem Reel theo từng user, **(b)** không có bảng hồ sơ sở thích tổng hợp, **(c)** không persist khung giờ hoạt động, **(d)** notification không đo lường được, **(e)** chưa khai thác transcript đàm thoại.

---

## PHẦN 1 — Cần lưu gì vào Database + INSIGHT rút ra từ mỗi loại dữ liệu

Nguyên tắc thiết kế giống TikTok/Facebook: **3 tầng dữ liệu**

```
Tầng 1: EVENT LOG (append-only, thô)      → ghi mọi hành vi, không xóa
Tầng 2: PROFILE/FEATURE (tổng hợp định kỳ) → job chạy nền gom event thành "hồ sơ sở thích"
Tầng 3: SERVING (đọc nhanh khi xếp feed)   → bảng nhỏ, index tốt, cache Redis
```

> 💡 **Nguyên tắc đọc insight (đọc trước khi xem từng bảng):** một dòng dữ liệu đơn lẻ KHÔNG phải insight — "user 42 xem reel 101 hết 12 giây" tự nó vô nghĩa. Insight chỉ xuất hiện khi tổng hợp theo 4 kiểu: **(1) tần suất** (lặp lại bao nhiêu lần), **(2) chuỗi** (hành động nào dẫn tới hành động nào), **(3) tỷ lệ chuyển đổi** (bao nhiêu % đi tiếp bước sau), **(4) so sánh nhóm** (nhóm này khác nhóm kia chỗ nào). Mọi mục 💡 bên dưới đều được suy theo 4 kiểu này. Bảng Tầng 1 = nguyên liệu; bảng Tầng 2 = insight đã được "vật chất hóa" thành cột để API đọc nhanh.

> 🔗 **BỔ SUNG v3 — Dữ liệu này chảy vào từ đâu?** Tài liệu này trả lời câu hỏi *lưu GÌ*; câu hỏi *ai ghi vào, ở tầng nào, làm sạch thế nào* được trả lời trong **`CatSpeak_KienTruc_Microservice.md`**. Tóm tắt để đọc tiếp cho liền mạch: mọi bảng Tầng 1 bên dưới **không được ghi trực tiếp** từ nhiều nơi, mà đi qua đúng một cửa `POST /v1/events` của Personalization Service; sự kiện nghiệp vụ (like, join queue, kết bạn, session end) bắn từ **tầng Application** (`cath-api`/`catspeak-ai` qua Outbox), còn hành vi UI (watch time, impression) bắn từ **tầng Client** và bị coi là không đáng tin cho tới khi qua đủ 7 chốt làm sạch. Ngoài 9 bảng A–I ở đây, cần thêm **4 bảng hạ tầng ingest**: `EventSchemaRegistry`, `EventFixtures`, `EventIngestRejects`, `IngestHealthDaily`.

### 1.1. Tầng 1 — Các bảng event mới cần thêm

**(A) `ReelViewEvents` — quan trọng số 1** (TikTok sống nhờ chính bảng này)

```sql
CREATE TABLE ReelViewEvents (
    Id              BIGSERIAL PRIMARY KEY,
    AccountId       INT NOT NULL,
    ReelId          INT NOT NULL,
    SessionUuid     UUID,            -- gom các view trong 1 phiên lướt
    WatchMs         INT NOT NULL,    -- xem được bao nhiêu ms
    VideoDurationMs INT NOT NULL,
    CompletionPct   REAL,            -- WatchMs/Duration, >100% nếu replay
    ReplayCount     INT DEFAULT 0,
    IsSkip          BOOLEAN,         -- lướt qua < 3s → tín hiệu ÂM
    Source          VARCHAR(20),     -- feed | profile | challenge | share_link
    FeedPosition    INT,             -- vị trí trong feed lúc hiển thị (để debias)
    CreatedAt       TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ix_rve_account_time ON ReelViewEvents(AccountId, CreatedAt DESC);
CREATE INDEX ix_rve_reel ON ReelViewEvents(ReelId);
```
Bản chất: bảng **log sự kiện append-only** — mỗi lần user lướt tới một video rồi rời đi là thêm 1 dòng mới (cùng user xem cùng video 3 lần = 3 dòng). Client gửi event khi user rời video (batch 5–10 event/request để đỡ tải). Kèm theo là bảng `ReelImpressions` (reel đã HIỂN THỊ nhưng chưa chắc xem) — cần cho việc tính CTR và tránh đề xuất lặp. Lưu ý: topic của video KHÔNG nằm ở bảng này mà ở `ReelFeatures` (mục G) — insight sinh ra khi JOIN hai bảng.

> 💡 **INSIGHT rút ra từ (A):**
> - **Khẩu vị thật của từng user** (khác cái họ bấm like): user like video Finance cho lịch sự nhưng watch time dồn hết vào Travel → Travel mới là sở thích thật. Watch time không biết nói dối — đây là lý do TikTok coi nó là tín hiệu số 1.
> - **Sở thích ÂM**: 5 lần skip liên tiếp video chủ đề X trong <3s = "đừng bao giờ hiện X nữa" — tín hiệu âm quý ngang tín hiệu dương, và không có nút "không quan tâm" nào bắt được nó tự nhiên bằng.
> - **Chất lượng thật của video/creator**: video 10k view nhưng completion 8% = title câu view, nội dung rỗng → hạ điểm phân phối. Ngược lại video ít view nhưng completion 85% = viên ngọc chưa được phát hiện → đẩy thêm. Đây là cách feed "công bằng" với creator mới.
> - **Điểm gãy nội dung**: phân bố WatchMs cho biết người xem rời ở giây thứ mấy → feedback cho content team và creator (video học tiếng nên dài bao nhiêu là vừa?).
> - **Chân dung phiên lướt** (`SessionUuid`): phiên lướt trung bình dài bao nhiêu video, bắt đầu chán ở video thứ mấy → quyết định điểm chèn nội dung "mồi" (challenge, gợi ý join queue) trước khi user thoát.

**(B) `ConversationSessionFacts` — chốt lại mỗi cuộc đàm thoại**

```sql
CREATE TABLE ConversationSessionFacts (
    Id              BIGSERIAL PRIMARY KEY,
    SessionId       INT NOT NULL,        -- FK VideoChatSession
    AccountId       INT NOT NULL,
    PartnerIds      INT[],               -- những ai trong call
    LanguageType    VARCHAR(20),
    Level           VARCHAR(20),
    TopicsDeclared  TEXT[],              -- topic user chọn khi join queue (copy từ QueueEntry)
    TopicsDetected  TEXT[],              -- topic trích từ transcript STT (giai đoạn 2)
    DurationSeconds INT,
    SpeakTimeSeconds INT NULL,           -- thời gian user thực sự nói (từ room-stt)
    EndedEarly      BOOLEAN,             -- rời trước 2 phút → tín hiệu match tồi
    FriendedAfter   BOOLEAN DEFAULT FALSE, -- có kết bạn trong 24h sau call không
    RatedScore      SMALLINT NULL,       -- nếu thêm màn hình rate sau call (nên làm!)
    CreatedAt       TIMESTAMPTZ DEFAULT now()
);
```
Nguồn: KHÔNG cần UI mới — job gom từ `QueueEntry` (topic user tự khai), `VideoChatParticipant.DurationSeconds`, `Friendship` khi session kết thúc (LiveKit webhook đã có). Với transcript STT: **DB chỉ lưu kết luận, không lưu nguyên liệu** — khi call kết thúc, chạy Gemini trích topic từ buffer transcript → ghi vài từ khóa vào `TopicsDetected` rồi vứt transcript thô (nhẹ DB + sạch về privacy).

> 💡 **INSIGHT rút ra từ (B):**
> - **Nhãn "match tốt/tồi" cho từng lần ghép cặp** — thứ hiện tại hoàn toàn không có: call ≥10 phút / kết bạn sau call / rate cao = tốt; EndedEarly = tồi. Có nhãn rồi mới trả lời được bằng số liệu: *ghép cùng level hay lệch 1 bậc thì call lâu hơn? Topic trùng quan trọng cỡ nào?* → chỉnh trọng số matching dựa trên bằng chứng thay vì cảm tính.
> - **Cặp nào nên gặp lại**: A–B từng nói chuyện 20 phút → boost re-match + gợi ý kết bạn "bạn từng đàm thoại với người này". Đây là gợi ý bạn bè có tỷ lệ chấp nhận cao nhất có thể làm.
> - **Khoảng cách khai báo vs thực tế**: user khai thích Politics nhưng transcript toàn nói về Food → TopicsDetected chỉnh lại profile chuẩn hơn cái user tự nghĩ về mình.
> - **Level thật**: SpeakTimeSeconds quá thấp trong phòng B2 nhiều lần liên tiếp = level tự khai đang cao hơn thực lực → `LevelEstimate` hạ xuống, gợi ý phòng dễ hơn → user bớt sợ, ở lại app.
> - **Sức khỏe matching theo phân khúc**: tỷ lệ EndedEarly cao bất thường ở (topic=Startups, level=A2, 15h chiều) = giờ đó phân khúc đó thiếu người trầm trọng → biết chính xác chỗ nào cần bơm thanh khoản (event, noti kéo người vào giờ đó).

**(C) `NotificationLog` — thay thế/mở rộng bảng Notification hiện tại**

```sql
CREATE TABLE NotificationLog (
    Id           BIGSERIAL PRIMARY KEY,
    AccountId    INT NOT NULL,
    Type         VARCHAR(40),      -- friend_request | reel_like | streak_reminder | event | match_ready
    Channel      VARCHAR(10),      -- push | in_app | email
    SentAt       TIMESTAMPTZ,
    OpenedAt     TIMESTAMPTZ NULL, -- NULL = chưa mở
    ClickedAt    TIMESTAMPTZ NULL,
    LedToSession BOOLEAN           -- trong 30 phút sau khi mở có phiên hoạt động không
);
```

> 💡 **INSIGHT rút ra từ (C):**
> - **Loại noti nào "sống", loại nào "chết" với từng user**: user A mở 40% noti streak nhưng 0% noti "ai đó like reel" → cá nhân hóa cả LOẠI noti chứ không chỉ giờ gửi. Không có bảng này thì noti là hộp đen — gửi mà không biết có ai đọc.
> - **Noti nào thật sự kéo người vào app** (`LedToSession`): mở noti nhưng không dùng app = noti chỉ gây phiền. Đây là metric ăn tiền hơn open rate.
> - **Ngưỡng bão hòa (fatigue)**: tỷ lệ mở giảm dần theo số noti/ngày của từng user → tìm được điểm gửi tối ưu trước khi user tắt notification vĩnh viễn — mất kênh push là mất kênh kéo về rẻ nhất, gần như không lấy lại được.

**(D) Persist presence:** thêm job mỗi 10–15 phút flush `PresenceTrackerService` xuống bảng `PresenceSnapshots(AccountId, HourBucket, Language)` — hoặc đơn giản hơn: dùng luôn `AccountOnlineSession` đã có, job đêm tính ra histogram 24 ô giờ cho mỗi user. (Không cần lưu từng heartbeat — chỉ lưu kết luận theo ô giờ.)

> 💡 **INSIGHT rút ra từ (D):** một dòng presence vô nghĩa, nhưng cộng dồn 2–4 tuần thì mỗi user hiện ra một **"nhịp sinh hoạt"**:
> - **Giờ vàng của từng user**: user này chỉ online 21–23h ngày thường → đó là khung giờ duy nhất đáng gửi noti; gửi 8h sáng là ném đá ao bèo và làm mòn thiện cảm.
> - **Ghép người trùng nhịp**: hai user có khung giờ online trùng ≥3 tiếng/tuần mới nên gợi ý kết bạn/match — đặc thù app đàm thoại: kết bạn với người không bao giờ online cùng lúc là kết bạn chết, không bao giờ gọi được cho nhau.
> - **Cảnh báo churn sớm**: nhịp đang thưa dần (tuần trước 5 phiên → tuần này 1 phiên, phiên ngắn dần) = sắp bỏ app → can thiệp TRƯỚC khi mất hẳn, không phải sau.
> - **Thanh khoản matching theo giờ** (mức cộng đồng): English peak 21h → xếp Event/Challenge giờ đó; user join queue giờ thấp điểm thì chủ động báo "21h tối nay có ~200 người online" thay vì để họ chờ mãi không match rồi bỏ đi với ấn tượng xấu.

**(E) Mở rộng `UserActivityEvent`** (bảng đã có, tận dụng cột `Metadata` JSON): thêm event type `reel_watch`, `reel_skip`, `queue_join`, `call_completed`, `call_left_early`, `story_accept`, `story_decline`, `notification_open`, `search_user`, `view_profile`. Quy ước metadata JSON thống nhất: `{"targetId":…, "topic":…, "durationMs":…, "source":…}`. Ví dụ 1 dòng thực tế: `(AccountId=42, EventType='reel_skip', Metadata='{"targetId":309,"source":"feed"}')`.

> 💡 **INSIGHT rút ra từ (E):** bảng này không cho MỘT insight cố định — nó là **nguyên liệu đếm được cho mọi câu hỏi hành vi**, kể cả câu hỏi 6 tháng nữa mới nghĩ ra (không ghi từ bây giờ thì lúc đó không có quá khứ để nhìn lại). Những gì đếm ra ngay:
> - **Phân loại persona người dùng**: 90% event là `reel_watch`, 0 lần `queue_join` = "người xem thụ động" → chiến dịch riêng dụ thử đàm thoại lần đầu (kèm thưởng); ngược lại "người gọi nhiều xem ít" → đẩy noti về reels. Mỗi persona một cách chăm sóc — đó là cá nhân hóa ở tầng chiến dịch, không chỉ tầng feed.
> - **Funnel chuyển đổi có số liệu**: `queue_join` → `call_completed` vs `call_left_early` theo từng topic/level = biết matching hỏng ở đâu; `notification_open` → có phiên hoạt động sau đó không = noti nào thật sự hiệu quả.
> - **Chuỗi hành vi báo trước ý định**: `view_profile` + `search_user` dồn về một người = tín hiệu xã hội mạnh → feed thẳng vào gợi ý kết bạn.
> - **Điểm mấu chốt**: một event không phải insight — insight nằm ở **tần suất, chuỗi, và tỷ lệ chuyển đổi giữa các event**. Chi phí bắt đầu ghi ≈ 0 vì bảng đã tồn tại.

### 1.2. Tầng 2 — Bảng hồ sơ tổng hợp (job nền cập nhật mỗi 15–60 phút)

*(Các bảng Tầng 2 chính là "insight đã vật chất hóa" — job nền JOIN các bảng Tầng 1, cộng điểm, rồi ghi kết luận vào đây để API đọc nhanh khi serving.)*

**(F) `UserInterestProfiles` — trái tim của cá nhân hóa**

```sql
CREATE TABLE UserInterestProfiles (
    AccountId       INT PRIMARY KEY,
    TopicScores     JSONB,   -- {"Music":0.82,"Travel":0.41,"Finance":0.05,...} (20 topic trong TopicType enum)
    HashtagScores   JSONB,   -- {"#ielts":0.9,"#daily_vlog":0.3,...} top 50
    CreatorScores   JSONB,   -- {"1024":0.7,...} creator hay xem, top 100
    LevelEstimate   VARCHAR(10),   -- level suy ra từ hành vi (khác level tự khai)
    ActiveHours     SMALLINT[24],  -- histogram giờ hoạt động (theo timezone user)
    ActiveDays      SMALLINT[7],
    AvgSessionMin   REAL,
    LastDecayedAt   TIMESTAMPTZ
);
```

**Cách cập nhật điểm topic — công thức EMA + decay (giống affinity của EdgeRank):**

```
score_topic(t) = score_topic(t-1) × λ + Σ(weight_action × strength)
λ (hệ số quên) = 0.95/ngày   → sở thích 1 tháng trước còn ~20% trọng số

weight_action gợi ý (chuẩn hóa từ hệ số các paper + chỉnh cho app học ngôn ngữ):
  xem hết reel (≥90%)      +1.0        like reel        +2.0
  replay                   +1.5        comment          +3.0
  share                    +4.0        skip (<3s)       −0.5
  chọn topic khi join queue +3.0 (declared intent — mạnh!)
  call cùng topic ≥ 10 phút +5.0 (engagement thật — mạnh nhất)
  rời call sớm khi match topic đó  −2.0
```

> 💡 **INSIGHT rút ra từ (F):** bảng này TRẢ LỜI câu "user này là ai" bằng con số, và có 2 insight mà chỉ CatSpeak có:
> - **Core interest = topic vừa XEM nhiều vừa chịu NÓI**: user xem 20 video Travel (passive) rồi vào phòng đàm thoại nói về Travel 15 phút (active) — tín hiệu chéo 2 nguồn này mạnh gấp trăm lần một cú like, và TikTok không bao giờ có vế thứ hai. Topic đạt cả 2 tiêu chí → ưu tiên tuyệt đối trong feed lẫn matching.
> - **`LevelEstimate` vs level tự khai**: user tự nhận B1 nhưng hành vi (video khó bị skip, nói ít trong phòng B1) cho thấy A2 → mọi thứ (reel, phòng, AI agent) tự hạ một bậc → giảm frustration, tăng retention. Sở thích còn có "hạn dùng" nhờ decay λ — hồ sơ luôn phản ánh user của THÁNG NÀY chứ không phải năm ngoái.

**(G) `ReelFeatures` — hồ sơ nội dung (phía item)**

```sql
CREATE TABLE ReelFeatures (
    ReelId          INT PRIMARY KEY,
    Topics          TEXT[],     -- gán từ hashtag + title/description (dùng Gemini đã tích hợp sẵn GeminiAIService)
    DifficultyLevel VARCHAR(10),-- A1..C2 — phân tích tốc độ nói + từ vựng từ subtitle (SubtitleService đã có!)
    Embedding       VECTOR(384) NULL, -- pgvector, giai đoạn 2
    -- thống kê engagement để tính điểm chất lượng:
    Impressions30d  INT, AvgCompletionPct REAL, LikeRate REAL, ShareRate REAL, SkipRate REAL
);
```

> 💡 **INSIGHT rút ra từ (G):** ngoài việc phục vụ feed (JOIN với TopicScores), bảng này cho insight **vận hành nội dung** mà hiện tại team hoàn toàn mù:
> - **Bản đồ cung–cầu content**: tổng hợp TopicScores toàn bộ user = CẦU; đếm ReelFeatures.Topics = CUNG. Chỗ nào cầu cao cung thấp (vd: nhiều người thích Finance nhưng chỉ có 12 video Finance) → content team biết chính xác cần đặt hàng/tự sản xuất video gì, challenge chủ đề gì. Đây là insight cho KẾ HOẠCH CONTENT, không chỉ cho thuật toán.
> - **Độ khó nào đang thiếu**: user A2 chiếm 60% nhưng 80% video là B2+ → giải thích tại sao completion rate thấp — vấn đề không phải thuật toán mà là kho content lệch.

**(H) `UserAffinityEdges` — đồ thị quan hệ người–người**

```sql
CREATE TABLE UserAffinityEdges (
    AccountIdA   INT, AccountIdB INT,
    MsgCount30d  INT,          -- từ Messages
    CallCount    INT,          -- số lần đàm thoại cùng nhau
    CallSeconds  INT,
    MutualFriends INT,
    LikesGiven   INT,          -- A like reel/post của B
    AffinityScore REAL,        -- tổng hợp có decay
    PRIMARY KEY (AccountIdA, AccountIdB)
);
```

> 💡 **INSIGHT rút ra từ (H):**
> - **Ai đang giữ chân ai**: với app xã hội, yếu tố retention mạnh nhất không phải content mà là CON NGƯỜI — user có ≥2 mối quan hệ AffinityScore cao gần như không churn (đây là lý do Facebook đo "7 friends in 10 days" làm north star thời kỳ đầu). Ngược lại user điểm affinity toàn bằng 0 sau 2 tuần = nguy cơ rời bỏ số 1 → mục tiêu can thiệp: giúp họ có MỘT mối quan hệ đầu tiên (ưu tiên match với "connector").
> - **Tìm ra "connector"** — user có nhiều edge mạnh, hay khiến người khác quay lại: đây là những người nên được chăm sóc đặc biệt (early access, badge), vì mất 1 connector là mất theo cả cụm bạn của họ.
> - **Boost feed theo độ thân**: nội dung của người AffinityScore cao xuất hiện trước — chính là chữ "u" (affinity) trong công thức EdgeRank của Facebook.

**(I) `UserStreaks` + `NotificationPreferences`**

```sql
CREATE TABLE UserStreaks (
    AccountId INT PRIMARY KEY,
    CurrentStreak INT, LongestStreak INT, LastActiveDate DATE,
    FreezeAvailable INT DEFAULT 1   -- "đóng băng streak" — vũ khí giữ chân của Duolingo
);
CREATE TABLE NotificationPreferences (
    AccountId INT PRIMARY KEY,
    BestHourLocal SMALLINT,   -- suy từ ActiveHours + NotificationLog.OpenedAt
    OpenRateByType JSONB,     -- {"streak_reminder":0.4,"reel_like":0.1}
    FatigueScore REAL         -- gửi nhiều mà không mở → tăng, tự động giảm tần suất
);
```

> 💡 **INSIGHT rút ra từ (I):** `CurrentStreak` là thước đo **momentum** — chỉ số dự báo retention đơn giản mà chính xác bậc nhất: user streak ≥7 ngày có xác suất ở lại tháng sau cao vượt trội, còn danh sách "streak sắp gãy tối nay" chính là danh sách can thiệp hằng ngày giá trị nhất của hệ thống noti (Duolingo xây cả đế chế retention quanh đúng insight này). `NotificationPreferences` là phiên bản chưng cất của (C)+(D): mỗi user một dòng trả lời sẵn "gửi GÌ, lúc MẤY GIỜ, tần suất BAO NHIÊU" — API noti chỉ việc đọc.

### 1.3. Tóm tắt: dữ liệu nào → insight nào (bảng tra nhanh)

| Dữ liệu | Insight cốt lõi | Trả lời câu hỏi |
|---|---|---|
| (A) ReelViewEvents | Khẩu vị thật (watch time không nói dối) + sở thích âm + chất lượng thật của content | User thích GÌ, ghét GÌ? Video nào đáng đẩy? |
| (B) ConversationSessionFacts | Nhãn match tốt/tồi + cặp nên gặp lại + level thật | Ghép AI với AI thì thành công? |
| (C) NotificationLog | Loại noti nào sống/chết với từng user, ngưỡng spam | Gửi noti GÌ thì được mở? |
| (D) Presence/OnlineSession | Nhịp sinh hoạt: giờ vàng, churn sớm, thanh khoản queue | Chạm vào user KHI NÀO? |
| (E) UserActivityEvent | Persona (người xem vs người gọi) + funnel + chuỗi hành vi | User là KIỂU người dùng gì, kẹt ở BƯỚC nào? |
| (F) UserInterestProfiles | Core interest (xem + nói) + level thật vs tự khai | Hồ sơ "user này là ai" bằng con số |
| (G) ReelFeatures | Bản đồ cung–cầu content theo topic và độ khó | Content team nên sản xuất GÌ? |
| (H) UserAffinityEdges | Ai giữ chân ai; user cô đơn = churn risk số 1; connector | Quan hệ NÀO đáng nuôi? |
| (I) Streaks/NotiPrefs | Momentum từng user + công thức gửi noti cá nhân | Hôm nay cần "cứu" ai? |

---

## PHẦN 2 — BẢNG MAPPING: Dữ liệu nào → suy ra gì → thuật toán gì → áp dụng ở đâu

| # | Dữ liệu thu thập (bảng) | Suy ra được gì | Thuật toán | Ứng dụng cụ thể trong app |
|---|---|---|---|---|
| 1 | `ReelViewEvents` (watch time, completion, skip, replay) | Mức độ quan tâm THẬT với từng video/chủ đề (implicit feedback — TikTok coi watch time là tín hiệu số 1) | **Giai đoạn 1:** cộng điểm EMA vào `UserInterestProfiles`. **Giai đoạn 2:** Item-based Collaborative Filtering ("người xem hết video này cũng xem hết video kia"). **Giai đoạn 3:** mô hình ranking dự đoán P(xem hết), P(like) | Feed Reels cá nhân hóa: `FinalScore = w1·P(watch) + w2·P(like) + w3·P(share) + freshness − seen_penalty` thay cho công thức decay chung hiện tại |
| 2 | `ReelViewEvents.IsSkip` | Chủ đề/creator user KHÔNG thích (negative signal) | Trừ điểm topic; lọc ứng viên (candidate filtering) | Không đề xuất lại video đã skip; giảm hẳn topic bị skip liên tục 5 lần |
| 3 | `QueueEntry.Topic/Level/Language` (đã có sẵn!) | Sở thích user TỰ KHAI (declared interest — quý hơn inferred) | Cộng thẳng điểm lớn vào TopicScores | Cold-start: user mới chưa xem gì nhưng vừa join queue topic "Travel" → feed Reels lập tức ưu tiên Travel |
| 4 | `ConversationSessionFacts` (duration, ended early, friended after, rated) | Chất lượng một cặp match; kiểu bạn nói chuyện user hợp | **Matching có trọng số:** `MatchScore = α·TopicOverlap + β·LevelFit + γ·PastPairQuality + δ·AffinityFoF − ε·RecentRepeat`; về sau học trọng số bằng logistic regression trên nhãn "call ≥10 phút / kết bạn sau call" | Nâng cấp matchmaking từ FIFO thuần → xếp cặp tối ưu; "Gặp lại người này?" nếu call trước tốt |
| 5 | `TopicsDetected` từ transcript STT (room-stt + Deepgram đã có) | Chủ đề user THỰC SỰ nói (khác cái họ khai); vốn từ, độ trôi chảy | Keyword/topic extraction bằng LLM (GeminiAIService có sẵn); ước lượng level bằng phân tích từ vựng | Cập nhật `LevelEstimate` → gợi ý phòng đúng trình độ; gợi ý reel có độ khó phù hợp (i+1 comprehensible input) |
| 6 | `Messages` + `Friendship` + cùng phòng đàm thoại | Đồ thị xã hội và độ thân | **Friend-of-Friend (FoF) + Adamic-Adar** trên đồ thị (bạn chung, phòng chung, event chung); về sau: Personalized PageRank / node embedding (cách LinkedIn PYMK, Facebook PYMK) | Mục "Gợi ý kết bạn": ưu tiên người (a) từng đàm thoại cùng, (b) nhiều bạn chung, (c) TopicScores tương đồng cosine, (d) cùng level + khung giờ online → tăng khả năng gặp lại |
| 7 | `UserAffinityEdges` | Ai quan trọng với user | Affinity score (chính là chữ "u" trong công thức EdgeRank `Σ u·w·d`) | Boost nội dung của bạn thân trong feed; sort danh sách chat; "bạn X vừa đăng reel đầu tiên" |
| 8 | `AccountOnlineSession` + `PresenceSnapshots` | Khung giờ vàng của từng user; nguy cơ churn (khoảng cách phiên giãn dần) | Histogram giờ → chọn giờ gửi noti; **survival analysis / RFM segmentation** cho churn | Push notification đúng giờ user hay mở app; user 3 ngày không vào + streak sắp mất → noti "cứu streak" |
| 9 | `NotificationLog` (opened, clicked, led_to_session) | Loại noti nào hiệu quả với user nào, giờ nào | **Multi-armed bandit (Thompson sampling)** chọn {giờ gửi × loại noti}; FatigueScore chặn spam | Hệ thống noti tự tối ưu: user A nhận "streak reminder 21h", user B nhận "bạn X online, vào nói chuyện không?" lúc 12h trưa |
| 10 | `UserStoryInteraction` (accept/decline) | Sở thích với format/nội dung story | Cộng/trừ điểm profile như #1 | Xếp thứ tự stories |
| 11 | `EventRegistration` + Country/City | Quan tâm sự kiện, vị trí | Content-based filtering theo topic + geo | "Sự kiện tiếng Anh gần bạn tuần này"; mời user hay hủy kèo bằng noti nhắc sớm hơn |
| 12 | `HashtagRegistry.UseCount` theo thời gian | Trend đang lên trong cộng đồng | Trending detection: z-score tăng trưởng theo giờ (giống Twitter Trends) | Tab "Đang thịnh hành"; gợi ý hashtag khi upload; tự tạo Challenge từ trend |
| 13 | `ReelFeatures.DifficultyLevel` × `LevelEstimate` | Video vừa sức user không | Rule: ưu tiên nội dung level user ±1 bậc | Người A2 không bị dí video C1 → giảm frustration, tăng watch time (đặc thù app học ngôn ngữ, TikTok không có) |
| 14 | `LoginCount`, `FirstLoginAt`, chuỗi ngày hoạt động | Giai đoạn vòng đời: new / activated / power / at-risk / churned | RFM + rule đơn giản trước, sau đó mô hình churn prediction (gradient boosting) | Chiến dịch theo vòng đời: ngày 1–7 hướng dẫn + match dễ; at-risk → email + noti khác thường lệ |
| 15 | `ReelImpressions` (đã hiển thị) | Tránh lặp; tính CTR thật | Bloom filter / bảng seen 7 ngày | Feed không hiện lại video đã xem (client đang tự gửi `excludeReelIds` — chuyển về server-side mới scale được) |

---

## PHẦN 3 — Lộ trình thuật toán 3 giai đoạn (đề xuất roadmap)

### Giai đoạn 1 (2–4 tuần, không cần ML): "Heuristic có cá nhân hóa"
Mục tiêu: thay công thức feed chung bằng công thức có hồ sơ user. Đủ dữ liệu là làm được ngay bằng SQL + C#.

```
FeedScore(user, reel) =
      2.0 × TopicMatch(UserInterestProfiles.TopicScores, ReelFeatures.Topics)
    + 1.5 × CreatorAffinity (đã follow/bạn bè/hay xem creator này)
    + 1.0 × QualityScore (AvgCompletionPct, LikeRate của reel — thay Likes+0.1·Views thô)
    + 0.8 × LevelFit (độ khó vừa sức)
    + decay thời gian như cũ
    − 5.0 × AlreadySeen
    + ε random (giữ 10–20% exploration — rất quan trọng, tránh filter bubble và giúp video mới có cửa)
```
Song song: bật `ReelViewEvents`, `NotificationLog`, `ConversationSessionFacts`, màn rate-sau-call, streak. **Thu thập dữ liệu ngay từ giai đoạn này là điều kiện cho giai đoạn 2–3.**

### Giai đoạn 2 (1–3 tháng): Collaborative Filtering + gợi ý bạn bè
- **Item-item CF** cho Reels: ma trận user×reel từ completion (implicit), tính similarity — hoặc dùng **ALS matrix factorization** (thư viện có sẵn: implicit/Spark MLlib; hoặc service Python riêng giống mô hình `catspeak-ai` đang có).
- **Friend suggestion**: FoF + Adamic-Adar + cosine(TopicScores) + "đã từng cùng phòng".
- **Matching v2**: học trọng số MatchScore bằng logistic regression, nhãn = call ≥ 10 phút hoặc kết bạn sau call.
- **Noti bandit**: Thompson sampling cho giờ gửi.

### Giai đoạn 3 (3–6 tháng, khi DAU đủ lớn): kiến trúc 2 tầng như TikTok/IG
```
Candidate Generation (retrieval)          Ranking
┌────────────────────────────┐    ┌─────────────────────────┐
│ • Two-tower embedding      │    │ Mô hình dự đoán đa mục   │
│   (user tower × item tower)│ →  │ tiêu: P(watch), P(like), │ → Re-rank (đa dạng
│ • CF neighbors             │    │ P(share), P(follow)      │    hóa, trộn topic,
│ • Trending + fresh pool    │    │ (GBDT trước, DNN sau)    │    quota video mới)
│ • Social (bạn bè đăng)     │    └─────────────────────────┘
└────────────────────────────┘
```
Lấy ~500 ứng viên từ nhiều nguồn → rank → trộn. Đây đúng là kiến trúc Instagram Explore và YouTube mô tả trong paper. Với quy mô startup, tầng ranking chỉ cần **GBDT (LightGBM)** là đủ, chưa cần deep model như Monolith.

### Vòng lặp gây nghiện (áp dụng Hooked Model) — khung sản phẩm bao quanh thuật toán
1. **Trigger**: noti đúng giờ vàng, đúng loại (insight C + D) + streak sắp mất (insight I)
2. **Action**: mở app → feed đã cá nhân hóa (insight A + F) → lướt/join queue 1 chạm
3. **Variable Reward**: video hay không đoán trước được (exploration ε) + match được người nói chuyện hợp (insight B) + like/comment đổ về
4. **Investment**: càng xem/càng nói → profile càng chuẩn → trải nghiệm càng hay; streak, bạn bè (insight H), level là "tài sản" khiến rời đi tốn kém

### Chỉ số đo (để biết thuật toán có "kích thích hành vi" thật không)
- Feed: avg watch time/session, completion rate, % skip, session length, retention D1/D7/D30 (API analytics đã có sẵn D1/D7/D30 — tận dụng)
- Matching: % match thành call ≥10 phút, % kết bạn sau call, repeat queue rate
- Noti: open rate theo type, % noti dẫn tới phiên hoạt động, opt-out rate
- Guardrail: report rate, thời lượng dùng quá mức (app học nên có "healthy engagement"), độ đa dạng topic của feed
- **Mọi thay đổi thuật toán phải chạy A/B test** — thêm bảng `ExperimentAssignments(AccountId, ExperimentKey, Variant)`.

> 🔗 **BỔ SUNG v3 — chỉ số hạ tầng phải theo dõi TRƯỚC chỉ số sản phẩm:** tỷ lệ event bị từ chối (> 2% là báo động), số event nhận/giờ so với trung bình 7 ngày (lệch > 30% = client hỏng chứ không phải user giảm), độ trễ ingest p99, và **tỷ lệ fallback** (bao nhiêu % request phải quay về công thức feed cũ). Mọi chỉ số sản phẩm ở trên đều đứng trên các chỉ số này — chi tiết ở mục 6.5 tài liệu Microservice.

### Việc cần làm ngay (tuần này)
1. Migration tạo `ReelViewEvents` + `ReelImpressions`; client bắn event watch/skip (batch).
2. Job gom `ConversationSessionFacts` từ LiveKit webhook khi session end.
3. Thay bảng `Notification` nghèo nàn bằng `NotificationLog` có type + opened tracking.
4. Persist presence/online-session thành histogram giờ.
5. Thêm màn rate 1–5 sao (hoặc 👍👎) sau mỗi call — 1 ngày dev, dữ liệu nhãn vô giá.
6. Job nightly build `UserInterestProfiles` phiên bản 1 (chỉ từ QueueEntry topics + ReelLike + view events).

> 🔗 **BỔ SUNG v3 — 3 việc phải chèn LÊN TRƯỚC 6 việc trên** (theo hướng microservice sếp yêu cầu):
> **0a.** Tạo **schema DB riêng** `personalization.*` + user Postgres riêng chỉ có quyền trên schema đó. *Tách DB muộn là cực khổ (phải gỡ hàng chục JOIN chéo đã lỡ viết); tách container muộn chỉ tốn một buổi chiều viết Dockerfile — nên làm đúng thứ tự này.*
> **0b.** Tạo 4 bảng hạ tầng ingest (`EventSchemaRegistry`, `EventFixtures`, `EventIngestRejects`, `IngestHealthDaily`) + pipeline 7 chốt làm sạch + endpoint `POST /v1/events` và `POST /v1/events:validate`.
> **0c.** Thêm bảng `OutboxEvents` trong `cath-api` để bắn event sang mà không mất và không chặn nghiệp vụ.
> Sau đó 6 việc trên vẫn giữ nguyên, chỉ khác: các bảng event được ghi **qua API của service** thay vì ghi thẳng. Đồng thời khi `ReelService` gọi sang `/v1/feed/reels`, **bắt buộc giữ lại công thức feed cũ làm fallback (timeout 200ms)** — service cá nhân hóa chết thì feed nhạt đi, chứ app không được lỗi.

---

## PHẦN 4 — Tài liệu tham khảo

### Các hệ thống thực tế của big tech
1. **TikTok/ByteDance — Monolith** (paper chính thức về hệ đề xuất real-time của ByteDance, online training, collisionless embedding): [arXiv 2209.07663](https://arxiv.org/abs/2209.07663) — bản [PDF](https://arxiv.org/pdf/2209.07663). Bài giải thích dễ đọc: [Paper review trên Medium](https://haneulkim.medium.com/paper-review-monolith-tiktoks-real-time-recommender-system-72b90bece653), [Simplifying TikTok's Monolith](https://medium.com/analytics-vidhya/simplifying-tiktoks-monolith-recommendation-algorithm-a7017f7ca3ab)
2. **Facebook News Feed ranking** (chính chủ Meta mô tả pipeline: inventory → signals → predictions → relevance score; hậu duệ của EdgeRank `Σ affinity × weight × decay`): [How ML powers Facebook's News Feed ranking](https://engineering.fb.com/2021/01/26/ml-applications/news-feed-ranking/)
3. **Instagram Explore** (kiến trúc retrieval → ranking nhiều tầng, account embeddings — mẫu chuẩn để CatSpeak copy ở giai đoạn 3): [Powered by AI: Instagram's Explore recommender system](https://instagram-engineering.com/powered-by-ai-instagrams-explore-recommender-system-7ca901d2a882) và bản mở rộng 2023: [Scaling the Instagram Explore recommendations system](https://engineering.fb.com/2023/08/09/ml-applications/scaling-instagram-explore-recommendations-system/)
4. **YouTube — Deep Neural Networks for YouTube Recommendations** (Covington et al., RecSys 2016 — paper kinh điển khai sinh kiến trúc candidate generation + ranking, dùng watch time làm mục tiêu): [PDF](https://cseweb.ucsd.edu/classes/fa17/cse291-b/reading/p191-covington.pdf), [Semantic Scholar](https://www.semanticscholar.org/paper/Deep-Neural-Networks-for-YouTube-Recommendations-Covington-Adams/5e383584ccbc8b920eaf3cfce3869da646ff5550), tóm tắt dễ hiểu: [the morning paper](https://blog.acolyer.org/2016/09/19/deep-neural-networks-for-youtube-recommendations/)
5. **Twitter/X — toàn bộ source code thuật toán For You được open-source** (đọc được cả trọng số like/retweet/reply thật, SimClusters, Real Graph — tham khảo trực tiếp cách chấm affinity người–người): [github.com/twitter/the-algorithm](https://github.com/twitter/the-algorithm), [the-algorithm-ml](https://github.com/twitter/the-algorithm-ml), blog chính chủ: [Twitter's Recommendation Algorithm](https://blog.x.com/engineering/en_us/topics/open-source/2023/twitter-recommendation-algorithm), bản chú giải cho dân recsys: [awesome-twitter-algo](https://github.com/igorbrigadir/awesome-twitter-algo)

### Riêng cho app học ngôn ngữ
6. **Duolingo — Half-Life Regression** (Settles & Meeder, ACL 2016 — mô hình spaced repetition dự đoán khi nào user quên từ, dùng để cá nhân hóa ôn tập và cả nội dung noti): [PDF chính chủ](https://research.duolingo.com/papers/settles.acl16.pdf), [code chính chủ trên GitHub](https://github.com/duolingo/halflife-regression), blog: [How we learn how you learn](https://blog.duolingo.com/how-we-learn-how-you-learn/), trang tổng hợp research: [research.duolingo.com](https://research.duolingo.com/)
7. **Nir Eyal — "Hooked: How to Build Habit-Forming Products"** (sách, khung Trigger–Action–Variable Reward–Investment dùng ở Phần 3) — Duolingo/TikTok đều là case study của mô hình này.

### Nền tảng thuật toán để đọc thêm
8. **Collaborative Filtering / Matrix Factorization**: Koren, Bell & Volinsky, *"Matrix Factorization Techniques for Recommender Systems"* (IEEE Computer 2009) — paper nền tảng từ Netflix Prize; và Hu, Koren & Volinsky, *"Collaborative Filtering for Implicit Feedback Datasets"* (ICDM 2008) — đúng bài toán của ta vì like/watch là implicit feedback.
9. **Two-tower retrieval**: Yi et al. (Google), *"Sampling-Bias-Corrected Neural Modeling for Large Corpus Item Recommendations"* (RecSys 2019) — kiến trúc retrieval giai đoạn 3.
10. **Meta DLRM** — mô hình deep learning recommendation mã nguồn mở của Meta: arXiv 1906.00091, github.com/facebookresearch/dlrm.
11. **Multi-armed bandits cho noti/exploration**: chương Bandit trong *"Reinforcement Learning: An Introduction"* (Sutton & Barto, bản free online) — đủ cho Thompson sampling ở mục noti.
12. **Open-source có thể dùng luôn**: [Gorse](https://gorse.io) (recommender engine self-host, Go, tích hợp qua REST — hợp để thử nhanh giai đoạn 2), thư viện Python `implicit` (ALS), `LightGBM` (ranking), `pgvector` (embedding trong chính Postgres đang dùng).

---

*Ghi chú cuối từ góc nhìn PM: lợi thế cạnh tranh của CatSpeak so với TikTok không phải là thuật toán giỏi hơn — mà là dữ liệu KHÔNG AI CÓ: người dùng vừa xem (passive) vừa NÓI (active). Một user xem 20 video về du lịch rồi vào phòng đàm thoại nói về du lịch 15 phút là tín hiệu sở thích mạnh gấp trăm lần một cú like. Thiết kế DB ở Phần 1 xoay quanh việc bắt trọn vòng lặp xem → nói → kết bạn → quay lại đó.*

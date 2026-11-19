# TAK12 ⇄ EduHub: thông báo chéo và sự kiện học tập

Sơ đồ các luồng tích hợp giữa TAK12, azVocab và EduHub (spec gốc: `eduhub/docs/specs/2026-11-12-dashboard-tong-hop-phu-huynh.md`, ADR-046).

Hai kênh, hai việc khác nhau — không trộn lẫn:

| Kênh | Chở gì | Chiều |
|---|---|---|
| **HTTP API** (`/api/eduhub/v1/*`, `X-Api-Key`) | Thông báo (hộp thư TAK12, EduHub pull), tài khoản, liên kết con | EduHub ⇄ TAK12 đồng bộ |
| **EventOutbox → NATS JetStream** | Sự kiện học tập bất biến (`quiz_attempt.completed`, `word.pass_completed`, `quiz.submitted`) | TAK12/azVocab → EduHub, at-least-once |

EduHub **không có bảng thông báo** (schema `notifications` chỉ có `CREATE SCHEMA`): hộp thư và trạng thái đã đọc nằm trọn ở TAK12 (`NotificationRecipients`, phân vùng cột `App`).

## 1. Màn Dashboard của phụ huynh — tải gì, từ bảng nào

```mermaid
sequenceDiagram
    autonumber
    actor P as Phụ huynh (trình duyệt)
    participant Web as eduhub-web (/app, BFF)
    participant Api as eduhub-api
    participant PG as PostgreSQL (eduhub)
    participant T as TAK12 /api/eduhub/v1

    P->>Web: GET /app
    Web->>Api: family: con active? (điều kiện hiện Dashboard)
    Api->>PG: family.guardianship, family.child
    PG-->>Api: children[]

    par Khối Thông báo
        Web->>Api: GET /v1/guardian/notifications
        Api->>PG: family.link_request (lời mời chờ, dashboard_dismissed_at)
        Api->>T: GET /guardians/{sub}/notifications?unread=true&limit=50 (X-Api-Key)
        T->>T: UserMappings (sub→UserId) → NotificationRecipients (App=eduhub, !IsDeleted, cutoff) + Campaign.LinkLabel
        T-->>Api: { items[], unreadCount } (bodyHtml lọc ở api trước khi ra trình duyệt)
        Api-->>Web: gộp 2 nguồn + số trên chuông (cache Valkey)
    and Khối Tình hình học tập
        Web->>Api: GET progress?childId&week
        Api->>PG: learning.progress_item (sum theo metric, local_date), learning.goal (phiên bản theo ngày)
        note over Api,PG: Lần mở đầu của mỗi con: tạo learning.backfill (pending) → job gọi History API lấy bù 12 tháng
    and Khối Đánh giá định kì
        Web->>Api: GET assessments
        Api->>PG: content.tak12_quiz (visible, cửa sổ) + learning.activity (lượt đầu theo quiz_id)
    and Hoạt động gần đây
        Web->>Api: GET activities
        Api->>PG: learning.activity (child_id, completed_at desc)
    end
    Web-->>P: Dashboard (goals chỉ gọi khi mở popup mục tiêu)
```

## 2. Con nộp bài trên TAK12 — publisher ghi gì, bảng nào

```mermaid
sequenceDiagram
    autonumber
    actor S as Học sinh (EK12)
    participant QS as QuizPublicAppService
    participant Pub as EduHubLearningEventPublisher
    participant DB as SQL Server (QuizSystem)
    participant Relay as EventOutboxRelayWorker (3s)
    participant NATS as NATS JetStream (CTH_TAK12)
    participant W as eduhub-worker
    participant PG as PostgreSQL (eduhub)

    S->>QS: SubmitQuizAttempt / StopQuizAttempt
    QS->>DB: QuizAttemptQuestions: chấm từng câu (IsCorrectStatus 0–2, CheckAnswer/CheckRetriedAnswer cập nhật CÙNG dòng khi làm lại)
    QS->>DB: QuizAttempts: AttemptPoint, FinishTimeUtc
    QS->>Pub: PublishQuizAttemptCompletedAsync(attempt) — cùng unit of work
    Pub->>DB: AbpUsers (Guid=CustomerId) → UserMappings → sub (không có sub → bỏ)
    Pub->>DB: EventOutbox: đã có EventId? (UUIDv5 theo attemptId → nộp/dừng lại không phát trùng)
    Pub->>DB: QuizAttemptQuestions: answeredCount (IsCorrectStatus 0–2, !IsParent, Quizzes: title, IsSpeakingQuiz)
    Pub->>DB: INSERT EventOutbox (Subject cth.tak12.pNN.quiz_attempt.completed.v1, Payload CloudEvents, Status=Pending)
    QS->>DB: COMMIT (bài + outbox một transaction)

    note over QS: Phiên dạng bài/chủ điểm bỏ ngang: job eduhub-close-idle-practice-attempts (5 phút) đóng sau 30 phút không trả lời thêm, rồi gọi đúng publisher này

    Relay->>DB: SELECT Pending (UPDLOCK, READPAST) ORDER BY Id
    Relay->>NATS: PublishAsync + Nats-Msg-Id = EventId
    NATS-->>Relay: ack → UPDATE Status=Sent, SentAt (lỗi: backoff 5s…30m, 10 lần → Dead)
    NATS->>W: durable pull consumer "eduhub"
    W->>W: validate zod (FamilyFacade.activeChildIdForSub con không có phụ huynh → ack, bỏ)
    W->>PG: 1 transaction: INSERT learning.event_log (PK source+event_id, trùng → ack) + upsert learning.activity + learning.progress_item
    note over W,PG: Dashboard thấy trong ≤ 60 giây (NFR-DATA-1)
```

## 3. Chiến dịch thông báo của admin → Dashboard phụ huynh

```mermaid
sequenceDiagram
    autonumber
    actor A as Admin TAK12 (AdminPanel)
    participant Svc as UserNotificationAppService
    participant HF as Hangfire
    participant Job as NotificationCampaignApplyJob
    participant DB as SQL Server (QuizSystem)
    participant GSvc as EduHubGuardianNotificationService
    participant Api as eduhub-api
    actor P as Phụ huynh (Dashboard)

    A->>Svc: CreateManualNotification (Audience: Students | EduhubGuardians | Both, LinkLabel ≤30)
    Svc->>DB: INSERT NotificationCampaigns (Status=Draft/Scheduled)
    opt Có ScheduledAt
        Svc->>HF: BackgroundJob.Schedule(ScheduledAt)
    end

    HF->>Job: ExecuteAsync (tới giờ, hoặc Send_Now_Queued)
    Job->>DB: TryClaim: UPDATE Status=Sending (1 dòng thắng)
    Job->>DB: ResolveAudience: AbpUsers (All / ImportedList / Filtered)
    alt Audience có phụ huynh EduHub
        alt TargetType = All
            Job->>DB: ExternalIdentityAccounts (Managed: CompletedAt, !Released, !Deleted) → toàn bộ phụ huynh EduHub
        else Học sinh đã chọn
            Job->>DB: UserMappings (học sinh → sub) → GuardianLinks (RevokedAt IS NULL) → sub phụ huynh → UserMappings → AbpUsers
        end
        Job->>DB: BulkInsert NotificationRecipients (App=eduhub, render %FULL_NAME%…)
    end
    opt Audience có học sinh
        Job->>DB: BulkInsert NotificationRecipients (App=tak12)
        Job->>Job: SignalR push CHỈ dòng App=tak12
    end
    Job->>DB: Campaign Status=Sent, RecipientCount

    P->>Api: mở /app (khối Thông báo + chuông)
    Api->>GSvc: GET /api/eduhub/v1/guardians/{sub}/notifications (X-Api-Key = EduHubApiKey)
    GSvc->>DB: UserMappings (sub→UserId) → NotificationRecipients (App=eduhub, unread, cutoff NotificationRetentionDays) + LinkLabel
    GSvc-->>Api: { items[{id chuỗi, title, bodyHtml, linkUrl, linkLabel, sentAt}], unreadCount }
    P->>Api: đóng thông báo
    Api->>GSvc: POST …/{id}/read → IsRead, ReadAt (dòng không thuộc sub → 404)
```

## 4. Dòng chảy NATS JetStream — azVocab và TAK12 vào EduHub

```mermaid
flowchart LR
    subgraph AZV["azVocab (PostgreSQL)"]
        LEARN["/api/learn/update-progress<br/>so boxes cũ/mới của mỗi nghĩa từ<br/>(cùng transaction updateLearnResult)"]
        AQUIZ["Nộp quiz trực tuyến<br/>(quiz_participants)"]
        AOUT[("event_outbox<br/>id · subject · payload · status<br/>attempts · next_retry_at")]
        POLLER["poller.js (PM2, chạy liên tục)<br/>FOR UPDATE SKIP LOCKED"]
        LEARN -- "word.pass_completed.v1<br/>data: learner{sub}, entryId<br/>id = UUIDv5(userId:entryId:defId:box.id)" --> AOUT
        AQUIZ -- "quiz.submitted.v1<br/>data: learner{sub}, quizId, title,<br/>score10 = accuracy/10" --> AOUT
        AOUT --> POLLER
    end

    subgraph TAK["TAK12 (SQL Server)"]
        SUBMIT["SubmitQuizAttempt / StopQuizAttempt<br/>+ job đóng phiên treo 30 phút"]
        TOUT[("EventOutbox<br/>Subject · EventId · Payload<br/>Status · Attempts · NextRetryAt")]
        TRELAY["EventOutboxRelayWorker<br/>UPDLOCK/READPAST, backoff, Dead sau 10 lần"]
        SUBMIT -- "quiz_attempt.completed.v1<br/>data: attemptId, learner{sub}, attemptType,<br/>quizId, practiceSessionId, title, skill,<br/>durationSeconds, answeredCount, score10<br/>id = UUIDv5(tak12:type:attemptId)" --> TOUT
        TOUT --> TRELAY
    end

    subgraph NATS["NATS JetStream (account EDUHUB)"]
        SAZ[["stream CTH_AZVOCAB<br/>cth.azvocab.pNN.>"]]
        STK[["stream CTH_TAK12<br/>cth.tak12.pNN.>"]]
    end

    POLLER -- "user azvocab, Nats-Msg-Id = id<br/>pNN = FNV-1a(learnersub) mod 16" --> SAZ
    TRELAY -- "user tak12, Nats-Msg-Id = EventId" --> STK

    subgraph EH["eduhub-worker → PostgreSQL"]
        CONS["durable pull consumer eduhub<br/>(mỗi stream một consumer, ack tường minh)"]
        VAL{"validate zod<br/>+ subject khớp type"}
        DEAD[("learning.ingest_dead_letter<br/>(giữ 30 ngày, replay CLI)")]
        FAM{"activeChildIdForSub?<br/>(family.guardianship)"}
        LOG[("learning.event_log<br/>PK (source, event_id) — khử trùng")]
        RM[("learning.activity (unique child+app+source_ref)<br/>learning.progress_item (dedupe_key:<br/>quiz:id:ngày · attempt:id · word:entry:ngày)")]
        CONS --> VAL
        VAL -- "sai schema → term" --> DEAD
        VAL -- hợp lệ --> FAM
        FAM -- "không có phụ huynh → ack, bỏ" --> DROP(("bỏ"))
        FAM -- có --> LOG
        LOG -- "cùng transaction" --> RM
    end

    SAZ --> CONS
    STK --> CONS
    RM --> DASH["Dashboard /app<br/>(tình hình học tập, hoạt động gần đây)<br/>p95 ≤ 60 giây"]
```

## Ghi nhớ khi sửa

- **Thông báo**: đổi gì ở hộp thư phụ huynh thì sửa TAK12 (`NotificationRecipients.App`, `EduHubGuardianNotificationService`) — EduHub chỉ là người đọc.
- **Sự kiện học tập**: `id` phải xác định (UUIDv5) để nộp lại/lấy bù không đếm trùng; thêm trường mới vào `data` là thay đổi tương thích trong cùng major, đổi nghĩa trường là `v2`.
- Hợp đồng chuẩn: `eduhub/packages/contracts/src/learning-events.ts` (zod) — TAK12 và azVocab phát đúng schema này, lệch là vào dead letter.

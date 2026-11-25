# User Nickname — thiết kế & migration

**Ngày:** 2026-11-25
**Phạm vi:** TAK12 (`CTH.QuizSystem`); AzVocab sẽ làm tương tự sau.
**Mục tiêu (EduHub):** Cha/mẹ có thể đặt/đổi tên cho con trên EduHub mà **không** làm đổi tên hiển thị của con trên TAK12 (và sau này AzVocab).

Thêm cột `AbpUsers.Nickname NVARCHAR(130) NULL`:

## Luồng tổng thể

```mermaid
flowchart TD
    P[1. Cha/mẹ submit form Đổi tên thật của con trên EduHub] --> BE[2. EduHub backend]
    BE --> KC[Gọi Keycloak<br/>cập nhật firstName / lastName]
    BE --> TK[Gọi API TAK12<br/>cập nhật Name / Surname]
    TK --> SN[AbpUsers.Surname / Name bị ghi đè<br/>— Nickname không ảnh hưởng]

    NK[(AbpUsers.Nickname)] --> UI[TAK12 hiển thị Nickname<br/>Leaderboard / Session / UserDto<br/>EK12 · AdminPanel]
```

## Luồng migration (script idempotent)

Chạy **một lần** (script vẫn idempotent — lỡ chạy lại chỉ đụng dòng còn NULL, không ghi đè nickname đã đặt):

```mermaid
flowchart TD
    S1["1. Thêm cột<br/>ALTER TABLE AbpUsers ADD Nickname NVARCHAR 130 NULL<br/>(guard sys.columns — đã có thì bỏ qua)"]
    S2["2. Backfill một lần<br/>UPDATE AbpUsers SET Nickname = TRIM Surname + ' ' + Name<br/>WHERE Nickname IS NULL<br/>(batch 5000 dòng; tên ghép rỗng → giữ NULL, không chép email)"]
    S1 --> S2 --> DONE([Hoàn tất])
```

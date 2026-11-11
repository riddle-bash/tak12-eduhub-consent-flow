CTH-TAK12-EDUHUB 

**tuổi = năm hiện tại (giờ VN) − năm sinh + 1**. Server tự tính lại tuổi, không tin cờ do client gửi.

| Bậc | Năm sinh (năm 2026) | Cha mẹ đồng ý | Con tự xác nhận (`selfConsent`) |
| --- | --- | --- | --- |
| ≤ 6 tuổi | 2020 trở về sau | Bắt buộc | `not_required`. Vào học ngay, không hỏi con |
| 7–15 tuổi | 2012 – 2019 | Bắt buộc | `pending` → `agreed` hoặc `ask_guardian`. Màn lời lẽ dễ hiểu ở lần đăng nhập đầu |
| ≥ 16 tuổi | 2011 trở về trước | Không | `pending` → con tự quyết. Chưa xác nhận thì chưa được học |

- **Test bắt buộc**Năm 2026: người sinh 2012 là 15 tuổi (cần cha mẹ), người sinh 2011 là 16 tuổi (tự quyết). Lấy đúng ví dụ ở §11.3 làm test.
- **Hai đường POST**`RegisterForm.jsx` và `views/AgentRegister` cùng gọi `Account/Register`. Luồng Đại sứ (người lớn) rơi vào bậc ≥ 16.


| Mã mục | Ai đồng ý | Loại | Ghi khi |
| --- | --- | --- | --- |
| `guardian_required` | Cha mẹ | Bắt buộc | Luồng 2 (trang xác nhận), luồng 3 (form), luồng 4 (`sourceApp = eduhub`), TAK12-21 |
| `guardian_optional` | Cha mẹ | Tuỳ chọn | Như trên. Từ chối cũng ghi một dòng |
| `child_required` | Con, từ 7 tuổi | Bắt buộc | Màn con tự xác nhận, form tự đăng ký ≥ 16 |
| `child_optional` | Con, từ 7 tuổi | Tuỳ chọn | Như trên |

- **Mỗi dòng**`subject, code, decision, policyVersion, decidedAt, sourceApp (eduhub | tak12)`.
- **Lưu văn bản**Khuôn `ConsentTexts` (văn bản + hash SHA-256) của DocumentForms vẫn dùng được để lưu câu chữ. Nó không thay được danh mục 4 mục. Vướng tenant vẫn còn: nên tạo bảng host-level riêng.

**Đồng ý cho con có TK và học (A)**Lưu ở kho đồng ý của TAK12, gồm 4 mục. Có thể đến từ trang xác nhận của TAK12 hoặc từ EduHUB (TAK12-4, TAK12-21).

**Liên kết PH–HS**Chỉ EduHUB ghi (`family.guardianship`). `GuardianLinks` ở TAK12 là bản sao do TAK12-21 ghi. Hợp đồng này **đã chạy**.

**Vai trò parent / student (mới, R11/R14)**Mỗi `sub` phải khớp vai trò ở hai bên: guardian EduHUB tương ứng `parent`, `learner_account` tương ứng `student`. Mọi đường tạo vai trò phải báo cho bên kia. Khi vai trò lệch thì cảnh báo, không tự sửa. Đổi lại, EduHUB không phải hỏi phụ huynh “TK này đang được ai dùng?” nữa.

## Diagram

```mermaid
flowchart LR
  subgraph KC["Keycloak realm cthad"]
    K1["user HS"]
    K2["user PH"]
  end
  subgraph T["TAK12"]
    T1[("AbpUsers + UserMapping<br/>HS và PH · vai trò parent/student")]
    T2[("Kho đồng ý 4 mục<br/>chỉ ghi thêm (MỚI)")]
    T3[("GuardianLinks · bản sao")]
    T7[("ExternalIdentityAccounts")]
    T5["Hai publisher (MỚI)<br/>FamilyRegistration · Consent"]
    T9["Gửi email / ZNS"]
  end
  subgraph E["EduHUB"]
    E1[("identity: guardian, learner_account,<br/>liên hệ đã xác thực")]
    E4[("family: child, guardianship,<br/>link_request, outbox (ĐÃ CHẠY)")]
    E5[("child_consent + product-api (MỚI)")]
  end
  T1 --- KC
  E -. "chỉ token endpoint" .-> KC
  E4 -- "outbox: students, guardian-links, guardians, accounts/search, emails" --> T1
  E4 -- "PUT guardian-links (TAK12-21)" --> T3
  T1 -- "learner-api (ĐÃ CHẠY)" --> E4
  T5 -- "POST /v1/product-api/family-registrations" --> E5
  T5 -- "consent (selfConsent, ask_guardian)" --> E5
  T5 -- "TryWriteAdoptedAsync" --> T7
  E -. "nhờ gửi (TAK12-18)" .-> T9
  
```

## Vòng đời TK HS

```mermaid
stateDiagram-v2
  direction LR
  state "TH 1 · Đồng ý của cha mẹ (bậc ≤ 15)" as G {
    state "Chờ cha mẹ: VÔ HIỆU (D1)" as P
    state "Đã đồng ý" as V
    state "Không cần (≥ 16)" as NR
    state "Hết hạn hoặc bị từ chối: xoá TK" as X
    [*] --> P: HS tự đăng ký (US04)
    [*] --> V: PH đăng ký cho con, hoặc TAK12-4
    [*] --> NR: tự đăng ký, ≥ 16
    P --> V: cha mẹ xác nhận (trong 3 ngày)
    P --> V: TAK12-21 active kèm consent
    P --> X: hết hạn hoặc cha mẹ từ chối
  }
  state "Trục 2 · Con tự xác nhận (selfConsent)" as S {
    state "not_required (≤ 6)" as SN
    state "pending" as SP
    state "agreed" as SA
    state "ask_guardian" as SG
    [*] --> SN
    [*] --> SP: từ 7 tuổi
    SP --> SA: con đồng ý
    SP --> SG: "Con muốn hỏi bố mẹ thêm"
    SG --> SA: sau khi hỏi bố mẹ
  }
  state "Trục 3 · Xác thực kênh (ĐÃ CÓ, theo từng kênh)" as C {
    state "phoneVerified / emailVerified = false" as U
    state "= true" as Ve
    [*] --> U
    U --> Ve: OTP hoặc link riêng tới ĐÚNG kênh đó
  }
  
```

- **Chặn học**HS dưới 16 tuổi chỉ học được khi đã có đồng ý của cha mẹ. Khi con mất liên kết `active` cuối cùng có kèm đồng ý, TAK12 chặn học.
- **Con ≥ 7 vẫn tự xác nhận**Kể cả khi cha mẹ đã đồng ý trên EduHUB. Chỉ bỏ được phần đồng ý của cha mẹ, không bỏ được phần của con (X7).
- **TK cũ**Không backfill đồng ý. Riêng vai trò thì có script backfill chạy một lần (§10.4).

## HS tự đăng ký, từ 16 tuổi

```mermaid
sequenceDiagram
  autonumber
  actor HS
  participant FE as RegisterForm.jsx
  participant AC as AccountController (Web.Mvc)
  participant KC as Keycloak
  participant PR as ProfileAppService
  participant PUB as IEduHubConsentPublisher
  HS->>FE: Làm quiz khách, form thông tin "Cho bản thân"
  AC->>AC: tuổi = năm VN − năm sinh + 1 ≥ 16
  FE-->>HS: Màn đồng ý TÁCH RIÊNG: child_required, child_optional (không đánh dấu sẵn)
  HS->>FE: Chọn từng ô
  FE->>AC: POST Account/Register {..., consent: {required, optional, policyVersion}}
  AC->>KC: Tạo Keycloak user
  AC->>AC: AbpUser, vai trò student, ghi 2 dòng đồng ý (sourceApp = tak12)
  AC->>AC: selfConsent = agreed nếu đã tick mục bắt buộc
  AC-->>FE: IsSignedIn = true
  opt Xác thực ngay
    FE->>PR: SendPhone/EmailVerificationCode, VerifyEmailCode
  end
  AC--)PUB: Enqueue(childUserId)
  FE-->>HS: Vào học
  
```

- **Chưa tick mục bắt buộc**TK vẫn được tạo nhưng `selfConsent = pending`: chưa được học, lần đăng nhập sau sẽ hỏi lại.
- **Keycloak lỗi**Nếu `IsSignedIn=false` thì bỏ qua bước OTP và rơi về banner nhắc.

## HS tự đăng ký, từ 15 tuổi trở xuống (US04)

```mermaid
sequenceDiagram
  autonumber
  actor HS
  participant FE as RegisterForm.jsx
  participant AC as AccountController (Web.Mvc)
  participant KC as Keycloak
  participant MSG as Email Register.* / ZNS
  actor PH
  participant GP as Trang xác nhận của cha mẹ
  participant PUB as Publishers
  HS->>FE: Làm quiz, "Cho bản thân", tuổi ≤ 15
  FE-->>HS: Khối "Xác nhận cho HS là trẻ em" + SĐT/email cha mẹ
  FE->>AC: POST Account/Register
  AC->>KC: Tạo user, VÔ HIỆU (enabled = false)
  AC->>AC: TK chờ, hạn 3 ngày, ẩn khỏi TAK12-5
  FE-->>HS: Màn chờ: "Đã gửi cho bố mẹ"
  par Mỗi kênh một token (D4)
    AC->>MSG: Email: link riêng cho kênh email
  and
    AC->>MSG: SMS: mã/link riêng cho kênh SĐT
  end
  PH->>GP: Mở link (token HMAC, mint/verify ở Web.Mvc)
  GP-->>PH: Màn đồng ý: guardian_required, guardian_optional
  PH->>GP: Xác nhận
  GP->>AC: Ghi đồng ý (sourceApp = tak12), kênh đã mở là verified
  AC->>KC: Bật user HS
  AC--)PUB: FamilyRegistration và Consent (cần đối chiếu §11 cho US04)
  alt HS từ 7 tuổi
    HS->>FE: Đăng nhập lần đầu: màn con tự xác nhận
  end
  opt Hết 3 ngày hoặc cha mẹ từ chối
    AC->>AC: Xoá TK chờ
  end
  
```

- **X6 · Xác thực theo kênh**`phoneVerified`/`emailVerified` chỉ là `true` khi đúng kênh đó được xác thực. Một link dùng chung cho cả hai kênh thì cả hai đều là `false`. Gửi mã cho cha mẹ rồi để HS nhập hộ thì SĐT đó cũng **không** được tính là đã xác thực.
- **Giữ nguyên từ kế hoạch**Token dạng HMAC + hạn + base64url, so sánh bằng `FixedTimeEquals`. Không truyền `?id=` trần. Trang theo khuôn `ActiveAccount`. Không dùng `EduHubControllerBase`.
- **Nếu D1 giữ “dùng bình thường”**Phải đưa lại pháp chế và sửa CR-DK-01 §1. Phương án thứ ba: tách “chờ xác thực” (bị chặn) khỏi “đã xác thực nhưng chưa hoàn tất hồ sơ” (không chặn). Phương án này phải được viết thành thiết kế riêng.
- **Nút «Vào EduHUB»**Mở `{PUBLIC_APP_URL}/#dang-nhap`. Không tự đặt ra đường dẫn khác.

## PH điền form “Cho con” trên TAK12

```mermaid
sequenceDiagram
  autonumber
  actor HS
  actor PH
  participant FE as RegisterForm.jsx
  participant AC as AccountController (Web.Mvc)
  participant PUB as IEduHubFamilyRegistrationPublisher
  participant E as EduHUB product-api
  HS->>FE: Làm quiz khách
  Note over HS,PH: PH tiếp quản máy tính
  PH->>FE: Form "Cho con": thông tin con (1 đến 10), SĐT/email PH
  AC->>AC: SĐT/email đã là parent hoặc PH EduHUB?
  alt Đã có
    AC-->>PH: Hướng sang đăng nhập, gắn con vào TK sẵn có
  else Chưa có
    FE-->>PH: Màn đồng ý tách riêng: guardian_required, guardian_optional
    FE->>AC: POST Account/Register (vai Cho con)
    AC->>AC: TK con (student) + TK PH (parent), username PH = 0 + 9 số
    AC->>AC: Mật khẩu trên form thuộc TK CON. PH nhận mật khẩu tạm qua SMS + email
    AC->>AC: Ghi đồng ý guardian_* (sourceApp = tak12), selfConsent theo tuổi
    opt OTP không bắt buộc
      FE->>AC: OTP tới SĐT PH, phoneVerified = true
    end
    AC--)PUB: Enqueue(guardianUserId)
    PUB->>E: family-registrations {guardian, children[] + selfConsent, guardianConsent}
    E-->>PUB: 200 {guardianId}
    PUB->>AC: TryWriteAdoptedAsync(externalId = guardianId)
  end
  
```

- **Tách mật khẩu**Nếu dùng chung một mật khẩu, con vào được TK của cha mẹ, đổi lựa chọn dữ liệu và gỡ liên kết.
- **Job bền**Tham số job **chỉ là ID người dùng**. Payload được dựng lúc job chạy. Thử lại với backoff, tối đa 24 giờ. Enqueue lại khi PH xác thực SĐT/email sau lần đẩy đầu.
- **Con từ 7 tuổi**Vẫn phải qua màn tự xác nhận ở lần đăng nhập đầu.

## PH tạo TK cho con từ EduHUB (TAK12-4)

```mermaid
sequenceDiagram
  autonumber
  actor PH
  participant E as EduHUB
  participant T as TAK12-4 /students
  participant KC as Keycloak
  actor HS
  PH->>E: "Tạo TK cho con", đồng ý guardian_*
  E->>T: POST /api/eduhub/v1/students {..., guardianConsent: {guardianSub, optional, policyVersion, decidedAt}}
  T->>KC: Tạo Keycloak user
  T->>T: AbpUsers + UserMapping, vai trò student
  T->>T: Ghi đồng ý guardian_* (sourceApp = eduhub)
  T-->>E: sub, userId, username, selfConsent (pending hoặc not_required)
  E-->>PH: Hiện username/mật khẩu một lần
  alt selfConsent = pending (từ 7 tuổi)
    HS->>T: Đăng nhập lần đầu, màn con tự xác nhận
  end
  
```

## Màn con tự xác nhận

```mermaid
sequenceDiagram
  autonumber
  actor HS
  participant T as TAK12
  participant PUB as IEduHubConsentPublisher
  participant E as EduHUB
  actor PH
  HS->>T: Đăng nhập, selfConsent = pending
  T-->>HS: Màn lời lẽ dễ hiểu: child_required, child_optional
  alt Con đồng ý
    HS->>T: Đồng ý
    T->>T: Ghi child_*, selfConsent = agreed
  else "Con muốn hỏi bố mẹ thêm"
    HS->>T: ask_guardian
    T->>T: selfConsent = ask_guardian, chưa được học
  end
  T--)PUB: Enqueue(childUserId)
  PUB->>E: consent
  opt ask_guardian
    E->>T: Nhờ gửi template EduHub.ChildAskedGuardian (TAK12-18)
    T->>PH: Email cho phụ huynh
  end
  
```

- **Khi nào enqueue**Khi con chọn trên màn này, khi TAK12 xác định con cần tự xác nhận, và khi TAK12-21 tạo dòng `GuardianLinks` `active` **đầu tiên** của con.

## Liên kết PH–HS

```mermaid
sequenceDiagram
  autonumber
  actor HS
  participant FA as EduHubFamilyAppService (TAK12)
  participant LA as EduHUB learner-api
  participant T as TAK12-18 emails
  actor PH
  participant GL as TAK12-21 guardian-links
  HS->>FA: Hồ sơ, "Mời bố mẹ", nhập liên hệ
  FA->>LA: POST /v1/learner-api/invitations + X-Learner-Sub
  LA->>T: Nhờ gửi EduHub.GuardianInvite
  T->>PH: Lời mời
  PH->>LA: Đăng nhập/đăng ký, chấp nhận, đồng ý guardian_*
  LA->>GL: PUT guardian-links/{id} {status: active, consent: {optional, policyVersion, decidedAt}}
  GL->>GL: Ghi bản sao, gán student nếu đang null, ghi đồng ý (sourceApp = eduhub)
  GL->>GL: Chỉ cập nhật khi decidedAt MỚI HƠN
  opt Liên kết active đầu tiên của con
    GL--)LA: IEduHubConsentPublisher.Enqueue(childUserId)
  end
  
```

- **Quy tắc**Chỉ EduHUB ghi quan hệ (spec 02/11 R7). TAK12 không tự ghi `GuardianLinks`.
- **revoked**Nếu con dưới 16 tuổi không còn liên kết `active` nào kèm đồng ý, TAK12 chặn học.

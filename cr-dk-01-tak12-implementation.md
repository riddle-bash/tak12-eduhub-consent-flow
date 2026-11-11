# CR-DK-01 · Phần TAK12 đã làm trong repo TiengAnhK12

Ghi lại yêu cầu đã nhận và phần đã cài, để team TAK12 review và để biết chính xác chỗ nào còn hở.

- **Nhánh:** `refine-guest-quiz-result`
- **Ngày:** 11/11/2026
- **Nguồn yêu cầu:** `src/.vscode/Luồng đăng ký TAK12 – EduHUB (1).md` (CR-DK-01 · BẢN 5, theo tài liệu phân việc 11/11/2026)

---

## 1. Yêu cầu đã nhận

> implement eduhub-tak12 (this repository) child approval and registration form and interface follow `Luồng đăng ký TAK12 – EduHUB (1).md`

Ba thứ được gọi tên: **child approval**, **registration form**, **interface**. Đối chiếu sang mục của tài liệu:

| Trong yêu cầu | Mục tài liệu | Nội dung |
| --- | --- | --- |
| interface | §14 (Việc 7) | `IEduHubAccountRoles`, `IEduHubConsentLedger` trong `EduHubGates.cs` — TAK12 cài bản thật |
| registration form | §7 US03, §8 US04, §9 US05, §10 HS ≥ 16 | Form chọn vai trò + năm sinh, màn đồng ý, màn chờ, Google/Facebook |
| child approval | §8 US04, §12 | Trang cha mẹ xác nhận cho con học (ảnh 17) và màn con tự xác nhận (ảnh 11, 12, 14) |

Ba thứ đó không đứng riêng được: cả ba đều đọc/ghi vai trò và kho đồng ý, nên **§2 (luật tuổi), §3 (kho đồng ý), §4 (vai trò)** — tức giai đoạn B của §16 — được làm kèm như nền.

### Ngoài phạm vi, có chủ ý

Theo §1 và §15, những phần sau **do team EduHUB viết** trên nhánh `fea-eduhub-dong-y-du-lieu`, TAK12 chỉ review:

- API `/api/eduhub/v1/*` (`students`, `accounts/search`, `guardians`, `adopt`, `guardian-links`, `accounts/{sub}/role`)
- Hai job `FamilyRegistration` và `Consent` — TAK12 chỉ gọi `Enqueue` đúng chỗ
- Script `BackfillAccountRoles`, `RepublishFamily`, `RepublishSelfConsent`
- Màn đồng ý trên EduHUB (US01, US02)

Phần này **chưa có trong repo**. Các `interface` publisher đã được định nghĩa sẵn để EduHUB cắm vào mà không phải sửa mã TAK12.

---

## 2. Tổng quan phần đã cài

**24 file mới (~3.305 dòng)** và **19 file sửa (+728 / −23 dòng)**.

```
Core/EduHub            hợp đồng + luật tuổi + 2 entity        593 dòng
Application/EduHub     bản cài thật + 3 service + 1 job     1.932 dòng
EntityFrameworkCore    1 script SQL idempotent                234 dòng
Web.Mvc                1 controller công khai                  82 dòng
EK12/ClientApp         2 màn hình + bản gương luật tuổi        464 dòng
```

---

## 3. Interface (Việc 7, §14)

### `src/CTH.QuizSystem.Core/EduHub/EduHubGates.cs` — 284 dòng

Hợp đồng duy nhất mà mã EduHUB trong repo TAK12 được phép đi qua:

- `enum EduHubAccountRole` (Parent/Student), `EduHubRoleAssignment` (Assigned/Unchanged/Conflict), `EduHubSelfConsentDecision` (NotRequired/Pending/Agreed/AskGuardian)
- `EduHubSelfConsent`, `EduHubGuardianConsent` (có thêm `IpAddress`, `UserAgent` theo §3)
- `IEduHubAccountRoles`, `IEduHubConsentLedger` — đúng chữ ký trong §14
- `IEduHubFamilyRegistrationPublisher`, `IEduHubConsentPublisher` — chỗ EduHUB cắm hai job vào
- `IEduHubSmsSender` — **thêm ngoài tài liệu**, xem mục 8
- Bốn/năm bản mặc định `NullEduHub…` và `EduHubGatesLog`

**Quyết định đáng chú ý.** Các lớp `NullEduHub…` **không** cài `ITransientDependency`/`ISingletonDependency`. Nếu cài, cả bản mặc định lẫn bản thật cùng đăng ký cho một service và bản nào thắng là chuyện may rủi theo thứ tự nạp module. Thay vào đó chúng được đăng ký tay dạng Castle `IsFallback()` trong `QuizSystemCoreModule.Initialize` — Castle chỉ dùng fallback khi không có bản nào khác phục vụ cùng service, nên bản thật ở tầng Application luôn thắng, bất kể thứ tự nạp.

### Bản cài thật

| File | Dòng | Ghi chú |
| --- | --- | --- |
| `Application/EduHub/Tak12EduHubAccountRoles.cs` | 106 | Vai trò ở `AbpUsers.EduHubRole` |
| `Application/EduHub/Tak12EduHubConsentLedger.cs` | 225 | Kho đồng ý, chỉ ghi thêm |

`AssignAsync` ghi bằng `UPDATE … WHERE Id = @id AND EduHubRole IS NULL` chứ không đọc-rồi-ghi: hai request song song (EduHUB gọi `/accounts/{sub}/role` đúng lúc TAK12 đang gắn con vào phụ huynh) cùng đọc ra `null` rồi cùng ghi thì vai trò sau đè vai trò trước mà không ai biết. Ghi có điều kiện biến chuyện đó thành «0 dòng bị sửa», và lớp này đọc lại để trả đúng `Unchanged`/`Conflict`.

`RecordGuardianConsentAsync` so trùng theo `DecidedAtUtc` **tới mili giây** — đó là lý do cột `datetime2(7)` và không được làm tròn ở bất kỳ đâu (§3).

`GetSelfConsentAsync` không bao giờ trả `null`. Một trường hợp biên đáng ghi lại: con đã bấm `ask_guardian` lúc 15 tuổi rồi sang 16 — màn của em giờ là ảnh 11 (chỉ có «đồng ý»), nên nếu vẫn báo `ask_guardian` thì EduHUB hiện một nhãn mà em không còn đường nào gỡ. Lớp này lùi về `pending`.

### Dòng log nghiệm thu

Sau khi khởi động, `QuizSystemApplicationModule.PostInitialize` in:

```
[EDUHUB] gates | roles=Tak12EduHubAccountRoles | ledger=Tak12EduHubConsentLedger
        | familyPublisher=NullEduHubFamilyRegistrationPublisher (unavailable)
        | consentPublisher=NullEduHubConsentPublisher (unavailable)
        | sms=NullEduHubSmsSender (unavailable)
```

Hai publisher và SMS còn `NullEduHub…` là **đúng hiện trạng** — chúng thuộc phần EduHUB viết, chưa có trong repo.

---

## 4. Mô hình dữ liệu (§2, §3, §4 — giai đoạn B)

### `Migrations/Scripts/20261111_EduHubRegistrationConsent.sql` — 234 dòng, idempotent

Theo `.cursor/rules`: **không** thêm EF migration `.cs`. Entity, `QuizSystemDbContext` và `QuizSystemDbContextModelSnapshot.cs` đã sửa tay cho khớp.

| Đối tượng | Nội dung |
| --- | --- |
| `AbpUsers.EduHubRole` | `tinyint NULL` — 1 parent, 2 student, NULL = chưa rõ |
| `EduHubConsentEntries` | Kho đồng ý, chỉ ghi thêm. 2 index |
| `EduHubChildApprovals` | Tài khoản chờ cha mẹ xác nhận (US04). 2 index |
| 3 `MessageTemplates` | `EduHub.ChildAskedGuardian`, `EduHub.GuardianApproveChild`, `EduHub.ChildAccountApproved` |

Script **không** chạy `UPDATE` gán vai trò hàng loạt cho dòng cũ (§4: «TK cũ để trống»). Việc đó thuộc `BackfillAccountRoles` do EduHUB viết, có `dryRun`, chỉ chạm 3 nhóm đã rõ.

`EduHubConsentEntries` không có khoá ngoại sang `AbpUsers`: kho đồng ý là bằng chứng pháp lý, phải sống sót qua mọi thao tác dọn tài khoản.

### Luật tuổi — `Core/EduHub/EduHubAgeRules.cs`

`Tuổi = năm hiện tại (giờ VN) − năm sinh + 1`, đúng §2. Bản gương ở `EK12/ClientApp/src/common/utils/eduhubAge.js`.

Quy ước chung hai bên: **không biết năm sinh ⇒ coi như phải hỏi / phải có cha mẹ**. Thà hỏi thừa một màn còn hơn cho một đứa trẻ vào học mà chưa ai hỏi nó.

### Mã mục — `Core/EduHub/EduHubConsentCodes.cs`

| Mã | Ai ghi |
| --- | --- |
| `child_required`, `child_optional` | TAK12 và EduHUB (mã dùng chung, §3) |
| `child_self_agreement` | **Chỉ TAK12.** Mã màn con tự xác nhận — tự đặt, xem mục 8 |

Quyết định ghi dưới dạng chuỗi `granted` / `declined` / `ask_guardian`: kho chỉ ghi thêm, nên «con muốn hỏi bố mẹ thêm» phải là một giá trị quyết định chứ không thể là việc sửa một dòng đã ghi.

---

## 5. Registration form (US03, US04, US05, HS ≥ 16)

### Máy chủ

| File | Vai trò |
| --- | --- |
| `Application/EduHub/EduHubRegistrationFlow.cs` (269) | Phân nhánh ba đường theo vai + tuổi |
| `Application/EduHub/EduHubFamilyAccountFactory.cs` (243) | Tìm / tạo tài khoản phụ huynh |
| `Web.Mvc/Controllers/AccountController.cs` (sửa) | Gọi flow sau `UserMapping`, bỏ đăng nhập cho TK chờ |
| `Web.Mvc/Models/Account/RegisterInputDto.cs` (sửa) | +4 trường |
| `Web.Mvc/Models/Account/RegisterOutputDto.cs` (sửa) | +3 trường |

**Ba đường:**

- **US03 / US05 (phụ huynh điền)** — tìm hoặc tạo tài khoản phụ huynh (tên đăng nhập = SĐT 10 số), gán `parent` + `student`, ghi kho đồng ý nguồn `tak12`, rồi `Enqueue(ID phụ huynh)`.
- **US04 (HS < 16 tự điền)** — mở dòng chờ, **vô hiệu user Keycloak** (`enabled = false`), gửi link cho cha mẹ, trả `requiresGuardianApproval` để form hiện màn chờ.
- **HS ≥ 16 tự điền** — chỉ gán `student`. Chưa có liên kết nên không gọi publisher (§10).

Chưa cài kho đồng ý thật (`IsAvailable == false`) thì **cả ba đường đều no-op** — bước 3 của §18 yêu cầu TAK12 chạy y như cũ.

**Thứ tự trong `Register` không phải chuyện thẩm mỹ:**

1. `ValidateBeforeCreate` chạy **trước khi chạm Keycloak** — chặn muộn hơn là để lại một tài khoản con không có dòng đồng ý nào đứng sau nó.
2. Flow chạy **sau** khi `UserMapping` của con đã ghi — job family-registrations đi từ `AbpUsers` sang sub Keycloak qua bảng đó; thiếu nó thì job thử lại tới hết lượt rồi bỏ (§7).
3. Flow nằm **trong** khối `try` có dọn Keycloak — hỏng ở đây phải xoá lại danh tính con như mọi lỗi khác.
4. `Enqueue` gắn vào `UnitOfWork.Completed`, không gọi thẳng — job đọc thẳng DB (§15).

**Tách liên hệ phụ huynh khỏi liên hệ của con.** Ô `email` trên form nằm trong khối «thông tin tài khoản học» và thuộc về con; nếu dùng nó cho tài khoản phụ huynh thì mật khẩu tạm của phụ huynh được gửi tới hộp thư của một đứa trẻ. Vì vậy có `GuardianEmail` riêng. Ở đường phụ huynh, ô SĐT nằm trong khối phụ huynh nên `PhoneNumber` **là** số của phụ huynh; ở đường US04 thì `PhoneNumber` là của con và liên hệ cha mẹ đi qua `GuardianPhoneNumber`.

### Form (EK12 ClientApp)

`RegisterForm.jsx` (+174), `registerValidation.js` (+38):

- Hai ô đồng ý (`child_required` / `child_optional`), chỉ hiện ở vai phụ huynh. Ô bắt buộc **không tick sẵn** — một ô đã tick sẵn không phải là sự đồng ý, nó chỉ là một ô người ta quên bỏ tick. Ô tuỳ chọn bật sẵn.
- Khối «Liên hệ của bố mẹ», chỉ hiện khi HS tự đăng ký và dưới 16 tuổi — xuất hiện/biến mất theo ô năm sinh ngay khi đang gõ, kèm dòng nói rõ **lý do**.
- Luật «ít nhất một kênh»: dòng lỗi của nó đặt dưới cả cặp ô chứ không gắn vào một ô, vì nó không thuộc về ô nào.
- Màn chờ US04 (ảnh 15) thay màn thành công, và **không có nút «Đăng nhập ngay»** — tài khoản đang bị vô hiệu, bấm vào chỉ dẫn tới một lần đăng nhập hỏng.
- Màn thành công US03 ghi «Mật khẩu: Đã gửi tới số điện thoại và email của anh/chị». Mật khẩu tạm **không bao giờ** xuống client.

**US05 (Google/Facebook)** đi qua đúng `AccountController.Register` nên thừa hưởng toàn bộ nhánh trên; không phải sửa riêng.

---

## 6. Child approval

### 6a. Cha mẹ xác nhận cho con học (US04, ảnh 17)

| File | Dòng |
| --- | --- |
| `Core/EduHub/EduHubChildApproval.cs` | 86 |
| `Application/EduHub/EduHubGuardianApprovalService.cs` | 423 |
| `Application/EduHub/EduHubGuardianApprovalToken.cs` | 164 |
| `Application/EduHub/EduHubGuardianApprovalDtos.cs` | 79 |
| `Application/EduHub/EduHubPurgeExpiredChildApprovalsJob.cs` | 54 |
| `Web.Mvc/Controllers/EduHubApprovalController.cs` | 82 |
| `EK12/.../views/GuardianApproval/GuardianApproval.jsx` | 227 |

**Mỗi kênh một token** (§8 D4). Format `{approvalId}.{channel}.{expiry}.{hmac}`, ký bằng khoá JWT sẵn có với tiền tố mục đích `eduhub-guardian-approval|` — cùng khuôn `QuizAttemptViewToken`, không phải cấp secret mới cho từng môi trường. Id nằm **trong** payload ký nên token gia đình A không mở được dòng gia đình B, và token kênh email không dùng để xác thực SĐT được.

Mở link **là** bằng chứng sở hữu kênh đó, và chỉ kênh đó: `MarkChannelConfirmed` chỉ đặt `IsPhoneNumberConfirmed`/`IsEmailConfirmed` cho đúng kênh trên token. EduHUB tin hai cờ này để cho đăng nhập bằng SĐT và gợi ý con theo SĐT — đánh dấu nhầm kênh là lộ dữ liệu của một đứa trẻ cho người lạ (§6).

Trang công khai, không đăng nhập (phụ huynh có thể chưa có tài khoản TAK12 nào). Mọi token hỏng — sai chữ ký, hết hạn, dòng đã bị dọn — đều trả **cùng một** trạng thái `Invalid`: nói ra token sai ở chỗ nào là giúp người dò token.

`Approve` bật user Keycloak **sau cùng**, khi mọi thứ đã ghi: bật trước rồi hỏng ở bước sau là một tài khoản đăng nhập được mà không có đồng ý nào đứng sau nó.

Hạn 7 ngày; quá hạn hoặc cha mẹ từ chối thì xoá TK chờ — **Keycloak trước, AbpUsers sau**: xoá bản ghi TAK12 mà để lại danh tính Keycloak là khoá vĩnh viễn tên đăng nhập đó khỏi lần đăng ký sau của chính đứa trẻ ấy. Job `eduhub-purge-expired-child-approvals` quét theo lô 200 dòng.

### 6b. Con tự xác nhận (§12, ảnh 11/12/14)

| File | Dòng |
| --- | --- |
| `Application/EduHub/EduHubSelfConsentAppService.cs` | 191 |
| `Application/EduHub/EduHubSelfConsentDtos.cs` | 43 |
| `EK12/.../features/Layout/SelfConsentDialog.jsx` | 148 |

Modal chặn ngay sau khi đăng nhập, không phải một mục trong trang hồ sơ: hỏi ở chỗ phải đi tìm mới thấy thì thực tế là không hỏi. Không có nút đóng — máy chủ cũng chặn học cho tới khi có câu trả lời, nên một nút đóng chỉ dẫn tới màn học trống không giải thích được.

Bậc 7–15 có hai lựa chọn; bậc ≥ 16 chỉ có «Tôi đồng ý». Đã bấm `ask_guardian` thì màn vẫn mở ở lần sau để con còn đường đồng ý (ảnh 14), kèm dòng nhắc bố mẹ đã được báo.

`Enqueue(childUserId)` gọi mỗi lần con bấm, gắn vào `UnitOfWork.Completed`.

Email `EduHub.ChildAskedGuardian` **không** gửi từ đây: theo §12, EduHUB nhận trạng thái `ask_guardian` rồi mới hiện nhãn và gửi thư một lần. TAK12 chỉ ghi dòng và gọi publisher. (Template vẫn được seed trong script để sẵn sàng; guard `NOT EXISTS` nên seed của EduHUB chạy sau cũng vô hại.)

### 6c. Chặn học — `Application/EduHub/EduHubLearningGate.cs` (135)

Một chỗ duy nhất trả lời «tài khoản này có được học không», để trang học, API làm bài và màn đăng nhập không trả lời lệch nhau. Bốn lý do: `None`, `SelfConsentRequired`, `GuardianLinkRevoked`, `AwaitingGuardianApproval`.

**Điều kiện là «đã từng có liên kết mà giờ không còn cái nào hoạt động», không phải «không có liên kết nào».** Tài khoản con vừa tạo qua US03/US05 chưa có dòng `GuardianLinks` nào — EduHUB đẩy liên kết về sau khoảng một phút — nên chặn vì «chưa có liên kết» sẽ chặn đúng những đứa trẻ mà cha mẹ vừa đăng ký hộ xong. Vòng đời §6 cũng nói thế: US03/US05 vào thẳng trạng thái «được học», chỉ chuyển sang «bị chặn» khi liên kết **cuối cùng** bị gỡ.

---

## 7. Kiểm chứng

| Hạng mục | Kết quả |
| --- | --- |
| `dotnet build "CTH.QuizSystem - Net 8.sln"` | **0 error** |
| `npm run build` (EK12 ClientApp) | Compiled — 104 warning, **đúng bằng** baseline, không warning nào thuộc file mới |
| `node scripts/test.js` (EK12) | **805 pass** (baseline 793), 39 fail / 11 suite — **đúng bằng** baseline, không suite đỏ mới |

Baseline đo bằng `git stash` trên cùng máy. 11 suite đỏ là lỗi sẵn có (`window is not defined` lúc nạp module), không liên quan thay đổi này.

**12 test mới:**

- `eduhubAge.test.js` (6) — bốn mốc §2 (2021→6, 2020→7, 2012→15, 2011→16), năm tính theo giờ VN, và nhánh không biết năm sinh.
- `registerValidation.test.js` (6) — ô đồng ý bắt buộc, luật «ít nhất một kênh», ranh giới 16 tuổi, lỗi định dạng từng kênh.

6 test cũ trong `registerValidation.test.js` được **sửa kỳ vọng** vì luật mới làm đổi kết quả thật (ví dụ form trống giờ thiếu thêm `guardianContact` — năm sinh trống ⇒ coi như dưới 16).

---

## 8. Còn hở — cần quyết

### 8.1 Chưa có nhà cung cấp SMS — **hở thật**

Repo không có đường gửi SMS nào. §7 và §8 đều cần SMS (mật khẩu tạm của phụ huynh, link xác nhận US04).

Đã thêm `IEduHubSmsSender` + `NullEduHubSmsSender` ghi log `[EDUHUB] sms_not_configured | <logTag>` (cố ý **không** ghi số điện thoại hay nội dung). Kênh email chạy đủ; kênh SMS **chưa gửi được**. Cắm nhà cung cấp = viết một lớp cài interface kèm `ITransientDependency`, không phải sửa chỗ nào khác.

### 8.2 Mã mục màn tự xác nhận — §17 để mở

Đã tự đặt `child_self_agreement` cho cả ảnh 11 lẫn ảnh 12/14 (khác biệt nằm ở lựa chọn có sẵn, không nằm ở việc hỏi điều gì). `DecidedAtUtc` của `pending`/`not_required` lấy mốc `AbpUsers.CreationTime` — cố định và luôn sớm hơn mọi lần con bấm về sau, đúng yêu cầu §2.

**Chưa được PO chốt.**

### 8.3 TK chờ bị vô hiệu — mâu thuẫn §17 còn nguyên

Người yêu cầu từng chốt «TK chờ dùng bình thường»; phân việc và CR-DK-01 đều yêu cầu vô hiệu. **Bản này làm theo phân việc.** Nên báo lại người yêu cầu trước bước 6 của §18.

### 8.4 Hết hạn 3 ngày (xác thực liên hệ) — chưa làm

§17 để TAK12 tự quyết nhắc / khoá / xoá. **Chưa quyết nên chưa cài.** Hạn 7 ngày của US04 thì đã cài đủ.

### 8.5 Câu chữ — chờ PO

Các khoá `EduHub*` trong `QuizSystem.xml` là bản nháp. §3 yêu cầu nội dung ảnh 7/17 phải khớp ảnh 2 bên EduHUB (chữ được phép khác). Sửa ở Admin → Message Templates cho email, hoặc file localization cho màn hình.

### 8.6 Địa chỉ EduHUB — chờ EduHUB

Chưa có `<địa chỉ EduHub>/#dang-nhap` cho từng môi trường, nên **chưa làm nút «Vào EduHUB»**.

### 8.7 Màn chọn vai trò cho TK cũ — chưa làm

§17 hỏi «có làm hay không». Chưa có câu trả lời nên chưa làm.

---

## 9. Trước khi lên môi trường

Theo thứ tự §18. Phần TAK12:

1. **Chạy `20261111_EduHubRegistrationConsent.sql` trước khi deploy.** EF đọc `AbpUsers.EduHubRole` ở mọi truy vấn `AbpUsers`; thiếu cột thì mọi truy vấn user hỏng.
2. Deploy (bước 3 của §18) — hai interface vẫn là bản mặc định, TAK12 chạy như cũ.
3. Kiểm dòng `[EDUHUB] gates` trong log. Còn thấy `roles=NullEduHub…` hoặc `ledger=NullEduHub…` là bản thật chưa đăng ký và toàn bộ luồng mới im lặng không làm gì.
4. Bật recurring job `eduhub-purge-expired-child-approvals` (mặc định chưa có lịch trong `RecurringJobSchedules`).
5. Chờ EduHUB bật hai publisher thì US03/US04/US05 mới đẩy được gia đình sang EduHUB. Trước đó mọi thứ vẫn ghi đúng ở TAK12, chỉ là chưa sang EduHUB.

---

## 10. Danh sách file

### Mới (24)

```
Core/EduHub/EduHubGates.cs                             284
Core/EduHub/EduHubChildApproval.cs                      86
Core/EduHub/EduHubAgeRules.cs                           81
Core/EduHub/EduHubConsentEntry.cs                       62
Core/EduHub/EduHubConsentCodes.cs                       56
Core/EduHub/EduHubClock.cs                              24

Application/EduHub/EduHubGuardianApprovalService.cs    423
Application/EduHub/EduHubRegistrationFlow.cs           269
Application/EduHub/EduHubFamilyAccountFactory.cs       243
Application/EduHub/Tak12EduHubConsentLedger.cs         225
Application/EduHub/EduHubSelfConsentAppService.cs      191
Application/EduHub/EduHubGuardianApprovalToken.cs      164
Application/EduHub/EduHubLearningGate.cs               135
Application/EduHub/Tak12EduHubAccountRoles.cs          106
Application/EduHub/EduHubGuardianApprovalDtos.cs        79
Application/EduHub/EduHubPurgeExpiredChildApprovalsJob.cs 54
Application/EduHub/EduHubSelfConsentDtos.cs             43

EntityFrameworkCore/Migrations/Scripts/20261111_EduHubRegistrationConsent.sql 234
Web.Mvc/Controllers/EduHubApprovalController.cs         82

EK12/ClientApp/src/views/GuardianApproval/GuardianApproval.jsx 227
EK12/ClientApp/src/views/GuardianApproval/index.js        1
EK12/ClientApp/src/features/Layout/SelfConsentDialog.jsx 148
EK12/ClientApp/src/common/utils/eduhubAge.js             44
EK12/ClientApp/src/common/utils/eduhubAge.test.js        44
```

### Sửa (19)

```
Core/QuizSystemCoreModule.cs                        +19   đăng ký fallback 5 cửa
Core/Authorization/Users/User.cs                    +11   EduHubRole
Core/Authentication/Keycloak/KeycloakService.cs     +38   TrySetUserEnabledAsync
Core/Authentication/Keycloak/IKeycloakService.cs     +3
Core/BackgroundJobs/RecurringJobScheduleKeys.cs      +3   khoá job dọn TK chờ
Core/Localization/SourceFiles/QuizSystem.xml        +46   câu chữ EduHub* (nháp)

Application/QuizSystemApplicationModule.cs          +24   dòng log [EDUHUB] gates
Application/BackgroundJobs/RecurringJobRegistrar.cs  +8   job dọn TK chờ

EntityFrameworkCore/EntityFrameworkCore/QuizSystemDbContext.cs  +23
EntityFrameworkCore/Migrations/QuizSystemDbContextModelSnapshot.cs +117  (sửa tay)

Web.Mvc/Controllers/AccountController.cs           +111   gọi flow, bỏ đăng nhập TK chờ
Web.Mvc/Models/Account/RegisterInputDto.cs          +26
Web.Mvc/Models/Account/RegisterOutputDto.cs         +15

EK12/.../RegisterForm/RegisterForm.jsx             +174   ô đồng ý, liên hệ cha mẹ, màn chờ
EK12/.../RegisterForm/registerValidation.test.js    +74
EK12/.../RegisterForm/registerValidation.js         +38
EK12/.../features/App/App.jsx                       +10   route GuardianApproval
EK12/.../features/Layout/Layout.jsx                  +8   SelfConsentDialog
EK12/ClientApp/src/data/links.js                     +3   /xac-nhan-cho-con-hoc
```

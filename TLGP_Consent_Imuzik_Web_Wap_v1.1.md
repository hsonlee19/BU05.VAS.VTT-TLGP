# TÀI LIỆU ĐẶC TẢ YÊU CẦU CHỨC NĂNG – KÊNH WEB/WAP

**Giải pháp cập nhật và quản lý Consent khách hàng trên Imuzik – áp dụng cho Website và Wapsite**

Tài liệu dẫn xuất từ `TLGP_Consent_Imuzik_v1.3`, giữ nguyên nghiệp vụ Consent và điều chỉnh theo kiến trúc Web/Wap (Yii2 server-render, đăng nhập bằng session).

| Thông tin | Nội dung |
|---|---|
| Tên yêu cầu / dự án | Cập nhật và quản lý Consent khách hàng trên Imuzik – Web/Wap |
| Phiên bản | 1.2 (Web/Wap) |
| Tài liệu gốc | TLGP_Consent_Imuzik_v1.3 (06/10/2026) |
| Ngày cập nhật | 06/10/2026 |

## 1. Lịch sử thay đổi

| Ngày | Phiên bản | Phạm vi thay đổi | Mô tả |
|---|---|---|---|
| 05/10/2026 | 1.0 | Toàn bộ | Ban hành baseline giải pháp Consent Imuzik để triển khai FE/BE/DB và tích hợp CM |
| 05/10/2026 | 1.1 | Mục 2–6 | Tách luồng riêng triển khai Web/Wap và tích hợp CM |
| 06/10/2026 | 1.2 | Mục 2–7 | Đồng bộ nghiệp vụ với App v1.3: mã lỗi, `policy_version`, Phụ lục JSON string, fail-open, `policy.note`, nút Quay lại, Menu Chính sách và dual-write `vt_member`; giữ khác biệt technical Web/Wap |

## 2. Thông tin tổng quan

### 2.1 Khác biệt technical so với TLGP App v1.3

| Hạng mục | TLGP App v1.3 | Bản Web/Wap |
|---|---|---|
| Xác thực | `token` + `authorization_code` | Session Yii (`Yii::$app->user`), CSRF cho POST |
| Kiểm tra Consent | FE gọi `GET policy/check-policy` sau login | Kiểm tra 1 lần tại `afterLogin`, lưu kết quả vào session; layout chỉ đọc session |
| Hiển thị popup | FE quyết định theo response API | Layout `main.php` đọc cờ session và render popup |
| Lấy nội dung | `GET policy/list-policy` | Server render dữ liệu từ `consent_config`/`policy`; trình duyệt parse Phụ lục JSON string theo `type` |
| Lưu Consent | `POST policy/policy` | `POST /policy/update-policy` AJAX, gửi `type`, `policy_version`, danh sách policy và CSRF |
| Popup nhóm tuổi | 3 action theo design | Giữ nguyên 3 action: `16 tuổi trở lên` / `Dưới 16 tuổi` / `Không phải bây giờ` |
| Menu Chính sách | Có, readonly | Có, readonly; lấy dữ liệu qua service/session/server-render thay vì API token |
| Rule nghiệp vụ/error code | Baseline | Giữ nguyên baseline App v1.3; chỉ khác cách technical implementation |

### 2.2 Phạm vi

**Trong phạm vi**

- Website (`frontend/`) và Wapsite (`wap/`) Imuzik.
- Mọi hình thức đăng nhập bằng số điện thoại trên Web/Wap:
  - Đăng nhập form (`LoginForm::login`);
  - Tự đăng nhập theo MSISDN nhận diện mạng 3G/4G (`AppController::beforeAction` → `MobileRecognized::getMsisdn()`).
- Kiểm tra cần Consent theo `CONSENT_CONFIG` → `log_privacy_policy` → CM.
- Popup chọn nhóm tuổi, popup Văn bản + Phụ lục + 06 điều khoản.
- Lưu Consent tại CM (`updateCustPolicy`) và `log_privacy_policy`.
- Menu **Chính sách** để khách hàng xem lại chính sách đã Consent ở chế độ readonly, cùng phạm vi nghiệp vụ với App.
- DB: tạo `consent_config`, `log_privacy_policy`; sử dụng `policy.note` hiện có và bổ sung `policy.is_editable`.
- Logic dùng chung đặt tại `common/` để giai đoạn sau App/Miniapp tái sử dụng.

**Ngoài phạm vi (giai đoạn này)**

- Code/API App/Miniapp không thuộc phạm vi triển khai của tài liệu Web/Wap này; nghiệp vụ App thực hiện theo TLGP App riêng.
- Đăng nhập Google/Facebook.
- CMS quản trị Văn bản/Phụ lục; thay đổi/rút Consent; xác minh người giám hộ.
- Đồng bộ bù CM khi SAVE CM thất bại.
- Build lại bundle `js/build/app.min.js` (JS mới viết inline trong view).

### 2.3 Thành phần thay đổi

| Thành phần | Xử lý |
|---|---|
| `common/libs/ConsentService.php` (mới) | Toàn bộ rule nghiệp vụ: lấy config current, check local log, gọi CM, đối chiếu policy bắt buộc, build `custPolicyDTO`, ghi log |
| `common/libs/CmConsentClient.php` (mới) | SOAP client `getCustPolicy`/`updateCustPolicy` dựa trên `WsSoapClient`; timeout + tối đa 3 lần gọi |
| `common/models/v2/ConsentConfigBase.php`, `common/models/v2/LogPrivacyPolicyBase.php` (+ `db/*DB.php`) (mới) | ActiveRecord cho 2 bảng mới |
| `common/models/v2/db/PolicyDB.php` | Bổ sung thuộc tính `name`, `is_editable`, `note` |
| `common/config/params.php` | Cấu hình `consentCm` (wsdl, timeout, maxCalls) |
| `frontend/config/main.php`, `wap/config/main.php` | Gắn handler `on afterLogin` cho component `user` |
| `frontend/controllers/SiteController.php`, `wap/controllers/SiteController.php` | `actionLogout` hỗ trợ cờ `consent_skip_autologin` |
| `frontend/controllers/AppController.php`, `wap/controllers/AppController.php` | Bỏ qua tự đăng nhập MSISDN khi có cờ `consent_skip_autologin` |
| `frontend/controllers/PolicyController.php`, `wap/controllers/PolicyController.php` | Sửa `actionUpdatePolicy` theo luồng mới (nhận `type`, `policy_version`, danh sách policy trong `formData`, gọi `ConsentService::saveConsent`) |
| `frontend/views/layouts/_confirm-policy.php`, `wap/views/layouts/_confirm-policy.php` | Thay điều kiện hiển thị; thêm popup chọn nhóm tuổi; popup văn bản render từ `consent_config` + `policy`; parse Phụ lục JSON string; hiển thị `policy.note`; có nút Quay lại; JS inline |
| `frontend/web/js/coder.js`, `wap/web/js/coder.js` | Giữ/reuse logic hiện có khi phù hợp; bổ sung phần parse Phụ lục JSON, `policy_version`, Quay lại theo cấu trúc Web/Wap |
| Menu **Chính sách** (Web/Wap) | Bổ sung điểm truy cập readonly; dữ liệu nghiệp vụ tương đương App UC02, cách route/view/service do Dev bố trí theo code hiện tại |
| `vt_member.is_update_policy`, `vt_member.policy_id` | Không dùng làm source of truth; dual-write sau khi lưu Consent local thành công để tương thích luồng cũ |

### 2.4 Nguyên tắc giải pháp (kế thừa TLGP App v1.3)

- `policy.id` là khóa kỹ thuật; `policy.name` là key mapping CM; `sortorder` xác định thứ tự Điều khoản 1–6.
- `is_required` xác định điều khoản bắt buộc; `is_editable` chỉ xác định trạng thái tick mặc định.
- `policy.note` là ghi chú độc lập của điều khoản; có dữ liệu thì hiển thị dưới `description`, không tham gia validation và không suy diễn từ `is_editable`.
- `consent_config` có 01 Văn bản HTML dùng chung + 02 Phụ lục **JSON string** theo `type` cho mỗi version; trình duyệt parse chuỗi JSON để render.
- `policy_version` khách hàng đã xem phải được gửi lại khi lưu Consent và validate với current version.
- Local log cùng `current_policy_version` → coi như đã hoàn tất Consent, không đối chiếu `confirm_ids`, không gọi CM.
- Lỗi đọc `consent_config` hoặc thiếu Văn bản/Phụ lục/danh sách policy do lỗi cấu hình dữ liệu → fail-open trong lần đăng nhập hiện tại; không ghi local Consent, lần đăng nhập sau kiểm tra lại.
- Sau khi ghi `log_privacy_policy` thành công, dual-write `vt_member.is_update_policy=1`, `vt_member.policy_id=<confirm_ids>` để tương thích; không dùng hai field này làm source of truth.
- Không dùng `consent`, `displayConsent`, `systemType` của CM.

## 3. Yêu cầu chức năng

### 3.1 UC01 – Kiểm tra và xác nhận Consent khi đăng nhập Web/Wap

| Thông tin | Nội dung |
|---|---|
| Actor | Khách hàng đăng nhập Web/Wap bằng số điện thoại (form hoặc tự nhận diện MSISDN) |
| Trigger | Sự kiện `afterLogin` của component `user` |
| Điều kiện đầu vào | Xác định được MSISDN của user (`vt_member.username`); đọc được `consent_config` current/active |
| Đầu ra thành công | User đáp ứng rule Consent hiện hành và dùng tiếp dịch vụ |
| Đầu ra không thành công | User chọn "Không phải bây giờ" → đăng xuất; lỗi request/auth/submit không thuộc fail-open → giữ popup/hiển thị lỗi; lỗi cấu hình/nội dung Consent thuộc fail-open → bỏ qua Consent trong lần đăng nhập hiện tại |

### 3.2 Sơ đồ luồng

```mermaid
sequenceDiagram
    autonumber
    actor KH as Khách hàng
    participant FE as Trình duyệt (Web/Wap)
    participant BE as Imuzik BE (Yii)
    participant SS as Session (Redis)
    participant DB as Imuzik DB
    participant CM as CM Consent API

    alt Đăng nhập form
        KH->>BE: POST /dang-nhap (LoginForm::login)
    else Tự đăng nhập MSISDN (3G/4G)
        FE->>BE: Request bất kỳ (guest, không có cờ consent_skip_autologin)
        BE->>BE: MobileRecognized::getMsisdn() → user->login()
    end
    BE->>BE: Sự kiện afterLogin → ConsentService::checkNeedConsent()
    BE->>DB: consent_config current/active
    BE->>DB: log_privacy_policy theo user_id
    alt Có log cùng current_policy_version
        BE->>SS: consent_check = {user_id, need_update:false, type, policy_ids}
    else Chưa có log / khác version
        BE->>CM: getCustPolicy(isdn) – tối đa 3 lần khi lỗi kỹ thuật
        alt Lỗi kỹ thuật sau 3 lần
            BE->>SS: need_update=false (fail-open lần này; không ghi local Consent)
        else code != 0 hoặc DTO rỗng
            BE->>SS: need_update=true
        else code = 0 và DTO có dữ liệu
            BE->>DB: policy active, is_required=1
            alt Tất cả field bắt buộc = 1
                BE->>SS: need_update=false
            else Thiếu
                BE->>SS: need_update=true
            end
        end
    end

    FE->>BE: Tải trang
    BE->>SS: Đọc consent_check
    alt user_id khớp và need_update=true
        BE-->>FE: Layout render Popup chọn nhóm tuổi (MH01)
        alt Chọn "16 tuổi trở lên" / "Dưới 16 tuổi"
            FE-->>KH: Popup văn bản (MH02) – dữ liệu đã render sẵn, JS parse Phụ lục JSON theo type
            KH->>FE: Tick điều khoản + Xác nhận chung → ĐỒNG Ý
            FE->>BE: POST /policy/update-policy (type, policy_version, policy_id, _csrf)
            BE->>DB: Validate policy_version + policy active + is_required
            alt Thiếu policy bắt buộc / sai định dạng
                BE-->>FE: Lỗi → giữ popup
            else Hợp lệ
                BE->>CM: updateCustPolicy(isdn, custPolicyDTO) – tối đa 3 lần khi lỗi kỹ thuật
                BE->>DB: INSERT/UPDATE log_privacy_policy (mọi kết quả CM)
                BE->>DB: Dual-write vt_member.is_update_policy/policy_id
                BE->>SS: need_update=false
                BE-->>FE: errorCode=000000 → reload trang
            end
        else Chọn "Không phải bây giờ"
            FE->>BE: GET /logout?consent_skip=1
            BE->>SS: logout (hủy session) + giữ cờ consent_skip_autologin
            BE-->>FE: Redirect màn hình đăng nhập
        end
    else Không có cờ / need_update=false
        BE-->>FE: Không hiển thị popup Consent (popup khảo sát yêu thích nếu có)
    end
```

### 3.3 Mô tả luồng xử lý

#### Bước 1 – Kích hoạt kiểm tra tại `afterLogin`

- Gắn handler cho sự kiện `yii\web\User::EVENT_AFTER_LOGIN` trong `frontend/config/main.php` và `wap/config/main.php`.
- Handler gọi `ConsentService::checkNeedConsent($member)` và ghi session:

```php
Yii::$app->session->set('consent_check', [
    'user_id'     => $member->id,
    'need_update' => true | false,
    'type'        => <0|1|null>,
    'policy_ids'  => <array>,
]);
```

- Dùng `afterLogin` thay vì `LoginForm`: dùng cả đăng nhập và tự động đăng nhập.
- Lưu kèm `user_id`: tránh dùng nhầm kết quả của số điện thoại khác nhau trong cùng session.
- `type`, `policy_ids` được lưu khi xác định được để phục vụ Menu Chính sách; semantics giữ như App, không tự suy diễn `type` khi không có dữ liệu local.

#### Bước 2 – `ConsentService::checkNeedConsent` (giữ nguyên rule nghiệp vụ TLGP App v1.3)

1. Lấy `consent_config` có `is_current=1` và `status='active'` → `current_policy_version`.
   - Lỗi đọc DB / không có bản ghi → ghi log lỗi, `need_update=false` theo fail-open; không tạo/cập nhật `log_privacy_policy`; lần đăng nhập sau kiểm tra lại.
2. Tra `log_privacy_policy` theo `user_id`:
   - Có bản ghi và `policy_version = current_policy_version` → `need_update=false`, đồng thời lấy `type`, `confirm_ids` để lưu session phục vụ Menu Chính sách.
   - Không có / khác version → Bước 3.
3. Gọi CM `getCustPolicy(isdn)`:
   - `code=0` + `custPolicyDTO` có dữ liệu → so sánh từng `policy.name` (active, `is_required=1`) với field cùng tên; tất cả `=1` → `false`, ngược lại → `true`.
   - `code=0` + DTO rỗng/null → `true`.
   - `code != 0` → `true` (không retry).
   - Timeout/lỗi kết nối/lỗi server → gọi lại, **tổng tối đa 3 lần**; vẫn lỗi → `false`.

#### Bước 3 – Layout hiển thị popup

`_confirm-policy.php` (render trong `layouts/main.php`) chỉ hiển thị Popup chọn nhóm tuổi khi đồng thời:

- User đã đăng nhập;
- `session['consent_check']` tồn tại, `user_id` = user hiện tại, `need_update = true`.

Các trường hợp khác:

| Trường hợp | Xử lý |
|---|---|
| Không có cờ (session đăng nhập trước thời điểm deploy) | Không kiểm tra, không hiển thị popup; sẽ kiểm tra ở lần đăng nhập kế tiếp |
| Có cờ nhưng `user_id` khác user hiện tại | Kiểm tra lại 1 lần (Bước 2) và ghi đè cờ |
| `need_update=false` | Không hiển thị popup Consent; nếu `session['showSurvey']` thì hiển thị popup khảo sát yêu thích như hiện tại |

Bỏ hoàn toàn điều kiện cũ `created_at >= 2023-07-01 && is_update_policy == 0`.

#### Bước 4 – Popup chọn nhóm tuổi (MH01)

<p align="center">
  <img src="D:\Công việc\BU05.VTT.VAS\Imuzik\TLGP\images\Frame 313322.png" height="400"><br>
  <b>MH01 – Popup xác minh người dùng (chọn nhóm tuổi)</b>
</p>

| Nút | Hành động |
|---|---|
| **16 tuổi trở lên** | `type=0` → Bước 5 |
| **Dưới 16 tuổi** | `type=1` → Bước 5 |
| **Không phải bây giờ** | Gọi `/logout` kèm cờ `consent_skip_autologin` → Bước 9 |

- Nội dung text theo design, thay "Myclip" bằng "Imuzik".
- Popup dạng `backdrop: static`, không đóng bằng click ngoài/ESC.

#### Bước 5 – Lấy nội dung (server render, không dùng AJAX)

Web/Wap không gọi `GET policy/list-policy`. Khi cần hiển thị Consent, server lấy dữ liệu từ `consent_config` + `policy` và render vào page; trình duyệt xử lý phần hiển thị theo `type`.

**Xử lý BE** (`ConsentService::getContent()`, gọi trong `_confirm-policy.php` khi cần hiển thị popup)

1. Lấy `consent_config` current/active và `policy_version`.
   - Lỗi đọc/không có config → ghi log và fail-open: không hiển thị Consent lần này, không ghi local Consent.
2. `document_content` rỗng → `130005`; xử lý fail-open.
3. `appendix_type_0_content`, `appendix_type_1_content` là **JSON string**. Sau khi khách hàng chọn `type`, nếu Phụ lục tương ứng bị thiếu → `130006` và xử lý fail-open; không dùng trạng thái Phụ lục của `type` còn lại để quyết định luồng hiện tại.
4. Lấy policy active theo `sortorder`, bao gồm `description`, `note`, `is_required`, `is_editable`; không có policy → `130007`; xử lý fail-open.
5. Render `policy_version` vào input ẩn để gửi lại khi SAVE.
6. Render dữ liệu Văn bản, 02 chuỗi Phụ lục và policy vào page. Khi khách hàng chọn `type`, JS chọn chuỗi Phụ lục tương ứng, parse JSON (`title`, `warning`, `parent_content`, `confirm_content`) và render.
7. Nếu fail-open tại bước này: không tạo/cập nhật `log_privacy_policy`, không coi khách hàng đã Consent; session chỉ bỏ yêu cầu popup trong lần đăng nhập hiện tại.

`policy.note` nếu có được hiển thị dưới `description`; `note` không tham gia validation.

#### Bước 6 – Popup Văn bản Consent (MH02)

<p align="center">
  <img src="D:\Công việc\BU05.VTT.VAS\Imuzik\TLGP\images\Frame 313325.png" height="400"><br>
  <b>MH02 – Popup Văn bản Consent</b>
</p>

- Render `document_content`; JS parse Phụ lục JSON string theo `type` và render các field `title`, `warning`, `parent_content`, `confirm_content` theo design.
- Render danh sách điều khoản theo thứ tự trả về, giữ cấu trúc DOM hiện có (`#form-confirm`, `.box-check`, `.container-checkbox.required-box|optional-box`, `#stt-check{n}`, `#footer-all-confirm`, `#confirm-all-policy`, `#btn-confirm`) để **tái sử dụng nguyên logic JS hiện có trong `coder.js`**:
  - Tick "Tôi xác nhận đồng ý…" (Xác nhận chung) → tự tick toàn bộ điều khoản;
  - Nút **ĐỒNG Ý** chỉ active khi toàn bộ điều khoản `is_required=1` đã tick.
- Hiển thị `note` (nếu có) dưới `description`.
- Trạng thái tick mặc định lấy theo `is_editable`; mapping kỹ thuật 0/1 và cách đồng bộ bộ đếm do Dev xử lý theo code/data hiện tại.
- Có nút **Quay lại** Popup chọn nhóm tuổi; khi quay lại chưa lưu Consent và khách hàng có thể chọn lại `type`.

#### Bước 7 – AJAX lưu Consent: `POST /policy/update-policy`

**Request** (`application/x-www-form-urlencoded`, giữ nguyên contract `acceptConfirm()` hiện có trong `coder.js`)

| Field | Bắt buộc | Mô tả |
|---|---:|---|
| `_csrf` | Có | CSRF token Yii |
| `formData` | Có | JSON `serializeArray()` của `#form-confirm`: `id_confirm[]` (= `policy.id` đã tick), `type`, `policy_version` (input ẩn lấy từ config đã render) |

**Response**: `{"status": 1|0, "errorCode": "...", "message": "..."}` – `coder.js` dùng `status` (1 → chuyển về trang chủ; 0 → `alert(message)`).

**Xử lý BE** (`ConsentService::saveConsent(...)`) – giữ outcome nghiệp vụ theo TLGP App v1.3:

1. Chưa đăng nhập → `000002`.
2. `type ∉ {0,1}` → `130004`.
3. `policy_id` sai định dạng → `130002`; có id không tồn tại/không active → `130003`.
4. Lấy `current_policy_version`; so sánh `policy_version` từ form với current:
   - khác version → `130009`, không gọi CM; FE reload/render lại nội dung hiện hành.
5. Thiếu policy bắt buộc → `130008` (không gọi CM).
6. Build `custPolicyDTO` 06 field theo `policy.name` (`1` nếu id có trong `policy_id`, ngược lại `0`) → `updateCustPolicy(isdn, dto)`:
   - `code=0` → tiếp tục;
   - `code!=0` → ghi log `code/description`, không retry, tiếp tục;
   - lỗi kỹ thuật → tổng tối đa 3 lần, vẫn lỗi → ghi log, tiếp tục.
7. INSERT/UPDATE `log_privacy_policy` (`user_id`, `isdn`, `confirm_ids`, `is_consent=1`, `type`, `policy_version`) → lỗi DB → `000001`.
8. Sau khi ghi local thành công, dual-write `vt_member.is_update_policy=1`, `vt_member.policy_id=<confirm_ids>` để tương thích luồng cũ; không dùng làm source of truth.
9. Cập nhật `session['consent_check']` với `need_update=false`, `type`, `policy_ids`.

**Response**: `{"errorCode":"000000","message":"Successful","data":null}` → FE reload trang hiện tại.

#### Bước 8 – Xử lý lỗi tại FE

| `errorCode` | Message | Xử lý FE |
|---|---|---|
| `000001` | Hệ thống đang bận, vui lòng thử lại sau. | Nếu lỗi xảy ra khi tải cấu hình/nội dung Consent → fail-open lần đăng nhập hiện tại; nếu lỗi khi ghi `log_privacy_policy` → giữ popup, hiển thị lỗi, không hoàn tất Consent |
| `000002` | Require login. / Vui lòng đăng nhập để thực hiện chức năng này | Reload/điều hướng đăng nhập theo cơ chế hiện tại |
| `130002` | Sai định dạng tham số truyền vào! | Giữ popup, hiển thị lỗi |
| `130003` | Nhập id không tồn tại! | Giữ popup, hiển thị lỗi |
| `130004` | Nhóm tuổi không hợp lệ. | Quay lại Popup chọn nhóm tuổi |
| `130005` | Không tìm thấy Văn bản Consent hiện hành. | Ghi log và fail-open lần đăng nhập hiện tại; không ghi local Consent |
| `130006` | Không tìm thấy Phụ lục Consent phù hợp. | Ghi log và fail-open lần đăng nhập hiện tại; không ghi local Consent |
| `130007` | Không tìm thấy danh sách điều khoản Consent. | Ghi log và fail-open lần đăng nhập hiện tại; không ghi local Consent |
| `130008` | Vui lòng xác nhận đầy đủ các điều khoản bắt buộc. | Giữ popup, hiển thị lỗi |
| `130009` | Văn bản Consent đã được cập nhật. Vui lòng tải lại nội dung. | Reload/render lại nội dung Consent current trước khi cho phép xác nhận |
| Lỗi mạng/HTTP | — | Xử lý theo cơ chế lỗi hiện tại; nếu là lỗi CM check thuộc rule fallback thì không hiện popup lần này |

Mã lỗi CM không trả về FE.

#### Bước 9 – "Không phải bây giờ" và tự đăng nhập MSISDN

Vấn đề: user dùng 3G/4G bị đăng xuất sẽ được `AppController` tự đăng nhập lại ở request kế tiếp → popup hiện lại liên tục.

Xử lý:

1. `actionLogout` nhận tham số `consent_skip=1` → sau `Yii::$app->user->logout()` (hủy session) ghi lại `session['consent_skip_autologin'] = true` (cùng cơ chế giữ `recent_keywords` hiện có).
2. `AppController::beforeAction`: bỏ qua nhánh tự đăng nhập MSISDN nếu có `consent_skip_autologin`.
3. Sau logout, redirect về màn hình đăng nhập. Khi user chủ động đăng nhập lại bằng form → xóa cờ `consent_skip_autologin` → luồng Consent chạy lại tại `afterLogin`.
4. Cờ hết hiệu lực khi session hết hạn.

### 3.4 UC02 – Xem lại chính sách đã Consent

Giữ nguyên nghiệp vụ Menu **Chính sách** theo TLGP App v1.3; Web/Wap chỉ khác cách lấy/render dữ liệu.

| Nội dung | Xử lý Web/Wap |
|---|---|
| Điểm truy cập | Menu **Chính sách** sau đăng nhập |
| Nguồn `type`, `policy_ids` | Dữ liệu tương đương kết quả check Consent, lưu trong `session['consent_check']` khi xác định được; có thể lấy lại qua `ConsentService` theo code hiện tại |
| Lấy Văn bản/Phụ lục | Server lấy `consent_config`/policy; nếu có `type`, trình duyệt parse Phụ lục JSON string tương ứng |
| Hiển thị policy đã Consent | Đánh dấu các policy có ID thuộc `policy_ids` ở trạng thái readonly |
| Quyền thao tác | Chỉ xem; không cho thay đổi/rút Consent |
| Trường hợp `type=null` | Giữ nguyên rule App: không tự suy diễn nhóm tuổi; hiển thị thông tin policy/Văn bản chung theo dữ liệu hiện có, không hiển thị Phụ lục theo nhóm tuổi |

Không bổ sung nghiệp vụ mới cho Menu Chính sách ngoài phạm vi đã có ở App v1.3.

## 4. Business Rules (Web/Wap)

| Rule | Mô tả |
|---|---|
| BR-W01 | Kiểm tra Consent đúng 1 lần cho mỗi lần đăng nhập (form hoặc MSISDN), tại `afterLogin`; không gọi CM theo từng lần tải trang |
| BR-W02 | Cờ session `consent_check` luôn gắn `user_id`; không dùng cờ của user khác |
| BR-W03 | Session tồn tại trước thời điểm deploy (chưa có cờ) không được kiểm tra; kiểm tra ở lần đăng nhập kế tiếp |
| BR-W04 | Popup chọn nhóm tuổi gồm 3 nút; "Không phải bây giờ" → đăng xuất và redirect màn hình đăng nhập |
| BR-W05 | Sau "Không phải bây giờ", tạm dừng tự đăng nhập MSISDN trong phiên để tránh vòng lặp |
| BR-W06 | Logic JS checkbox hiện có (`coder.js`) được giữ nguyên; không build lại `app.min.js` trong phạm vi này |
| BR-W07 | Dùng cùng danh mục mã lỗi App: `130002` sai định dạng, `130003` ID không tồn tại, `130004` nhóm tuổi, `130005-130007` lỗi dữ liệu nội dung, `130008` thiếu policy bắt buộc, `130009` lệch version |
| BR-W08 | CM: mỗi operation tối đa 3 lần gọi (tính cả lần đầu), timeout 5s/lần, chỉ gọi lại khi lỗi kỹ thuật; không có cờ bật/tắt tích hợp |
| BR-W09 | Các rule nghiệp vụ của TLGP App v1.3 giữ nguyên hiệu lực; Web/Wap chỉ khác cơ chế session/server-render/AJAX |
| BR-W10 | Lỗi đọc `consent_config` hoặc thiếu Văn bản/Phụ lục/danh sách policy → fail-open trong lần đăng nhập hiện tại; không ghi local Consent, lần sau kiểm tra lại |
| BR-W11 | `policy_version` render cho khách hàng phải được gửi lại khi SAVE và validate với current; khác version → `130009` |
| BR-W12 | `policy.note` hiển thị dưới `description` khi có dữ liệu, không tham gia validation và không suy diễn từ `is_editable` |
| BR-W13 | MH02 có nút **Quay lại** MH01 để chọn lại nhóm tuổi; chưa lưu Consent |
| BR-W14 | Sau khi ghi local thành công, dual-write `vt_member.is_update_policy=1`, `vt_member.policy_id=<confirm_ids>` để tương thích; không dùng làm source of truth |
| BR-W15 | Menu **Chính sách** có trên Web/Wap, readonly và giữ nguyên nghiệp vụ App |

## 5. Tích hợp và dữ liệu

### 5.1 Tích hợp CM (theo PYC-64913 mục 1.4.1, 1.4.2)

| Hạng mục | Giá trị |
|---|---|
| WSDL | `http://10.58.71.238:8701/SALE_SERVICE/bpm/sale/externalSystem/InterfaceSaleMyclip?wsdl` |
| `getCustPolicy` | Chỉ truyền `isdn`; dùng `code`, `custPolicyDTO.{6 field}` |
| `updateCustPolicy` | `isdn` + `custPolicyDTO` 06 field (`"1"`/`"0"`) |
| Client | `WsSoapClient` (`common/libs/WsSoapClient.php`) |
| Cấu hình | `params['consentCm'] = ['wsdl' => …, 'timeout' => 5, 'maxCalls' => 3]` |
| Log | Ghi log request/response (che MSISDN theo quy định) và mã lỗi CM vào category riêng `consent` |

Lưu ý từ PYC-64913: `updateCustPolicy` luôn lưu 2 điều khoản đầu = 1; `getCustPolicy` chỉ trả `custPolicyDTO` khi KH đã consent ≥ X điều khoản (X mặc định 2).

### 5.2 Bảng `policy` (DB `imuzik`) – reuse

Hiện trạng: id 1–6 `is_active=0` (bộ cũ, giữ nguyên); id 7–12 `is_active=1`, `sortorder` 1–6, `is_required` = 1,1,0,0,0,0; có cột `note`; chưa có `is_editable`. `note` được dùng để lưu ghi chú hiển thị độc lập dưới `description`, không tham gia validation.

```sql
ALTER TABLE policy ADD COLUMN is_editable TINYINT(1) NOT NULL DEFAULT 0;

UPDATE policy SET name = 'provideProduct'       WHERE id = 7;
UPDATE policy SET name = 'supportCustomer'      WHERE id = 8;
UPDATE policy SET name = 'improveQuality'       WHERE id = 9;
UPDATE policy SET name = 'marketingAdvertising' WHERE id = 10;
UPDATE policy SET name = 'researchMarket'       WHERE id = 11;
UPDATE policy SET name = 'tradePromotion'       WHERE id = 12;

-- Giá trị `is_editable` thực tế do Dev xác nhận theo code/data hiện tại;
-- BA chỉ khóa semantics: field này xác định trạng thái tick mặc định.
```

### 5.3 Bảng `consent_config` (DB `imuzik`)

```sql
CREATE TABLE consent_config (
    id                      INT AUTO_INCREMENT PRIMARY KEY,
    policy_version          VARCHAR(20)  NOT NULL,
    document_content        LONGTEXT     NOT NULL,
    appendix_type_0_content LONGTEXT     NOT NULL,
    appendix_type_1_content LONGTEXT     NOT NULL,
    is_current              TINYINT(1)   NOT NULL DEFAULT 0,
    status                  VARCHAR(20)  NOT NULL DEFAULT 'active',
    created_at              DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at              DATETIME     NULL ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_policy_version (policy_version),
    KEY idx_current (is_current, status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

Dữ liệu khởi tạo version `1.0`:

`policy_version` được xác lập khi INSERT version mới; UPDATE bản ghi hiện tại không tự tăng `policy_version`.

- `document_content`: Điều 1–12 của "Văn bản chấp thuận về xử lý và bảo vệ dữ liệu cá nhân tại VTT", HTML dùng class hiện có (`legal-doc`, `article-title`, `legal-num`, `legal-sub-num`, `legal-bullet`).
- `appendix_type_0_content`, `appendix_type_1_content`: lưu **JSON string** theo cấu trúc `title`, `warning`, `parent_content`, `confirm_content`; không lưu Phụ lục thành HTML riêng.
- Bảng 06 điều khoản lấy từ `policy`; `policy.note` hiển thị dưới `description` khi có dữ liệu.
- Trình duyệt chọn chuỗi Phụ lục theo `type`, parse JSON và render theo design.

Ví dụ cấu trúc chuỗi Phụ lục:

```json
{
  "title": "Văn bản chấp thuận về việc xử lý và bảo vệ dữ liệu cá nhân",
  "warning": "",
  "parent_content": "Tôi xác nhận đã đọc, hiểu và đồng ý đối với toàn bộ nội dung và các mục đích xử lý dữ liệu theo quy định tại Văn bản chấp thuận về xử lý và bảo vệ dữ liệu cá nhân.",
  "confirm_content": ""
}
```

### 5.4 Bảng `log_privacy_policy` (DB `imuzik`)

```sql
CREATE TABLE log_privacy_policy (
    id             BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id        BIGINT       NOT NULL,
    isdn           VARCHAR(20)  NOT NULL,
    confirm_ids    VARCHAR(255) NOT NULL,
    is_consent     TINYINT(1)   NOT NULL DEFAULT 1,
    type           TINYINT(1)   NOT NULL,
    policy_version VARCHAR(20)  NOT NULL,
    created_at     DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at     DATETIME     NULL ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_user (user_id),
    KEY idx_isdn (isdn)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

Mỗi user 1 bản ghi hiện hành (INSERT nếu chưa có, UPDATE nếu đã có).

### 5.5 Danh mục mã lỗi Consent Web/Wap

Web/Wap dùng cùng semantic mã lỗi với App:

| `errorCode` | `message` | Phạm vi |
|---|---|---|
| `130002` | `Sai định dạng tham số truyền vào!` | `policy_id` sai định dạng |
| `130003` | `Nhập id không tồn tại!` | `policy_id` không tồn tại/không active |
| `130004` | `Nhóm tuổi không hợp lệ.` | `type` thiếu/ngoài `{0,1}` |
| `130005` | `Không tìm thấy Văn bản Consent hiện hành.` | Thiếu `document_content`; fail-open khi tải nội dung |
| `130006` | `Không tìm thấy Phụ lục Consent phù hợp.` | Thiếu Phụ lục theo `type`; fail-open khi tải nội dung |
| `130007` | `Không tìm thấy danh sách điều khoản Consent.` | Không có policy active; fail-open khi tải nội dung |
| `130008` | `Vui lòng xác nhận đầy đủ các điều khoản bắt buộc.` | Thiếu policy `is_required=1` khi submit |
| `130009` | `Văn bản Consent đã được cập nhật. Vui lòng tải lại nội dung.` | `policy_version` submit khác current version |

## 6. Ràng buộc triển khai

| STT | Ràng buộc |
|---:|---|
| 1 | Chỉ áp dụng triển khai Web/Wap; code App/Miniapp không thuộc phạm vi tài liệu này và thực hiện theo TLGP App riêng. |
| 2 | Server Web/Wap (test, prod) phải mở kết nối tới `10.58.71.238:8701` |
| 3 | Script DB (Mục 5) chạy trước khi deploy code |
| 4 | Không build lại `app.min.js`; JS mới viết inline trong `_confirm-policy.php` |
| 5 | `document_content` lưu HTML; 02 Phụ lục lưu JSON string và được parse/render theo `type`. |
| 6 | Menu **Chính sách** thuộc phạm vi và chỉ cho xem; không triển khai thay đổi/rút Consent, CMS, xác minh người giám hộ hoặc đồng bộ bù CM. |
| 7 | `policy_version` phải được gửi lại khi SAVE và validate với current version. |
| 8 | Lỗi cấu hình/nội dung Consent thuộc fail-open: bỏ qua Consent lần đăng nhập hiện tại, không ghi local, lần sau kiểm tra lại. |
| 9 | Sau khi lưu local thành công, dual-write `vt_member.is_update_policy/policy_id` chỉ để tương thích; `log_privacy_policy` vẫn là source of truth. |

## 7. Các điểm chốt và technical cần Dev xác nhận

| # | Nội dung | Đề xuất |
|---:|---|---|
| 1 | Giá trị kỹ thuật `is_editable` từng điều khoản | Dev xác nhận theo code/data hiện tại; BA chỉ khóa semantics trạng thái tick mặc định |
| 2 | Vị trí script SQL | Commit `docs/sql/PYC-84821_consent.sql` hoặc gửi DBA ngoài repo |
| 3 | Nội dung `consent_config` | `document_content` là HTML; 02 Phụ lục là JSON string theo Mục 5.3; version xác lập khi INSERT version mới |
| 4 | Tương thích luồng cũ | **Đã chốt:** sau khi Web/Wap lưu local thành công, dual-write `vt_member.is_update_policy=1`, `policy_id=<confirm_ids>`; không dùng làm source of truth |
| 5 | Nút "Quay lại" trên popup văn bản | **Đã chốt:** có, quay về Popup chọn nhóm tuổi; chưa lưu Consent |
| 6 | Lỗi cấu hình/nội dung Consent | **Đã chốt fail-open:** `need_update=false` trong lần đăng nhập hiện tại + ghi log; không ghi `log_privacy_policy`; lần đăng nhập sau kiểm tra lại |
| 7 | `policy.note` | **Đã chốt:** lưu/đọc từ `policy.note`, hiển thị dưới `description`, không tham gia validation |
| 8 | `policy_version` khi SAVE | **Đã chốt:** gửi lại version đã render; khác current → `130009` |
| 9 | Menu Chính sách | **Giữ nguyên nghiệp vụ App:** có trên Web/Wap, readonly; chưa bổ sung yêu cầu mới ngoài baseline |
| 10 | Thông tin CM cần xác nhận | URL prod, xác thực, định dạng `isdn` (84…/0…), dùng chung interface `InterfaceSaleMyclip` |
| 11 | `session.gc_maxlifetime` thực tế trên server | Hỏi vận hành |
| 12 | Phạm vi Miniapp | Ngoài phạm vi giai đoạn Web/Wap |

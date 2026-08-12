# API Response Contract

> Chuẩn hình dạng response cho mọi API trong pack: thành công trả gì, thất bại trả gì, phân trang ra sao, mã lỗi đặt tên thế nào.
> Phần normative — chỉ chứa luật. Edge case, nơi đặt code theo stack, **api client mẫu cho FE (appendix section 5)**, và đánh đổi thiết kế nằm ở [`API_RESPONSE_CONTRACT_APPENDIX.md`](API_RESPONSE_CONTRACT_APPENDIX.md).

## 1. Nguyên tắc gốc

**HTTP đã có sẵn kênh phân biệt thành công/thất bại là status code. Không dựng kênh thứ hai trong body.**

Envelope chỉ tồn tại ở đúng 2 chỗ mà hình dạng bắt buộc phải máy đọc được: **lỗi** và **list phân trang**. Mọi chỗ khác trả thẳng resource.

Hệ quả: vì success không có envelope nên `204`, file download, streaming, redirect, webhook, `/healthz` **không phải là ngoại lệ của chuẩn** — chúng chỉ là response HTTP bình thường. Chuẩn này vì thế không có danh sách ngoại lệ dài để người đọc bỏ qua.

## 2. Phạm vi áp dụng

Chuẩn này nói cách **sinh** response. Gần như mọi dự án còn **gọi** API của người khác (partner, Zalo, Graph, cổng thanh toán, sidecar) — không cái nào tuân `api-1`, và luật cho vai consumer nằm ở [appendix section 2.5](API_RESPONSE_CONTRACT_APPENDIX.md).

| Nhóm | Áp dụng | Ghi chú |
|---|---|---|
| `web/backend-only/*`, `web/fullstack/*` | Toàn bộ | Đây là nhóm sinh HTTP API |
| `web/frontend-only/*` | Vai consumer | Api client + type mirror. Nuxt: `server/api/**` là producer thật, phải tuân toàn bộ chuẩn; trang SSR cho browser thì không — xem appendix section 2.6 |
| `service/*` | Health surface + vai consumer | `/healthz` + `/readyz` bắt buộc **khi service có HTTP surface** (daemon chạy dài). Service one-shot/cron không dựng HTTP server thì miễn — nhưng vẫn dùng `error.code` + `retryable` làm căn cứ retry/backoff khi gọi API ngoài |
| `desktop/wails-go-vue` | Phần **error** qua bridge | Xem appendix section 2 |
| `desktop/dotnet-wpf`, `dotnet-winform` | Vai consumer | |
| `browser-extension/js` | Vai consumer + messaging error | |

## 3. Tier bắt buộc

Chia 3 tầng theo **điều kiện của dự án**, không theo mode thư mục. Mode `simple`/`structured` nói về khối lượng code, không nói về độ tinh vi của API — một MVP `simple` phục vụ app mobile có nghĩa vụ nặng hơn một hệ thống `structured` chỉ có admin web nội bộ.

### Tier 0 — bắt buộc mọi repo có HTTP surface, kể cả MVP một người

1. **Status code nói đúng sự thật.** Không bao giờ `200` cho một request thất bại.
2. **Mọi response status ≥ 400 dưới `/api/v1` đều là** `{"error": {...}}` với `Content-Type: application/json` — kể cả 404 route không tồn tại, 405, và 500 do panic. Trang lỗi mặc định của framework phải bị override. Miễn trừ là **luật vị trí**, không phải danh sách ngoại lệ: mọi thứ được miễn đều nằm NGOÀI `/api/v1` (luật 8) — `/readyz` (section 1.6 của [appendix](API_RESPONSE_CONTRACT_APPENDIX.md)), webhook receiver (1.8), endpoint browser điều hướng trả HTML/302 (1.3, 1.7, 1.9), và endpoint implement spec ngoài như RFC 6749 của OAuth. Mỗi chỗ miễn trừ phải có comment `// contract-exempt:` và một dòng trong `docs/api-contract.md`.
   **Tầng trước app cũng phải tuân luật này.** nginx/IIS/Caddy tự sinh HTML cho 413, 502, 504 — request không bao giờ tới handler, nên writer trong app vô dụng. Reverse proxy phải override chúng thành JSON envelope và tự sinh `X-Request-Id` khi request chưa có.
3. **Bắt buộc 4 field trong `error`:** `code`, `message`, `trace_id`, `retryable`.
4. **`trace_id`** bằng đúng request id trong log, mirror ra header `X-Request-Id` trên **mọi** response kể cả success.
5. **List endpoint luôn trả** `{"items": [...], "page": {...}}` — không bare array, không nhét pagination vào header.
6. **Wire conventions:** JSON key `snake_case`, timestamp RFC3339 UTC kết thúc `Z`, mọi `id` là string.
7. **500 không bao giờ lộ** stack trace, câu SQL, path nội bộ, connection string. `message` của 500 là chuỗi cố định.
8. **Mọi endpoint FE/public nằm dưới prefix `/api/v1`.** Mọi thứ miễn trừ chuẩn này phải nằm NGOÀI prefix. Repo đã dựng `/api/v2` (section 11, hoặc nhánh B của [`MIGRATION_CHECKLIST.md`](../02-checklists/MIGRATION_CHECKLIST.md)) thì mọi luật viết `/api/v1` áp cho **mọi** prefix `/api/v<n>` đang phục vụ; "ngoài prefix" nghĩa là ngoài tất cả chúng.

Đủ 8 dòng là compliant. Không cần OpenAPI, không cần package `contracts/`, không cần project `.Contracts`, không cần codegen, không cần catalog i18n.

**Luật precedence — đọc trước khi vào phần sau.** Section 4–10 là **chi tiết hóa của Tier 0**: chúng nói 8 dòng trên trông ra sao trong từng tình huống, không phải thêm tầng nghĩa vụ mới. Chữ "bắt buộc" ở đó có nghĩa "bắt buộc **khi** tình huống đó xảy ra trong dự án của bạn" — API không có tác vụ nền thì luật 202 không áp, không có form thì luật 422 không áp. Chỉ những món có tên trong bảng **Tier 2** mới được hoãn, và hoãn theo điều kiện ghi ở đó.

### Tier 1 — bắt buộc KHI điều kiện xảy ra

| Điều kiện | Bắt buộc thêm |
|---|---|
| Có form / validation input | `422` + `code: validation_failed` + `details[]` đầy đủ |
| Có rate limit | header `Retry-After` trên 429 |
| Có collection lớn dần theo thời gian | `page.total` chính xác, `limit` có trần và clamp, sort deterministic tie-break bằng `id` (section 8.2) |
| Có auth | header `WWW-Authenticate` trên 401, scheme đúng cái server thật sự chấp nhận (`Bearer`, `Basic`, `Negotiate` cho Windows Auth) |
| Có tác vụ chạy nền | `202` + header `Location` + job resource |
| Có gửi `user_message` đã localize | header `Content-Language` |
| API có consumer ngoài repo (mobile, đối tác) | `user_message` + OpenAPI |

### Tier 2 — bắt buộc khi điều kiện xảy ra, được phép hoãn theo cột "Được hoãn khi"

Tier 2 **không** được gate bằng mode thư mục. Mode `structured` chọn theo khối lượng code (core section 4), chẳng liên quan gì tới độ tinh vi của API — một MVP `simple` phục vụ app mobile cần idempotency hơn hẳn một hệ thống `structured` chỉ có admin web nội bộ.

| Món | Bắt buộc khi | Được hoãn khi |
|---|---|---|
| `Idempotency-Key` | Client **không kiểm soát được** tự retry POST: mobile app, đối tác tích hợp, worker, desktop offline-first. Hoặc endpoint đụng tiền | Chỉ có FE trong cùng repo và không có thao tác đụng tiền |
| Cursor pagination | Endpoint là feed ghi liên tục, hoặc `COUNT(*)` đo được là chậm (section 8.2) | Dataset còn nhỏ |
| ETag / `If-Match` / `428` | Nhiều người sửa cùng một resource và ghi đè nhau là mất dữ liệu thật | Một người sửa, hoặc sửa đè là chấp nhận được |
| `Deprecation` / `Sunset` | API có consumer ngoài repo | Client nằm cùng repo, deploy cùng lúc |
| Batch non-atomic | Thao tác thật sự không transactional được (section 1.4 của appendix) | Còn transactional được — giữ atomic |

**Chọn `structured` không tự động kích hoạt Tier 2, và chọn `simple` không miễn Tier 2.** Điều kiện ở cột giữa mới là thứ quyết định.

### Không bao giờ bắt buộc trong pack này

Wrapper `{"data": ...}` cho success · mã lỗi dạng số · RFC 9457 `application/problem+json` · HATEOAS · `meta` trên mọi response · shared contracts package · codegen · `207 Multi-Status`.

---

## 4. Thành công

**Không có envelope. Body chính là resource.** Ngoại lệ duy nhất: list.

### 4.1 Một object — `GET /api/v1/orders/{id}` → 200

```json
{
  "id": "01J8ZP9K3M4N5Q6R7S8T9V0W1X",
  "status": "pending_payment",
  "customer_id": "01J8ZP9K3M4N5Q6R7S8T9V0W22",
  "note": null,
  "total": { "amount": "1299000.00", "currency": "VND" },
  "is_paid": false,
  "paid_at": null,
  "created_at": "2026-08-12T03:14:07Z",
  "updated_at": "2026-08-12T03:14:07Z"
}
```

### 4.2 List — `GET /api/v1/orders?limit=20&offset=0&sort=-created_at` → 200

```json
{
  "items": [
    { "id": "01J8ZP9K3M4N5Q6R7S8T9V0W1X", "status": "pending_payment", "created_at": "2026-08-12T03:14:07Z" }
  ],
  "page": { "limit": 20, "offset": 0, "total": 137 }
}
```

List rỗng vẫn là `{"items": [], "page": {"limit": 20, "offset": 0, "total": 0}}` — không 404, không `null`.

### 4.3 Tạo mới — `POST /api/v1/orders` → 201

Bắt buộc header `Location: /api/v1/orders/<id>`. Body là **full resource vừa tạo**, không phải `{"id": "..."}` — FE gần như luôn cần render ngay, tránh một round-trip GET.

### 4.4 Void action — 204

Body **rỗng tuyệt đối**: không `Content-Type`, không `{}`, không `null`. Cấm `200 {"success": true}`.

Rule:

- DELETE lặp lại trên resource đã xóa vẫn trả **204**, không phải 404 — DELETE là idempotent (RFC 9110 §9.2.2). Không có luật này, client retry sau network timeout ăn lỗi giả.
- Action **làm đổi trạng thái** thì trả `200` + resource sau khi đổi, không trả 204. Ví dụ `POST /api/v1/orders/{id}/cancel` → `200` + order với `status: "cancelled"`.

---

## 5. Thất bại

**Mọi status ≥ 400 đều đúng hình dạng này. Đúng 1 key `error` ở top level, không có key nào khác.**

### 5.1 Mức tối thiểu — 4 field bắt buộc

```json
{
  "error": {
    "code": "not_found",
    "message": "order 01J8ZP9K3M4N5Q6R7S8T9V0W1X does not exist",
    "trace_id": "01J8ZQ7K9M3PQ4RSTVWXYZ0004",
    "retryable": false
  }
}
```

### 5.2 Mức đầy đủ

```json
{
  "error": {
    "code": "validation_failed",
    "message": "request body failed validation: 3 field(s)",
    "trace_id": "01J8ZQ7K9M3PQ4RSTVWXYZ0008",
    "retryable": false,
    "user_message": "Vui lòng kiểm tra lại thông tin đã nhập.",
    "details": [
      { "field": "email", "in": "body", "code": "invalid_format", "message": "must be a valid email address", "user_message": "Email không hợp lệ." },
      { "field": "billing.postal_code", "in": "body", "code": "required", "message": "is required" },
      { "field": "created_after", "in": "query", "code": "invalid_format", "message": "must be RFC3339 UTC" }
    ]
  }
}
```

### 5.3 500 — `message` là chuỗi CỐ ĐỊNH

```json
{
  "error": {
    "code": "internal",
    "message": "internal server error",
    "trace_id": "01J8ZQ7K9M3PQ4RSTVWXYZ0007",
    "retryable": false,
    "user_message": "Hệ thống đang gặp sự cố. Vui lòng thử lại hoặc gửi mã 01J8ZQ7K9M3PQ4RSTVWXYZ0007 cho hỗ trợ."
  }
}
```

Stack trace đầy đủ nằm ở log, join bằng `trace_id`. `trace_id` là cầu nối duy nhất giữa user và log — đó là lý do nó bắt buộc chứ không optional.

### 5.4 Bảng field

| Field | Kiểu | Bắt buộc | Ý nghĩa |
|---|---|---|---|
| `error` | object | khi status ≥ 400 | Toàn bộ lỗi nằm gọn trong đúng key này. Response lỗi không có key nào khác ở top level; response success không bao giờ chứa key này. **Ngoại lệ duy nhất:** job resource và batch item — ở đó `error` là một **field mô tả trạng thái của dữ liệu**, không phải error envelope của request (request vẫn 200). Xem appendix section 1.4 và 1.5 |
| `error.code` | string snake_case | Có | Khóa máy đọc, ổn định vĩnh viễn như tên bảng DB. Client `switch/case` trên field này. Cấm parse `message` để đoán loại lỗi |
| `error.message` | string (en) | Có | Câu tiếng Anh cho developer và log. Cấm stack trace, SQL, path nội bộ, connection string. Viết sao cho lỡ FE hiện thẳng ra cũng không rò gì |
| `error.trace_id` | string | Có | Bằng đúng request id trong log JSON và bằng header `X-Request-Id` |
| `error.retryable` | bool | Có | Lỗi có tính **tạm thời** hay không — thử lại lát nữa có thể ra kết quả khác. Là **thuộc tính của `code`** (tra bảng section 6), không phải quyết định từng lần. **Không phải giấy phép phát lại request** — xem luật ngay dưới section 6.1 |
| `error.user_message` | string đã localize | Optional | Hiện thẳng cho end-user. Chỉ gửi khi server thật sự có bản dịch. Vắng thì omit, không gửi `null` |
| `error.details` | array | Optional; **bắt buộc khi** `code = validation_failed` | Danh sách lỗi con. **Không chỉ dành cho validation** — đây là **kênh máy đọc được duy nhất** để lỗi 4xx mang dữ liệu nghiệp vụ, vì `message` bị anti-pattern 8 cấm parse và `user_message` là chuỗi cho người. Lỗi domain được phép kèm `details[]` + `params` kể cả khi `code ≠ validation_failed`: `wallet.insufficient_funds` gửi `params: {"shortfall": "50000.00", "currency": "VND"}` để FE dựng câu "còn thiếu 50.000đ". **Cấm thêm key mới trong `error`** cho mục đích này |
| `error.details[].field` | string | Optional trong item | Đường dẫn theo **tên field trên wire** client đã gửi: `email`, `billing.postal_code`, `line_items[0].quantity`. Omit khi lỗi cross-field |
| `error.details[].in` | enum | Optional trong item | `body` \| `query` \| `path` \| `header` \| `cookie` \| `form` (part của multipart, `field` = tên part). Không có nó thì `limit` ở query và `limit` ở body nhìn y hệt nhau |
| `error.details[].code` | string snake_case | Có trong item | Tập đóng: `required`, `invalid_format`, `invalid_enum`, `invalid_type`, `min`, `max`, `too_short`, `too_long`, `not_unique`, `immutable`, `conflict`, `not_found` |
| `error.details[].message` | string (en) | Có trong item | `must be a valid email address` |
| `error.details[].user_message` | string | Optional trong item | Bản localize riêng cho field, gắn dưới input trong form |
| `error.details[].params` | object scalar | Optional trong item | Tham số để FE nội suy chuỗi i18n: `{"min": 1}` → `Tối thiểu {min}` |

Rule:

- **Không có `error_subcode` dạng số.** Sub-code gộp thẳng vào `code` qua namespace: `payment.card_declined`.
- **Không lồng** `{"error": {"error": ...}}`; không kèm `success: false`.
- `user_message` và HTTP status **không bao giờ** được sinh ở tầng service/usecase. Domain trả typed error, **adapter HTTP mới map** sang status + code + message.

---

## 6. Bảng mã lỗi

**Format: `<reserved>` hoặc `<module>.<reason>` — snake_case, ASCII, chữ thường.**

### 6.1 Tập reserved — 17 mã, khóa 1-1 với status

Bảng này **phải được implement thành pure function**, không để dưới dạng văn xuôi. Vừa test được bằng table-driven test, vừa cho worker và desktop tái dùng taxonomy dù không đi qua HTTP.

**Được phép — và nên — tách làm 2 nửa theo đúng ranh giới kiến trúc:**

| Nửa | Nội dung | Đặt ở |
|---|---|---|
| Domain | `Code` + `retryable(code) bool` | Tầng domain (`model/`, `internal/domain/`, `.Application/`). "Lỗi này có tạm thời không" là câu hỏi nghiệp vụ, và `usecase`/`worker` cần trả lời nó mà không được biết HTTP |
| Transport | `status(code) int` | Cùng package với response writer. HTTP status là **phép chiếu** của code sang một transport cụ thể — desktop bridge và worker không dùng tới nó |

Tách như vậy thì tầng domain không phải import `net/http` chỉ để lấy vài hằng số nguyên, và không vi phạm rule "domain không biết HTTP" mà mọi template structured của pack đang giữ. Repo nhỏ giữ một file cũng được, miễn file đó không nằm trong package HTTP.

| code | status | retryable |
|---|---:|---|
| `bad_request` | 400 | false |
| `unauthenticated` | 401 | false |
| `permission_denied` | 403 | false |
| `not_found` | 404 | false |
| `method_not_allowed` | 405 | false |
| `conflict` | 409 | false |
| `precondition_failed` | 412 | false |
| `payload_too_large` | 413 | false |
| `unsupported_media_type` | 415 | false |
| `validation_failed` | 422 | false |
| `rate_limited` | 429 | **true** |
| `internal` | 500 | false |
| `not_implemented` | 501 | false |
| `dependency_failed` | 502 | **true** |
| `unavailable` | 503 | **true** |
| `not_ready` | 503 | **true** |
| `timeout` | 504 | **true** |

`503` tách 2 mã: `unavailable` (dependency chết — `message` phải chỉ đích danh dependency nào, ví dụ `"redis unavailable"`) và `not_ready` (đang deploy hoặc chạy migration). Gộp chung làm mất đúng thông tin dashboard cần nhất.

**Ngoại lệ duy nhất của bảng:** lỗi transient mang status không-retryable (deadlock DB, connection reset) được phép override `retryable: true`. Đây chính là lý do `retryable` phải lên wire chứ không để client tự suy từ status.

### 6.1b `retryable` không phải giấy phép phát lại request

`retryable: true` nói về **lỗi**, không nói về **request**. Nó nghĩa là "nguyên nhân có tính tạm thời", không nghĩa là "cứ gửi lại đi".

- **Method idempotent** (GET, PUT, DELETE, HEAD): `retryable: true` thì client được tự động phát lại, có backoff.
- **Method không idempotent** (POST tạo mới, POST tính tiền): client **chỉ** được tự động phát lại khi request có `Idempotency-Key`. Không có key thì `retryable: true` chỉ có nghĩa "người dùng bấm lại được", **không** phải "client tự bấm hộ".

Vì sao phải viết ra: `timeout` (504) và `dependency_failed` (502) đều mặc định `retryable: true`, mà đó đúng là hai mã xuất hiện khi request **đã chạy xong ở server nhưng response không về tới client**. Client tự retry ở đó là tạo đơn trùng, trừ tiền trùng, gửi tin trùng. Đây là hậu quả tốn kém nhất mà chuẩn này có thể gây ra, nên luật nằm ở phần normative chứ không ở appendix.

### 6.2 Domain code — `<module>.<reason>`

- `<module>` = tên bounded context, **khớp đúng tên folder**: `services/<module>/` (go-vue-services), `modules/<module>/` (node-express, python-fastapi structured), hoặc tên aggregate (`order`, `payment`, `wallet`).
- `<reason>` = trạng thái nghiệp vụ, thể chủ động, không có tiền tố `error_`: `order.already_paid`, `wallet.insufficient_funds`, `auth.token_expired`, `inventory.out_of_stock`.
- Domain code luôn đi với 4xx, thường 409 hoặc 422. **Không có domain code nào gắn với 5xx.**
- **Domain code cũng phải có entry trong cùng pure function** `code -> (http_status, retryable)` ở section 6.1, không được để handler tự chọn status. Nếu không, cùng một `catalog.out_of_stock` sẽ ra 409 ở handler này và 422 ở handler kia — và trong kiến trúc nhiều module (`go-vue-services`), lỗi của module B đi ra qua handler của module A thì không ai biết ai quyết. Bảng là nơi quyết; handler chỉ tra bảng.

### 6.3 Luật chống phình mã

**Chỉ tạo domain code mới khi FE cần rẽ nhánh xử lý khác nhau.**

- `user_not_found`, `order_not_found`, `product_not_found` là **sai** — dùng `not_found` cho cả ba, vì FE xử lý y hệt nhau.
- `wallet.insufficient_funds` là **đúng**, vì FE phải mở màn hình nạp tiền.

### 6.4 Nơi khai báo — mỗi repo đúng 1 file

| ngôn ngữ | tên file và dạng khai báo |
|---|---|
| Go | `errors.go` — `const` string |
| C# | `ErrorCodes.cs` — static class, `const string` |
| TypeScript / JavaScript | `error-codes.ts` (hoặc `.js` nếu repo dùng JS) — union type hoặc `as const` object |
| Python | `errors.py` — `StrEnum` |

**Đường dẫn khác nhau theo từng template** — Go có 5 template với 5 layout khác nhau, và service không có HTTP thì file này nằm ở tầng domain chứ không ở package HTTP. Tra bảng "nơi đặt code theo từng stack" ở [appendix section 3](API_RESPONSE_CONTRACT_APPENDIX.md), đừng suy từ bảng này.

Nguyên tắc chọn chỗ: đặt ở tầng mà **mọi nơi cần rẽ nhánh theo `code` đều import xuôi chiều được**. Với backend có HTTP đó là package response/httpx; với worker đó là domain — đừng bắt `usecase` import ngược vào adapter chỉ để đọc một hằng số.

Phía FE, file khai báo là bản **mirror**, không phải nguồn sự thật. Nguồn sự thật là `docs/api-contract.md` của repo backend (section 11); khi backend thêm code mới thì FE cập nhật mirror theo file đó, không hỏi miệng.

Test tối thiểu: mọi chuỗi code xuất hiện trong response phải tồn tại trong file khai báo. Một test grep/regex là đủ.

### 6.5 Luật tương thích

- **Thêm code mới = non-breaking.** Client **bắt buộc** fallback theo status class khi gặp code lạ (4xx hiện `user_message` hoặc câu chung; 5xx hiện "hệ thống lỗi" + `trace_id`). Client crash vì code lạ là bug của client.
- **Đổi tên hoặc xóa code đã public = breaking** → `/api/v2`, hoặc giữ song song rồi deprecate.
- **Đổi ý nghĩa code mà giữ nguyên tên = tệ hơn breaking** (im lặng làm sai logic client). Cấm tuyệt đối.

---

## 7. Bảng map tình huống → status → code

| Tình huống | Status | Code |
|---|---|---|
| GET một resource; PUT/PATCH xong có body | 200 | — |
| Tạo resource mới | 201 + `Location` | — (body = full resource) |
| POST trả **kết quả thao tác**, không tạo resource nào (login, đổi mật khẩu, preview export) | 200, **không** `Location` | — (body = kết quả thao tác, ví dụ `{"access_token": ..., "expires_at": ...}`) |
| Nhận việc chạy nền | 202 + `Location: /api/v1/jobs/{id}` | — (body = job resource) |
| Xóa xong, hoặc action void | 204 (body rỗng tuyệt đối) | — |
| DELETE lặp trên resource đã xóa | 204 | — |
| Batch non-atomic xong, có item lỗi lẫn item OK | 200 | — (lỗi từng item nằm trong `results[].error`) |
| JSON hỏng, query param sai kiểu, thiếu param bắt buộc | 400 | `bad_request` |
| Thiếu credential, token sai chữ ký hoặc hết hạn | 401 + `WWW-Authenticate` | `unauthenticated`, `auth.token_expired` |
| Đã xác thực nhưng thiếu quyền, thiếu scope, sai tenant | 403 | `permission_denied` |
| Resource không tồn tại, HOẶC caller không được phép biết nó tồn tại | 404 | `not_found` |
| Route hoàn toàn không tồn tại (catch-all của framework) | 404 | `not_found` — **vẫn phải là JSON envelope** |
| Sai HTTP method trên route có thật | 405 + header `Allow` | `method_not_allowed` |
| **Vi phạm state machine** ("đơn đã giao không hủy được"), unique trùng, optimistic lock lệch version | 409 | `conflict` hoặc `order.already_paid` |
| `If-Match` / ETag không khớp | 412 | `precondition_failed` |
| Body hoặc upload vượt giới hạn | 413 | `payload_too_large` |
| `Content-Type` không hỗ trợ | 415 | `unsupported_media_type` |
| Sai format field, min/max, enum lạ | 422 | `validation_failed` + `details[]` |
| **Vi phạm rule trên dữ liệu hiện tại** ("không đủ số dư") | 422 | `wallet.insufficient_funds` |
| Batch atomic có 1 item sai, rollback cả lô | 422 | `validation_failed`, `details[].field = "items[3].sku"` |
| Vượt rate limit hoặc quota | 429 + `Retry-After` | `rate_limited` |
| Panic hoặc exception không bắt được | 500 | `internal` (message cố định) |
| Route đã khai báo nhưng chưa implement | 501 | `not_implemented` |
| Upstream, third-party, dependency lỗi hoặc unreachable | 502 | `dependency_failed` |
| Dependency chết, graceful shutdown, bảo trì | 503 + `Retry-After` | `unavailable` |
| Đang deploy hoặc chạy migration | 503 + `Retry-After` | `not_ready` |
| Upstream vượt deadline, DB query timeout | 504 | `timeout` |
| Client tự hủy request (đóng tab, component unmount) | không trả gì | — log level info, **không alert, không đếm vào error rate** |

Rule:

- **Ranh giới 409 và 422:** 409 = vi phạm state machine; 422 = vi phạm rule trên dữ liệu hiện tại.
- **Ranh giới 403 và 404, chốt một lần cho cả API:** resource **tồn tại nhưng caller thuộc tenant/tổ chức khác** → `404`, vì `403` đã tiết lộ rằng id đó có thật. `403` chỉ dành cho resource caller **được phép nhìn thấy** nhưng thiếu quyền cho **hành động** này. Chọn hai kiểu ở hai endpoint là rò rỉ thông tin qua đúng chỗ không ai review.
- **Lỗi nghiệp vụ luôn là 4xx, không bao giờ 5xx.** 5xx phải đồng nghĩa với "có người cần bị đánh thức".
- **Bên thứ 3 từ chối yêu cầu nghiệp vụ** (thẻ bị từ chối, tài khoản bị khoá phía đối tác) là **4xx với domain code**, không phải `502`. `502 dependency_failed` dành cho dependency hỏng, không dành cho dependency trả lời "không".

---

## 8. Phân trang

**Offset là mặc định. Cursor là escape hatch có hình dạng cố định.**

Lý do chọn offset dù cursor đúng kỹ thuật hơn: mọi template `web/fullstack/*` trong pack đều có `apps/admin-web/`, mà admin table nào cũng cần "trang 5/20" và cần `total`. Cursor không cho nhảy trang. Thêm nữa `LIMIT ? OFFSET ?` chạy được ngay trên mọi stack, còn cursor cần sort key ổn định, encode/decode token, và xử lý tie-break.

**Container giống hệt nhau ở cả 2 mode, chỉ `page` khác:**

```json
{ "items": [ ], "page": { "limit": 20, "offset": 40, "total": 137 } }
{ "items": [ ], "page": { "limit": 20, "next_cursor": "eyJpZCI6IjAxSjhaUCJ9", "has_more": true } }
```

Giữ `items` + `page` cho cả hai là điểm mấu chốt: FE table component chỉ có **một** code path; đổi endpoint từ offset sang cursor không phải viết lại renderer.

### 8.1 Request params

| param | default | max | ghi chú |
|---|---|---|---|
| `limit` | 20 | 100 | Server **clamp im lặng** về max, không trả 422. Vì thế `page.limit` phải echo giá trị thực tế — client đọc `page.limit`, không giả định theo param đã gửi |
| `offset` | 0 | — | 0-based. Âm → 400 `bad_request` |
| `cursor` | — | — | Chỉ ở cursor mode. Opaque base64; client cấm parse hoặc tự sinh. Sai hoặc hết hạn → 400 `bad_request` |
| `sort` | theo endpoint | — | `sort=-created_at,name` — dấu `-` là DESC. Field không cho sort → 400 `bad_request` |
| filter | — | — | Query param phẳng snake_case: `?status=pending_payment&created_after=2026-08-01T00:00:00Z`. **Không tự chế DSL** kiểu `filter[status][eq]`. Đa giá trị dùng **lặp param** (`?status=a&status=b`), không dùng CSV. Khoảng dùng hậu tố `_after` / `_before`, nửa mở: `_after` là `>=`, `_before` là `<` |

### 8.2 Rule

- **Mọi lỗi tham số phân trang / sort / filter đều là 400 `bad_request`, không phải 422.** 422 dành cho dữ liệu nghiệp vụ trong body; query param sai là request hỏng. Riêng `limit` vượt max thì **clamp im lặng**, không lỗi — vì thế `page.limit` phải echo giá trị thực tế.
- **Offset pagination phải có sort deterministic, luôn tie-break bằng `id`.** `ORDER BY created_at DESC` không đủ — hai record cùng timestamp sẽ nhảy qua lại giữa các trang. Luôn là `ORDER BY created_at DESC, id DESC`.
- **`total` bắt buộc ở offset mode.** Không có `total: null` hay `total: -1` để né `COUNT(*)`. COUNT quá đắt chính là tín hiệu phải đổi endpoint sang cursor — đó là forcing function biến quyết định "offset hay cursor" thành tiêu chí đo được thay vì cảm tính.
- **Cursor được phép khi và chỉ khi** một trong ba: endpoint là feed ghi liên tục (log, notification, activity stream); bảng đã lớn và `COUNT(*)` đo được là chậm; hoặc client đọc **tuần tự toàn bộ** collection (đối tác sync, export máy) — đó là ca offset vừa chậm vừa bỏ sót bản ghi khi có insert giữa chừng. Khi dùng phải ghi rõ trong `docs/api-contract.md`. Cursor là opaque nên server được phép encode `{offset: N}` bên trong — đó là đường di cư offset sang keyset mà không breaking.
- **Chế độ phân trang là một phần của contract, không phải chi tiết implement.** Đổi `page` từ offset mode sang cursor mode là **breaking** theo section 11 (xóa `total` và `offset` mà client đang đọc). Với endpoint có consumer ngoài repo, chốt chế độ ngay khi thiết kế; đổi sau phải lên `/api/v2`.
- **Miễn trừ bounded collection:** danh sách tĩnh có trần cứng (countries, roles, enum options) được khai báo "không phân trang" trong `docs/api-contract.md` và trả `{"items": [...]}` không kèm `page`.
- **Miễn trừ endpoint tổng hợp:** list ghép dữ liệu từ nhiều bounded context, hoặc sort theo giá trị tính toán, được phép trả `page` **không có `total`**. Lý do: trong kiến trúc modular monolith, `go-vue-services` **cấm join chéo bảng của module khác**, nên một `COUNT(*)` chính xác qua 3 module là bất khả thi nếu không phá chính rule đó. Endpoint loại này phải khai trong `docs/api-contract.md` kèm trần cứng cho `limit`, và trả 400 `bad_request` khi client xin vượt trần. Đây là miễn trừ hẹp — list của **một** module vẫn bắt buộc `total`.
  Đường thoát tốt hơn khi màn hình admin thật sự cần số trang: dựng **read model** (view hoặc bảng projection) **read-only**, khai chủ sở hữu rõ ràng trong `docs/architecture.md`, cập nhật qua event của các module nguồn. Read model đọc chéo là hợp lệ vì nó không phá invariant của module nào — thứ bị cấm là **ghi** chéo và **join trực tiếp bảng nghiệp vụ** của module khác.
- **Metadata nằm trong body, không phải header.** Không dùng `Link` (RFC 8288) hay `X-Total-Count`: header bị CORS chặn mặc định cho tới khi ai đó nhớ set `Access-Control-Expose-Headers`, không type được bằng TS, và không hiện trong tab Preview JSON của devtools.
- **Nested list** nhúng trong một resource (`order.line_items`) là array trần, không có `page`. Khi nó có khả năng dài vô hạn thì tách endpoint riêng, lúc đó mới có `{items, page}`. Không lồng `{items, page}` bên trong một resource.

---

## 9. Validation

**Luôn là 422 + `validation_failed` + `details[]`. Trả TẤT CẢ field sai trong một lần — cấm dừng ở field đầu tiên.** Bắt user sửa từng field một là cách nhanh nhất khiến họ bỏ form.

**Ranh giới decode và validate — hai tầng khác nhau:**

- **Decode hỏng** (JSON không parse được, `"abc"` cho field số, thiếu field bắt buộc ở tầng kiểu) → `400 bad_request`. Được phép dừng ở lỗi đầu tiên: parser dừng là dừng, và body hỏng thì không có gì để validate tiếp.
- **Decode xong nhưng vi phạm rule** (email sai format, min/max, enum lạ) → `422 validation_failed` + **tất cả** field sai.

Không gộp hai tầng: gộp lại thì hoặc là bạn trả 400 cho lỗi format email (client không biết field nào), hoặc là bạn hứa "trả tất cả field sai" ở một tầng không giữ được lời hứa đó.

Ví dụ đầy đủ: xem section 5.2.

### 9.1 Luật về `field` path

- **Dùng tên field trên wire mà client đã gửi**, không phải tên struct nội bộ: `postal_code` chứ không phải `PostalCode` hay `BillingAddressVO.Zip`. Đây là lỗi hay gặp nhất khi dùng validator của framework (FluentValidation, class-validator, Pydantic) — phải config để nó xuất tên đã serialize.
- Nested dùng dấu chấm (`billing.postal_code`); array dùng index ngoặc vuông (`line_items[0].quantity`).
- Path phải parse được bằng lodash `get()` để FE map thẳng vào form state. **Không dùng JSON Pointer** (`/items/1/quantity`) — khó đọc hơn và không map trực tiếp được.
- **Lỗi cross-field:** omit `field`, FE hiện ở đầu form. Không dùng `field: ""`, không dùng `field: "_form"`.

### 9.2 i18n

**Mặc định: FE dịch, server không dịch.** Server trả `code` + `details[].code` + `field` + `params`; FE có key i18n dạng `errors.validation.min` với `{min}` nội suy. FE đã có sẵn i18n runtime (vue-i18n, react-intl), server thì chưa — ép server dựng catalog dịch là ceremony bị bỏ qua đầu tiên khi gấp.

**Server dịch (bật `user_message`) chỉ khi** một trong ba: (a) API phục vụ client không nằm trong repo — mobile app, đối tác tích hợp; (b) message cần dữ liệu chỉ server có ("Còn 3 sản phẩm trong kho"); (c) đã có sẵn catalog i18n server-side. Khi bật: đọc `Accept-Language`, fallback về ngôn ngữ mặc định của project, và trả header `Content-Language`.

**Thứ tự fallback phía FE, cố định:**

```text
details[].user_message
  → t("errors." + details[].code, details[].params)
    → error.user_message
      → t("errors." + error.code)
        → câu chung "Đã có lỗi xảy ra" + hiện trace_id
```

Bậc cuối luôn hiện `trace_id` — kể cả khi mọi thứ khác thiếu, user vẫn có mã đưa cho support.

---

## 10. Wire conventions

| Hạng mục | Quyết định | Lý do |
|---|---|---|
| Key casing | `snake_case` cho JSON key, query param, error code, enum value. Header giữ chuẩn HTTP (`X-Request-Id`) | Một luật casing duy nhất cho toàn contract. camelCase bắt phải sống chung 2 luật (`createdAt` nhưng `validation_failed`) |
| Timestamp | RFC3339 UTC, luôn kết thúc `Z`. Field tên `*_at`. Ngày thuần `YYYY-MM-DD`, field `*_date` | Response **không bao giờ** có `+07:00` (request thì chấp nhận). Giết class bug "server VN, DB UTC, FE lệch 7 tiếng". Không dùng epoch int |
| Money | Object `{"amount": "1299000.00", "currency": "VND"}`, `amount` là **string decimal** | String không mất chính xác qua JS float; object đảm bảo amount không mồ côi khỏi currency. Project single-currency được phép dùng string decimal trần + `currency` ở cấp resource, nhưng phải khai trong `docs/api-contract.md` |
| Decimal khác | Cần chính xác tuyệt đối (số lượng lẻ, tỉ giá) → string. Chấp nhận sai số (%, điểm) → number | |
| Null vs omit | Field đã khai báo trong schema resource thì **luôn có mặt**, rỗng thì `null`. Trong `error` thì ngược lại: optional vắng mặt thì **omit** | Resource: giết class bug `undefined` vs `null` phía TS |
| Null trong PATCH | Field vắng mặt = giữ nguyên. Field bằng `null` = xóa giá trị | |
| Enum | String snake_case chữ thường (`pending_payment`). Không dùng int, không localize. **Client phải chịu được giá trị lạ** — fallback, không crash | Thêm enum value mới chỉ non-breaking khi client tolerant |
| ID | **Luôn là string**, kể cả khi DB là bigint. Khuyến nghị ULID/UUIDv7 cho project mới. **Luôn sort bằng `created_at`, không bao giờ bằng `id`** (`"10" < "9"`) | JS `Number.MAX_SAFE_INTEGER` là 2^53, int64 id bị truncate im lặng |
| Boolean | `is_*` hoặc `has_*`. Không dùng bool nullable — 3 trạng thái thì dùng enum | |
| Array | Rỗng là `[]`, không bao giờ `null` | |

### Header bắt buộc

| Header | Khi nào | Ghi chú |
|---|---|---|
| `X-Request-Id` | **Mọi** response — success lẫn error, 204 lẫn file download | Nhận từ client nếu hợp lệ, không thì server sinh. Đây là thứ thay thế cho việc phải bọc success trong envelope để có chỗ nhét trace |
| `Location` | 201 và 202 | 201 trỏ resource vừa tạo; 202 trỏ URL poll job |
| `Retry-After` | 429 và 503 | Nguồn sự thật duy nhất cho việc chờ bao lâu. Body không cần mirror |
| `WWW-Authenticate` | 401 | RFC 9110 yêu cầu. Scheme phải là cái server thật sự chấp nhận, không mặc định `Bearer` |
| `Allow` | 405 | RFC 9110 yêu cầu |
| `Content-Language` | Khi có gửi `user_message` đã localize | |

Nhớ set `Access-Control-Expose-Headers` cho `X-Request-Id`, `Retry-After`, `Deprecation`, `Sunset` — nếu không FE đọc không thấy.

---

## 11. Versioning

Hai trục tách biệt, đừng nhầm:

- **Contract version** — cái frame: envelope, error shape, conventions. Hiện tại là `api-1`, thuộc sở hữu của pack.
- **API surface version** — endpoint và field của project. Prefix URL `/api/v1/...`.

Version bằng URL path chứ không bằng header hay content negotiation: grep được, cache được, gõ được vào thanh địa chỉ browser, nginx route được bằng một dòng.

**Project khai báo ở đúng 2 chỗ:**

1. `docs/api-contract.md` của project — dòng đầu ghi `API contract: structure_standard_pack/api-1`, kèm bảng domain error code và danh sách endpoint có ngoại lệ (cursor mode, batch non-atomic, bounded collection).
2. Runtime, trong body `/healthz`: `{"status":"ok","service":"billing-api","version":"1.4.2","contract":"api-1"}`.

**Non-breaking** (không bump gì): thêm field optional, thêm error code mới, thêm enum value mới, nới lỏng validation, thêm endpoint. Điều này **chỉ đúng khi client tolerant** — nên "client phải chịu được field lạ và enum lạ" là luật bắt buộc của chuẩn này, không phải lời khuyên.

**Breaking** (phải lên `/api/v2` hoặc endpoint mới): xóa, đổi tên, đổi kiểu field; siết validation; đổi status code của một tình huống; đổi ý nghĩa của một `code` đã có.

**Contract v2 chỉ ra đời khi frame đổi** — ví dụ thêm `meta` bắt buộc. Pack cam kết frame `api-1` không đổi; sửa chữa trong `api-1` chỉ là làm rõ chữ nghĩa.

Deprecate endpoint: header `Deprecation: true` + `Sunset: <http-date>` (Tier 2), luôn kèm dòng trong CHANGELOG của project.

---

## 12. Anti-patterns

1. `200 OK` kèm `{"success": false}` hoặc `{"status": "error"}`. Hủy toàn bộ giá trị của status code; mọi client, mọi proxy, mọi dashboard đều phải parse body mới biết thất bại. **Cấm tuyệt đối.**
2. Đưa `err.Error()`, `ex.ToString()`, `str(e)`, stack trace, hoặc câu SQL vào `message`. Đây là con đường DSN và tên bảng rò ra internet.
3. Bare JSON array ở top level. Không có chỗ thêm `page` sau này, và có lịch sử lỗ hổng JSON hijacking.
4. Để nguyên trang lỗi mặc định của framework trên route API: HTML của Express, developer exception page của ASP.NET, và đặc biệt `{"detail": "..."}` của FastAPI.
5. Tên field không nhất quán giữa các endpoint: `userId` chỗ này, `user_id` chỗ kia, `customerId` cho cùng một thứ.
6. Envelope lồng envelope: `{"data":{"data":...}}`, `{"error":{"error":...}}`.
7. Service hoặc usecase layer tự sinh câu tiếng Việt, hoặc tự chọn HTTP status. Domain trả typed error, adapter HTTP mới map.
8. Nhét thông tin phân loại vào `message` rồi FE `message.includes("not found")`.
9. Dùng 5xx cho lỗi nghiệp vụ (số dư không đủ, hết hàng).
10. Dùng 404 cho endpoint chưa implement (đúng: 501), hoặc 403 cho chưa đăng nhập (đúng: 401).
11. `total: -1` hoặc `total: null` để né `COUNT(*)`.
12. Mỗi module tự viết response writer riêng — rủi ro số 1 của `go-vue-services`.
13. `retryable: true` cho lỗi 4xx do input sai. Client retry vô nghĩa, tự DDoS chính mình.
14. Trả error envelope của mình cho webhook của bên thứ 3. Provider gặp body lạ sẽ retry storm rồi disable endpoint.
15. Tạo `not_found` riêng cho từng entity (`user_not_found`, `order_not_found`) trong khi FE xử lý y hệt nhau.
16. Sort danh sách theo `id` khi id là string. Luôn sort bằng `created_at`.

---

## 13. Tự kiểm khi áp chuẩn

Danh sách để tự soát khi vừa áp chuẩn vào một repo. Repo cũ đang migrate dùng mục 5b của [`02-checklists/MIGRATION_CHECKLIST.md`](../02-checklists/MIGRATION_CHECKLIST.md).

- [ ] Helper response + writer nằm đúng folder mà template của stack chỉ định; không handler nào tự serialize
- [ ] Đúng 1 file khai báo error code; không có chuỗi code literal rải trong handler
- [ ] Bảng `code -> (status, retryable)` là pure function, không phải chuỗi `if` — tách 2 nửa theo section 6.1 thì **cả hai** nửa đều là pure function
- [ ] Middleware recover/exception toàn cục đã đăng ký; 404 và 405 của framework đã bị override thành JSON
- [ ] `X-Request-Id` có trên mọi response và bằng đúng request id trong log
- [ ] Bẫy framework của stack đã xử lý (section 4 của appendix)
- [ ] **Đã gọi thử một endpoint success và mắt thấy `created_at` / `customer_id` là snake_case.** Sai casing ở nhánh success là lỗi im lặng hay lọt nhất
- [ ] Reverse proxy (nginx/IIS/Caddy) đã override 413, 502, 504 thành JSON envelope và tự sinh `X-Request-Id` khi request chưa có
- [ ] `docs/api-contract.md` đã có: contract version, bảng domain error code, danh sách endpoint ngoại lệ

### Khai `code` thành một kiểu riêng

Rẻ nhất và chắc nhất: khai kiểu riêng cho `code` (`type Code string` ở Go, `StrEnum` ở Python, union `as const` ở TS, `const string` trong static class ở C#) và ép writer **chỉ nhận kiểu đó**. Compiler chặn code lạ ngay lúc build, không cần cơ chế nào khác. Repo JS thuần không có kiểu thì đây là chỗ duy nhất cần một test grep thay thế.

### Test — khuyến nghị, không bắt buộc

Chuẩn này không ép dự án nào phải viết test. Nhưng ba test dưới đây rẻ, và là thứ duy nhất bắt được drift mà code review hay bỏ sót — cân nhắc theo quy mô và tuổi thọ dự án:

| Test | Bắt gì | Đáng làm khi |
|---|---|---|
| Table-driven cho bảng mã | Code thiếu entry, hai nửa bảng lệch key sau khi tách | Có domain code, hoặc bảng bắt đầu dài |
| Smoke duyệt mọi route đã mount | Drift hình dạng — cả `error.code`/`error.trace_id` lẫn `items`/`page`. Đây là chỗ duy nhất chạm nhánh success | Nhiều người cùng thêm endpoint, hoặc nhiều module trong một binary |
| E2e cố ý gây panic | 500 rò stack trace ra ngoài | API có consumer ngoài repo, hoặc dữ liệu nhạy cảm |

Không viết test thì các luật ở trên sống nhờ code review — chấp nhận được với repo nhỏ, rủi ro tăng dần theo số người cùng chạm.

---

## Metadata

- Contract version: `api-1`
- Pack version: `3.1`
- Owner: `https://github.com/TranTaiDakLak/`
- Maintainer: `Engineering / Architecture`

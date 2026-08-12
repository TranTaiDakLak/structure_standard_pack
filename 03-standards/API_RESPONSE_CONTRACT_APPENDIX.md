# API Response Contract — Appendix

> Phần bổ trợ cho [`API_RESPONSE_CONTRACT.md`](API_RESPONSE_CONTRACT.md): edge case (section 1), transport không HTTP + **vai consumer khi gọi API không theo chuẩn** (section 2), nơi đặt code theo từng stack (section 3), bẫy framework (section 4), **api client mẫu cho FE** (section 5), đánh đổi và rủi ro (section 6–8).
> File normative đứng độc lập được — đọc file này khi gặp một tình huống cụ thể hoặc khi triển khai vào một stack cụ thể.

## 1. Edge case

### 1.1 File download

Success path miễn trừ contract: trả binary với `Content-Type` thật, `Content-Disposition: attachment; filename*=UTF-8''bao-cao-2026.xlsx` (dùng `filename*` cho tên tiếng Việt), và `Content-Length` khi biết trước.

Rule:

- **Lỗi xảy ra TRƯỚC byte đầu tiên vẫn phải là JSON envelope** với status đúng. Nghĩa là mọi check quyền và check tồn tại phải chạy xong **trước khi** set header stream.
- Client bắt buộc kiểm tra `res.ok` trước khi đọc blob, nếu không sẽ lưu ra file `.xlsx` chứa JSON lỗi.
- **Ngưỡng chuyển sang job 202 + `result_url`, đo được chứ không cảm tính:** khi thời gian generate vượt ~50% `proxy_read_timeout` của reverse proxy (mặc định nginx là 60s), hoặc file vượt ~50MB. Dưới ngưỡng thì giữ đồng bộ cho gọn.
- **Upload cũng bị chặn ở tầng proxy trước khi tới app.** `client_max_body_size` của nginx mặc định 1MB và tự sinh HTML 413 — giới hạn ở proxy phải khớp giới hạn app khai, nếu không app không bao giờ có cơ hội trả 413 đúng envelope.

### 1.2 Streaming — SSE, NDJSON, chunked

Envelope không áp cho từng chunk. Vấn đề cốt lõi: status đã gửi rồi, không đổi được nữa.

**Tách 2 loại stream — luật khác nhau:**

| Loại | Ví dụ | Kết thúc là |
|---|---|---|
| **Hữu hạn** — có điểm kết tự nhiên | export NDJSON, tail log tới hết file, kết quả job | Sự kiện cần khẳng định. Bắt buộc frame terminal |
| **Vô hạn** — chạy tới khi ai đó dừng | dashboard realtime, `?follow=true`, notification feed | Chuyện bình thường. **Không** bắt buộc frame terminal |

Rule:

- **Stream hữu hạn phải kết thúc bằng frame terminal tường minh**, và client coi "hết stream mà không có frame terminal" là **thất bại**. Không có luật này, mất kết nối giữa chừng bị hiểu nhầm thành "đã xong" và sinh dữ liệu thiếu âm thầm.
- **Stream vô hạn thì ngược lại**: đứt kết nối là điều kiện vận hành bình thường (proxy timeout, wifi chuyển mạng, laptop ngủ). Client phải reconnect có backoff, và server phải hỗ trợ nối lại từ vị trí cũ — SSE dùng `id:` + header `Last-Event-ID`, NDJSON dùng query `?since=<cursor>`. Bắt client coi mỗi lần đứt là lỗi sẽ tạo báo động giả suốt ngày rồi bị tắt đi.
- **Mọi dòng NDJSON phải mang discriminator**, không dựa vào vị trí: `{"type":"item","data":{...}}` · `{"type":"done"}` · `{"type":"error","error":{...}}`. Luật "dòng cuối là lỗi" không dùng được khi client đọc theo dòng và không biết dòng nào là cuối.
- SSE tương ứng: `event: item` / `event: done` / `event: error` + `data: {"error":{...}}`.
- **SSE dưới `/api/v1` phải consume bằng `fetch` + `ReadableStream`, không dùng `EventSource`.** `EventSource` không gắn được header `Authorization` (buộc phải nhét token vào query — rò vào access log), không đọc được body lỗi, và tự reconnect theo cách mình không kiểm soát. Xem hàm `stream()` ở section 5.
- Token hết hạn giữa stream: server đóng stream bằng frame `error` với `code: auth.token_expired`, client refresh token rồi reconnect. Không im lặng cắt.
- Không phân trang trong stream.

### 1.3 Redirect

Tách rõ 2 loại endpoint:

| Loại | Xử lý |
|---|---|
| Browser điều hướng (OAuth callback, payment return, short link) | 302/303 là đúng, không cần body |
| XHR/fetch | **Tuyệt đối không redirect.** `fetch` follow redirect trong suốt và CORS ăn mất `Location`, FE nhận về HTML trang login mà tưởng là JSON. Thay vào đó trả `200` + `{"redirect_url": "https://..."}` để FE tự `window.location.assign()` |

Cần giữ nguyên method (POST sang POST) thì dùng 307/308, không dùng 301/302.

### 1.4 Batch và partial success

**Mặc định bắt buộc là atomic**: cả lô thành công hoặc rollback toàn bộ. Response là success hoặc error bình thường, không cần hình dạng đặc biệt.

Chỉ khi thao tác thật sự không transactional được (mỗi item gọi một API bên thứ 3) mới được dùng shape non-atomic, và **phải khai báo tường minh trong `docs/api-contract.md`**:

```json
{
  "results": [
    { "index": 0, "status": "ok", "data": { "id": "01J8ZP..." } },
    { "index": 1, "status": "error", "error": { "code": "inventory.out_of_stock", "message": "sku KB-87 has 0 left", "retryable": false } }
  ],
  "summary": { "total": 2, "succeeded": 1, "failed": 1 }
}
```

**Ranh giới với anti-pattern `200 + success:false`:** ở đây 200 là đúng vì *thao tác batch đã thực thi thành công*; kết quả từng item là **dữ liệu**, không phải lỗi của request. Đây là chỗ dễ bị viện dẫn sai nhất để lách luật, nên ranh giới phải giữ gắt: batch atomic là mặc định, non-atomic phải khai báo trong docs.

Rule:

- `results[].error` dùng lại đúng error object nhưng bỏ `trace_id` — đã có một cái ở header `X-Request-Id`.
- `index` khớp thứ tự input.
- Cấm dùng `207 Multi-Status`.
- Batch atomic thất bại thì trả 422 với `details[].field = "items[3].sku"`.

### 1.5 Long-running job — 202 + polling

```text
POST /api/v1/reports
→ 202
  Location: /api/v1/jobs/01J9...
  { "job_id": "01J9...", "status": "queued", "created_at": "2026-08-12T03:14:07Z" }
```

Poll `GET /api/v1/jobs/{job_id}` **luôn trả HTTP 200**, kể cả khi job failed — vì bản thân request poll đã thành công, trạng thái nằm ở field `status`. Trả 202 khi poll là sai: client không phân biệt được "job chưa xong" với "request poll bị lỗi".

| Field | Giá trị |
|---|---|
| `status` | `queued` \| `running` \| `succeeded` \| `failed` \| `cancelled` |
| `result` / `result_url` | `succeeded` → `result` inline nếu nhỏ, `result_url` nếu lớn |
| `error` | `failed` → đúng error object, vẫn HTTP 200 |

Rule:

- **Job đã hoàn tất phải giữ tối thiểu 24h** rồi mới trả 404. Không có luật này, client poll chậm thấy 404 và tưởng job không tồn tại.
- Gợi ý nhịp poll qua header `Retry-After` trên response 202.

### 1.6 Health và readiness

Chuẩn hóa cho mọi stack có HTTP surface:

| Endpoint | Ý nghĩa | Response |
|---|---|---|
| `/healthz` | liveness — **không gọi DB**, luôn 200 nếu process sống | `{"status":"ok","service":"billing-api","version":"1.4.2","contract":"api-1"}` |
| `/readyz` | readiness — có gọi dependency | 200 `{"status":"ok","checks":{"db":"ok","redis":"ok"}}` hoặc **503** `{"status":"degraded","checks":{"db":"ok","redis":"fail"}}` |

Rule:

- **Miễn trừ error envelope một cách tường minh.** Readiness fail là một *báo cáo trạng thái*, không phải lỗi của caller. Probe (Docker healthcheck, systemd) chỉ đọc status code, và nhét nó vào `{"error":{...}}` làm mất thông tin check nào đang chết.
- Hai endpoint này nằm ngoài prefix `/api/v1`, không auth, không rate limit, không log mỗi request.
- Field `contract` trong `/healthz` là hằng số, không đụng DB — vẫn đúng tinh thần "healthz chỉ kiểm tra process sống".

### 1.7 Response không phải JSON

**Danh sách đóng** các content type được phép rời khỏi `application/json`: binary (file), `text/event-stream`, `application/x-ndjson`, `text/csv`, `text/plain` (`/metrics`), `text/html` (trang cho browser điều hướng). Đóng tập là điều kiện để luật "`/api/v1` ⇒ JSON" kiểm tra được bằng grep.

Lỗi trước byte đầu tiên: endpoint gọi bởi máy trả JSON envelope bất kể `Accept`; endpoint dành cho browser được render trang HTML lỗi nhưng **phải giữ đúng status code và phải in `trace_id` lên trang** để user copy gửi support. Cấm trả HTML lỗi với status 200.

### 1.8 Webhook receiver — bên thứ 3 quyết định hình dạng

Chuẩn này **không** áp cho request body lẫn response. Luật cứng:

1. **Tuyệt đối không trả error envelope của mình ra ngoài.** Provider gặp body lạ hoặc 5xx sẽ retry storm rồi tự disable endpoint.
2. **Sai chữ ký → 401/403** để provider ngừng retry. Lỗi nội bộ tạm thời → 5xx để provider có retry. Chọn nhầm cặp này gây một trong hai hậu quả: mất event vĩnh viễn, hoặc kẹt vòng retry rồi bị disable endpoint.
3. **Persist raw event rồi trả 2xx ngay**, mọi xử lý đẩy sang worker. Không để business logic quyết định status trả về provider.
4. Idempotent theo event id của provider — retry là chắc chắn xảy ra.
5. Đánh dấu `// contract-exempt: webhook receiver` trong code để reviewer không "sửa cho đúng chuẩn".

Đường dẫn quy ước `/webhooks/<provider>`, nằm ngoài `/api/v1`.

### 1.8b Webhook mình GỬI ĐI

Đây không phải response, nên nằm ngoài frame `api-1`. Nhưng nó dùng chung wire conventions ở section 10 của chuẩn, và bỏ trống thì mỗi dự án tự chế một kiểu.

```json
{
  "id": "01J8ZQ...",
  "type": "order.paid",
  "api_version": "v1",
  "created_at": "2026-08-12T03:14:07Z",
  "data": { }
}
```

Rule:

- `id` để receiver **dedupe** — retry là chắc chắn xảy ra, và receiver không dedupe được nếu event không có id ổn định.
- `type` dạng `<resource>.<past_tense>`: `order.paid`, `refund.completed`. Không dùng thì hiện tại.
- `api_version` tách khỏi version của API — thêm field vào `data` là non-breaking, đổi/xóa field thì bump.
- `data` theo đúng wire conventions section 10 (snake_case, RFC3339 UTC, id là string, money là object).
- Ký payload bằng HMAC, gửi ở header cùng timestamp để receiver chống replay.
- Retry có backoff, có trần, rồi vào dead-letter. Coi mọi 2xx của receiver là thành công; 4xx là **ngừng retry** (endpoint của họ sai, retry vô ích); 5xx và timeout mới retry.
- Ghi rõ danh sách `type` đang phát trong `docs/api-contract.md` — đó là contract với người tích hợp, y như endpoint.

### 1.9 OAuth callback và third-party callback

`GET /auth/callback/<provider>` là endpoint browser điều hướng: luôn 302 về URL của FE, mang kết quả qua query (`?login=ok` hoặc `?error=auth.oauth_denied`), không bao giờ trả JSON. Không hiện lỗi thô của provider cho user.

Chiều ngược lại — khi chính mình gọi token endpoint của provider: response của họ theo RFC 6749 (`{"error":"invalid_grant","error_description":"..."}`). **Không reshape, không forward nguyên si ra client**, mà map tại adapter boundary thành code của mình (`auth.oauth_exchange_failed`, status 502 `dependency_failed`) và log nguyên văn lỗi provider kèm `trace_id`.

### 1.10 Panic và exception không bắt được

Middleware recover hoặc exception handler toàn cục là **bắt buộc** ở mọi stack:

| stack | cơ chế |
|---|---|
| Go | `recover()` trong middleware |
| .NET | `UseExceptionHandler` |
| Express | error middleware 4 tham số, đăng ký **cuối cùng** |
| FastAPI | `@app.exception_handler(Exception)` |

Log full stack + request id + method + path. Response là 500 + `code: internal` với `message` là **chuỗi cố định** — không bao giờ `err.Error()`, `ex.ToString()`, `str(e)`.

API có consumer ngoài repo hoặc chạm dữ liệu nhạy cảm thì nên có một test e2e cố ý gây panic rồi assert body không chứa chuỗi nào của stack trace — đây là luật khó tự soát bằng mắt nhất vì nó chỉ lộ đúng lúc có sự cố.

### 1.11 Timeout

- Mỗi request có deadline server-side, `context`/`CancellationToken` truyền xuống tận DB driver chứ không chỉ ở tầng HTTP.
- Upstream hoặc DB vượt deadline → 504 `timeout`, `retryable: true`.
- Client tự hủy (đóng tab, component unmount) → không có response để trả; log ở mức info, **không đếm vào error rate và không alert**. Nếu không, dashboard sẽ đầy "lỗi" giả mỗi lần user bấm nhanh.
- FE api client bắt buộc có timeout riêng (AbortController), **ngắn hơn** timeout của reverse proxy.

### 1.12 Rate limit

Header `Retry-After` tính bằng giây là **bắt buộc** và là nguồn sự thật duy nhất. `X-RateLimit-Limit`, `-Remaining`, `-Reset` là optional. Áp cho cả login (chống brute force), OTP, và endpoint tốn tài nguyên.

FE interceptor: gặp 429 thì backoff theo `Retry-After`, không retry ngay, và **không retry request non-idempotent** (POST tạo đơn) trừ khi có idempotency key.

### 1.13 Idempotency (Tier 2)

Client gửi `Idempotency-Key: <uuid>` trên POST không idempotent. Server lưu key kèm response trong tối thiểu 24h.

**Với API có consumer ngoài repo, key này là BẮT BUỘC** trên mọi POST có hệ quả tiền hoặc kho, và thiếu key thì trả `400 bad_request` ngay. Lý do: bạn không sửa được client của đối tác, không ép được họ retry đúng cách, và "đối tác retry sau network timeout" không phải rủi ro giả định — nó là chuyện chắc chắn xảy ra. Từ chối nhận request không có key rẻ hơn nhiều so với đối soát giao dịch trùng.

**Ba trạng thái, không phải hai.** Trạng thái thứ ba mới là chỗ bug thật sự nằm:

| Trạng thái | Xử lý |
|---|---|
| Chưa có key | **Ghi key với trạng thái `in_progress` TRƯỚC khi bắt đầu xử lý**, không phải sau khi có response |
| Key đã có response | Trả lại **response cũ nguyên văn** + header `Idempotency-Replayed: true` |
| Key đang `in_progress` | `409 conflict` + `Retry-After` — request gốc còn đang chạy |

Vì sao trạng thái thứ ba là bắt buộc: client timeout 10s trong khi charge mất 12s. Retry đến ở giây thứ 10, lúc đó request gốc chưa ghi response nên key chưa tồn tại — implement kiểu 2 trạng thái sẽ coi đây là request mới và **charge lần hai**. Đây chính xác là ca mà idempotency sinh ra để chặn.

So sánh body giữa 2 lần dùng cùng key bằng hash của bản canonicalize (sort key, bỏ whitespace), không so sánh chuỗi thô. Key trùng nhưng body khác → 409 `conflict`.

---

## 2. Ngoài luồng HTTP request → response chuẩn

Phần này gom mọi thứ không phải "app của mình trả response cho caller": transport không có status code, và chiều ngược lại — mình đi tiêu thụ API của người khác.

**Nguyên tắc: áp phần error, không áp phần success.** Success vẫn trả giá trị trần theo idiom của transport.

### 2.0 `trace_id` khi không có HTTP request

Luật 4 của Tier 0 (section 3 của file normative) định nghĩa `trace_id` bằng request id của HTTP request. Luồng khởi phát ngoài HTTP — cron tick, worker xử lý message, bridge desktop khi offline — không có nguồn đó. Luật:

- **Sinh tại điểm khởi phát**, không phải tại chỗ xảy ra lỗi: một `run_id` cho mỗi lần chạy cron, một id cho mỗi row webhook event đã persist, một id cho mỗi job.
- **Persist cùng bản ghi** mà nó mô tả, để tra ngược được từ dữ liệu ra log.
- **Việc phái sinh kế thừa id của việc cha.** Order tạo qua HTTP → worker gửi email dùng lại `trace_id` của request đã tạo order. Đó là thứ cho phép nối một khiếu nại của khách với toàn bộ chuỗi xử lý.
- Với desktop/extension: sinh tại client, ghi vào log local. Khi request lên server thì gửi kèm ở header `X-Request-Id` để hai bên log cùng một khóa.

### 2.0b Code cho lỗi chỉ tồn tại ở client

Máy không có mạng, user bấm huỷ, cửa sổ đóng giữa chừng — đây không phải lỗi của server và không có status code nào đúng. Dùng namespace `client.*` với `retryable` theo đúng nghĩa tạm thời:

| code | khi nào | retryable |
|---|---|---|
| `client.offline` | không có kết nối mạng | true |
| `client.cancelled` | user huỷ, component unmount, cửa sổ đóng | false |
| `client.storage_failed` | ghi SQLite/IndexedDB local thất bại | false |

Không dùng `unavailable` hay `timeout` cho các ca này: cả hai mang nghĩa "server có vấn đề", sẽ làm hỏng dashboard và làm retry logic nhắm sai chỗ.

### 2.1 Wails bridge — `desktop/wails-go-vue`

Wails ép Go `error` thành Promise reject với một **chuỗi trần**, mất sạch `code`, `user_message`, `retryable`. Vì thế:

- Bound method trả `(T, *AppError)` — cùng một kiểu với mọi stack Go khác của pack, không đẻ tên riêng cho desktop — và bridge wrapper serialize error thành **đúng object** `{code, message, trace_id, retryable, user_message?}` như HTTP.
- `recover()` trong wrapper map sang `internal` + `trace_id`, ghi vào log file local — để support desktop có cùng khóa join với server.
- Nơi đặt: `app.go` (simple) hoặc `internal/app/` (structured) — đúng chỗ template đã ghi là "bridge expose ra FE".

### 2.2 Browser extension messaging

Messaging của extension hỏng theo **ba** cơ chế khác nhau, và chỉ bắt một cơ chế là để lọt hai cái kia:

| Cơ chế | Biểu hiện | Code |
|---|---|---|
| Extension bị reload/disable giữa chừng | `chrome.runtime.sendMessage` **throw đồng bộ** hoặc promise reject — không đi qua callback | `ext.context_invalidated` |
| Không có receiver (background đã ngủ, content script chưa inject) | `chrome.runtime.lastError` được set, callback vẫn chạy với `response === undefined` | `ext.no_receiver` |
| Port đóng giữa luồng dài | `port.onDisconnect` bắn, `chrome.runtime.lastError` trên port | `ext.port_disconnected` |

Bắt cả ba rồi wrap thành cùng error object để popup và content script `switch` trên một union `code` duy nhất với lỗi API thật. Nơi đặt: `src/api-client.js` (simple — một file, không đẻ folder) hoặc `src/platform/messaging/` (structured).

### 2.3 Worker và background service

`service/go-service` và `service/dotnet-worker-service` dùng `error.code` + `retryable` của chính taxonomy này làm **căn cứ quyết định retry/backoff** khi gọi API nội bộ. Điều này nối thẳng vào rule đã có trong template: *"Retry/backoff phải rõ cho lỗi tạm thời; không loop lỗi liên tục làm nghẽn log/CPU"*.

**Với service one-shot, exit code LÀ contract** — nó thay hoàn toàn vai trò của status code, và là thứ cron/systemd/Task Scheduler thật sự đọc. Phải khai trong `docs/` của project:

- `0` = **mọi** bước bắt buộc đã hoàn tất. Không được trả `0` khi có nguồn bắt buộc thất bại, dù các nguồn khác xong — cron sẽ báo xanh trong lúc dữ liệu thiếu.
- `!= 0` = cần người can thiệp. Alert dựa trên đây.
- Project phải liệt kê nguồn/bước nào là **bắt buộc** và nguồn nào là **best-effort** (thất bại vẫn `0` nhưng phải log ở mức warn). Không liệt kê thì "2/3 nguồn OK" trở thành một quyết định ngẫu hứng của người viết code hôm đó.
- Bản ghi hỏng giữa chừng (một dòng CSV sai format) đi vào bảng/thư mục lỗi kèm `trace_id` của lần chạy, không làm đổ cả job — trừ khi tỉ lệ hỏng vượt ngưỡng project tự khai.

Về health surface: `/healthz` + `/readyz` theo section 1.6 là **bắt buộc khi service có HTTP surface** — tức daemon chạy dài, vốn đã cần probe cho Docker/systemd. Service one-shot hoặc chạy theo lịch rồi thoát (cron, Windows Task Scheduler) **không** phải dựng HTTP server chỉ để có health endpoint; liveness của nó là exit code và log của lần chạy. Ép health surface lên loại service này là đẻ thêm cả web stack cho một binary không cần — đúng thứ nguyên tắc "không thêm layer khi chưa có trách nhiệm rõ" muốn chặn.

### 2.4 Desktop .NET

`dotnet-wpf` và `dotnet-winform` chỉ ở vai consumer. Đặt error mapping trong `Services/` (simple) hoặc `.Infrastructure` (structured).

### 2.5 Tiêu thụ API KHÔNG tuân chuẩn `api-1`

Chuẩn này nói cách **sinh** response. Nhưng gần như mọi dự án còn **gọi** API của người khác — partner API, Zalo OA, Facebook Graph, cổng thanh toán, sidecar exe, hệ thống nội bộ cũ. Không cái nào trong đó tuân `api-1`, và một số làm đúng cái anti-pattern số 1 của chuẩn này.

**Luật gốc: mỗi upstream có một adapter riêng, và adapter đó quyết định thành công/thất bại theo luật CỦA UPSTREAM, không theo luật của chuẩn này.** Chỉ sau khi đã quyết xong mới normalize về `AppError` với code của mình.

```text
upstream trả gì đó  →  [adapter riêng của upstream]  →  AppError code của mình  →  usecase
                        ^ luật của HỌ                    ^ luật của MÌNH
```

#### Vì sao không được dùng chung một client

Api client mẫu ở section 5 kiểm tra `res.ok` rồi trả body. Dùng nó cho upstream sau đây là sai im lặng:

| Upstream | Hình dạng thật | Nếu dùng client của `api-1` |
|---|---|---|
| Zalo OA | `200` + `{"error": -216, "message": "..."}` khi **thất bại** | `res.ok` true → báo "đã gửi" trong khi tin không tới |
| Facebook Graph | `{"error": {"message", "code", "error_subcode", "fbtrace_id"}}` | Trùng tên key `error` nên duck-type `body.error` tưởng là envelope của mình, đọc `error.code` ra số thay vì chuỗi |
| Partner API tự chế | `{"status": "error", "msg": "quota exceeded"}` + `429` | Không có `error.code`, không có `retryable` → logic retry của mình mù |

**Cấm duck-type `body.error`.** Key `error` xuất hiện ở rất nhiều API và mang nghĩa khác nhau. Adapter phải biết trước nó đang gọi ai.

#### Nguồn sự thật cho quyết định retry, theo từng loại upstream

| Upstream | Đọc gì để biết có retry không |
|---|---|
| API tự khai `contract: api-1` | `error.code` + `error.retryable` trong body |
| Mọi thứ còn lại | **HTTP status code + header `Retry-After`** — đây là hợp đồng duy nhất họ thật sự tuân. Cộng field riêng của họ nếu tài liệu của họ nói rõ |

Câu "không suy từ status code trần" trong chuẩn chỉ áp cho API tự khai `api-1`. Với bên thứ 3 thì status code **chính là** nguồn sự thật, vì không còn gì khác.

#### Rule

- Mỗi upstream một file adapter, đặt ở vùng integration của template (`repository/`, `internal/adapter/external/`, `.Infrastructure`; extension dùng `src/providers/` ở simple và `src/platform/providers/` ở structured — **không** nhét vào api client của API nội bộ).
- Adapter là nơi **duy nhất** biết hình dạng lỗi của upstream. Ra khỏi adapter chỉ còn `AppError` của mình.
- Map upstream sang code của mình theo ngữ nghĩa, không theo status: upstream trả 400 vì thẻ bị từ chối → `payment.card_declined` (4xx), không phải `bad_request`. Upstream trả 500 → `dependency_failed` (502) vì đó là lỗi của dependency, không phải lỗi của caller mình.
- Log nguyên văn response của upstream kèm `trace_id` của mình. Đó là thứ duy nhất tra ngược được khi đối tác nói "bên tôi không nhận được gì".
- Sidecar exe (xem quy ước `sidecar/` ở core section 5) tính là một upstream — nó có hình dạng lỗi riêng, cần adapter riêng, và thêm một trạng thái mà API mạng không có: **chưa chạy**. Dùng `dependency_failed` + `message` nói rõ sidecar nào, đừng để `ECONNREFUSED` rò lên UI.

### 2.6 BFF và SSR — vừa consumer vừa producer

Nuxt `server/`, Nitro route, Next route handler, hay bất kỳ tầng nào gọi backend rồi phục vụ tiếp: **nghĩa vụ producer chỉ áp cho endpoint machine-facing**, không áp cho trang cho người xem.

| Loại | Ví dụ | Nghĩa vụ |
|---|---|---|
| Machine-facing | `server/api/**` — client gọi bằng `$fetch`/XHR | Áp **toàn bộ** contract như một backend thật: error envelope, status đúng, `X-Request-Id` |
| Browser-facing | SSR render `pages/products/[slug].vue` | **Không** áp. Slug không tồn tại thì trả trang 404 HTML cho SEO, không trả JSON. Status code vẫn phải đúng (404 thật, không phải 200) và `trace_id` phải in lên trang |

**Trusted upstream passthrough.** Khi upstream **cũng** khai `contract: api-1`, BFF **forward nguyên văn** status + object `error` cho mọi 4xx thay vì bọc lại thành `dependency_failed`:

- 4xx của upstream là lỗi của người dùng cuối, không phải lỗi hạ tầng. Bọc thành 502 làm mất `code` mà FE cần rẽ nhánh, mất `details[]` mà form cần, và biến một lỗi nghiệp vụ thành báo động vận hành giả.
- 5xx và timeout của upstream **thì bọc**: đó mới thật là dependency hỏng → `502 dependency_failed`, và log nguyên văn kèm `trace_id`.
- Forward luôn `trace_id` của upstream nếu BFF không tự sinh, để hai bên log nối được với nhau.

Khai rõ trong `docs/api-contract.md` upstream nào được coi là trusted. Một BFF gọi cả backend nhà lẫn API bên thứ 3 thì hai loại đi hai đường khác nhau — bên thứ 3 luôn qua adapter ở section 2.5.

---

## 3. Nơi đặt code theo từng stack

Không đẻ layer mới — dùng đúng chỗ template đã có.

| stack | simple | structured |
|---|---|---|
| `web/backend-only/go` | thêm `response/` ngang cấp `model/` (`envelope.go`, `write.go`); bảng code + nửa `retryable` ở `model/errors.go`, nửa `status` cùng writer | writer ở `internal/adapter/http/response/` (kèm nửa `status`); **bảng code + typed error + nửa `retryable` ở `internal/domain/errors.go`** để usecase rẽ nhánh không phải import adapter |
| `web/backend-only/dotnet` | `Dtos/Common/` (`ApiError`, `ErrorDetail`, `PagedResult<T>`) + `Middlewares/ExceptionHandlingMiddleware.cs` | type ở `.Application`, middleware ở `.Api` — **không** đẻ project `.Contracts` |
| `web/backend-only/node-express` | thêm `src/http/` (response helper + class `ApiError`); error middleware vào `src/middlewares/` đã có sẵn | `src/platform/http/` |
| `web/backend-only/python-fastapi` | `app/schemas/common.py` + handler đăng ký ở `app/main.py` | schema ở `app/schemas/`, handler ở `app/adapters/http/` |
| `web/fullstack/go-vue` | writer + nửa `status` ở `apps/api/handler/response/` cạnh `handler/health/`; **bảng code + nửa `retryable` ở `apps/api/model/errors.go`** — `service/` phải dùng được mà không import `handler/` | writer + nửa `status` ở `apps/api/internal/platform/httpx/` (khớp rule "cross-cutting đặt dưới `internal/platform/`"); bảng code + nửa `retryable` ở `internal/domain/errors.go` |
| `web/fullstack/go-vue-services` | `apps/api/platform/httpresponse/`, cạnh `httpserver/` và `logger/` | `apps/api/internal/shared/` — tree đã khai báo sẵn "errors, pagination"; mapping ở `internal/platform/` |
| `web/fullstack/dotnet-vue` | `backend/src/<ServiceName>/Dtos/` + middleware đăng ký trong `Program.cs` | type ở `.Application`, middleware ở `.Api` |
| `web/fullstack/node-react` | thêm `backend/src/http/` cạnh `models/` | type ở `backend/src/shared/`, middleware ở `backend/src/platform/` |
| `web/frontend-only/vue-vite` | `src/api/client.ts` + `src/api/types.ts` | `src/services/http.ts` + `src/types/api.ts` |
| `web/frontend-only/react-vite` | `src/services/client.ts` + `src/services/types.ts` | `src/services/http.ts` + `src/types/api.ts` |
| `web/frontend-only/nuxt` | **consumer:** wrapper quanh `$fetch` trong `composables/useApi.ts`. **producer:** writer + bảng code ở `server/utils/response.ts` (auto-import của Nitro), override lỗi ở `nitro.errorHandler` | `types/api.ts` + `composables/` cho consumer; `server/utils/response.ts` + `nitro.errorHandler` cho producer. Nghĩa vụ producer chỉ áp cho `server/api/**` — xem section 2.6 |
| `desktop/wails-go-vue` | `app.go` | `internal/app/` |
| `desktop/dotnet-wpf`, `dotnet-winform` | `Services/` | `.Infrastructure` |
| `service/go-service` | bảng code ở `model/errors.go`; `/healthz` + `/readyz` khi có HTTP surface; `retryable` làm input cho retry/backoff luôn áp | bảng code ở `internal/domain/errors.go` — đặt ở `domain` để `usecase`/`worker` rẽ nhánh theo `code` mà không import ngược adapter |
| `service/dotnet-worker-service` | bảng code ở `src/<ServiceName>/Models/ErrorCodes.cs`; health surface và `retryable` như trên | bảng code ở `src/<ServiceName>.Application/Dtos/`; `.Infrastructure` chỉ map vào, không khai code riêng |
| `browser-extension/js` | `src/api-client.js` (api client cho API nội bộ `api-1` + type mirror + wrapper messaging); adapter upstream không tuân chuẩn ở `src/providers/` | `src/platform/api-client/` + type mirror ở `src/shared/`; `src/platform/messaging/` wrap `chrome.runtime.lastError` về cùng error object; adapter upstream không tuân chuẩn ở `src/platform/providers/` |

---

## 4. Bẫy framework — phải ship snippet chạy được, không viết văn xuôi

Đây là nhóm rủi ro cao nhất của chuẩn này: framework **mặc định làm sai** và không ai phát hiện cho tới khi FE gọi thử.

| stack | bẫy | bắt buộc làm |
|---|---|---|
| **.NET có `[ApiController]`** (`backend-only/dotnet`, `fullstack/dotnet-vue`) | `[ApiController]` **tự động** sinh RFC 7807 `ValidationProblemDetails` cho lỗi model-binding. Nguy hiểm hơn: MVC và middleware dùng **hai bộ JSON options khác nhau**, nên cấu hình nhầm chỗ sẽ ra success camelCase + error snake_case trong cùng một API | Xem 3 khối `Program.cs` bắt buộc trong template của stack. Điểm mấu chốt: `AddControllers().AddJsonOptions(...)` cho controller (`Mvc.JsonOptions`) và `ConfigureHttpJsonOptions(...)` cho `WriteAsJsonAsync` của middleware (`Http.Json.JsonOptions`) là **hai thứ khác nhau, phải set cả hai**. Đặt mỗi `ConfigureHttpJsonOptions` là bẫy chết người. Đo trên .NET 10: chỉ lỗi **ghi từ middleware** (`WriteAsJsonAsync`) ra snake_case, còn success **và** lỗi trả bằng `ObjectResult` của controller (422 từ ValidationFilter) đều ra camelCase — body 422 lộ `traceId`/`userMessage`. Nghĩa là soát bằng một request lỗi đi qua middleware sẽ thấy xanh trong lúc toàn bộ nhánh success đã sai — phải gọi thử một endpoint success mới thấy. `SnakeCaseLower` cần .NET 8+ |
| **.NET không có `[ApiController]`** (`dotnet-worker-service` khi expose Minimal API; `dotnet-wpf`, `dotnet-winform` ở vai consumer) | `System.Text.Json` mặc định camelCase | Chỉ cần `JsonNamingPolicy.SnakeCaseLower` — ở `Program.cs` khi serialize, ở `JsonSerializerOptions` của `HttpClient` khi deserialize. Không có `[ApiController]` nên không phải tắt `ProblemDetails`. Quên là deserialize hụt **toàn bộ** field, im lặng, giá trị mặc định |
| **FastAPI** | mặc định trả `{"detail": ...}` cho **cả** `HTTPException` **lẫn** `RequestValidationError` — sai hình dạng ngay từ hộp | Override cả hai bằng `@app.exception_handler`. Đăng ký ở `app/main.py` (simple) hoặc `app/adapters/http/` (structured). Chỉ ghi "bắt buộc override" mà không ship stub thì solo dev sẽ không làm |
| **Express** | trang lỗi HTML mặc định cho unhandled error và 404 | Error middleware 4 tham số đăng ký **cuối cùng**, cộng một catch-all 404 handler cũng trả JSON envelope |
| **Go** | Không có gì tự động, nên dễ mỗi handler tự `json.Marshal`. Cộng 2 bẫy vi phạm Tier 0 **im lặng** — không lỗi build, không lỗi test: `time.Time` marshal ra `+07:00` khi process không chạy UTC (mặc định trên server VN), và `id` kiểu `int64` marshal thành JSON number **Snippet chạy được: [`snippets/go-api-contract.md`](snippets/go-api-contract.md)** — phủ 5 template Go của pack. Mọi handler đi qua helper chung; cấm ghi thẳng ra `http.ResponseWriter`. Trong DTO: `CreatedAt` luôn `.UTC()` trước khi trả (hoặc `TZ=UTC` trong container), và `ID` khai kiểu `string` với `strconv.FormatInt` ở boundary — đừng để `json:"id"` trên một field `int64`. Snippet còn xử lý 2 bẫy nữa mà văn xuôi không cảnh báo: `chi` trả 405 body rỗng, và `encoding/json` không phân biệt `null` với field vắng mặt trong PATCH |
| **Reverse proxy** (nginx, Caddy, IIS — áp cho mọi template deploy sau proxy) | Proxy sinh lỗi HTML **trước khi request tới app**, nên writer trong app vô dụng: `client_max_body_size` mặc định 1MB → 413; `proxy_read_timeout` mặc định 60s → 504; upstream chết → 502; IIS app pool recycle → 502.3/503 | `error_page 413 502 503 504` trỏ về một location trả JSON envelope với `$request_id`; `add_header X-Request-Id $request_id always`; `client_max_body_size` khớp giới hạn upload app khai; `proxy_read_timeout` **lớn hơn** deadline server-side của app, và với SSE thì thêm `proxy_buffering off`. Đặt ở `infra/nginx/` (hoặc `infra/iis/`) của template |
| **go-vue-services** | mỗi module tự viết `adapter/http/` nên drift giữa các module gần như chắc chắn khi thêm module mới | Shared writer trong `internal/platform/` (bảng error code ở `internal/shared/`) **cộng** một integration test smoke duyệt mọi route đã mount, gọi một request lỗi cố ý và assert body có đủ `error.code` và `error.trace_id`. Cả hai là bắt buộc, không phải gợi ý |

---

## 5. Tích hợp phía FE — toàn bộ chi phí

```ts
export type FieldError = {
  field?: string
  in?: 'body' | 'query' | 'path' | 'header' | 'cookie' | 'form'
  code: string
  message: string
  user_message?: string
  params?: Record<string, unknown>
}

export class ApiError extends Error {
  constructor(
    readonly status: number, readonly code: string, message: string,
    readonly traceId: string, readonly retryable: boolean,
    readonly userMessage?: string, readonly details: FieldError[] = [],
  ) { super(message) }
}

// Một chỗ duy nhất dựng ApiError từ response lỗi — api(), download(), stream() dùng chung.
// Body lỗi KHÔNG phải lúc nào cũng là JSON: nginx/IIS sinh HTML 413/502/504 trước khi
// request tới app (section 4), nên .catch() ở đây là bắt buộc chứ không phải phòng xa.
async function apiErrorFrom(res: Response): Promise<ApiError> {
  const e = (res.headers.get('content-type')?.includes('json')
    ? (await res.json().catch(() => null))?.error : null) ?? {}
  return new ApiError(
    res.status, e.code ?? 'internal', e.message ?? res.statusText,
    e.trace_id ?? res.headers.get('x-request-id') ?? '',
    e.retryable ?? false, e.user_message, e.details ?? [],
  )
}

export async function api<T>(path: string, init: RequestInit = {}): Promise<T> {
  const res = await fetch(`${import.meta.env.VITE_API_BASE_URL}${path}`, {
    ...init, headers: { 'content-type': 'application/json', ...init.headers },
  })
  if (!res.ok) throw await apiErrorFrom(res)
  if (res.status === 204) return undefined as T
  return (res.headers.get('content-type')?.includes('json') ? await res.json() : null) as T
}
```

Không generic wrapper type, không `unwrap()`, không `res.data.data`. Đây là bài kiểm tra để biết chuẩn này có thật sự tối giản hay không.

Hai bổ sung khi dự án cần — dùng lại đúng `ApiError` ở trên, không đẻ lớp lỗi thứ hai:

```ts
// Download: PHẢI kiểm tra res.ok trước khi đọc blob, nếu không sẽ lưu file chứa JSON lỗi.
export async function download(path: string): Promise<Blob> {
  const res = await fetch(`${import.meta.env.VITE_API_BASE_URL}${path}`)
  if (!res.ok) throw await apiErrorFrom(res)
  return res.blob()
}

// SSE/NDJSON: fetch + ReadableStream, KHÔNG EventSource — EventSource không gắn được
// Authorization header và không đọc được body lỗi.
// finite=true (export, tail tới hết file): thiếu frame terminal là THẤT BẠI (section 1.2).
// finite=false (dashboard, ?follow=true): hết stream là bình thường, caller tự reconnect.
export async function* stream<T>(
  path: string, init: RequestInit = {}, finite = true,
): AsyncGenerator<T> {
  const res = await fetch(`${import.meta.env.VITE_API_BASE_URL}${path}`, init)
  if (!res.ok) throw await apiErrorFrom(res)
  const isSSE = res.headers.get('content-type')?.includes('text/event-stream')
  const reader = res.body!.pipeThrough(new TextDecoderStream()).getReader()
  let buf = '', terminated = false

  const handle = function* (raw: string): Generator<T> {
    // SSE: discriminator là dòng `event:` (section 1.2); data là các dòng `data:` nối
    // bằng \n. NDJSON: mỗi dòng một frame, discriminator là khóa `type` trong JSON.
    let ev = '', payload = ''
    if (isSSE) {
      const data: string[] = []
      for (const line of raw.split(/\r\n|[\r\n]/)) {
        if (line.startsWith('event:')) ev = line.slice(6).trim()
        else if (line.startsWith('data:')) data.push(line.slice(5).replace(/^ /, ''))
      }
      payload = data.join('\n')
    } else {
      payload = raw.trim()
    }
    if (!ev && !payload) return // comment `: ping` giữ kết nối, và block rỗng
    const frame = payload ? JSON.parse(payload) : {}
    const type = ev || frame.type
    if (type === 'error' || frame.error) {
      terminated = true
      const e = frame.error ?? {}
      throw new ApiError(res.status, e.code ?? 'internal', e.message ?? 'stream failed',
        e.trace_id ?? '', e.retryable ?? false, e.user_message)
    }
    if (type === 'done') { terminated = true; return }
    yield (frame.data ?? frame) as T
  }

  for (;;) {
    const { done, value } = await reader.read()
    if (done) break
    buf += value
    let parts: string[]
    if (isSSE) {
      // SSE kết block bằng dòng trống và cho phép CR, LF lẫn CRLF -> chuẩn hoá về \n.
      // Giữ lại \r ở cuối buffer: nó có thể là nửa đầu của một \r\n bị cắt ngang chunk.
      const cr = buf.endsWith('\r') ? '\r' : ''
      parts = (cr ? buf.slice(0, -1) : buf).replace(/\r\n|\r/g, '\n').split('\n\n')
      buf = (parts.pop() ?? '') + cr
    } else {
      parts = buf.split('\n')
      buf = parts.pop() ?? ''
    }
    for (const p of parts) { yield* handle(p); if (terminated) return }
  }
  if (buf.trim()) { yield* handle(buf); if (terminated) return }
  if (finite && !terminated) {
    throw new ApiError(res.status, 'internal', 'stream ended without terminal frame',
      res.headers.get('x-request-id') ?? '', true)
  }
}
```

`stream()` đọc discriminator ở **dòng `event:`** cho SSE, đúng như section 1.2 quy định (`event: item` / `event: done` / `event: error`), và vẫn chấp nhận `type` nằm trong JSON để tương thích NDJSON. Bản trước chỉ đọc `data:` nên một server SSE làm **đúng** chuẩn sẽ vừa bị yield thêm một item rác vừa bị báo "thiếu frame terminal".

Lưu ý về casing trong snippet: `FieldError` giữ `snake_case` vì nó là **hình dạng trên wire**, parse thẳng từ JSON không qua map. `ApiError` dùng `camelCase` vì nó là **class của client**, do FE tự dựng. Ranh giới nằm ở đúng hàm `api()` — đó là chỗ duy nhất trong FE biết tên field wire. Nếu thấy `traceId` xuất hiện ngoài class này, hoặc `trace_id` xuất hiện trong component, là đã rò ranh giới.

---

## 6. Đánh đổi đã chấp nhận

| Quyết định | Được | Mất |
|---|---|---|
| Success không có wrapper | 204, file, stream, redirect, webhook, healthz không còn là ngoại lệ; FE không có `res.data.data` | Không có chỗ đặt `meta` cấp response. Thêm sau là breaking, phải lên `/api/v2`. Giảm thiểu: header là kênh metadata chính thức (`X-Request-Id`, `Deprecation`, `Sunset`, `RateLimit-*`) |
| Hai hình dạng success (resource trần và `{items, page}`) | Tránh envelope cho mọi thứ | Luật "list luôn có wrapper" **không lint được**. Sẽ có người trả `{"orders": [...]}` vì "đọc hay hơn" — chỉ code review bắt được |
| Offset mặc định | Chạy được ngay ở mọi stack; admin table có số trang và `total` | Không an toàn với dataset ghi liên tục (insert giữa lúc lật trang gây lặp hoặc nhảy item); `total` là một `COUNT(*)` mỗi request |
| `code` dạng string, không có registry số | Grep được, đọc được khi debug, tự giải thích | Dễ đẻ code trùng nghĩa giữa các repo (`order.already_paid` vs `orders.already_paid`). Pack cấm shared package mặc định nên file khai báo bị copy tay — chưa có cơ chế rẻ nào chặn |
| `snake_case` trên wire | Một luật casing duy nhất cho field, error code, enum, query param; native với FastAPI/Pydantic; khớp tên cột DB | FE JS/TS thấy `created_at` lạc quẻ; .NET phải nhớ set `JsonNamingPolicy.SnakeCaseLower` |
| Money là object với `amount` string | Không mất chính xác qua JS float; amount không mồ côi khỏi currency | FE vẫn sẽ `parseFloat` để cộng giỏ hàng, tái tạo đúng bug mà string định phòng. Chỉ chặn được bằng helper trong `services/` + code review |
| `id` luôn là string | Đổi int sang uuid/ULID không breaking; tránh JS truncate int64 | Mỗi mapping DTO thêm một dòng cast; sort theo id ở FE sẽ sai nếu id là số tự tăng |
| i18n mặc định đẩy về FE | Không ép server dựng catalog dịch từ ngày đầu | App không có FE trong repo (đối tác, mobile) chỉ nhận `code` + `message` tiếng Anh cho tới khi bật `user_message` |
| Không bắt OpenAPI khi mọi client nằm trong repo (Tier 1 chỉ bắt khi API có consumer ngoài repo, bất kể mode thư mục) | Đúng core §1.7 "không mặc định contracts-first" | Không có contract kiểm tra được bằng máy; consistency phụ thuộc code review và một file helper |
| Không dùng RFC 9457 `application/problem+json` | Envelope ngắn hơn; không phải giải thích `type`/`instance` URI cho ai cả | Mất tương thích với tooling hiểu sẵn 7807; phải tắt `ProblemDetails` built-in của ASP.NET Core. Nếu cần phục vụ đối tác ngoài, viết ~10 dòng middleware map sang problem+json như một adapter optional — làm ở đúng một chỗ |
| Danh sách miễn trừ | Ngắn và có luật vị trí (`/api/v1`) thay vì luật niềm tin | Vẫn cần comment `// contract-exempt:` cho webhook. Người mới có thể đọc thành "chỗ nào cũng được phá luật" |

---

## 7. Các phương án đã cân nhắc và lý do loại

**Wrapper `{"data": ..., "meta": ..., "error": null}` cho mọi response** (JSend, kiểu Laravel). Loại. Nhân đôi kênh báo lỗi vốn đã có ở status code, tạo `res.data.data` ở mọi FE dùng axios, và quan trọng nhất là tạo ra một danh sách dài ngoại lệ (204 thì wrap gì? file download? SSE? webhook? `/healthz`?). Mỗi ngoại lệ là một dòng doc mà dev sẽ không đọc. Bỏ wrapper khiến những case đó **không còn là ngoại lệ**.

**RFC 9457 `application/problem+json`.** Loại, dù đây là phương án đúng chuẩn nhất. `{type, title, status, detail, instance}` bắt phải nghĩ ra URI cho `type` (mỗi repo tự bịa một domain, 20 repo cùng khai một chuỗi copy từ template — va thẳng vào quality gate "rule trung tính, không phụ thuộc hạ tầng riêng"), lặp `status` vào body, và không có chỗ chuẩn cho field-level error. Envelope ở đây chính là 7807 đã rút gọn: `code` thay `type` (string thay URI), `message` thay `detail`, bỏ `title`/`status`/`instance`.

**Mã lỗi dạng số kèm subcode** (kiểu Facebook: `code: 10`, `error_subcode: 2859017`). Loại. Số buộc phải có bảng tra cứu được maintain, không grep được trong codebase, không đọc được khi debug. `payment.card_declined` tự giải thích và namespace của nó đã đóng vai subcode. **Giữ lại từ Facebook:** tách dev-message với user-message, cờ transient (`is_transient` → `retryable`), và trace id cho support (`fbtrace_id` → `trace_id`).

**Cursor-first pagination.** Loại làm mặc định — đúng về kỹ thuật, sai về ngân sách: mọi template fullstack trong pack đều có `apps/admin-web/` cần số trang.

**Pagination bằng header `Link` hoặc `X-Total-Count`.** Loại: CORS chặn mặc định, không type được, không thấy trong devtools JSON view.

**camelCase trên wire.** Loại: sẽ có hai luật casing trong cùng một contract (`snake_case` cho error code và enum, `camelCase` cho field).

**ID kiểu số.** Loại: int64 lớn hơn 2^53 bị JS truncate im lặng.

---

## 8. Rủi ro đã biết

| Mức | Rủi ro | Giảm thiểu |
|---|---|---|
| Cao | **.NET drift im lặng** — `[ApiController]` tự sinh problem+json cho validation, `System.Text.Json` mặc định camelCase | Ship snippet `Program.cs` thật trong template, không viết văn xuôi. Xem section 4 |
| Cao | **FastAPI drift im lặng** — mặc định `{"detail": ...}` sai hình dạng ngay từ hộp | Ship 2 exception handler stub thật trong template |
| Cao | **One-way door của success trần** — không có chỗ đặt `meta`, thêm sau là breaking | Tuyên bố header là kênh metadata chính thức; nhớ `Access-Control-Expose-Headers` |
| Cao | **Không có shared package** (core §1.7 cấm mặc định) nên file error code bị copy tay giữa các repo cùng product, code phân kỳ chính tả | Chưa có cơ chế rẻ nào chặn. Chấp nhận và ghi rõ ở đây |
| Trung bình | Hai hình dạng success không lint được | Shared writer + smoke test duyệt route; với `go-vue-services` là bắt buộc |
| Trung bình | `total` bắt buộc bị vi phạm âm thầm trên bảng lớn (dev trả số cached thay vì di cư sang cursor) | Forcing function chỉ hoạt động nếu có người review — pack không mặc định CODEOWNERS |
| Trung bình | Danh sách miễn trừ dễ bị đọc thành "chỗ nào cũng được phá luật" | Luật prefix `/api/v1` biến nó thành luật vị trí thay vì luật niềm tin, grep một dòng là kiểm tra được |
| Thấp | Batch trả 200 kèm item lỗi trông giống anti-pattern số 1 | Ranh giới phát biểu gắt ở section 1.4 + bắt buộc khai báo endpoint non-atomic trong `docs/api-contract.md` |

---

## Metadata

- Contract version: `api-1`
- Pack version: `3.1`
- Owner: `https://github.com/TranTaiDakLak/`
- Maintainer: `Engineering / Architecture`

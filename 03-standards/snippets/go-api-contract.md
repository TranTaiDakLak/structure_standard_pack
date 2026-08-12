# Go API Contract Snippets

> Code Go tham chiếu để áp [`API_RESPONSE_CONTRACT.md`](../API_RESPONSE_CONTRACT.md) (`api-1`) trong một buổi thay vì 1–2 ngày.
> **Đây là snippet để copy, KHÔNG phải package chung để import.** Copy vào đúng folder mà template của stack chỉ định — tra bảng ở [appendix](../API_RESPONSE_CONTRACT_APPENDIX.md) section 3 (`response/`, `handler/response/`, `internal/platform/httpx/`, `internal/adapter/http/response/`, `model/errors.go`...). Đổi `package response` cho khớp tên folder đó.

Stdlib + [`chi/v5`](https://github.com/go-chi/chi) + [`validator/v10`](https://github.com/go-playground/validator) (chỉ section 7). Khác biệt khi dùng `gin` ghi ngay tại chỗ.

**Snippet trải theo 2 package, đúng như mọi template Go của pack yêu cầu.** Section 1 (`errors.go` — `Code`, `AppError`, `Retryable()`) đi vào `model/` hoặc `internal/domain/` để `service`/`usecase` rẽ nhánh theo `code` mà **không import `net/http`**; section 2–4, 6–7 đi vào package writer (`response/`, `handler/response/`, `internal/adapter/http/response/`, `internal/platform/httpx/`...).

Bảng section 6.1 của chuẩn tách 2 nửa theo đúng ranh giới đó: **`retryable` ở domain** (câu hỏi nghiệp vụ, worker và Wails bridge cần) và **`status` ở writer** (phép chiếu sang một transport cụ thể). Sau khi tách, file writer import package bảng mã và qualify: `model.AppError`, `model.Retryable`, `model.CodeInternal`, `model.MsgInternal`, `[]model.Detail`.

## 1. `errors.go` — bảng 17 mã reserved + `AppError`

Bảng section 6.1 phải là **pure function**, không phải chuỗi `if` trong handler. Domain code thêm vào **cùng bảng này**. Nửa `status` nằm ở package writer (section 2) — package này không import `net/http`.

```go
package model // hoặc domain — KHÔNG phải package HTTP (chuẩn section 6.1)

type Code string

const (
	CodeBadRequest           Code = "bad_request"
	CodeUnauthenticated      Code = "unauthenticated"
	CodePermissionDenied     Code = "permission_denied"
	CodeNotFound             Code = "not_found"
	CodeMethodNotAllowed     Code = "method_not_allowed"
	CodeConflict             Code = "conflict"
	CodePreconditionFailed   Code = "precondition_failed"
	CodePayloadTooLarge      Code = "payload_too_large"
	CodeUnsupportedMediaType Code = "unsupported_media_type"
	CodeValidationFailed     Code = "validation_failed"
	CodeRateLimited          Code = "rate_limited"
	CodeInternal             Code = "internal"
	CodeNotImplemented       Code = "not_implemented"
	CodeDependencyFailed     Code = "dependency_failed"
	CodeUnavailable          Code = "unavailable"
	CodeNotReady             Code = "not_ready"
	CodeTimeout              Code = "timeout"
)

// MsgInternal: message của 500 là chuỗi CỐ ĐỊNH (section 5.3). Không bao giờ err.Error().
const MsgInternal = "internal server error"

// Bảng CHỈ giữ retryable — "lỗi này có tạm thời không" là câu hỏi nghiệp vụ.
// Phép chiếu code -> HTTP status nằm ở package writer (section 2b), vì status là
// chi tiết của MỘT transport; worker và desktop bridge không dùng tới nó.
// Đây là lý do package này KHÔNG import net/http (chuẩn section 6.1).
var retryableTable = map[Code]bool{
	CodeBadRequest:           false,
	CodeUnauthenticated:      false,
	CodePermissionDenied:     false,
	CodeNotFound:             false,
	CodeMethodNotAllowed:     false,
	CodeConflict:             false,
	CodePreconditionFailed:   false,
	CodePayloadTooLarge:      false,
	CodeUnsupportedMediaType: false,
	CodeValidationFailed:     false,
	CodeRateLimited:          true,
	CodeInternal:             false,
	CodeNotImplemented:       false,
	CodeDependencyFailed:     true,
	CodeUnavailable:          true,
	CodeNotReady:             true,
	CodeTimeout:              true,

	// Domain code <module>.<reason> thêm vào ĐÂY, không để handler tự chọn status
	// (section 6.2). Luôn 4xx, không bao giờ 5xx:
	// Code("order.already_paid"):        false,
	// Code("wallet.insufficient_funds"): false,
}

// Retryable là nửa domain của pure function ở Definition of done — test 1 là
// table-driven, phủ đủ 17 mã reserved VÀ mọi domain code đã thêm bên trên.
// Code lạ rơi về false; kiểu `Code` ở trên là thứ chặn code lạ ngay lúc compile.
func Retryable(c Code) bool { return retryableTable[c] }

// Codes trả mọi code có trong bảng. Cần cho test 1 (section 8): sau khi tách package,
// retryableTable nằm ở đây còn statusTable nằm ở writer, cả hai đều unexported — không
// có accessor này thì assert "hai nửa cùng tập key" không viết được từ package nào cả.
func Codes() []Code {
	out := make([]Code, 0, len(retryableTable))
	for c := range retryableTable {
		out = append(out, c)
	}
	return out
}

type Detail struct {
	Field       string         `json:"field,omitempty"`
	In          string         `json:"in,omitempty"` // body | query | path | header | cookie | form
	Code        string         `json:"code"`
	Message     string         `json:"message"`
	UserMessage string         `json:"user_message,omitempty"`
	Params      map[string]any `json:"params,omitempty"`
}

type AppError struct {
	Code        Code
	Message     string // tiếng Anh, cho developer và log. Cấm SQL, path, connection string
	UserMessage string // chỉ set khi thật sự có bản dịch; rỗng thì omit, không gửi null
	Details     []Detail
	Retryable   *bool // non-nil = override bảng, chỉ cho lỗi transient (deadlock DB, conn reset)
	cause       error
}

func (e *AppError) Error() string { return string(e.Code) + ": " + e.Message }
func (e *AppError) Unwrap() error { return e.cause }

func New(c Code, msg string) *AppError { return &AppError{Code: c, Message: msg} }

func Wrap(c Code, msg string, cause error) *AppError {
	return &AppError{Code: c, Message: msg, cause: cause}
}
```

## 2. `write.go` — writer duy nhất của repo

Chữ ký **bắt buộc nhận `r *http.Request`**: `trace_id` nằm trong context của request, không có nguồn nào khác. Viết `WriteError(w, code, msg)` trước rồi thêm `trace_id` sau nghĩa là sửa lại **toàn bộ** call site — đây là chỗ tốn thời gian nhất khi áp chuẩn muộn.

```go
package response

import (
	"encoding/json"
	"errors"
	"log/slog"
	"net/http"

	"example.com/app/model" // đổi thành import path thật của repo
)

// Nửa transport của bảng section 6.1: phép chiếu code -> HTTP status.
// Nằm ở ĐÂY chứ không ở package model, vì status là chi tiết của một transport.
// Worker và Wails bridge dùng model.Retryable() mà không cần bảng này.
var statusTable = map[model.Code]int{
	model.CodeBadRequest:           400,
	model.CodeUnauthenticated:      401,
	model.CodePermissionDenied:     403,
	model.CodeNotFound:             404,
	model.CodeMethodNotAllowed:     405,
	model.CodeConflict:             409,
	model.CodePreconditionFailed:   412,
	model.CodePayloadTooLarge:      413,
	model.CodeUnsupportedMediaType: 415,
	model.CodeValidationFailed:     422,
	model.CodeRateLimited:          429,
	model.CodeInternal:             500,
	model.CodeNotImplemented:       501,
	model.CodeDependencyFailed:     502,
	model.CodeUnavailable:          503,
	model.CodeNotReady:             503,
	model.CodeTimeout:              504,

	// Domain code thêm vào ĐÂY (status) và vào retryableTable ở section 1 (retryable).
	// Luôn 4xx, không bao giờ 5xx (chuẩn section 6.2).
	// model.Code("order.already_paid"):        409,
	// model.Code("wallet.insufficient_funds"): 422,
}

// statusOf: code lạ rơi về 500 thay vì panic. Kiểu Code chặn code lạ ngay lúc compile.
func statusOf(c model.Code) int {
	if st, ok := statusTable[c]; ok {
		return st
	}
	return http.StatusInternalServerError
}

type Page struct {
	Limit  int   `json:"limit"` // echo giá trị THỰC TẾ sau clamp, không echo param client gửi
	Offset int   `json:"offset"`
	Total  int64 `json:"total"`
}

type errBody struct {
	Code        model.Code     `json:"code"`
	Message     string         `json:"message"`
	TraceID     string         `json:"trace_id"`
	Retryable   bool           `json:"retryable"`
	UserMessage string         `json:"user_message,omitempty"`
	Details     []model.Detail `json:"details,omitempty"`
}

func WriteJSON(w http.ResponseWriter, r *http.Request, status int, v any) {
	if status == http.StatusNoContent || v == nil {
		w.WriteHeader(status) // 204: body rỗng tuyệt đối, KHÔNG set Content-Type
		return
	}
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.WriteHeader(status)
	if err := json.NewEncoder(w).Encode(v); err != nil {
		slog.ErrorContext(r.Context(), "write response failed", "error", err)
	}
}

func WriteError(w http.ResponseWriter, r *http.Request, err error) {
	var ae *model.AppError
	if !errors.As(err, &ae) {
		ae = model.New(model.CodeInternal, model.MsgInternal) // error trần = lỗi chưa map, không đoán status
	}
	status, retryable := statusOf(ae.Code), model.Retryable(ae.Code)
	if ae.Retryable != nil {
		retryable = *ae.Retryable
	}
	body := errBody{ae.Code, ae.Message, RequestIDFrom(r.Context()), retryable, ae.UserMessage, ae.Details}
	if status >= 500 {
		slog.ErrorContext(r.Context(), "request failed", "code", ae.Code, "error", err,
			"trace_id", body.TraceID, "method", r.Method, "path", r.URL.Path)
	}
	if status == http.StatusInternalServerError {
		body.Message = model.MsgInternal // chi tiết chỉ nằm ở log, join bằng trace_id
	}
	WriteJSON(w, r, status, struct {
		Error errBody `json:"error"`
	}{body})
}

func WriteList[T any](w http.ResponseWriter, r *http.Request, items []T, p Page) {
	if items == nil {
		items = []T{} // array rỗng là [], không bao giờ null
	}
	WriteJSON(w, r, http.StatusOK, struct {
		Items []T  `json:"items"`
		Page  Page `json:"page"`
	}{items, p})
}
```

Rule:

- Cursor mode dùng `Page` khác (`limit`, `next_cursor`, `has_more`) nhưng **giữ nguyên container `{items, page}`** — FE table chỉ có một code path (section 8).
- Handler không bao giờ gọi `json.NewEncoder(w).Encode(...)` trực tiếp. Đây là luật grep được.

## 3. `middleware.go` — request id + recover

`X-Request-Id` set ở middleware nên phủ **mọi** response, kể cả 204, file download và panic. Không tin id client gửi mà chưa validate: nó đi thẳng vào log.

**File này nằm cùng package với writer, không nằm trong `handler/`.** `WriteError` gọi `RequestIDFrom`, nên đẩy `middleware.go` sang package `handler/` là tạo import cycle `handler → response → handler` — Go không build được. Dòng "exception/recover middleware toàn cục: `handler/`" trong template nói về **nơi đăng ký**: `handler/` mount `response.RequestID` và `response.Recover` vào router (section 4), không phải nơi khai chúng.

```go
package response

import (
	"context"
	"crypto/rand"
	"encoding/hex"
	"log/slog"
	"net/http"
	"regexp"
	"runtime/debug"

	"example.com/app/model" // đổi thành import path thật của repo
)

const HeaderRequestID = "X-Request-Id"

type ctxKey struct{}

func RequestIDFrom(ctx context.Context) string {
	id, _ := ctx.Value(ctxKey{}).(string)
	return id
}

var reqIDPattern = regexp.MustCompile(`^[A-Za-z0-9._-]{8,128}$`)

// chi có middleware.RequestID nhưng nó KHÔNG ghi header ra response và không validate
// input — dùng cái này thay thế, đừng mount cả hai.
func RequestID(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		id := r.Header.Get(HeaderRequestID)
		if !reqIDPattern.MatchString(id) {
			var b [16]byte
			_, _ = rand.Read(b[:])
			id = hex.EncodeToString(b[:]) // đổi sang ULID/UUIDv7 nếu repo đã có sẵn generator
		}
		w.Header().Set(HeaderRequestID, id)
		next.ServeHTTP(w, r.WithContext(context.WithValue(r.Context(), ctxKey{}, id)))
	})
}

func Recover(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			rec := recover()
			if rec == nil {
				return
			}
			if rec == http.ErrAbortHandler {
				panic(rec) // panic nội bộ của net/http, phải để nó bay tiếp
			}
			slog.ErrorContext(r.Context(), "panic recovered", "panic", rec,
				"stack", string(debug.Stack()), "trace_id", RequestIDFrom(r.Context()),
				"method", r.Method, "path", r.URL.Path)
			WriteError(w, r, model.New(model.CodeInternal, model.MsgInternal))
		}()
		next.ServeHTTP(w, r)
	})
}
```

## 4. Wiring router — bẫy 404/405 im lặng

Đo trên `chi` v5: route lạ trả `text/plain` `"404 page not found"`, sai method trả **405 body RỖNG hoàn toàn, không cả `Content-Type`**. `gin` trả `404 page not found` dạng `text/plain` cho cả sai route lẫn sai method. Đều vi phạm Tier 0 luật 2 mà không có lỗi build, không có test đỏ — chỉ FE gọi sai route mới lộ.

File này nằm ở package dựng router (`handler/`, `internal/app/`, `internal/adapter/http/`...), tức package thứ ba — nên import **cả hai**: `response` (writer) và package bảng mã (`model`/`domain`).

```go
r := chi.NewRouter()
r.Use(response.RequestID, response.Recover) // RequestID trước: Recover cần trace_id

r.NotFound(func(w http.ResponseWriter, req *http.Request) {
	response.WriteError(w, req, model.New(model.CodeNotFound,
		"route "+req.URL.Path+" does not exist"))
})
r.MethodNotAllowed(func(w http.ResponseWriter, req *http.Request) {
	// RFC 9110 bắt buộc header Allow trên 405. chi chỉ set hộ ở default handler —
	// đăng ký custom handler là mất luôn Allow. Set tay, đúng method API này có.
	w.Header().Set("Allow", "GET, POST, PUT, PATCH, DELETE")
	response.WriteError(w, req, model.New(model.CodeMethodNotAllowed,
		req.Method+" is not allowed on "+req.URL.Path))
})

r.Route("/api/v1", func(r chi.Router) { /* mount route nghiệp vụ */ })
r.Get("/healthz", healthHandler) // ngoài /api/v1 — miễn trừ theo luật vị trí
```

Với `gin`: `r.NoRoute(h)` + `r.NoMethod(h)`, và **phải bật `r.HandleMethodNotAllowed = true`** — mặc định là `false`, nên sai method rơi vào `NoRoute` và trả 404 thay vì 405.

## 5. DTO — 2 bẫy im lặng của Go

`time.Time` marshal ra `+07:00` khi process không chạy UTC (mặc định trên server VN), và `int64` marshal thành JSON number rồi bị JS truncate ở 2^53. Cả hai đều không có lỗi build.

```go
type OrderDTO struct {
	ID         string     `json:"id"`         // KHÔNG bao giờ đặt json:"id" trên một field int64
	CustomerID string     `json:"customer_id"`
	Note       *string    `json:"note"`       // field đã khai trong schema thì luôn có mặt, rỗng thì null
	IsPaid     bool       `json:"is_paid"`
	PaidAt     *time.Time `json:"paid_at"`
	CreatedAt  time.Time  `json:"created_at"`
}

func toOrderDTO(o domain.Order) OrderDTO {
	return OrderDTO{
		ID:         strconv.FormatInt(o.ID, 10),
		CustomerID: strconv.FormatInt(o.CustomerID, 10),
		Note:       o.Note,
		IsPaid:     o.IsPaid,
		PaidAt:     utcPtr(o.PaidAt),
		CreatedAt:  o.CreatedAt.UTC(), // thiếu .UTC() → "2026-08-12T10:14:07+07:00"
	}
}

func utcPtr(t *time.Time) *time.Time {
	if t == nil {
		return nil
	}
	u := t.UTC()
	return &u
}
```

Muốn không phụ thuộc trí nhớ của người viết mapper thì khai một kiểu tự ép UTC và dùng nó trong mọi DTO:

```go
type UTCTime time.Time

func (t UTCTime) MarshalJSON() ([]byte, error) {
	return []byte(`"` + time.Time(t).UTC().Format(time.RFC3339) + `"`), nil
}
```

`TZ=UTC` trong container là lớp phòng thủ thứ hai, không thay thế được `.UTC()` ở boundary — dev máy local vẫn chạy giờ VN.

## 6. PATCH — `Optional[T]` cho null-vs-absent

`encoding/json` **không** phân biệt `{"note": null}` với `note` vắng mặt: cả hai để `*string` là `nil`. Section 10 bắt buộc phân biệt (vắng = giữ nguyên, `null` = xóa). Không có kiểu này, mọi PATCH đều xóa sạch field client không gửi.

```go
type Optional[T any] struct {
	Defined bool // key CÓ mặt trong body
	Value   *T   // nil khi client gửi null
}

// UnmarshalJSON chỉ được gọi khi key có mặt — đó chính là tín hiệu cần bắt.
func (o *Optional[T]) UnmarshalJSON(b []byte) error {
	o.Defined = true
	if string(b) == "null" {
		o.Value = nil
		return nil
	}
	var v T
	if err := json.Unmarshal(b, &v); err != nil {
		return err
	}
	o.Value = &v
	return nil
}

type PatchOrderReq struct {
	Note     Optional[string] `json:"note"`
	Discount Optional[int64]  `json:"discount"`
}

// Trong usecase:
if req.Note.Defined {
	order.Note = req.Note.Value // nil = xóa giá trị, non-nil = set
}
```

Chỉ dùng cho request body. Field trong DTO response vẫn là `*T` bình thường.

## 7. Validation — `field` phải là tên trên wire

Mặc định `validator` xuất tên struct (`PostalCode`), section 9.1 bắt buộc tên đã serialize (`postal_code`). Sửa bằng đúng một `RegisterTagNameFunc`.

```go
var validate = newValidator()

func newValidator() *validator.Validate {
	v := validator.New(validator.WithRequiredStructEnabled())
	v.RegisterTagNameFunc(func(f reflect.StructField) string {
		name := strings.SplitN(f.Tag.Get("json"), ",", 2)[0]
		if name == "" || name == "-" {
			return f.Name
		}
		return name
	})
	return v
}

// tag của validator -> tập đóng details[].code ở section 5.4
var detailCodes = map[string]string{
	"required": "required", "email": "invalid_format", "uuid": "invalid_format",
	"oneof": "invalid_enum", "min": "min", "gte": "min", "max": "max", "lte": "max",
}

// Validate trả TẤT CẢ field sai trong một lần — validator không dừng ở field đầu tiên.
// Hàm này nằm ở package writer nên vẫn qualify `model.` như section 2.
func Validate(v any) *model.AppError {
	err := validate.Struct(v)
	if err == nil {
		return nil
	}
	var ve validator.ValidationErrors
	if !errors.As(err, &ve) {
		return model.Wrap(model.CodeInternal, model.MsgInternal, err) // sai cách gọi validator, không phải lỗi input
	}
	details := make([]model.Detail, 0, len(ve))
	for _, fe := range ve {
		// Namespace đã dùng json tag nhờ RegisterTagNameFunc:
		// "CreateOrderReq.billing.postal_code" -> "billing.postal_code"
		field := fe.Namespace()
		if i := strings.IndexByte(field, '.'); i >= 0 {
			field = field[i+1:]
		}
		code, ok := detailCodes[fe.Tag()]
		if !ok {
			code = "invalid_format"
		}
		d := model.Detail{Field: field, In: "body", Code: code, Message: fe.Tag() + " constraint failed"}
		if fe.Param() != "" {
			d.Params = map[string]any{fe.Tag(): fe.Param()} // FE nội suy: "Tối thiểu {min}"
		}
		details = append(details, d)
	}
	return &model.AppError{
		Code:    model.CodeValidationFailed,
		Message: fmt.Sprintf("request body failed validation: %d field(s)", len(details)),
		Details: details,
	}
}
```

Rule:

- Lỗi param phân trang / sort / filter là `bad_request` (400), **không** đi qua hàm này (section 8.2).
- Lỗi cross-field: omit `Field`, không dùng `""` hay `"_form"`.

## 8. Còn thiếu gì

Copy hết 7 khối trên vẫn chưa đủ Definition of done (section 13). Còn lại đúng 3 test bắt buộc của DoD, không đổi tên không đổi số:

- **Test 1** — table-driven cho `Retryable()` và `statusOf()`, phủ 17 mã reserved **và mọi domain code**. Thêm một assert rằng hai nửa bảng cùng một tập key — lệch nhau là cách bảng này hỏng. Test này đặt **trong package writer** (nó cần `statusTable` vốn unexported) và duyệt tập key qua `model.Codes()`:

  ```go
  for _, c := range model.Codes() {
  	if _, ok := statusTable[c]; !ok {
  		t.Errorf("code %q có trong retryableTable nhưng thiếu ở statusTable", c)
  	}
  }
  // và chiều ngược lại: mọi key của statusTable phải có trong model.Codes()
  ```
- **Test 2** — smoke test duyệt **mọi route đã mount**: một request lỗi cố ý assert body có `error.code` + `error.trace_id`; một request list assert body đúng 2 khóa `items` + `page`. Đây là test **duy nhất** chạm nhánh success — bỏ nó thì DTO sai casing hoặc thiếu `.UTC()` (section 5) không có gì bắt được. Đừng thay bằng test grep: grep chỉ soi chuỗi code, `type Code string` ở trên đã chặn code lạ từ lúc compile rồi.
- **Test 3** — e2e cố ý panic, assert body không chứa chuỗi nào của stack trace.

Cộng `docs/api-contract.md` và override 413/502/504 ở reverse proxy.

## Metadata

- Contract version: `api-1`
- Pack version: `3.1`
- Owner: `https://github.com/TranTaiDakLak/`
- Maintainer: `Engineering / Architecture`

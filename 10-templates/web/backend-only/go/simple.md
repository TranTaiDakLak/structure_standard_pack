# Go Backend Simple

## Khi nào dùng

- team 1–3 người
- API Go nhỏ hoặc vừa
- chưa cần tách lớp sâu

## Cây thư mục

```text
<service-name>/
├── docs/                 # tài liệu, flow, ghi chú kỹ thuật
├── infra/                # docker, nginx, deployment config
├── scripts/              # script build/run/migrate
├── config/               # file cấu hình mẫu (không chứa secret thật)
├── tests/                # test nằm ngoài source chính
├── handler/              # router + HTTP handler (Gin/Echo/net-http)
├── service/              # business logic (use case)
├── repository/           # truy cập DB / data source
├── model/                # struct dùng chung (entity, DTO), errors.go (Code, AppError, nửa retryable)
├── response/             # envelope + response writer HTTP + nửa status của bảng mã
├── main.go               # entrypoint (được phép ở root cho simple)
├── go.mod                # khai báo module + version
├── go.sum                # checksum dependency
├── README.md             # hướng dẫn repo
└── .gitignore            # bỏ qua bin/, tmp/, vendor/...
```

## Vai trò thư mục

- `handler/`: router, HTTP handlers
- `service/`: business logic
- `repository/`: DB access
- `model/`: structs dùng chung; `errors.go` giữ `Code`, `AppError`, và nửa `retryable` để `service/` rẽ nhánh mà không import `response/`
- `response/`: envelope + response writer HTTP dùng chung cho mọi handler, kèm nửa `status` của bảng mã
- `config/`: config mẫu hoặc loading đơn giản

## Rule

- `main.go` được phép ở root
- business logic không nhét vào `main.go`
- nếu `service/` quá to, cân nhắc nâng lên structured
- Config mẫu không chứa secret; runtime config nạp từ env/file theo deploy target.
- Nếu có DB, cần `migrations/` hoặc script migrate rõ.
- API production nên có `/healthz`, `/readyz`, structured log và graceful shutdown.

## API response contract

Theo [`03-standards/API_RESPONSE_CONTRACT.md`](../../../../03-standards/API_RESPONSE_CONTRACT.md), contract `api-1` — không tự chế hình dạng response, không tự chế bảng mã lỗi.

- Response helper + writer: `response/` — chỉ `envelope.go` + `write.go`, ngang cấp `model/`. Không khai mã lỗi ở đây; chỉ giữ nửa `status` của bảng mã, cùng package với writer.
- File khai báo error code + typed error, đúng 1 file cho cả repo: `model/errors.go` — `Code`, `AppError`, và nửa `retryable`, để `service/` rẽ nhánh theo `code` mà không import ngược `response/` (business logic không import HTTP layer). Tách 2 nửa theo chuẩn section 6.1; mỗi domain code phải có entry ở cả hai.
- Exception/recover middleware toàn cục: `handler/`
- Go không có handler mặc định nào — mọi handler bắt buộc ghi response qua helper trong `response/`; cấm `json.Marshal` thẳng ra `http.ResponseWriter`.
- Vai consumer: mỗi upstream không tuân `api-1` một file adapter riêng trong `repository/`; status code + header `Retry-After` là nguồn sự thật để retry, cấm duck-type `body.error` — [appendix section 2.5](../../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md).
- `docs/api-contract.md`: contract version, bảng domain error code, danh sách endpoint ngoại lệ.
- Code Go tham chiếu (bảng 17 mã, writer, middleware, wiring 404/405 của `chi`, bẫy DTO, PATCH null-vs-absent): [`03-standards/snippets/go-api-contract.md`](../../../../03-standards/snippets/go-api-contract.md) — section 1 (`Code`, `AppError`, `Retryable()`) copy vào `model/errors.go`, section 2–3 (writer + `statusTable` + request id/recover) copy vào `response/`, đổi `package` cho khớp folder; không import chung.

Không đẻ layer mới cho việc này — mọi path trên đều nằm trong cây thư mục ở trên.

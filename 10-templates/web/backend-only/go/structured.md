# Go Backend Structured

## Khi nào dùng

- team 3–5 người
- service sống lâu hơn
- cần rõ entrypoint, business, integration

## Cây thư mục

```text
<service-name>/
├── docs/                           # tài liệu, flow, ghi chú kỹ thuật
├── infra/                          # docker, nginx, deployment config
├── scripts/                        # script build/run/migrate
├── config/                         # file cấu hình mẫu
├── tests/                          # test nằm ngoài source chính
│   ├── integration/                # test tích hợp (có DB, HTTP thật)
│   └── e2e/                        # end-to-end qua API công khai
├── cmd/                            # mỗi binary 1 sub-folder
│   └── <service-name>/             # tên binary = tên folder
│       └── main.go                 # entrypoint — chỉ wiring
├── internal/                       # code private, không export ngoài module
│   ├── app/                        # bootstrap wiring (DI, router)
│   ├── domain/                     # entity, rule cốt lõi, errors.go (Code, AppError, nửa retryable) — không biết HTTP/DB
│   ├── usecase/                    # application flow, orchestrate domain
│   ├── adapter/                    # cổng ra ngoài (hexagonal)
│   │   ├── http/                   # handler, middleware HTTP
│   │   │   └── response/           # envelope + response writer HTTP + nửa status của bảng mã
│   │   ├── repository/             # SQL/NoSQL implement cho domain
│   │   └── external/               # client gọi service ngoài
│   └── config/                     # load config runtime
├── migrations/                     # SQL migrations (golang-migrate/Goose)
├── go.mod                          # khai báo module + version
├── go.sum                          # checksum dependency
├── README.md                       # hướng dẫn repo
└── .gitignore
```

## Vai trò thư mục

- `cmd/`: entrypoint
- `internal/domain/`: rules cốt lõi; `errors.go` giữ `Code`, `AppError`, và nửa `retryable` của bảng mã
- `internal/usecase/`: application flow
- `internal/adapter/`: http, DB, integrations — `http/response/` giữ envelope + writer + nửa `status` của bảng mã, `external/` giữ client gọi API ngoài
- `migrations/`: SQL migrations

## Rule

- `cmd/*/main.go` chỉ wiring
- domain không biết DB/HTTP framework
- đừng tạo package con quá vụn
- `usecase` đi qua interface cho persistence/external dependency; adapter implement.
- Unit test domain/usecase; integration test repository/HTTP quan trọng.
- API production nên có health/readiness, migration script, structured log, graceful shutdown.
- Nếu `internal/` chỉ có vài file mỏng, dùng lại `simple.md`.

## API response contract

Theo [`03-standards/API_RESPONSE_CONTRACT.md`](../../../../03-standards/API_RESPONSE_CONTRACT.md), contract `api-1` — không tự chế hình dạng response, không tự chế bảng mã lỗi.

- Response helper + writer: `internal/adapter/http/response/` — chỉ `envelope.go` + `write.go`. Không khai mã lỗi ở đây; chỉ giữ nửa `status` của bảng mã, cùng package với writer.
- File khai báo error code + typed error, đúng 1 file cho cả repo: `internal/domain/errors.go` — `Code`, `AppError`, và nửa `retryable`, để `usecase/` rẽ nhánh theo `code` mà không import ngược adapter.
- Bảng mã tách 2 nửa theo chuẩn section 6.1: `retryable(code) bool` ở `internal/domain/errors.go`, `status(code) int` ở `internal/adapter/http/response/` cùng writer. Cả 2 nửa là pure function, table-driven test được; mỗi domain code phải có entry ở **cả hai**.
- Exception/recover middleware toàn cục: `internal/adapter/http/`
- Go không có handler mặc định nào — mọi handler bắt buộc ghi response qua helper trong `internal/adapter/http/response/`; cấm `json.Marshal` thẳng ra `http.ResponseWriter`.
- Vai consumer: mỗi upstream không tuân `api-1` một file adapter riêng trong `internal/adapter/external/`; status code + header `Retry-After` là nguồn sự thật để retry, cấm duck-type `body.error` — [appendix section 2.5](../../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md).
- `docs/api-contract.md`: contract version, bảng domain error code, danh sách endpoint ngoại lệ.
- Code Go tham chiếu (bảng 17 mã, writer, middleware, wiring 404/405 của `chi`, bẫy DTO, PATCH null-vs-absent): [`03-standards/snippets/go-api-contract.md`](../../../../03-standards/snippets/go-api-contract.md) — section 1 (`Code`, `AppError`, `Retryable()`) copy vào `internal/domain/errors.go`, section 2–3 (writer + `statusTable` + request id/recover) copy vào `internal/adapter/http/response/`, đổi `package` cho khớp folder; không import chung.

`internal/domain/` và `internal/usecase/` không biết HTTP status, không sinh `user_message`; chỉ adapter map typed error sang status + code.

Không đẻ layer mới cho việc này — mọi path trên đều nằm trong cây thư mục ở trên.

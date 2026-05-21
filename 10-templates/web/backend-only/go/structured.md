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
│   ├── domain/                     # entity + rule cốt lõi, không biết HTTP/DB
│   ├── usecase/                    # application flow, orchestrate domain
│   ├── adapter/                    # cổng ra ngoài (hexagonal)
│   │   ├── http/                   # handler, middleware HTTP
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
- `internal/domain/`: rules cốt lõi
- `internal/usecase/`: application flow
- `internal/adapter/`: http, DB, integrations
- `migrations/`: SQL migrations

## Rule

- `cmd/*/main.go` chỉ wiring
- domain không biết DB/HTTP framework
- đừng tạo package con quá vụn
- `usecase` đi qua interface cho persistence/external dependency; adapter implement.
- Unit test domain/usecase; integration test repository/HTTP quan trọng.
- API production nên có health/readiness, migration script, structured log, graceful shutdown.
- Nếu `internal/` chỉ có vài file mỏng, dùng lại `simple.md`.

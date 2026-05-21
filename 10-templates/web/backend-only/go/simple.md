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
├── model/                # struct dùng chung (entity, DTO)
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
- `model/`: structs dùng chung
- `config/`: config mẫu hoặc loading đơn giản

## Rule

- `main.go` được phép ở root
- business logic không nhét vào `main.go`
- nếu `service/` quá to, cân nhắc nâng lên structured
- Config mẫu không chứa secret; runtime config nạp từ env/file theo deploy target.
- Nếu có DB, cần `migrations/` hoặc script migrate rõ.
- API production nên có `/healthz`, `/readyz`, structured log và graceful shutdown.

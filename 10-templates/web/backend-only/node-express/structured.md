# Node.js + Express Backend Structured

## Khi nào dùng

- repo Node đã bắt đầu nhiều module
- cần boundary rõ hơn giữa platform và modules

## Cây thư mục

```text
<service-name>/
├── docs/                       # tài liệu, flow, ghi chú kỹ thuật
├── infra/                      # docker, nginx, deployment config
├── scripts/                    # script build/run/migrate
├── config/                     # .env.example, file cấu hình mẫu
├── tests/                      # test nằm ngoài source chính
│   ├── integration/            # test tích hợp (DB thật, HTTP thật)
│   └── e2e/                    # end-to-end qua API công khai
├── src/                        # vùng source chính (khuyến nghị dùng TypeScript)
│   ├── app/                    # bootstrap wiring (DI container, router mount)
│   ├── modules/                # feature/domain module — độc lập nhau
│   │   ├── auth/               # ví dụ module auth (controller + service + repo)
│   │   ├── users/              # ví dụ module users
│   │   └── billing/            # ví dụ module billing
│   ├── platform/               # hạ tầng dùng chung
│   │   ├── db/                 # connection pool, migration bootstrap
│   │   ├── http/               # Express app factory, middleware chung
│   │   └── cache/              # Redis client, cache wrapper
│   ├── shared/                 # helper dùng chung — rất tiết chế
│   └── server.js               # entrypoint — chỉ boot (hoặc server.ts)
├── package.json                # dependency + script
├── README.md                   # hướng dẫn repo
└── .gitignore
```

## Vai trò thư mục

- `modules/`: feature/domain modules
- `platform/`: DB, HTTP server, cache, infra concerns
- `shared/`: shared helpers rất tiết chế
- `app/`: bootstrap wiring

## Rule

- module không phụ thuộc lung tung vào module khác
- `shared/` không thành bãi rác
- structured là để dễ tìm code, không phải để thêm ceremony
- Module nên expose boundary rõ, không import sâu vào file nội bộ của module khác.
- API production nên có healthcheck, request id, validation, error middleware và secret loading rõ.
- Unit test module/service; integration test route/DB quan trọng.
- Nếu chỉ có vài route nhỏ, quay lại `simple.md`.

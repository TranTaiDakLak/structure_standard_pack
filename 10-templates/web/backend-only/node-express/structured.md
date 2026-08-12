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

## API response contract

Theo [`03-standards/API_RESPONSE_CONTRACT.md`](../../../../03-standards/API_RESPONSE_CONTRACT.md), contract `api-1` — không tự chế hình dạng response, không tự chế bảng mã lỗi.

- Response helper + writer: `src/platform/http/` — response writer + class `ApiError`. Module trong `src/modules/` chỉ throw `ApiError`, cấm tự viết writer riêng.
- File khai báo error code, đúng 1 file cho cả repo: `src/platform/http/error-codes.ts`
- Exception/recover middleware toàn cục: `src/platform/http/` — error handler + catch-all 404, mount trong `src/app/`
- Error middleware phải có đủ 4 tham số `(err, req, res, next)` và đăng ký CUỐI CÙNG sau mọi route; catch-all 404 cũng phải trả JSON envelope, nếu không Express trả trang HTML mặc định.
- **Express 4 KHÔNG đưa lỗi của handler `async` vào error middleware.** Handler `async` throw (hoặc promise reject) trên Express 4 thành `unhandledRejection` — middleware không bao giờ chạy, client không nhận envelope, và Node mặc định **giết luôn process**. Đo được: cùng một đoạn code trên Express 5 trả đúng 500 envelope, trên Express 4.22 làm process exit. Chọn một: dùng Express 5, hoặc `import 'express-async-errors'`, hoặc bọc mọi handler async bằng `catch(next)`. Ghi rõ lựa chọn trong README của project.
- `docs/api-contract.md`: contract version, bảng domain error code, danh sách endpoint ngoại lệ.

Không đẻ layer mới cho việc này — mọi path trên đều nằm trong cây thư mục ở trên.

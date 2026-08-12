# Node.js + Express Backend Simple

## Khi nào dùng

- API nhỏ-vừa
- team 1–3 người
- cần setup nhanh

## Cây thư mục

```text
<service-name>/
├── docs/                    # tài liệu, flow, ghi chú kỹ thuật
├── infra/                   # docker, nginx, deployment config
├── scripts/                 # script build/run/migrate
├── config/                  # .env.example, file cấu hình mẫu
├── tests/                   # test nằm ngoài source chính
├── src/                     # vùng source chính (có thể dùng TS thay cho JS)
│   ├── routes/              # khai báo route, map URL → controller
│   ├── controllers/         # nhận request, validate, gọi service
│   ├── services/            # business logic
│   ├── repositories/        # truy cập DB (Prisma/Knex/Mongoose)
│   ├── models/              # schema/entity dùng chung
│   ├── middlewares/         # auth, error handler, logging
│   ├── http/                # response writer, ApiError, error code
│   └── app.js               # build Express app (chưa listen)
├── server.js                # entrypoint — chỉ boot (app.listen)
├── package.json             # dependency + script
├── README.md                # hướng dẫn repo
└── .gitignore               # bỏ qua node_modules/, dist/, .env
```

## Vai trò thư mục

- `routes/`: route map
- `controllers/`: request handling
- `services/`: business logic
- `repositories/`: DB access
- `middlewares/`: auth, error, logging wrappers
- `http/`: response writer, class `ApiError`, bảng error code dùng chung

## Rule

- `server.js` chỉ boot app
- controller không xử lý business logic dài
- service không biết chi tiết response HTTP
- Config mẫu dùng `.env.example`; `.env` thật phải gitignore.
- API production nên có healthcheck, request id, error middleware, validation và graceful shutdown.
- Nếu có DB, migration/seed script phải nằm trong `scripts/` hoặc package script rõ.

## API response contract

Theo [`03-standards/API_RESPONSE_CONTRACT.md`](../../../../03-standards/API_RESPONSE_CONTRACT.md), contract `api-1` — không tự chế hình dạng response, không tự chế bảng mã lỗi.

- Response helper + writer: `src/http/` — response writer + class `ApiError`
- File khai báo error code, đúng 1 file cho cả repo: `src/http/error-codes.js` (`.ts` nếu repo dùng TypeScript)
- Exception/recover middleware toàn cục: `src/middlewares/` — error handler + catch-all 404
- Error middleware phải có đủ 4 tham số `(err, req, res, next)` và đăng ký CUỐI CÙNG sau mọi route; catch-all 404 cũng phải trả JSON envelope, nếu không Express trả trang HTML mặc định.
- **Express 4 KHÔNG đưa lỗi của handler `async` vào error middleware.** Handler `async` throw (hoặc promise reject) trên Express 4 thành `unhandledRejection` — middleware không bao giờ chạy, client không nhận envelope, và Node mặc định **giết luôn process**. Đo được: cùng một đoạn code trên Express 5 trả đúng 500 envelope, trên Express 4.22 làm process exit. Chọn một: dùng Express 5, hoặc `import 'express-async-errors'`, hoặc bọc mọi handler async bằng `catch(next)`. Ghi rõ lựa chọn trong README của project.
- `docs/api-contract.md`: contract version, bảng domain error code, danh sách endpoint ngoại lệ.

Không đẻ layer mới cho việc này — mọi path trên đều nằm trong cây thư mục ở trên.

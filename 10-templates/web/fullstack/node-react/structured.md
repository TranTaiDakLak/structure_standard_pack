# Fullstack Structured — Node.js + React

## Khi nào dùng

- backend và frontend đều bắt đầu nhiều modules
- cần boundary rõ hơn

## Cây thư mục

```text
<product-name>/
├── docs/                       # tài liệu chung của product
├── infra/                      # deployment config chung
├── scripts/                    # script dev/build/deploy chung
├── config/                     # file cấu hình mẫu
├── backend/                    # Express API — như node-express/structured
│   ├── src/                    # source BE
│   │   ├── app/                # bootstrap wiring
│   │   ├── modules/            # feature/domain module độc lập
│   │   ├── platform/           # DB, HTTP, cache — hạ tầng dùng chung
│   │   ├── shared/             # helper dùng chung — rất tiết chế
│   │   └── server.js           # entrypoint BE
│   ├── tests/                  # test BE
│   └── package.json            # dependency BE
├── frontend/                   # React app — như react-vite/structured
│   ├── public/                 # tài nguyên tĩnh
│   ├── src/                    # source FE
│   │   ├── components/         # component generic
│   │   ├── features/           # logic theo domain
│   │   ├── hooks/              # hook dùng chung
│   │   ├── pages/              # page mỏng
│   │   ├── routes/             # khai báo route
│   │   ├── services/           # API client gọi BE
│   │   ├── types/              # TypeScript type
│   │   ├── App.tsx             # root component
│   │   └── main.tsx            # entrypoint
│   ├── tests/                  # test FE
│   └── package.json            # dependency FE
├── README.md                   # hướng dẫn repo
└── .gitignore
```

## Vai trò thư mục

- backend structured theo modules/platform
- frontend structured theo features/hooks/services

## Rule

- structured chỉ có ý nghĩa nếu team thật sự bắt đầu đau vì simple
- không thêm package/shared layer khi chưa cần
- Backend và frontend giữ test/build pipeline riêng; root script chỉ orchestration.
- API contract ổn định nên có OpenAPI hoặc tài liệu tương đương, không bắt buộc codegen.
- Production cần healthcheck, logging, validation, migration, secret loading và deploy target rõ.
- Nếu workspace/shared package chưa có nhu cầu thật, không thêm layer chung.

## API response contract

Theo [`03-standards/API_RESPONSE_CONTRACT.md`](../../../../03-standards/API_RESPONSE_CONTRACT.md), contract `api-1` — không tự chế hình dạng response, không tự chế bảng mã lỗi.

- Response helper + writer: type envelope ở `backend/src/shared/`, writer thật ở `backend/src/platform/http/` — mọi module trong `backend/src/modules/` dùng chung một writer, cấm mỗi module tự viết
- File khai báo error code, đúng 1 file cho cả repo: `backend/src/shared/error-codes.ts` — domain code `<module>.<reason>` với `<module>` khớp đúng tên folder trong `backend/src/modules/`
- Exception/recover middleware toàn cục: `backend/src/platform/http/`, wiring trong `backend/src/app/` — module trả typed error, chỉ tầng HTTP mới map sang status
- Express để nguyên sẽ trả trang lỗi HTML cho unhandled error và route lạ — thứ tự đăng ký trong `backend/src/app/` là bắt buộc, error middleware phải 4 tham số và nằm CUỐI CÙNG:

  ```js
  app.use(routes);
  app.use((req, res) => notFound(res));                 // catch-all 404, vẫn trả JSON envelope
  app.use((err, req, res, next) => onError(err, res));  // 4 tham số, đăng ký cuối cùng
  ```

  Trên **Express 4** khối này vẫn để lọt lỗi của handler `async`: throw trong handler async thành `unhandledRejection`, error middleware không chạy và Node mặc định giết process. Đo được: cùng code này Express 5 trả 500 envelope, Express 4.22 làm process exit. Dùng Express 5, hoặc `express-async-errors`, hoặc bọc handler bằng `catch(next)`.

- FE React: api client trong `frontend/src/services/`, `frontend/src/types/api.ts` mirror union `ErrorCode` dùng chung đúng tên code với backend — cùng repo thì đây là lợi thế duy nhất, đừng bỏ
- `docs/api-contract.md`: contract version, bảng domain error code, danh sách endpoint ngoại lệ.

Không đẻ layer mới cho việc này — mọi path trên đều nằm trong cây thư mục ở trên.

# Fullstack Simple — Node.js + React

## Khi nào dùng

- team nhỏ
- 1 frontend + 1 backend
- ship nhanh

## Cây thư mục

```text
<product-name>/
├── docs/                       # tài liệu chung của product
├── infra/                      # deployment config chung
├── scripts/                    # script dev/build/deploy chung
├── config/                     # file cấu hình mẫu
├── backend/                    # Express API — như node-express/simple
│   ├── src/                    # source BE (khuyến nghị TypeScript)
│   │   ├── routes/             # khai báo route
│   │   ├── controllers/        # nhận request, gọi service
│   │   ├── services/           # business logic
│   │   ├── repositories/       # truy cập DB
│   │   ├── models/             # schema/entity
│   │   ├── middlewares/        # auth, error handler, logging
│   │   ├── http/               # response writer, ApiError, error code
│   │   └── app.js              # build Express app (chưa listen)
│   ├── server.js               # entrypoint BE (app.listen)
│   └── package.json            # dependency BE
├── frontend/                   # React app — như react-vite/simple
│   ├── public/                 # tài nguyên tĩnh
│   ├── src/                    # source FE
│   │   ├── assets/             # ảnh/font/css
│   │   ├── components/         # component UI dùng lại
│   │   ├── pages/              # page screen
│   │   ├── routes/             # khai báo route (react-router)
│   │   ├── services/           # API client gọi BE
│   │   ├── App.tsx             # root component
│   │   └── main.tsx            # entrypoint
│   ├── vite.config.ts          # cấu hình Vite
│   └── package.json            # dependency FE
├── README.md                   # hướng dẫn repo
└── .gitignore
```

## Vai trò thư mục

- `backend/`: Express API
- `frontend/`: React app

## Rule

- chưa cần workspace tool
- vẫn giữ fullstack simple nếu code còn gọn
- Backend có `.env.example`, healthcheck, error middleware, validation và migration script nếu có DB.
- Frontend dùng env cho API base URL, không hardcode endpoint production.
- Root README phải có lệnh run/build/test cho cả backend và frontend.
- Nếu backend hoặc frontend phình trước, nâng riêng phần đó sang structured.

## API response contract

Theo [`03-standards/API_RESPONSE_CONTRACT.md`](../../../../03-standards/API_RESPONSE_CONTRACT.md), contract `api-1` — không tự chế hình dạng response, không tự chế bảng mã lỗi.

- Response helper + writer: `backend/src/http/`, cạnh `models/` — envelope lỗi, `{items, page}`, class `ApiError`; mọi controller đi qua đây, cấm tự `res.json()` shape riêng
- File khai báo error code, đúng 1 file cho cả repo: `backend/src/http/error-codes.js` (`.ts` nếu repo dùng TypeScript)
- Exception/recover middleware toàn cục: error middleware 4 tham số đặt trong folder middleware của backend (`backend/src/middlewares/`, kế thừa từ node-express/simple), đăng ký trong `backend/src/app.js`
- Express để nguyên sẽ trả trang lỗi HTML cho unhandled error và route lạ — thứ tự đăng ký trong `backend/src/app.js` là bắt buộc, error middleware phải 4 tham số và nằm CUỐI CÙNG:

  ```js
  app.use(routes);
  app.use((req, res) => notFound(res));                 // catch-all 404, vẫn trả JSON envelope
  app.use((err, req, res, next) => onError(err, res));  // 4 tham số, đăng ký cuối cùng
  ```

  Trên **Express 4** khối này vẫn để lọt lỗi của handler `async`: throw trong handler async thành `unhandledRejection`, error middleware không chạy và Node mặc định giết process. Đo được: cùng code này Express 5 trả 500 envelope, Express 4.22 làm process exit. Dùng Express 5, hoặc `express-async-errors`, hoặc bọc handler bằng `catch(next)`.

- FE React: api client trong `frontend/src/services/`, thêm `frontend/src/services/types.ts` mirror union `ErrorCode` dùng chung đúng tên code với backend — cùng repo thì đây là lợi thế duy nhất, đừng bỏ
- `docs/api-contract.md`: contract version, bảng domain error code, danh sách endpoint ngoại lệ.

Không đẻ layer mới cho việc này — mọi path trên đều nằm trong cây thư mục ở trên.

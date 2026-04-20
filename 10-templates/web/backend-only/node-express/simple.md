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

## Rule

- `server.js` chỉ boot app
- controller không xử lý business logic dài
- service không biết chi tiết response HTTP

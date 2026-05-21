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

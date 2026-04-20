# Fullstack Simple — Go + Vue

## Khi nào dùng

- team 1–3 người
- 1 frontend + 1 backend
- cần ship nhanh

## Cây thư mục

```text
<product-name>/
├── docs/                       # tài liệu chung của product
├── infra/                      # deployment config dùng chung (BE + FE)
│   ├── docker/                 # Dockerfile, docker-compose
│   ├── iis/                    # cấu hình IIS (nếu host Windows)
│   └── nginx/                  # reverse proxy cho BE + FE
├── scripts/                    # script dev/build/deploy dùng chung
├── config/                     # file cấu hình mẫu cho cả product
├── backend/                    # Go API — cấu trúc giống go/simple
│   ├── handler/                # router + HTTP handler
│   ├── service/                # business logic
│   ├── repository/             # truy cập DB
│   ├── model/                  # struct dùng chung
│   ├── main.go                 # entrypoint
│   ├── go.mod                  # khai báo module
│   └── go.sum                  # checksum dependency
├── frontend/                   # Vue app — cấu trúc giống vue-vite/simple
│   ├── public/                 # tài nguyên tĩnh
│   ├── src/                    # source Vue
│   │   ├── assets/             # ảnh/font/css
│   │   ├── components/         # component UI dùng lại
│   │   ├── pages/              # page screen
│   │   ├── router/             # vue-router
│   │   ├── stores/             # Pinia store
│   │   ├── api/                # wrapper gọi backend Go
│   │   ├── App.vue             # root component
│   │   └── main.ts             # entrypoint
│   ├── vite.config.ts          # cấu hình Vite
│   └── package.json            # dependency FE
├── README.md                   # hướng dẫn repo
└── .gitignore
```

## Vai trò thư mục

- `backend/`: Go API
- `frontend/`: Vue app
- `infra/`: docker / reverse proxy / deploy

## Rule

- chưa cần monorepo tool
- chưa cần shared package nếu chưa có thật
- fullstack small dùng cấu trúc này là hợp lý

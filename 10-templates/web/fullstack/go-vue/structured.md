# Fullstack Structured — Go + Vue

## Khi nào dùng

- product bắt đầu sống lâu hơn
- backend và frontend đều đã nhiều module

## Cây thư mục

```text
<product-name>/
├── docs/                           # tài liệu chung của product
├── infra/                          # deployment config chung
├── scripts/                        # script dev/build/deploy chung
├── config/                         # file cấu hình mẫu
├── backend/                        # Go API — cấu trúc như go/structured
│   ├── docs/                       # tài liệu riêng BE
│   ├── cmd/                        # entrypoint cho mỗi binary
│   │   └── api/                    # binary API
│   │       └── main.go             # chỉ wiring
│   ├── internal/                   # code private của module Go
│   │   ├── app/                    # bootstrap, DI
│   │   ├── domain/                 # entity, rule cốt lõi
│   │   ├── usecase/                # application flow
│   │   └── adapter/                # http/repository/external
│   ├── migrations/                 # SQL migrations
│   ├── tests/                      # test BE
│   ├── go.mod                      # khai báo module
│   └── go.sum                      # checksum dependency
├── frontend/                       # Vue app — cấu trúc như vue-vite/structured
│   ├── docs/                       # tài liệu riêng FE
│   ├── public/                     # tài nguyên tĩnh
│   ├── src/                        # source Vue
│   │   ├── components/             # component generic
│   │   ├── composables/            # composable dùng chung
│   │   ├── features/               # logic theo domain
│   │   ├── pages/                  # page mỏng
│   │   ├── router/                 # vue-router
│   │   ├── services/               # API client gọi BE
│   │   ├── stores/                 # Pinia store
│   │   ├── types/                  # TypeScript type
│   │   ├── App.vue                 # root component
│   │   └── main.ts                 # entrypoint
│   ├── tests/                      # test FE
│   ├── vite.config.ts              # cấu hình Vite
│   └── package.json                # dependency FE
├── README.md                       # hướng dẫn repo
└── .gitignore
```

## Vai trò thư mục

- backend structured theo Go idiomatic layout
- frontend structured theo feature-based Vue layout
- root chỉ giữ concern chung của product

## Rule

- vẫn chưa phải monorepo nâng cao
- chỉ tách rõ 2 phía, không thêm ceremony thừa

# Vue + Vite Frontend Structured

## Khi nào dùng

- app Vue bắt đầu nhiều feature
- nhiều người sửa cùng lúc

## Cây thư mục

```text
<app-name>/
├── docs/                       # tài liệu, flow, ghi chú kỹ thuật
├── infra/                      # docker, nginx, deployment config
├── scripts/                    # script build/preview/deploy
├── config/                     # file cấu hình mẫu
├── tests/                      # test nằm ngoài source chính
│   ├── unit/                   # unit test (Vitest)
│   └── e2e/                    # end-to-end (Playwright/Cypress)
├── public/                     # tài nguyên tĩnh
├── src/                        # vùng source chính
│   ├── assets/                 # ảnh/font/css import qua bundler
│   ├── components/             # component generic dùng chung nhiều feature
│   ├── composables/            # composable/hook dùng chung (useXxx)
│   ├── features/               # logic theo domain — mỗi feature độc lập
│   │   ├── auth/               # feature auth (component + store + api)
│   │   ├── dashboard/          # feature dashboard
│   │   └── settings/           # feature settings
│   ├── layouts/                # layout bọc page (default, auth, admin...)
│   ├── pages/                  # page mỏng — compose từ feature
│   ├── router/                 # khai báo vue-router
│   ├── stores/                 # Pinia store dùng toàn cục
│   ├── services/               # tích hợp ngoài (API client, SDK wrapper)
│   ├── types/                  # TypeScript type dùng chung
│   ├── utils/                  # helper thuần (không state) — rất tiết chế
│   ├── App.vue                 # root component
│   └── main.ts                 # entrypoint
├── index.html                  # HTML gốc Vite dùng
├── vite.config.ts              # cấu hình Vite
├── package.json                # dependency + script
├── README.md                   # hướng dẫn repo
└── .gitignore
```

## Vai trò thư mục

- `features/`: logic theo domain
- `components/`: generic shared UI
- `composables/`: hooks/composables dùng chung
- `services/`: integrations
- `pages/`: mỏng, compose từ feature

## Rule

- đừng để `pages/` thành nơi chứa đủ thứ
- `components/` root không chứa component chỉ dùng cho 1 feature
- `utils/` phải rất tiết chế
- Feature nên chứa component/composable/api riêng nếu chỉ dùng trong feature đó.
- Config dùng `.env.example` và biến `VITE_*`; không hardcode endpoint production.
- Unit test feature/composable/store; e2e cho flow chính; build output luôn gitignore.
- Nếu feature layer chỉ là folder rỗng, quay lại `simple.md`.

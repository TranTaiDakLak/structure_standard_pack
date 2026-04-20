# React + Vite Frontend Structured

## Khi nào dùng

- app React nhiều feature hơn
- cần tách state, hooks, feature logic

## Cây thư mục

```text
<app-name>/
├── docs/                   # tài liệu, flow, ghi chú kỹ thuật
├── infra/                  # docker, nginx, deployment config
├── scripts/                # script build/preview/deploy
├── config/                 # file cấu hình mẫu
├── tests/                  # test nằm ngoài source chính
│   ├── unit/               # unit test (Vitest)
│   └── e2e/                # end-to-end (Playwright)
├── public/                 # tài nguyên tĩnh
├── src/                    # vùng source chính
│   ├── assets/             # ảnh/font/css import qua bundler
│   ├── components/         # component generic dùng chung nhiều feature
│   ├── features/           # logic theo domain
│   │   ├── auth/           # feature auth (component + hook + api)
│   │   ├── dashboard/      # feature dashboard
│   │   └── settings/       # feature settings
│   ├── hooks/              # hook dùng chung (useXxx)
│   ├── lib/                # infra/helper (client, logger...) — tiết chế
│   ├── pages/              # page mỏng — compose từ feature
│   ├── routes/             # khai báo route
│   ├── services/           # API client, SDK wrapper
│   ├── types/              # TypeScript type dùng chung
│   ├── App.tsx             # root component
│   └── main.tsx            # entrypoint
├── index.html              # HTML gốc Vite dùng
├── vite.config.ts          # cấu hình Vite
├── package.json            # dependency + script
├── README.md               # hướng dẫn repo
└── .gitignore
```

## Vai trò thư mục

- `features/`: domain logic
- `hooks/`: shared hooks
- `lib/`: infra/helper rất tiết chế
- `pages/`: compose từ features

## Rule

- tránh để `components/` và `features/` chồng chéo
- không biến `lib/` thành thùng rác
- structured là để code dễ tìm hơn

# Nuxt Frontend Structured

## Khi nào dùng

- Nuxt app có nhiều feature
- page bắt đầu nhiều logic

## Cây thư mục

```text
<app-name>/
├── docs/                   # tài liệu, flow, ghi chú kỹ thuật
├── infra/                  # docker, deployment config
├── scripts/                # script build/preview/deploy
├── config/                 # file cấu hình mẫu
├── tests/                  # test nằm ngoài source chính
│   ├── unit/               # unit test (Vitest)
│   └── e2e/                # end-to-end (Playwright)
├── assets/                 # ảnh/font/scss import qua bundler
├── components/             # component generic (auto-import)
├── composables/            # composable dùng chung (useXxx — auto-import)
├── features/               # logic theo domain
│   ├── auth/               # feature auth
│   ├── billing/            # feature billing
│   └── settings/           # feature settings
├── layouts/                # layout Nuxt
├── pages/                  # page mỏng — compose từ feature
├── public/                 # tài nguyên tĩnh
├── server/                 # route server / BFF (Nitro)
├── types/                  # TypeScript type dùng chung
├── utils/                  # helper thuần — rất tiết chế
├── app.vue                 # root component Nuxt
├── nuxt.config.ts          # cấu hình Nuxt
├── package.json            # dependency + script
├── README.md               # hướng dẫn repo
└── .gitignore
```

## Vai trò thư mục

- `features/`: logic theo domain
- `pages/`: mỏng, compose từ feature
- `server/`: routes/BFF nhẹ
- `composables/`: shared composables

## Rule

- vẫn tôn trọng Nuxt conventions
- không nhét toàn bộ logic vào `pages/`
- `server/` không thành backend vô tổ chức
- Runtime config dùng Nuxt config/env; secret server-side không rò sang client.
- Unit test composable/feature; e2e cho flow chính.
- Nếu server routes trở thành backend thật, tách backend riêng theo template web backend.

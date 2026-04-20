# Nuxt Frontend Simple

## Khi nào dùng

- landing/marketing site
- Nuxt app nhỏ-vừa

## Cây thư mục

```text
<app-name>/
├── docs/                 # tài liệu, flow, ghi chú kỹ thuật
├── infra/                # docker, deployment config
├── scripts/              # script build/preview/deploy
├── config/               # file cấu hình mẫu, .env.example
├── tests/                # test nằm ngoài source chính
├── assets/               # ảnh/font/scss import qua bundler
├── components/           # component UI dùng lại (auto-import)
├── layouts/              # layout Nuxt (default, auth, admin...)
├── pages/                # page Nuxt — file = route (file-based routing)
├── public/               # tài nguyên tĩnh phục vụ thẳng
├── server/               # route server / BFF nhẹ (Nitro)
├── app.vue               # root component Nuxt
├── nuxt.config.ts        # cấu hình Nuxt (module, runtimeConfig)
├── package.json          # dependency + script
├── README.md             # hướng dẫn repo
└── .gitignore            # bỏ qua .nuxt/, .output/, node_modules/
```

## Vai trò thư mục

- `pages/`: route pages
- `components/`: reusable UI
- `layouts/`: layouts
- `server/`: server routes/BFF nhẹ
- `assets/`: styles, images

## Rule

- giữ convention Nuxt
- chưa cần `features/` nếu app còn nhỏ

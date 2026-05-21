# Vue + Vite Frontend Simple

## Khi nào dùng

- web app đơn
- team 1–3 người
- chưa có nhiều feature

## Cây thư mục

```text
<app-name>/
├── docs/                 # tài liệu, flow, ghi chú kỹ thuật
├── infra/                # docker, nginx, deployment config
├── scripts/              # script build/preview/deploy
├── config/               # file cấu hình mẫu, .env.example
├── tests/                # test nằm ngoài source chính
├── public/               # tài nguyên tĩnh phục vụ thẳng (favicon, robots)
├── src/                  # vùng source chính
│   ├── assets/           # ảnh/font/css import qua bundler
│   ├── components/       # component UI dùng lại (Button, Modal...)
│   ├── pages/            # page-level screen (map với route)
│   ├── router/           # khai báo vue-router
│   ├── stores/           # Pinia store
│   ├── api/              # wrapper gọi API (axios/fetch)
│   ├── App.vue           # root component
│   └── main.ts           # entrypoint (tạo app, mount #app)
├── index.html            # HTML gốc Vite dùng
├── vite.config.ts        # cấu hình Vite (alias, plugin)
├── package.json          # dependency + script
├── README.md             # hướng dẫn repo
└── .gitignore            # bỏ qua node_modules/, dist/
```

## Vai trò thư mục

- `components/`: reusable UI trong app
- `pages/`: page-level screens
- `router/`: route definitions
- `stores/`: Pinia stores
- `api/`: API wrappers

## Rule

- page lớn mới nên chứa logic page
- `api/` không ôm business rules
- nếu feature nhiều lên, chuyển dần sang structured
- Config dùng `.env.example` và biến `VITE_*`; không hardcode endpoint production.
- Test tối thiểu component/composable/store quan trọng và e2e cho flow chính.
- `dist/`, coverage, cache, node_modules phải gitignore.

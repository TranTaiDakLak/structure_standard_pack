# React + Vite Frontend Simple

## Khi nào dùng

- web app React nhỏ-vừa
- team nhỏ
- ưu tiên ship nhanh

## Cây thư mục

```text
<app-name>/
├── docs/                 # tài liệu, flow, ghi chú kỹ thuật
├── infra/                # docker, nginx, deployment config
├── scripts/              # script build/preview/deploy
├── config/               # file cấu hình mẫu, .env.example
├── tests/                # test nằm ngoài source chính
├── public/               # tài nguyên tĩnh phục vụ thẳng
├── src/                  # vùng source chính
│   ├── assets/           # ảnh/font/css import qua bundler
│   ├── components/       # component UI dùng lại
│   ├── pages/            # page-level screen (map với route)
│   ├── routes/           # khai báo route (react-router)
│   ├── services/         # wrapper gọi API/SDK ngoài
│   ├── App.tsx           # root component
│   └── main.tsx          # entrypoint (createRoot, render App)
├── index.html            # HTML gốc Vite dùng
├── vite.config.ts        # cấu hình Vite (alias, plugin)
├── package.json          # dependency + script
├── README.md             # hướng dẫn repo
└── .gitignore            # bỏ qua node_modules/, dist/
```

## Vai trò thư mục

- `components/`: reusable UI
- `pages/`: page screens
- `routes/`: route setup
- `services/`: API/integration wrappers

## Rule

- page không nên ôm toàn bộ logic app
- `services/` không ôm state UI
- nếu feature tăng nhanh, nâng lên structured

# Fullstack Structured — Node.js + React

## Khi nào dùng

- backend và frontend đều bắt đầu nhiều modules
- cần boundary rõ hơn

## Cây thư mục

```text
<product-name>/
├── docs/                       # tài liệu chung của product
├── infra/                      # deployment config chung
├── scripts/                    # script dev/build/deploy chung
├── config/                     # file cấu hình mẫu
├── backend/                    # Express API — như node-express/structured
│   ├── src/                    # source BE
│   │   ├── app/                # bootstrap wiring
│   │   ├── modules/            # feature/domain module độc lập
│   │   ├── platform/           # DB, HTTP, cache — hạ tầng dùng chung
│   │   ├── shared/             # helper dùng chung — rất tiết chế
│   │   └── server.js           # entrypoint BE
│   ├── tests/                  # test BE
│   └── package.json            # dependency BE
├── frontend/                   # React app — như react-vite/structured
│   ├── public/                 # tài nguyên tĩnh
│   ├── src/                    # source FE
│   │   ├── components/         # component generic
│   │   ├── features/           # logic theo domain
│   │   ├── hooks/              # hook dùng chung
│   │   ├── pages/              # page mỏng
│   │   ├── routes/             # khai báo route
│   │   ├── services/           # API client gọi BE
│   │   ├── types/              # TypeScript type
│   │   ├── App.tsx             # root component
│   │   └── main.tsx            # entrypoint
│   ├── tests/                  # test FE
│   └── package.json            # dependency FE
├── README.md                   # hướng dẫn repo
└── .gitignore
```

## Vai trò thư mục

- backend structured theo modules/platform
- frontend structured theo features/hooks/services

## Rule

- structured chỉ có ý nghĩa nếu team thật sự bắt đầu đau vì simple
- không thêm package/shared layer khi chưa cần
- Backend và frontend giữ test/build pipeline riêng; root script chỉ orchestration.
- API contract ổn định nên có OpenAPI hoặc tài liệu tương đương, không bắt buộc codegen.
- Production cần healthcheck, logging, validation, migration, secret loading và deploy target rõ.
- Nếu workspace/shared package chưa có nhu cầu thật, không thêm layer chung.

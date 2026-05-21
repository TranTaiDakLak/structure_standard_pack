# Fullstack Structured — ASP.NET Core + Vue

## Khi nào dùng

- product dài hạn hơn
- backend và frontend cùng nhiều module

## Cây thư mục

```text
<product-name>/
├── docs/                                   # tài liệu chung của product
├── infra/                                  # deployment config chung
├── scripts/                                # script dev/build/deploy chung
├── config/                                 # file cấu hình mẫu
├── backend/                                # ASP.NET Core — như dotnet/structured
│   ├── src/                                # multi-project
│   │   ├── <ServiceName>.Api/              # controller, middleware, Program.cs
│   │   ├── <ServiceName>.Application/      # use case, DTO, orchestration
│   │   ├── <ServiceName>.Domain/           # entity, rule — không phụ thuộc framework
│   │   └── <ServiceName>.Infrastructure/   # EF Core, HTTP client, adapter
│   ├── tests/                              # test BE (unit + integration)
│   ├── <ServiceName>.sln                   # solution
│   ├── Directory.Build.props               # thiết lập build chung
│   └── Directory.Packages.props            # quản lý version NuGet tập trung
├── frontend/                               # Vue app — như vue-vite/structured
│   ├── public/                             # tài nguyên tĩnh
│   ├── src/                                # source Vue
│   │   ├── components/                     # component generic
│   │   ├── composables/                    # composable dùng chung
│   │   ├── features/                       # logic theo domain
│   │   ├── pages/                          # page mỏng
│   │   ├── router/                         # vue-router
│   │   ├── services/                       # API client gọi BE
│   │   ├── stores/                         # Pinia store
│   │   ├── types/                          # TypeScript type
│   │   ├── App.vue                         # root component
│   │   └── main.ts                         # entrypoint
│   ├── tests/                              # test FE
│   ├── vite.config.ts                      # cấu hình Vite
│   └── package.json                        # dependency FE
├── README.md                               # hướng dẫn repo
└── .gitignore
```

## Vai trò thư mục

- backend structured theo .NET layered layout
- frontend structured theo Vue feature-based layout

## Rule

- structured để dễ maintain, không phải để giả enterprise
- nếu một phía còn nhỏ, có thể chỉ nâng phía đó trước
- Backend và frontend giữ test/build pipeline riêng; root script chỉ orchestration.
- API contract ổn định nên có OpenAPI hoặc tài liệu tương đương, không bắt buộc codegen.
- Production cần healthcheck, logging, migration, secret loading và deploy target rõ.
- Nếu shared package/contracts chưa có nhu cầu thật, không thêm layer chung.

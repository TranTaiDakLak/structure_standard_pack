# ASP.NET Core Backend Simple

## Khi nào dùng

- API nhỏ hoặc vừa
- team 1–3 người
- MVP hoặc internal tool

## Cây thư mục

```text
<ServiceName>/
├── docs/                               # tài liệu, flow, ghi chú kỹ thuật
├── infra/                              # iis, nginx, deployment config
├── scripts/                            # script build/publish/migrate
├── config/                             # appsettings.example.json, không secret thật
├── tests/                              # test nằm ngoài source chính
│   └── <ServiceName>.Tests/            # unit test project
├── src/                                # vùng source chính
│   └── <ServiceName>/                  # project ASP.NET Core duy nhất
│       ├── Controllers/                # API endpoint (Attribute Routing)
│       ├── Services/                   # business logic
│       ├── Repositories/               # truy cập DB (EF Core / Dapper)
│       ├── Models/                     # entity / domain model đơn giản
│       ├── Dtos/                       # request/response DTO
│       ├── Program.cs                  # entrypoint + DI + middleware pipeline
│       └── appsettings.json            # cấu hình runtime
├── <ServiceName>.sln                   # solution file
├── Directory.Build.props               # thiết lập build dùng chung
├── README.md                           # hướng dẫn repo
└── .gitignore                          # bỏ qua bin/, obj/, *.user...
```

## Vai trò thư mục

- `Controllers/`: API endpoints
- `Services/`: business logic
- `Repositories/`: data access
- `Models/`: entity/model đơn giản
- `Dtos/`: request/response DTO

## Rule

- controller không ôm logic dài
- `src/` và `tests/` tách riêng
- nếu flow bắt đầu rối, nâng lên structured
- Config mẫu dùng `appsettings.example.json` hoặc env mapping; secret thật không nằm trong repo.
- Nếu có DB, migration phải có script hoặc hướng dẫn rõ, không chạy ngầm ngoài kiểm soát.
- API production nên có healthcheck, logging có correlation id, validation và error response thống nhất.

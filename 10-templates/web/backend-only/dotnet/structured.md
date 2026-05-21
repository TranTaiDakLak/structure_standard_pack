# ASP.NET Core Backend Structured

## Khi nào dùng

- service sống lâu
- cần phân lớp rõ hơn
- team 3–5 người

## Cây thư mục

```text
<ServiceName>/
├── docs/                                  # tài liệu, flow, ghi chú kỹ thuật
├── infra/                                 # iis, nginx, deployment config
├── scripts/                               # script build/publish/migrate
├── config/                                # appsettings mẫu
├── tests/                                 # test nằm ngoài source chính
│   ├── <ServiceName>.UnitTests/           # test Domain + Application (không I/O)
│   ├── <ServiceName>.IntegrationTests/    # test Infrastructure (DB, HTTP thật)
│   └── <ServiceName>.E2ETests/            # end-to-end qua API công khai
├── src/                                   # vùng source chính (multi-project)
│   ├── <ServiceName>.Api/                 # controller, middleware, Program.cs
│   ├── <ServiceName>.Application/         # use case, orchestration, DTO
│   ├── <ServiceName>.Domain/              # entity, rule — KHÔNG phụ thuộc framework
│   └── <ServiceName>.Infrastructure/      # EF Core, HTTP client, message bus...
├── <ServiceName>.sln                      # solution file
├── Directory.Build.props                  # thiết lập build dùng chung
├── Directory.Packages.props               # quản lý version NuGet tập trung
├── README.md                              # hướng dẫn repo
└── .gitignore
```

## Vai trò thư mục

- `.Api`: controllers, middleware
- `.Application`: use cases
- `.Domain`: rules, entities
- `.Infrastructure`: EF Core, external clients

## Rule

- `.Api` không xử lý nghiệp vụ
- `.Domain` không phụ thuộc web framework
- chưa cần `.Contracts` nếu chưa có nhu cầu share thật
- `.Application` định nghĩa abstraction; `.Infrastructure` implement, không đảo chiều phụ thuộc.
- Unit test Domain/Application; integration test Infrastructure/API quan trọng.
- API production nên có health/readiness, migration script, structured logging và secret loading rõ.
- Nếu layer làm chậm mà domain còn nhỏ, quay lại `simple.md`.

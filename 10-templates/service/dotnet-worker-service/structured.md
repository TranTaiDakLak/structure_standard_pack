# Service Structured — .NET Worker Service

## Khi nào dùng

- Service sống lâu (> 6 tháng), nhiều worker / job, nhiều tích hợp ngoài
- Cần test được business logic độc lập khỏi runtime Host
- Team ≥ 3 người cùng chạm
- Có vài service tương tự nhau (ETL, sync, scheduled) muốn dùng chung domain

## Cây thư mục

```text
<ServiceName>/
├── docs/                                   # tài liệu, flow, ghi chú kỹ thuật
├── infra/                                  # config triển khai
│   └── service-install/                    # script cài Windows Service (sc.exe / nssm)
├── scripts/                                # script build/publish/migrate
├── config/                                 # appsettings mẫu
├── tests/                                  # test nằm ngoài source chính
│   ├── <ServiceName>.UnitTests/            # test Domain + Application (không I/O)
│   └── <ServiceName>.IntegrationTests/     # test Infrastructure (DB, HTTP thật)
├── src/                                    # vùng source chính (multi-project)
│   ├── <ServiceName>.Host/                 # host chạy worker — entrypoint
│   │   ├── Workers/                        # class kế thừa BackgroundService
│   │   ├── Jobs/                           # scheduled job (Quartz/Hangfire) — nếu có
│   │   ├── Program.cs                      # CreateHostBuilder + DI + UseWindowsService
│   │   └── appsettings.json                # cấu hình runtime
│   ├── <ServiceName>.Application/          # use case, orchestration
│   │   ├── UseCases/                       # mỗi use case 1 class (SyncOrder, ImportFile...)
│   │   ├── Dtos/                           # DTO truyền giữa Host và Application
│   │   └── Abstractions/                   # interface Infrastructure sẽ implement
│   ├── <ServiceName>.Domain/               # nghiệp vụ lõi — KHÔNG phụ thuộc framework
│   │   ├── Entities/                       # entity nghiệp vụ
│   │   ├── ValueObjects/                   # value object
│   │   └── Services/                       # domain service (logic thuần)
│   └── <ServiceName>.Infrastructure/       # tích hợp ngoài
│       ├── Persistence/                    # EF Core / Dapper / ADO.NET
│       ├── Http/                           # HttpClient gọi API ngoài
│       ├── Files/                          # đọc/ghi file, FTP, SFTP
│       └── Messaging/                      # RabbitMQ / Kafka / Azure Service Bus
├── <ServiceName>.sln                       # solution
├── Directory.Build.props                   # thiết lập build chung
├── Directory.Packages.props                # quản lý version NuGet tập trung
├── README.md                               # hướng dẫn repo
└── .gitignore                              # bỏ qua bin/, obj/, publish/
```

## Vai trò thư mục

- `.Host`: chỉ là runtime wrapper — `Workers/`, `Jobs/`, cấu hình Host. KHÔNG chứa logic nghiệp vụ.
- `.Application`: orchestrate use case, định nghĩa `Abstractions/` (interface) để Infrastructure implement.
- `.Domain`: nghiệp vụ thuần — entity, value object, domain service. KHÔNG tham chiếu project khác.
- `.Infrastructure`: implement interface của `.Application.Abstractions` — persistence, HTTP, file, messaging.

## Rule

- `Domain` KHÔNG tham chiếu `Microsoft.Extensions.*` hoặc bất kỳ project khác.
- `Application` KHÔNG tham chiếu `Infrastructure` (DI đảo ngược qua interface).
- Worker KHÔNG new trực tiếp repository/HttpClient — inject qua constructor.
- Mỗi Worker nên gọn: chỉ 1 use case chính. Nếu phình → tách thành Worker riêng.
- Job handler (Quartz/Hangfire) cũng chỉ là 1 lớp mỏng gọi vào `Application.UseCases`.
- Structured Worker Service KHÔNG có nghĩa là enterprise nặng — vẫn giữ tinh thần gọn.
- Nếu chưa đủ lớn để justify 4 project → quay về simple.

# Service Simple — .NET Worker Service

## Khi nào dùng

- 1–2 worker chạy ngầm, logic còn gọn
- Team 1–3 người
- Service quét file / sync dữ liệu / chạy job định kỳ nội bộ
- 1 project .NET duy nhất là đủ

## Cây thư mục

```text
<ServiceName>/
├── docs/                               # tài liệu, flow, ghi chú kỹ thuật
├── infra/                              # config triển khai
│   └── service-install/                # script cài Windows Service (sc.exe / nssm / PowerShell)
├── scripts/                            # script build/publish/run
├── config/                             # appsettings.example.json, không secret thật
├── tests/                              # test nằm ngoài source chính
│   └── <ServiceName>.Tests/            # unit test project
├── src/                                # vùng source chính
│   └── <ServiceName>/                  # project Worker Service duy nhất
│       ├── Workers/                    # class kế thừa BackgroundService
│       ├── Jobs/                       # scheduled job (Quartz/Hangfire) — nếu có
│       ├── Services/                   # business logic
│       ├── Models/                     # POCO / DTO
│       ├── Program.cs                  # Host.CreateDefaultBuilder().UseWindowsService()
│       └── appsettings.json            # cấu hình runtime
├── <ServiceName>.sln                   # solution
├── Directory.Build.props               # thiết lập build chung
├── README.md                           # hướng dẫn repo
└── .gitignore                          # bỏ qua bin/, obj/, publish/
```

## Vai trò thư mục

- `Workers/`: mỗi worker = 1 class kế thừa `BackgroundService` (override `ExecuteAsync`). KHÔNG nhét business logic dài vào đây — chỉ orchestrate.
- `Jobs/`: nếu dùng Quartz.NET hoặc Hangfire → job handler nằm ở đây (khác `Workers/` vì job được scheduler kích, không tự loop).
- `Services/`: business logic thật, worker/job gọi vào.
- `Models/`: POCO/DTO dùng chung.
- `Program.cs`: cấu hình Host + DI + `.UseWindowsService()` (nếu cài Windows Service) hoặc `.RunAsync()` (nếu chạy console/Docker).
- `infra/service-install/`: script PowerShell / `sc.exe create` / `nssm install` để cài service — file example, không chứa secret.

## Rule

- Worker KHÔNG chứa business logic nặng — gọi xuống Service.
- KHÔNG dùng `Task.Delay` thủ công trong vòng lặp dài — ưu tiên `PeriodicTimer` (.NET 6+) hoặc Quartz.
- Graceful shutdown: luôn tôn trọng `CancellationToken` truyền từ `ExecuteAsync`.
- Logging đi qua `ILogger<T>` — KHÔNG `Console.WriteLine`.
- `appsettings.json` chứa key runtime; secret (connection string thật, API key) nạp qua environment variable hoặc User Secrets lúc dev.
- `build/`, `publish/`, `*.user` phải gitignore.
- Nếu service bắt đầu có nhiều worker + nhiều domain → cân nhắc nâng lên structured.

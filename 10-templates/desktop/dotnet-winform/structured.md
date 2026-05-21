# Desktop Structured — .NET WinForms

## Khi nào dùng

- App WinForms sống lâu (> 6 tháng), nhiều Form, nhiều module nghiệp vụ
- Cần tách domain rõ khỏi UI để test được
- Có tích hợp ngoài (DB, Web API, file nội bộ) nhiều và đa dạng
- Team ≥ 3 người cùng chạm vào codebase

## Cây thư mục

```text
<AppName>/
├── docs/                                  # tài liệu, flow, ghi chú kỹ thuật
├── infra/                                 # config build/triển khai
│   └── installer/                         # script tạo installer (Inno Setup, WiX...)
├── scripts/                               # script build/publish/run/migrate
├── config/                                # file cấu hình mẫu
├── build/                                 # output build — gitignore
├── sidecar/                               # (optional) binary prebuilt ship kèm app
│   └── <service-name>/                    # mỗi sidecar 1 sub-folder
│       ├── README.md                      # mục đích, version, start/stop, port
│       └── <service-name>.exe             # binary (API local, updater, helper)
├── src/                                   # vùng source chính (multi-project)
│   ├── <AppName>.App/                     # project WinForms — UI layer
│   │   ├── Forms/                         # các Form (.cs + .Designer.cs + .resx)
│   │   ├── Presenters/                    # presenter cho từng Form (MVP)
│   │   ├── Views/                         # interface IView cho Form (contract của UI)
│   │   ├── Resources/                     # ảnh, icon, string resource
│   │   ├── Properties/                    # AssemblyInfo, Settings WinForms
│   │   └── Program.cs                     # entrypoint + DI bootstrap
│   ├── <AppName>.Application/             # use case / application service
│   │   ├── UseCases/                      # mỗi use case 1 class (CreateOrder, ImportFile...)
│   │   ├── Dtos/                          # DTO truyền giữa App và UI
│   │   └── Abstractions/                  # interface mà Infrastructure sẽ implement
│   ├── <AppName>.Domain/                  # nghiệp vụ lõi — KHÔNG phụ thuộc UI/DB
│   │   ├── Entities/                      # entity nghiệp vụ
│   │   ├── ValueObjects/                  # value object
│   │   └── Services/                      # domain service (logic thuần nghiệp vụ)
│   └── <AppName>.Infrastructure/          # tích hợp ngoài: DB, API, file, Windows API
│       ├── Persistence/                   # EF Core / Dapper / ADO.NET
│       ├── Http/                          # HttpClient wrapper gọi API ngoài
│       └── Files/                         # đọc/ghi file, Excel, CSV
├── tests/                                 # test nằm ngoài source chính
│   ├── <AppName>.UnitTests/               # unit test cho Domain + Application
│   └── <AppName>.IntegrationTests/        # integration test Infrastructure
├── <AppName>.sln                          # solution file
├── Directory.Build.props                  # thiết lập build dùng chung
├── Directory.Packages.props               # quản lý version NuGet tập trung
├── README.md                              # hướng dẫn repo
└── .gitignore                             # bỏ qua bin/, obj/, build/, *.user
```

## Vai trò thư mục

- `.App`: UI layer WinForms — Form, Presenter, View interface, entrypoint.
- `.Application`: orchestrate use case, dùng Domain + Abstractions.
- `.Domain`: nghiệp vụ lõi, KHÔNG tham chiếu bất kỳ project nào khác.
- `.Infrastructure`: implement các interface trong `.Application.Abstractions` (DB, HTTP, file...).
- `Presenters/` + `Views/`: tách rõ contract UI (IView) khỏi implement (Form) → Presenter test được.

## Rule

- `Domain` KHÔNG được tham chiếu `System.Windows.Forms` hoặc bất kỳ project khác.
- `Application` KHÔNG tham chiếu `Infrastructure` (đảo ngược qua interface).
- `Form` KHÔNG new trực tiếp Service — inject qua Presenter/constructor.
- Structured WinForms không có nghĩa là enterprise nặng — vẫn giữ tinh thần gọn.
- Nếu dự án chưa đủ lớn để justify 4 project → quay lại simple.
- Unit test Domain/Application; integration test Infrastructure quan trọng.
- `.App` chỉ wiring UI + Presenter, không tham chiếu trực tiếp persistence/API ngoài.
- Installer, publish output, sidecar artifact phải tách khỏi source và gitignore đúng.

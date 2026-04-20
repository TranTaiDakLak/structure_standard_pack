# Desktop Structured — .NET WPF

## Khi nào dùng

- app WPF sống lâu hơn
- nhiều module hơn
- cần phân lớp rõ hơn

## Cây thư mục

```text
<AppName>/
├── docs/                               # tài liệu, flow, ghi chú kỹ thuật
├── infra/                              # deployment config
│   └── installer/                      # script tạo installer
├── scripts/                            # script build/publish/run
├── config/                             # file cấu hình mẫu
├── build/                              # output build — gitignore
├── sidecar/                            # (optional) binary prebuilt ship kèm app
│   └── <service-name>/                 # mỗi sidecar 1 sub-folder
│       ├── README.md                   # mục đích, version, start/stop, port
│       └── <service-name>.exe          # binary (API local, updater, helper)
├── src/                                # vùng source chính (multi-project)
│   ├── <AppName>.App/                  # WPF UI layer
│   │   ├── Views/                      # XAML view
│   │   ├── ViewModels/                 # presentation logic (MVVM)
│   │   └── App.xaml                    # App resource + startup
│   ├── <AppName>.Application/          # use case, application service
│   ├── <AppName>.Domain/               # entity + rule — KHÔNG phụ thuộc UI/DB
│   └── <AppName>.Infrastructure/       # EF Core, HTTP client, file/Windows API
├── tests/                              # test nằm ngoài source chính
│   ├── <AppName>.UnitTests/            # unit test Domain + Application
│   └── <AppName>.IntegrationTests/     # integration test Infrastructure
├── <AppName>.sln                       # solution
├── Directory.Build.props               # thiết lập build chung
├── Directory.Packages.props            # quản lý version NuGet tập trung
└── README.md                           # hướng dẫn repo
```

## Vai trò thư mục

- `.App`: WPF UI layer
- `.Application`: use cases/app services
- `.Domain`: rules/entities
- `.Infrastructure`: storage, external integrations

## Rule

- structured WPF không có nghĩa là enterprise nặng
- MVVM vẫn nằm ở `.App`, domain không phụ thuộc UI

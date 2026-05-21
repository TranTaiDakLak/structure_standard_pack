# Desktop Simple — .NET WPF

## Khi nào dùng

- app WPF nhỏ-vừa
- team 1–3 người
- cần rõ Views / ViewModels / Services

## Cây thư mục

```text
<AppName>/
├── docs/                           # tài liệu, flow, ghi chú kỹ thuật
├── infra/                          # deployment config
│   └── installer/                  # script tạo installer (Inno Setup, WiX...)
├── scripts/                        # script build/publish/run
├── config/                         # file cấu hình mẫu
├── build/                          # output build — gitignore
├── sidecar/                        # (optional) binary prebuilt ship kèm app
│   └── <service-name>/             # mỗi sidecar 1 sub-folder
│       ├── README.md               # mục đích, version, start/stop, port
│       └── <service-name>.exe      # binary (API local, updater, helper)
├── src/                            # vùng source chính
│   └── <AppName>/                  # project WPF duy nhất
│       ├── Views/                  # XAML view (UserControl, Window)
│       ├── ViewModels/             # presentation logic (MVVM)
│       ├── Services/               # app service / integration đơn giản
│       ├── Models/                 # domain model đơn giản
│       ├── Resources/              # ảnh, icon, ResourceDictionary
│       ├── App.xaml                # App resource + startup
│       └── MainWindow.xaml         # cửa sổ chính
├── tests/                          # test nằm ngoài source chính
│   └── <AppName>.Tests/            # unit test project
├── <AppName>.sln                   # solution
├── Directory.Build.props           # thiết lập build chung
├── README.md                       # hướng dẫn repo
└── .gitignore                      # bỏ qua bin/, obj/, build/, *.user
```

## Vai trò thư mục

- `Views/`: XAML views
- `ViewModels/`: presentation logic
- `Services/`: app services/integration đơn giản
- `Models/`: model/domain đơn giản

## Rule

- View không ôm business logic nặng
- Service không bị dính chặt vào View
- `build/` và installer output phải gitignore
- Config mẫu và resource dùng chung phải nằm ở chỗ rõ, không rải trong từng View.
- Test ưu tiên ViewModel/Service; UI automation chỉ thêm khi workflow thật sự quan trọng.
- Nếu nhiều module nghiệp vụ hoặc nhiều integration ngoài, nâng sang `structured.md`.

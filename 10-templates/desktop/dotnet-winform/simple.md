# Desktop Simple — .NET WinForms

## Khi nào dùng

- App WinForms nhỏ-vừa
- Team 1–3 người
- Cần rõ Forms / Presenters / Services / Models trong 1 project duy nhất
- Ship nhanh cho nội bộ: tool nhập liệu, tool cấu hình, utility nhỏ

## Cây thư mục

```text
<AppName>/
├── docs/                                  # tài liệu, flow, ghi chú kỹ thuật
├── infra/                                 # config build, triển khai
│   └── installer/                         # script/cấu hình tạo installer (Inno Setup, WiX...)
├── scripts/                               # script hỗ trợ build/publish/run
├── config/                                # file cấu hình mẫu (appsettings.example.json)
├── build/                                 # output build (bin/obj publish) — gitignore
├── sidecar/                               # (optional) binary prebuilt ship kèm app
│   └── <service-name>/                    # mỗi sidecar 1 sub-folder
│       ├── README.md                      # mục đích, version, start/stop, port
│       └── <service-name>.exe             # binary (API local, updater, helper)
├── src/                                   # vùng source chính
│   └── <AppName>/                         # project WinForms duy nhất
│       ├── Forms/                         # các Form UI (.cs + .Designer.cs + .resx)
│       ├── Presenters/                    # presenter/controller xử lý logic cho Form (MVP)
│       ├── Services/                      # business logic + service tích hợp
│       ├── Models/                        # POCO, DTO, entity đơn giản
│       ├── Resources/                     # ảnh, icon, string resource
│       ├── Properties/                    # AssemblyInfo, Settings của WinForms
│       ├── Program.cs                     # entrypoint (Application.Run)
│       └── <AppName>.csproj               # file project
├── tests/                                 # test nằm ngoài source chính
│   └── <AppName>.Tests/                   # unit test project
├── <AppName>.sln                          # solution file
├── Directory.Build.props                  # thiết lập build dùng chung
├── README.md                              # hướng dẫn repo
└── .gitignore                             # bỏ qua bin/, obj/, build/, *.user...
```

## Vai trò thư mục

- `Forms/`: UI layer — chỉ render + forward event, không chứa business logic.
- `Presenters/`: nhận event từ Form, gọi Service, trả kết quả cho View (pattern MVP).
- `Services/`: business logic + tích hợp ngoài (API, file, DB client đơn giản).
- `Models/`: POCO/DTO, entity đơn giản.
- `Resources/`: ảnh, icon, string resource cho UI.
- `Properties/`: mặc định WinForms sinh ra — giữ nguyên, không nhét logic vào.
- `Program.cs`: `Application.Run(new MainForm())` hoặc wiring DI nhẹ.

## Rule

- Form KHÔNG chứa business logic nặng; tất cả logic gọi qua Presenter/Service.
- KHÔNG viết logic trong file `*.Designer.cs` — file này do Form Designer sinh ra.
- Service KHÔNG biết gì về Form hay `System.Windows.Forms`.
- `build/` và file installer output phải gitignore (`bin/`, `obj/`, `*.user`, `publish/`).
- Nếu app chạy nhiều Form và bắt đầu có state dùng chung → cân nhắc nâng sang structured.

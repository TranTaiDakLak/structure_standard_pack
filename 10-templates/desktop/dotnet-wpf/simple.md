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

## API response contract

Consumer của [`03-standards/API_RESPONSE_CONTRACT.md`](../../../03-standards/API_RESPONSE_CONTRACT.md), contract `api-1`. App không expose HTTP, nhưng khi gọi API ngoài thì:

- Http client + deserialize error đặt ở `src/<AppName>/Services/`, không rải `HttpClient` trong `Views/` hay `ViewModels/`.
- Map `error.code` sang thông báo hiển thị tại đúng một chỗ; `retryable` quyết định có retry hay không.
- `error.trace_id` phải được hiện cho user ở dialog lỗi để gửi support, và ghi vào log local.
- `JsonSerializerOptions` phải set `PropertyNamingPolicy = JsonNamingPolicy.SnakeCaseLower`, nếu không sẽ deserialize hụt toàn bộ field.
- Mỗi upstream **không tuân `api-1`** (partner API, sidecar exe, hệ thống nội bộ cũ) có một adapter riêng, đặt cùng chỗ với http client ở `src/<AppName>/Services/`: adapter quyết định thành công/thất bại theo luật của upstream rồi mới normalize về code của mình, cấm duck-type `body.error` — [appendix section 2.5](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md).
- Lỗi phía máy người dùng (mất mạng, user huỷ, ghi file local hỏng) dùng nhóm `client.*` ở [appendix section 2.0b](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md), không mượn `unavailable`/`timeout` vì hai mã đó nghĩa là server có vấn đề.

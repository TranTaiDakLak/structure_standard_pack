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
├── README.md                           # hướng dẫn repo
└── .gitignore                          # bỏ qua bin/, obj/, build/, *.user
```

## Vai trò thư mục

- `.App`: WPF UI layer
- `.Application`: use cases/app services
- `.Domain`: rules/entities
- `.Infrastructure`: storage, external integrations

## Rule

- structured WPF không có nghĩa là enterprise nặng
- MVVM vẫn nằm ở `.App`, domain không phụ thuộc UI
- `Application` định nghĩa abstraction cho integration; `Infrastructure` implement, không đảo chiều phụ thuộc.
- Test unit cho Domain/Application; integration test cho Infrastructure quan trọng.
- Installer/publish output phải nằm ngoài source và được gitignore.
- Nếu 4 project làm team chậm hơn mà domain còn nhỏ, quay lại `simple.md`.

## API response contract

Consumer của [`03-standards/API_RESPONSE_CONTRACT.md`](../../../03-standards/API_RESPONSE_CONTRACT.md), contract `api-1`. App không expose HTTP, nhưng khi gọi API ngoài thì:

- Http client + deserialize error đặt ở `src/<AppName>.Infrastructure/`, không rải `HttpClient` trong `Views/` hay `ViewModels/` của `.App`.
- Map `error.code` sang thông báo hiển thị tại đúng một chỗ; `retryable` quyết định có retry hay không.
- `error.trace_id` phải được hiện cho user ở dialog lỗi để gửi support, và ghi vào log local.
- `JsonSerializerOptions` phải set `PropertyNamingPolicy = JsonNamingPolicy.SnakeCaseLower`, nếu không sẽ deserialize hụt toàn bộ field.
- Mỗi upstream **không tuân `api-1`** (partner API, sidecar exe, hệ thống nội bộ cũ) có một adapter riêng, đặt cùng chỗ với http client ở `src/<AppName>.Infrastructure/`: adapter quyết định thành công/thất bại theo luật của upstream rồi mới normalize về code của mình, cấm duck-type `body.error` — [appendix section 2.5](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md).
- Lỗi phía máy người dùng (mất mạng, user huỷ, ghi file local hỏng) dùng nhóm `client.*` ở [appendix section 2.0b](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md), không mượn `unavailable`/`timeout` vì hai mã đó nghĩa là server có vấn đề.

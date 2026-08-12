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
- `Program.cs`: cấu hình Host + DI + `.UseWindowsService()` (nếu cài Windows Service) hoặc `.RunAsync()` (nếu chạy console/systemd).
- `infra/service-install/`: script PowerShell / `sc.exe create` / `nssm install` để cài service — file example, không chứa secret.

## Rule

- Worker KHÔNG chứa business logic nặng — gọi xuống Service.
- KHÔNG dùng `Task.Delay` thủ công trong vòng lặp dài — ưu tiên `PeriodicTimer` (.NET 6+) hoặc Quartz.
- Graceful shutdown: luôn tôn trọng `CancellationToken` truyền từ `ExecuteAsync`.
- Logging đi qua `ILogger<T>` — KHÔNG `Console.WriteLine`.
- `appsettings.json` chứa key runtime; secret (connection string thật, API key) nạp qua environment variable hoặc User Secrets lúc dev.
- `build/`, `publish/`, `*.user` phải gitignore.
- Nếu service bắt đầu có nhiều worker + nhiều domain → cân nhắc nâng lên structured.
- Retry/backoff phải rõ cho lỗi tạm thời; không loop lỗi liên tục làm nghẽn log/CPU.
- Nếu expose health/admin endpoint, chỉ bind nội bộ hoặc sau proxy bảo vệ.
- Có script cài/gỡ service hoặc hướng dẫn systemd tương ứng deploy target thật.

## API response contract

Theo [`03-standards/API_RESPONSE_CONTRACT.md`](../../../03-standards/API_RESPONSE_CONTRACT.md), contract `api-1`. Service không phải web API, nhưng dính chuẩn ở 2 chỗ:

**1. Hợp đồng với hệ thống chạy service — health surface và exit code.**
- `/healthz` (liveness, KHÔNG gọi DB) + `/readyz` (readiness, có gọi dependency) bắt buộc khi service có HTTP surface. Hình dạng body và luật **miễn trừ error envelope** lấy nguyên ở [appendix](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md) section 1.6 — không tự chế body khác. Daemon chạy dài bắt buộc có cả 2.
- Worker chạy theo lịch rồi thoát (Task Scheduler) thì **không** kéo cả ASP.NET Core hosting vào chỉ để có health endpoint. Ở đó **exit code LÀ contract, thay vai trò status code**: `0` = mọi bước BẮT BUỘC hoàn tất, `!= 0` = cần người can thiệp và alert bám vào đây. Phải liệt kê trong `docs/` bước/nguồn nào bắt buộc, nguồn nào best-effort (fail vẫn `0` nhưng log mức warn). Bản ghi hỏng giữa chừng đẩy vào bảng/thư mục lỗi kèm `trace_id` của lần chạy, KHÔNG đổ cả job. Luật đầy đủ: [appendix](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md) section 2.3. Trả exit code qua `Environment.ExitCode` (hoặc giá trị trả về của `Main`), không `return 0` mặc định sau khi đã nuốt exception.
- `trace_id` của job one-shot là `run_id` sinh lúc bắt đầu lần chạy, persist cùng bản ghi (appendix section 2.0).

**2. Consumer của API khác — nguồn sự thật để retry khác nhau theo loại upstream.**
- **Upstream tự khai `contract: api-1`:** policy retry (Polly hay tự viết) rẽ nhánh theo `error.code` + `error.retryable` trong body, KHÔNG suy từ status code trần, KHÔNG parse `message` để đoán loại lỗi.
- **Upstream bên thứ 3** (partner API, cổng thanh toán, hệ thống nội bộ cũ) — ca phổ biến nhất của service: **HTTP status code + header `Retry-After` CHÍNH LÀ nguồn sự thật**, vì đó là hợp đồng duy nhất họ thật sự tuân. Mỗi upstream một class client riêng trong `Services/`, map về taxonomy của mình rồi mới ra ngoài; cấm duck-type `body.error`. Luật đầy đủ: [appendix](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md) section 2.5.
- Sau khi đã map: `retryable: false` thì **dừng và đẩy sang dead-letter / bảng lỗi**, không backoff vô hạn; `retryable: true` mới backoff. Đừng để `HttpResponseMessage` hay mã lỗi của bên thứ 3 rò vào `Workers/`.
- `src/<ServiceName>/Models/ErrorCodes.cs`: nơi đặt type envelope + bảng code (static class, `const string`), đúng 1 file cho cả repo.

Nếu service có thêm HTTP endpoint nội bộ (admin, trigger, dashboard mini) thì các endpoint đó áp TOÀN BỘ contract như một API thật. Khi đã expose Minimal API trong `Program.cs`, vẫn phải set `JsonNamingPolicy.SnakeCaseLower` — mặc định của ASP.NET là camelCase, sai wire convention ngay từ health response.

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
- Unit test Domain/Application; integration test Infrastructure có DB/HTTP/queue thật hoặc test container.
- Retry/backoff, idempotency, lock/concurrency phải ghi rõ cho worker/job có tác động dữ liệu.
- Deploy target phải có script install/run và log path rõ.

## API response contract

Theo [`03-standards/API_RESPONSE_CONTRACT.md`](../../../03-standards/API_RESPONSE_CONTRACT.md), contract `api-1`. Service không phải web API, nhưng dính chuẩn ở 2 chỗ:

**1. Hợp đồng với hệ thống chạy service — health surface và exit code.**
- `/healthz` (liveness, KHÔNG gọi DB) + `/readyz` (readiness, có gọi dependency) bắt buộc khi service có HTTP surface. Hình dạng body và luật **miễn trừ error envelope** lấy nguyên ở [appendix](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md) section 1.6 — không tự chế body khác. Daemon chạy dài bắt buộc có cả 2.
- Worker chạy theo lịch rồi thoát (Task Scheduler) thì **không** kéo cả ASP.NET Core hosting vào chỉ để có health endpoint. Ở đó **exit code LÀ contract, thay vai trò status code**: `0` = mọi bước BẮT BUỘC hoàn tất, `!= 0` = cần người can thiệp và alert bám vào đây. Phải liệt kê trong `docs/` bước/nguồn nào bắt buộc, nguồn nào best-effort (fail vẫn `0` nhưng log mức warn). Bản ghi hỏng giữa chừng đẩy vào bảng/thư mục lỗi kèm `trace_id` của lần chạy, KHÔNG đổ cả job. Luật đầy đủ: [appendix](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md) section 2.3. `.Host` là nơi duy nhất quyết exit code — `.Application` trả typed error, không tự gọi `Environment.Exit`.
- `trace_id` của job one-shot là `run_id` sinh lúc bắt đầu lần chạy, persist cùng bản ghi (appendix section 2.0).

**2. Consumer của API khác — nguồn sự thật để retry khác nhau theo loại upstream.**
- **Upstream tự khai `contract: api-1`:** policy retry rẽ nhánh theo `error.code` + `error.retryable` trong body, KHÔNG suy từ status code trần, KHÔNG parse `message` để đoán loại lỗi.
- **Upstream bên thứ 3** (partner API, cổng thanh toán, hệ thống nội bộ cũ) — ca phổ biến nhất của service: **HTTP status code + header `Retry-After` CHÍNH LÀ nguồn sự thật**, vì đó là hợp đồng duy nhất họ thật sự tuân. Mỗi upstream một class adapter riêng trong `src/<ServiceName>.Infrastructure/Http/` (SFTP/file thì `Files/`), map về taxonomy của mình rồi mới ra ngoài; cấm duck-type `body.error`. Luật đầy đủ: [appendix](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md) section 2.5.
- Sau khi đã map: `retryable: false` thì **dừng và đẩy sang dead-letter / bảng lỗi**, không backoff vô hạn; `retryable: true` mới backoff. Rule "retry/backoff, idempotency, lock/concurrency phải ghi rõ" ở trên lấy đây làm căn cứ máy đọc được.
- `Http/`, `Files/`, `Messaging/` trong `.Infrastructure` là nơi DUY NHẤT biết hình dạng lỗi của upstream; đừng để mã lỗi của bên thứ 3 rò vào `.Application.UseCases` hay `Workers/`.
- `src/<ServiceName>.Application/Dtos/`: nơi đặt type envelope + bảng code (`ErrorCodes` static class, `const string`), đúng 1 chỗ cho cả solution; `.Infrastructure` chỉ map vào, không tự khai code riêng.

Nếu service có thêm HTTP endpoint nội bộ (admin, trigger, dashboard mini) thì các endpoint đó áp TOÀN BỘ contract như một API thật. Khi đã expose Minimal API trong `src/<ServiceName>.Host/Program.cs`, vẫn phải set `JsonNamingPolicy.SnakeCaseLower` — mặc định của ASP.NET là camelCase, sai wire convention ngay từ health response.

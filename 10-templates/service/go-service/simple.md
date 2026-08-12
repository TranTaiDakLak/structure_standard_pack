# Service Simple — Go

## Khi nào dùng

- 1–2 worker/job Go, logic còn gọn
- Team 1–3 người
- Service quét file / sync dữ liệu / consumer queue nhỏ
- Không cần tách cmd/internal

## Cây thư mục

```text
<service-name>/
├── docs/                       # tài liệu, flow, ghi chú kỹ thuật
├── infra/                      # config triển khai
│   └── service-install/        # systemd unit / Windows Service script / Dockerfile
├── scripts/                    # script build/run/package
├── config/                     # file cấu hình mẫu (config.example.yaml)
├── tests/                      # test nằm ngoài source chính
├── worker/                     # long-running loop (consume queue, watch file...)
├── job/                        # scheduled job (robfig/cron) — nếu có
├── service/                    # business logic
├── repository/                 # truy cập DB / data source / client gọi API ngoài
├── model/                      # struct dùng chung + errors.go (bảng error code)
├── main.go                     # entrypoint — setup signal, start worker/job
├── go.mod                      # khai báo module + version
├── go.sum                      # checksum dependency
├── README.md                   # hướng dẫn repo
└── .gitignore                  # bỏ qua bin/, tmp/, vendor/
```

## Vai trò thư mục

- `worker/`: mỗi worker = 1 struct có method `Run(ctx)` chạy loop. KHÔNG chứa business logic dài — chỉ orchestrate.
- `job/`: scheduled job kích bởi cron (robfig/cron) hoặc chạy 1 lần rồi thoát. Khác `worker/` vì không tự loop liên tục.
- `service/`: business logic thật, worker/job gọi xuống.
- `repository/`: data access.
- `model/`: struct dùng chung (entity, DTO, config) + `errors.go` giữ bảng error code + typed error của cả repo.
- `main.go`: parse config, init DI thủ công, khởi động worker/job, handle signal (`SIGINT`/`SIGTERM`) để graceful shutdown.
- `infra/service-install/`: ví dụ — `service.service` (systemd), `install-windows.ps1` (dùng `sc.exe` với binary Go đã build + tham số).

## Rule

- Worker KHÔNG chứa business logic nặng — gọi xuống `service/`.
- KHÔNG `time.Sleep` trần trong loop dài — dùng `time.Ticker` hoặc `context.WithTimeout`.
- Graceful shutdown BẮT BUỘC: `context.Context` phải truyền xuống và được tôn trọng. Nhận `SIGINT`/`SIGTERM` → cancel context → chờ worker dừng → exit.
- Logging qua logger chuẩn (slog / zap / zerolog) — KHÔNG `fmt.Println`.
- Config nạp từ file + env (viper hoặc tự đọc); secret KHÔNG commit.
- Nếu service có > 3 worker / nhiều domain → cân nhắc nâng lên structured.
- Retry/backoff phải rõ cho lỗi tạm thời; không loop lỗi liên tục làm nghẽn log/CPU.
- Nếu expose health/metrics/admin endpoint, chỉ bind nội bộ hoặc sau proxy bảo vệ.
- Có script build/install/run tương ứng Docker, systemd, Windows Service, cron hoặc Task Scheduler.

## API response contract

Theo [`03-standards/API_RESPONSE_CONTRACT.md`](../../../03-standards/API_RESPONSE_CONTRACT.md), contract `api-1`. Service không phải web API, nhưng dính chuẩn ở 2 chỗ:

**1. Hợp đồng với hệ thống chạy service — health surface và exit code.**
- `/healthz` (liveness, KHÔNG gọi DB) + `/readyz` (readiness, có gọi dependency) bắt buộc khi service có HTTP surface. Hình dạng body và luật **miễn trừ error envelope** lấy nguyên ở [appendix](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md) section 1.6 — không tự chế body khác. Daemon chạy dài bắt buộc có cả 2 — probe của Docker/systemd vốn đã cần.
- Job chạy một lần rồi thoát (cron) thì **không** dựng HTTP server chỉ để có health endpoint. Ở đó **exit code LÀ contract, thay vai trò status code**: `0` = mọi bước BẮT BUỘC hoàn tất, `!= 0` = cần người can thiệp và alert bám vào đây. Phải liệt kê trong `docs/` bước/nguồn nào bắt buộc, nguồn nào best-effort (fail vẫn `0` nhưng log mức warn). Bản ghi hỏng giữa chừng đẩy vào bảng/thư mục lỗi kèm `trace_id` của lần chạy, KHÔNG đổ cả job. Luật đầy đủ: [appendix](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md) section 2.3.
- `trace_id` của job one-shot là `run_id` sinh lúc bắt đầu lần chạy, persist cùng bản ghi (appendix section 2.0).

**2. Consumer của API khác — nguồn sự thật để retry khác nhau theo loại upstream.**
- **Upstream tự khai `contract: api-1`:** đọc `error.code` + `error.retryable` trong body, KHÔNG suy từ status code trần, KHÔNG parse `message` để đoán loại lỗi.
- **Upstream bên thứ 3** (partner API, cổng thanh toán, hệ thống nội bộ cũ) — ca phổ biến nhất của service: **HTTP status code + header `Retry-After` CHÍNH LÀ nguồn sự thật**, vì đó là hợp đồng duy nhất họ thật sự tuân. Mỗi upstream một adapter riêng, map về taxonomy của mình rồi mới ra ngoài; cấm duck-type `body.error`. Luật đầy đủ: [appendix](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md) section 2.5.
- Sau khi đã map: `retryable: false` thì **dừng và đẩy sang dead-letter / bảng lỗi**, không backoff vô hạn; `retryable: true` mới backoff.
- Client gọi API ngoài đặt trong `repository/` ở mode này (`repository/` là vùng integration, không chỉ DB), mỗi upstream một file. Đừng để mã lỗi của bên thứ 3, hay chính `http.Response`, rò lên `worker/` hoặc `service/`.
- `model/errors.go`: nơi đặt type + bảng code, đúng 1 file cho cả repo — `service/` và `repository/` dùng chung, không mỗi chỗ tự khai lại.
- Code Go tham chiếu cho bảng 17 mã, `AppError`, và writer/middleware khi có HTTP surface: [`03-standards/snippets/go-api-contract.md`](../../../03-standards/snippets/go-api-contract.md) — copy phần section 1 vào `model/errors.go`, không import chung.

Nếu service có thêm HTTP endpoint nội bộ (admin, trigger, dashboard mini) thì các endpoint đó áp TOÀN BỘ contract như một API thật.

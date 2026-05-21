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
├── repository/                 # truy cập DB / data source
├── model/                      # struct dùng chung
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
- `model/`: struct dùng chung (entity, DTO, config).
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

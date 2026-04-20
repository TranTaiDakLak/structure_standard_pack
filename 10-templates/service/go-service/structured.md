# Service Structured — Go

## Khi nào dùng

- Service Go sống lâu, nhiều worker / job
- Nhiều tích hợp ngoài (DB, queue, HTTP, file)
- Cần test được business logic độc lập khỏi worker runtime
- Team ≥ 3 người cùng chạm

## Cây thư mục

```text
<service-name>/
├── docs/                           # tài liệu, flow, ghi chú kỹ thuật
├── infra/                          # config triển khai
│   └── service-install/            # systemd unit / Windows Service script / Dockerfile
├── scripts/                        # script build/run/migrate
├── config/                         # file cấu hình mẫu
├── tests/                          # test nằm ngoài source chính
│   ├── integration/                # test tích hợp (DB, queue thật)
│   └── e2e/                        # end-to-end (nếu service có entrypoint kích ngoài)
├── cmd/                            # mỗi binary 1 sub-folder
│   └── <service-name>/             # tên binary = tên folder
│       └── main.go                 # chỉ wiring + signal handling
├── internal/                       # code private, không export ngoài module
│   ├── app/                        # bootstrap (DI, config, logger)
│   ├── worker/                     # long-running loop — orchestrate usecase
│   ├── job/                        # scheduled job (robfig/cron)
│   ├── domain/                     # entity + rule cốt lõi, không biết HTTP/DB
│   ├── usecase/                    # application flow — orchestrate domain + adapter
│   ├── adapter/                    # cổng ra ngoài (hexagonal)
│   │   ├── repository/             # SQL/NoSQL implement port
│   │   ├── external/               # client gọi service ngoài
│   │   └── messaging/              # consumer/producer queue (RabbitMQ, Kafka...)
│   └── config/                     # load config runtime
├── go.mod                          # khai báo module + version
├── go.sum                          # checksum dependency
├── README.md                       # hướng dẫn repo
└── .gitignore
```

## Vai trò thư mục

- `cmd/<service>/main.go`: CHỈ wiring — parse config, init logger, build DI, register worker/job, start, handle signal.
- `internal/app/`: factory tạo dependency, gom wiring để `main.go` gọn.
- `internal/worker/`: mỗi worker struct có `Run(ctx)` — không chứa business logic; gọi `usecase`.
- `internal/job/`: job handler cho cron — cũng chỉ gọi `usecase`.
- `internal/domain/`: entity, value object, rule thuần. KHÔNG import framework.
- `internal/usecase/`: use case, phối hợp domain + adapter qua interface.
- `internal/adapter/`: implement port trong `usecase` — persistence, HTTP, messaging.
- `internal/config/`: struct config + loader từ file/env.

## Rule

- `domain` KHÔNG biết gì về DB, HTTP, queue, framework.
- `usecase` định nghĩa interface cho adapter; adapter implement ngược lên (DI đảo ngược).
- Worker/Job KHÔNG gọi trực tiếp `adapter` — đi qua `usecase`.
- Graceful shutdown: `ctx` từ `main.go` truyền xuống tất cả worker; `SIGTERM` → `cancel()` → chờ `wg.Wait()` → exit.
- Logger và metrics là cross-cutting — inject qua `app/`, không mỗi worker tự khởi tạo.
- Đừng tạo package con quá vụn. Structured là để dễ tìm code, không phải để đẹp chuẩn.
- Nếu chưa đủ lớn để justify `cmd/` + `internal/` → quay về simple.

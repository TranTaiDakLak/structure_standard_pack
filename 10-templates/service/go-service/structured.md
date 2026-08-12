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
│   ├── domain/                     # entity + rule cốt lõi + errors.go (bảng error code) — không biết HTTP/DB
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
- Unit test domain/usecase; integration test adapter với DB/queue/file thật khi rủi ro cao.
- Retry/backoff, idempotency, lock/concurrency phải ghi rõ cho worker/job có tác động dữ liệu.
- Deploy target phải có script install/run và log path rõ.

## API response contract

Theo [`03-standards/API_RESPONSE_CONTRACT.md`](../../../03-standards/API_RESPONSE_CONTRACT.md), contract `api-1`. Service không phải web API, nhưng dính chuẩn ở 2 chỗ:

**1. Hợp đồng với hệ thống chạy service — health surface và exit code.**
- `/healthz` (liveness, KHÔNG gọi DB) + `/readyz` (readiness, có gọi dependency) bắt buộc khi service có HTTP surface. Hình dạng body và luật **miễn trừ error envelope** lấy nguyên ở [appendix](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md) section 1.6 — không tự chế body khác. Daemon chạy dài bắt buộc có cả 2 — probe của Docker/systemd vốn đã cần.
- Job chạy một lần rồi thoát (cron) thì **không** dựng HTTP server chỉ để có health endpoint. Ở đó **exit code LÀ contract, thay vai trò status code**: `0` = mọi bước BẮT BUỘC hoàn tất, `!= 0` = cần người can thiệp và alert bám vào đây. Phải liệt kê trong `docs/` bước/nguồn nào bắt buộc, nguồn nào best-effort (fail vẫn `0` nhưng log mức warn). Bản ghi hỏng giữa chừng đẩy vào bảng/thư mục lỗi kèm `trace_id` của lần chạy, KHÔNG đổ cả job. Luật đầy đủ: [appendix](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md) section 2.3.
- `trace_id` của job one-shot là `run_id` sinh lúc bắt đầu lần chạy, persist cùng bản ghi (appendix section 2.0).

**2. Consumer của API khác — nguồn sự thật để retry khác nhau theo loại upstream.**
- **Upstream tự khai `contract: api-1`:** đọc `error.code` + `error.retryable` trong body, KHÔNG suy từ status code trần, KHÔNG parse `message` để đoán loại lỗi.
- **Upstream bên thứ 3** (partner API, cổng thanh toán, hệ thống nội bộ cũ) — ca phổ biến nhất của service: **HTTP status code + header `Retry-After` CHÍNH LÀ nguồn sự thật**, vì đó là hợp đồng duy nhất họ thật sự tuân. Mỗi upstream một adapter riêng trong `internal/adapter/external/`, map về taxonomy của mình rồi mới ra ngoài; cấm duck-type `body.error`. Luật đầy đủ: [appendix](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md) section 2.5.
- Sau khi đã map: `retryable: false` thì **dừng và đẩy sang dead-letter / bảng lỗi**, không backoff vô hạn; `retryable: true` mới backoff. Rule "retry/backoff, idempotency, lock/concurrency phải ghi rõ" ở trên lấy đây làm căn cứ máy đọc được.
- Adapter boundary — `internal/adapter/external/` và `internal/adapter/messaging/` — là nơi DUY NHẤT biết hình dạng lỗi của upstream; đừng để mã lỗi của bên thứ 3 rò vào `internal/usecase/` hay `internal/worker/`.
- `internal/domain/errors.go`: nơi đặt type + bảng code, đúng 1 file cho cả repo. Đặt ở `domain` chứ **không** ở `adapter` để `usecase` và `worker` rẽ nhánh theo `code` mà không phải import ngược vào adapter — giữ đúng chiều DI đảo ngược của template này. Adapter chỉ map lỗi bên ngoài **vào** tập code này.
- Code Go tham chiếu: [`03-standards/snippets/go-api-contract.md`](../../../03-standards/snippets/go-api-contract.md) — section 1 (`Code` + `AppError` + `Retryable()`, KHÔNG import `net/http`) copy vào `internal/domain/errors.go`; section 2–4 (writer + middleware) chỉ copy khi service có HTTP surface, đặt trong package writer dưới `internal/adapter/`, không import chung.

Nếu service có thêm HTTP endpoint nội bộ (admin, trigger, dashboard mini) thì các endpoint đó áp TOÀN BỘ contract như một API thật.

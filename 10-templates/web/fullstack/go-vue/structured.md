# Fullstack Structured — Go + Vue

## Khi nào dùng

- Product sống lâu, BE và FE đều phình
- Nhiều worker/job, nhiều integration ngoài (DB, queue, S3, payment...)
- Cần test business logic độc lập khỏi HTTP handler và worker runtime
- Team ≥ 3 người cùng chạm
- Vẫn 1 product, chưa đủ tách microservice

## Cây thư mục

```text
<product-name>/
├── docs/                                  # tài liệu chung
│   ├── architecture.md                    # quyết định kiến trúc chính, boundary app
│   ├── runbook.md                         # start/stop, deploy, rollback, backup/restore
│   ├── server-registration.md             # project name, domain, port, network, scale profile
│   └── security.md                        # auth, CORS/CSRF, secret, audit, permission
├── infra/
│   ├── compose/
│   │   ├── dev.yml                        # compose tổng dev local
│   │   ├── ci.yml                         # compose cho CI test
│   │   └── prod.yml                       # production source of truth: network, db, redis, app
│   ├── nginx/                             # reverse proxy production
│   └── systemd/                           # unit quản lý docker compose trên Linux server
├── scripts/
│   ├── dev-up.sh
│   ├── test.sh                            # unit + integration test entrypoint
│   ├── migrate.sh                         # apply migration có kiểm soát
│   ├── deploy.sh                          # deploy production bằng compose/systemd wrapper
│   ├── rollback.sh                        # rollback image/tag hoặc compose version trước đó
│   ├── backup.sh                          # tar storage + pg_dump
│   └── restore.sh
├── config/                                # config mẫu chung product-level
├── apps/
│   ├── api/                               # Go module — chứa api + worker + job
│   │   ├── docs/                          # tài liệu riêng BE
│   │   │   └── openapi.yaml               # contract API public/admin nếu có HTTP API ổn định
│   │   ├── cmd/
│   │   │   ├── api/main.go                # entrypoint HTTP — chỉ wiring
│   │   │   └── worker/main.go             # entrypoint worker — wiring + signal
│   │   ├── internal/
│   │   │   ├── app/                       # bootstrap (DI, config, logger) — share api+worker
│   │   │   ├── domain/                    # entity + rule cốt lõi
│   │   │   ├── usecase/                   # application flow — share api+worker
│   │   │   ├── adapter/                   # hexagonal — cổng ra ngoài
│   │   │   │   ├── http/                  # handler, middleware — binary api dùng
│   │   │   │   │   └── health/            # /healthz, /readyz, /metrics nếu bật
│   │   │   │   ├── repository/            # SQL/NoSQL — cả 2 binary dùng
│   │   │   │   ├── messaging/             # queue consumer/producer — chủ yếu worker
│   │   │   │   └── external/              # client gọi service ngoài
│   │   │   ├── platform/                  # logging, metrics, tracing, clock, id generator
│   │   │   ├── worker/                    # long-running loop, gọi usecase
│   │   │   ├── job/                       # scheduled job (robfig/cron)
│   │   │   └── config/                    # struct config + loader
│   │   ├── migrations/                    # SQL migrations
│   │   ├── tests/
│   │   │   ├── integration/               # với DB/queue thật
│   │   │   └── e2e/                       # qua HTTP công khai
│   │   ├── config/
│   │   ├── storage/
│   │   │   └── .gitkeep
│   │   ├── Dockerfile                     # multi-stage, multi-target (api/worker)
│   │   ├── docker-compose.yml             # standalone/smoke/deploy API riêng khi cần
│   │   ├── .env.example
│   │   ├── .gitignore
│   │   ├── go.mod                         # 1 module duy nhất
│   │   ├── go.sum
│   │   └── README.md
│   ├── admin-web/                         # Vue admin
│   │   ├── docs/
│   │   ├── public/
│   │   ├── src/
│   │   │   ├── components/                # component generic
│   │   │   ├── composables/
│   │   │   ├── features/                  # logic theo domain
│   │   │   ├── pages/
│   │   │   ├── router/
│   │   │   ├── services/                  # API client
│   │   │   ├── stores/
│   │   │   ├── types/
│   │   │   ├── App.vue
│   │   │   └── main.ts
│   │   ├── tests/
│   │   ├── Dockerfile
│   │   ├── docker-compose.yml
│   │   ├── .env.example
│   │   ├── vite.config.ts
│   │   ├── package.json
│   │   └── README.md
│   └── client-web/                        # Vue end-user (optional, structure giống admin-web)
│       └── ...
├── README.md
└── .gitignore
```

## Vai trò thư mục

- `apps/api/cmd/api/main.go`: parse config, init logger, build DI từ `internal/app`, register routes từ `adapter/http`, start server, handle signal.
- `apps/api/cmd/worker/main.go`: parse config, init logger, build DI từ `internal/app`, register worker + job, start, handle `SIGTERM` → cancel → `wg.Wait()` → exit.
- `internal/app/`: factory dependency — share giữa 2 binary. 1 `NewContainer(cfg)` trả về tập DB pool, repo, usecase. `main.go` chỉ pick cái cần.
- `internal/domain/`: entity, value object, rule thuần. KHÔNG import framework.
- `internal/usecase/`: định nghĩa interface cho adapter; adapter implement ngược. **Đây là chỗ share thật giữa api và worker.**
- `internal/adapter/http/`: chỉ binary `api` import.
- `internal/adapter/http/health/`: endpoint `/healthz` chỉ kiểm tra process sống; `/readyz` kiểm tra DB/queue dependency; `/metrics` chỉ bật khi có hệ thống scrape.
- `internal/adapter/messaging/`: chủ yếu binary `worker`. API có thể publish event qua đây.
- `internal/platform/`: cross-cutting code không thuộc domain như logger, metrics, tracing, request id, clock; inject qua `internal/app`.
- `internal/worker/` + `internal/job/`: orchestrate `usecase`, KHÔNG chứa business logic.
- `migrations/`: SQL migrations dùng chung 2 binary — apply 1 lần qua `scripts/` hoặc init job, KHÔNG để binary tự migrate khi start.
- `infra/compose/prod.yml`: source of truth cho production khi product có nhiều app; app-level compose chỉ dùng cho smoke test hoặc deploy tách rời.
- `docs/runbook.md`: tài liệu vận hành bắt buộc cho production: lệnh deploy, rollback, backup, restore, log, ports, cron, domain, owner.
- `docs/server-registration.md`: ghi thông tin register lên server: deploy path, scale profile, domain, port registry, DB/Redis mode, Docker networks, resource limit.
- `docs/security.md`: ghi rõ auth/session, RBAC admin, CORS/CSRF, rate limit, secret, audit log, quyền DB user.

## Rule

- **Docker bắt buộc** — không có path bypass.
- `domain` KHÔNG biết DB, HTTP, queue, framework.
- `usecase` định nghĩa interface; adapter implement (DI đảo ngược).
- API handler KHÔNG gọi trực tiếp `adapter/repository` — đi qua `usecase`.
- Worker/Job KHÔNG gọi trực tiếp `adapter` — đi qua `usecase`.
- 2 binary share `internal/usecase` nhưng wiring khác nhau ở `cmd/*/main.go`.
- Graceful shutdown: `ctx` từ `main.go` truyền xuống tất cả worker + HTTP server.
- Logger và metrics inject qua `app/`, không tự khởi tạo trong handler/worker.
- Migrations apply qua `scripts/` hoặc init job riêng, KHÔNG tự migrate ngầm khi start.
- Tool migration phải thống nhất trong repo (`goose` hoặc `golang-migrate`) và được gọi qua `scripts/migrate.sh`.
- Health/readiness là bắt buộc: `/healthz` không gọi DB; `/readyz` được phép kiểm tra DB/queue/cache.
- API ổn định cho FE hoặc tích hợp ngoài phải có `apps/api/docs/openapi.yaml` hoặc tài liệu contract tương đương. Không bắt buộc codegen.
- Production log dùng JSON, có request id/correlation id. Worker/job log phải có job name, run id, duration, status.
- Metrics/tracing là cross-cutting, đặt dưới `internal/platform/` và inject qua `internal/app`.
- Security production: không wildcard CORS, không hardcode secret, rate limit endpoint public/admin nhạy cảm, audit log thao tác admin quan trọng.
- CI tối thiểu: lint, unit test, integration test với `infra/compose/ci.yml`, build Docker image, migration dry-run.
- Deploy production qua `scripts/deploy.sh`; rollback qua `scripts/rollback.sh`; không SSH vào server chạy lệnh rời rạc không ghi lại.
- Khi deploy lên Linux server nhiều dự án, phải cập nhật `PORT-REGISTRY.md` và `DOMAIN-REGISTRY.md`; app/API/web mặc định dùng `expose`, không bind host port.
- Compose production dùng container name theo chuẩn nội bộ và external networks public/internal/data đúng nhu cầu.
- Scale nhỏ/vừa dùng shared PostgreSQL/Redis với DB user và Redis prefix riêng; chỉ dedicated khi project quan trọng, workload nặng hoặc cần cô lập.
- FE build ra Docker image nginx serve tĩnh; production qua `infra/nginx/` reverse proxy.
- Backup script đụng cả `apps/api/storage/` và DB dump. Backup phải có retention, checksum, và restore drill định kỳ.
- Đừng tạo package con quá vụn. Structured để dễ tìm code, không phải để đẹp chuẩn.
- Nếu worker khác hẳn API về domain, không còn share `usecase` → cân nhắc tách repo riêng dùng `service/go-service`.

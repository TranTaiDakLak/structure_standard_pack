# Fullstack Multi-service Structured — Go + Vue

## Khi nào dùng

- Product multi-service sống lâu, mỗi bounded context có nghiệp vụ thật và phình
- Nhiều integration ngoài per module (DB, queue, S3, payment, external API)
- Cần test business logic của từng module độc lập khỏi HTTP handler và worker runtime
- Team ≥ 3 người, chia nhau theo module
- Vẫn 1 product / 1 deployable — **chưa** đủ lý do tách microservices

## Cây thư mục

```text
<product-name>/
├── docs/
│   ├── architecture.md                    # danh sách module, quan hệ gọi nhau qua port, schema mỗi module
│   ├── runbook.md                         # start/stop, deploy, rollback, backup/restore
│   ├── server-registration.md             # project name, domain, port, network, scale profile
│   └── security.md                        # auth, RBAC, CORS/CSRF, secret, audit, permission
├── infra/
│   ├── compose/
│   │   ├── dev.yml                        # compose tổng dev local
│   │   ├── ci.yml                         # compose cho CI test
│   │   └── prod.yml                       # production source of truth: network, db, redis, app
│   ├── nginx/
│   └── systemd/
├── scripts/
│   ├── dev-up.sh
│   ├── test.sh                            # unit + integration test entrypoint
│   ├── migrate.sh
│   ├── deploy.sh
│   ├── rollback.sh
│   ├── backup.sh
│   └── restore.sh
├── config/
├── apps/
│   ├── api/                               # 1 Go module — api + worker, nhiều service module
│   │   ├── docs/
│   │   │   └── openapi.yaml               # contract API public/admin nếu ổn định
│   │   ├── cmd/
│   │   │   ├── api/main.go                # entrypoint HTTP — chỉ gọi internal/app build + start
│   │   │   └── worker/main.go             # entrypoint worker — build + register worker + signal
│   │   ├── internal/
│   │   │   ├── app/                       # composition root: chọn & wiring module nào được mount
│   │   │   │   ├── container.go           # NewContainer(cfg) → platform + tất cả module module.Module
│   │   │   │   ├── http.go                # mount HTTP của module vào router
│   │   │   │   └── worker.go              # mount worker/job của module
│   │   │   ├── platform/                  # cross-cutting: logger, metrics, tracing, db, http, clock, id
│   │   │   ├── shared/                    # kiểu/dùng chung thuần (errors, pagination) — tiết chế, không thành thùng rác
│   │   │   └── services/                  # ⭐ mỗi sub-folder = 1 bounded context hexagonal
│   │   │       ├── identity/
│   │   │       │   ├── domain/            # entity + rule thuần, KHÔNG import framework/DB
│   │   │       │   ├── usecase/           # application flow + interface port của module
│   │   │       │   ├── adapter/
│   │   │       │   │   ├── http/          # handler, middleware riêng module
│   │   │       │   │   ├── repository/    # SQL chạm bảng/schema của module
│   │   │       │   │   ├── external/      # client gọi service ngoài
│   │   │       │   │   └── messaging/     # queue consumer/producer của module
│   │   │       │   └── module.go          # đăng ký routes + worker + job + dependency của module này
│   │   │       ├── billing/               # cấu trúc giống identity/
│   │   │       └── catalog/
│   │   ├── migrations/                    # SQL migrations, prefix theo module (001_identity_*)
│   │   ├── tests/
│   │   │   ├── integration/               # với DB/queue thật, theo module
│   │   │   └── e2e/                       # qua HTTP công khai
│   │   ├── config/
│   │   ├── storage/
│   │   │   └── .gitkeep
│   │   ├── Dockerfile                     # multi-stage, multi-target (api/worker)
│   │   ├── docker-compose.yml
│   │   ├── .env.example
│   │   ├── .gitignore
│   │   ├── go.mod                         # 1 module duy nhất
│   │   ├── go.sum
│   │   └── README.md
│   ├── admin-web/                         # Vue admin (như web/frontend-only/vue-vite structured)
│   │   ├── docs/  public/
│   │   ├── src/
│   │   │   ├── components/  composables/  features/  pages/  router/
│   │   │   ├── services/                  # API client
│   │   │   ├── stores/  types/
│   │   │   ├── App.vue
│   │   │   └── main.ts
│   │   ├── tests/
│   │   ├── Dockerfile  docker-compose.yml  .env.example  vite.config.ts  package.json  README.md
│   └── client-web/                        # Vue end-user (optional, structure giống admin-web)
│       └── ...
├── README.md
└── .gitignore
```

## Vai trò thư mục

- `apps/api/cmd/api/main.go` / `cmd/worker/main.go`: chỉ wiring — build container từ `internal/app`, mount HTTP hoặc worker, start, handle signal.
- `internal/app/container.go`: composition root. `NewContainer(cfg)` khởi tạo `platform/` rồi build từng module qua `module.New(...)`. `main.go` chỉ pick HTTP hoặc worker.
- `internal/app/http.go` / `worker.go`: gom việc mount route/worker của tất cả module — thêm module mới = thêm 1 dòng ở đây, không sửa rải rác.
- `internal/platform/`: cross-cutting không thuộc domain — logger, metrics, tracing, db pool, http server gốc, clock, id generator. Inject xuống module.
- `internal/shared/`: kiểu dùng chung thuần (error types, pagination, result). Tiết chế — nếu bắt đầu chứa nghiệp vụ thì sai chỗ.
- `internal/services/<module>/domain/`: entity, value object, rule thuần. KHÔNG biết DB/HTTP/queue.
- `internal/services/<module>/usecase/`: application flow của module + **định nghĩa interface port** (cả port nội bộ cho adapter implement, lẫn port export cho module khác gọi).
- `internal/services/<module>/adapter/`: hexagonal — `http` (binary api dùng), `repository` (chạm bảng riêng), `external`, `messaging` (chủ yếu worker).
- `internal/services/<module>/module.go`: nơi module tự khai báo dependency, routes, worker, job. `app/` chỉ gọi `module.New()` + `module.MountHTTP()` / `module.RegisterWorkers()`.
- `migrations/`: SQL chung 1 DB, prefix module để biết bảng thuộc ai; apply qua `scripts/`, KHÔNG tự migrate khi start.
- `tests/integration`, `tests/e2e`: test với dependency thật, tổ chức theo module.
- `docs/architecture.md`: bắt buộc — bản đồ module, ai gọi ai qua usecase port nào, bảng/schema mỗi module sở hữu.

## Rule

- **Docker bắt buộc** — không có path bypass.
- **1 deployable nhiều module** — KHÔNG network call giữa các module trong cùng process; module gọi nhau bằng **gọi hàm qua usecase port**, in-process.
- `domain` của module KHÔNG biết DB, HTTP, queue, framework.
- `usecase` định nghĩa interface; adapter implement (DI đảo ngược). HTTP handler và worker KHÔNG gọi thẳng `repository` — đi qua `usecase`.
- **Module A phụ thuộc module B chỉ qua port export ở `B/usecase`** — cấm import `B/adapter/repository`, `B/domain`. Quan hệ này phải xuất hiện trong `docs/architecture.md`.
- **Mỗi module sở hữu bảng/schema riêng** — không join chéo bảng module khác; cần dữ liệu thì gọi usecase của module sở hữu.
- `internal/app` là chỗ DUY NHẤT biết toàn bộ module — thêm/bớt module chỉ sửa ở đây + folder module.
- 2 binary (api/worker) share `internal/services/*/usecase` nhưng wiring khác ở `cmd/*/main.go`.
- Graceful shutdown: `ctx` từ `main.go` truyền xuống mọi worker + HTTP server.
- Logger/metrics/tracing inject qua `platform/` + `app/`, không tự khởi tạo trong module; log/metrics gắn nhãn `module`.
- Migrations apply qua `scripts/migrate.sh`, tool thống nhất (`goose`/`golang-migrate`).
- `/healthz` không gọi DB; `/readyz` kiểm tra dependency của **mọi module được mount**.
- API ổn định cho FE/tích hợp ngoài có `apps/api/docs/openapi.yaml`. Không bắt buộc codegen.
- Security production: không wildcard CORS, không hardcode secret, rate limit endpoint nhạy cảm, audit log thao tác admin.
- CI tối thiểu: lint, unit test (domain/usecase từng module), integration test qua `infra/compose/ci.yml`, build image, migration dry-run.
- Deploy qua `scripts/deploy.sh`, rollback qua `scripts/rollback.sh`.
- Khi deploy Linux nhiều dự án: `expose` mặc định, cập nhật `PORT-REGISTRY.md` + `DOMAIN-REGISTRY.md`, networks public/internal/data, shared PostgreSQL/Redis với DB user + Redis prefix riêng cho scale `shared`; dedicated DB/Redis khi `dedicated`/`isolated`.
- FE build ra image nginx serve tĩnh; production qua `infra/nginx/`.
- Backup đụng `apps/api/storage/` + DB dump, có retention, checksum, restore drill định kỳ.

## Chống over-engineering

- **Đừng tách module quá sớm** — chỉ tạo `services/<module>/` cho bounded context thật có nghiệp vụ riêng. 2 cái CRUD na ná nhau không đáng tách 2 module.
- **Đừng full hexagonal cho module mỏng** — module chỉ có vài endpoint CRUD có thể giữ phẳng kiểu `simple.md` ngay cả khi các module khác đã hexagonal. Không bắt mọi module cùng độ sâu.
- **Đừng biến in-process call thành RPC** — còn 1 deployable thì gọi hàm, không HTTP/gRPC nội bộ.
- **`shared/` và `platform/` không phải bãi rác** — thứ thuộc 1 module thì để trong module đó.
- Khi 1 module cần deploy/scale/đội ngũ độc lập, giao tiếp đã thưa và ổn định → **tách repo riêng** (`web/backend-only/go` hoặc `service/go-service`), KHÔNG giữ trong repo này tới mức thành cụm microservices không quản nổi.

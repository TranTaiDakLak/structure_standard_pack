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
│   ├── api-contract.md                    # contract api-1, bảng domain error code, endpoint ngoại lệ
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

## API response contract

Theo [`03-standards/API_RESPONSE_CONTRACT.md`](../../../../03-standards/API_RESPONSE_CONTRACT.md), contract `api-1` — không tự chế hình dạng response, không tự chế bảng mã lỗi.

- Response helper + writer: `apps/api/internal/platform/` — writer nằm cạnh logger/metrics/db pool/http server gốc, inject xuống module qua `internal/app`. **Nửa `status` của bảng mã (`status(code) int`) nằm ở ĐÂY, cùng package với writer**: status chỉ là phép chiếu của code sang một transport, worker không dùng tới.
- File khai báo error code, đúng 1 file cho cả repo: `apps/api/internal/shared/errors.go` — cây đã khai sẵn "errors, pagination" ở `internal/shared/`, dùng đúng chỗ đó, KHÔNG đẻ folder mới. Chứa `Code`, `AppError` và **nửa `retryable`**; KHÔNG import `net/http`, để `domain`/`usecase`/worker rẽ nhánh theo code mà không biết HTTP (chuẩn section 6.1). Type `{items, page}` cũng lấy từ đây.
- Exception/recover middleware toàn cục: `apps/api/internal/platform/` (http server gốc), gắn ở `internal/app/http.go` **trước khi** mount module nào.
- **Rule cứng 1 (không phải gợi ý)**: mọi module ghi response qua ĐÚNG MỘT writer ở `internal/platform/`; `services/<module>/adapter/http/` cấm tự `json.Marshal` rồi ghi thẳng vào `http.ResponseWriter`. Đây là bản cụ thể hoá của rule "module chỉ phụ thuộc nhau qua usecase port" — hình dạng response là contract chung của repo. `domain`/`usecase` trả typed error từ `internal/shared/`, chỉ `adapter/http` mới được chọn HTTP status. `/readyz` (rule sẵn có) là ngoại lệ tường minh: khi fail vẫn trả 503 nhưng body giữ hình dạng health ở [appendix](../../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md) section 1.6 — `{"status":"degraded","checks":{...}}` — không plain text, và cũng KHÔNG bọc `{"error":{...}}` vì bọc là mất thông tin check nào đang chết. Mã `not_ready`/`unavailable` dành cho endpoint nghiệp vụ khi đang deploy hoặc dependency chết, không dành cho probe.
- **Khuyến nghị mạnh cho riêng stack này**: 1 integration test smoke ở `apps/api/tests/integration/` duyệt mọi route đã mount ở `internal/app/http.go` — một request lỗi cố ý assert body có đủ `error.code` + `error.trace_id`, và một request tới route list assert body có đúng 2 khóa `items` + `page`. Nhiều module cùng viết `adapter/http/` là nơi drift hình dạng dễ xảy ra nhất, và test này bắt được ngay lần thêm module mới đầu tiên. Không bắt buộc — xem chuẩn section 13.
- **List tổng hợp lọc theo dữ liệu module khác** (`GET /api/v1/orders?payment_status=...`) có 2 đường hợp lệ theo chuẩn section 8.2, chọn 1: (a) khai endpoint trong `docs/api-contract.md` là endpoint tổng hợp, **miễn `total`**, nhưng phải có trần cứng cho `limit` và trả 400 `bad_request` khi client xin vượt trần; (b) dựng **read model read-only** do đúng 1 module sở hữu (adapter `repository/` của module đó), khai chủ sở hữu trong `docs/architecture.md`, cập nhật qua event của module nguồn. Read model đọc chéo là hợp lệ — thứ rule module cấm là **ghi** chéo và **join trực tiếp bảng nghiệp vụ** của module khác. List của một module vẫn bắt buộc `total`.
- Domain error code đặt `<module>.<reason>`, `<module>` KHỚP ĐÚNG tên folder trong `internal/services/`: `identity.token_expired`, `billing.already_paid`. Đọc mã lỗi là biết ngay chủ sở hữu.
- **Mỗi domain code phải có entry ở CẢ HAI nửa bảng** — `retryable` ở `internal/shared/errors.go`, `status` ở `internal/platform/` (chuẩn section 6.2). `billing` trả `billing.already_paid`; `catalog` gọi qua usecase port của `billing` và nhận đúng mã đó, rồi để lỗi đi ra qua `adapter/http` của chính mình — status vẫn tra từ đúng bảng chung. KHÔNG module nào tự chọn status.
- `docs/api-contract.md`: contract version, bảng domain error code, danh sách endpoint ngoại lệ.
- Code Go tham chiếu (bảng 17 mã, writer, middleware, wiring 404/405 của `chi`, bẫy DTO, PATCH null-vs-absent): [`03-standards/snippets/go-api-contract.md`](../../../../03-standards/snippets/go-api-contract.md) — copy section 1 vào `apps/api/internal/shared/errors.go`, section 2–3 (writer + request id/recover) vào `internal/platform/`, section 4 (wiring 404/405) áp ở `internal/app/http.go` nơi mount router; đổi `package` cho khớp folder, không import chung.

Không đẻ layer mới cho việc này — mọi path trên đều nằm trong cây thư mục ở trên.

## Chống over-engineering

- **Đừng tách module quá sớm** — chỉ tạo `services/<module>/` cho bounded context thật có nghiệp vụ riêng. 2 cái CRUD na ná nhau không đáng tách 2 module.
- **Đừng full hexagonal cho module mỏng** — module chỉ có vài endpoint CRUD có thể giữ phẳng kiểu `simple.md` ngay cả khi các module khác đã hexagonal. Không bắt mọi module cùng độ sâu.
- **Đừng biến in-process call thành RPC** — còn 1 deployable thì gọi hàm, không HTTP/gRPC nội bộ.
- **`shared/` và `platform/` không phải bãi rác** — thứ thuộc 1 module thì để trong module đó.
- Khi 1 module cần deploy/scale/đội ngũ độc lập, giao tiếp đã thưa và ổn định → **tách repo riêng** (`web/backend-only/go` hoặc `service/go-service`), KHÔNG giữ trong repo này tới mức thành cụm microservices không quản nổi.

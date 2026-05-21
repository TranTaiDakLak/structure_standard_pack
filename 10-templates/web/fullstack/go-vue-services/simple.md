# Fullstack Multi-service Simple — Go + Vue

## Khi nào dùng

- team 2–5 người, đã biết product chia thành vài bounded context
- 1 Go backend (1 binary) gồm nhiều **service module** còn nhỏ + Vue admin web (+ optional client-web, optional worker)
- Mỗi module còn mỏng, chưa cần tách hexagonal `domain/usecase/adapter`
- Deploy Linux qua Docker compose

## Cây thư mục

```text
<product-name>/
├── docs/                              # tài liệu chung của product
│   ├── architecture.md                # danh sách module, ai gọi ai, schema mỗi module sở hữu
│   ├── runbook.md                     # vận hành: deploy, rollback, backup/restore, log, port
│   └── server-registration.md         # project name, domain, port, network, scale profile
├── infra/
│   ├── compose/
│   │   ├── dev.yml                    # compose tổng — dev local all-in-one
│   │   └── prod.yml                   # production source of truth
│   ├── nginx/                         # reverse proxy production (FE + /api)
│   └── systemd/                       # (optional) unit quản lý docker compose
├── scripts/
│   ├── dev-up.sh                      # wrap docker compose -f infra/compose/dev.yml up
│   ├── migrate.sh                     # apply DB migration có kiểm soát
│   ├── deploy.sh                      # deploy production
│   ├── rollback.sh                    # rollback image/tag trước đó
│   ├── backup.sh                      # tar gz apps/*/storage + pg_dump
│   └── restore.sh
├── config/                            # config mẫu chung (nếu share giữa apps)
├── apps/
│   ├── api/                           # 1 Go module, nhiều service module trong 1 process
│   │   ├── cmd/
│   │   │   ├── api/main.go            # composition root: mount tất cả service module + start HTTP
│   │   │   └── worker/main.go         # (optional) chạy background cho các module
│   │   ├── platform/                  # cross-cutting dùng chung mọi module
│   │   │   ├── db/                     # kết nối DB pool
│   │   │   ├── httpserver/             # router gốc, middleware chung, /healthz /readyz
│   │   │   ├── logger/                 # JSON logger + request id
│   │   │   └── config/                 # load + struct config
│   │   ├── services/                  # ⭐ mỗi sub-folder = 1 bounded context
│   │   │   ├── identity/
│   │   │   │   ├── handler.go          # HTTP handler của module (đăng ký route)
│   │   │   │   ├── service.go          # business logic của module
│   │   │   │   ├── repository.go       # truy cập bảng/schema của module này
│   │   │   │   ├── model.go            # struct domain của module
│   │   │   │   └── port.go             # interface module này EXPORT cho module khác dùng
│   │   │   ├── billing/                # cấu trúc giống identity/
│   │   │   └── catalog/
│   │   ├── migrations/                # SQL migrations (đặt tên có prefix module: 001_identity_*)
│   │   ├── config/                    # config mẫu (config.example.yaml)
│   │   ├── storage/                   # runtime data mount volume — chỉ .gitkeep
│   │   │   └── .gitkeep
│   │   ├── Dockerfile                 # multi-stage, multi-target (api / worker)
│   │   ├── docker-compose.yml         # standalone/smoke/deploy API riêng khi cần
│   │   ├── .env.example               # template env, KHÔNG commit .env thật
│   │   ├── .gitignore                 # ignore storage/* trừ .gitkeep
│   │   ├── go.mod                     # 1 module duy nhất
│   │   ├── go.sum
│   │   └── README.md
│   ├── admin-web/                     # Vue admin (cấu trúc như web/frontend-only/vue-vite simple)
│   │   ├── public/
│   │   ├── src/
│   │   │   ├── assets/  components/  pages/  router/  stores/
│   │   │   ├── api/                   # wrapper gọi apps/api
│   │   │   ├── App.vue
│   │   │   └── main.ts
│   │   ├── Dockerfile                 # build Vue → nginx serve static
│   │   ├── docker-compose.yml
│   │   ├── .env.example               # VITE_API_BASE_URL...
│   │   ├── vite.config.ts
│   │   ├── package.json
│   │   └── README.md
│   └── client-web/                    # (optional) Vue end-user — cấu trúc giống admin-web/
│       └── ...
├── README.md
└── .gitignore
```

## Vai trò thư mục

- `apps/api/cmd/api/main.go`: composition root — load config, init `platform/`, gọi từng module để đăng ký route vào router gốc, start server, handle signal. KHÔNG chứa business logic.
- `apps/api/platform/`: code không thuộc nghiệp vụ module nào — db pool, router gốc, middleware, logger, config. Mọi module nhận dependency từ đây.
- `apps/api/services/<module>/`: 1 bounded context. `handler.go` đăng ký route; `service.go` chứa nghiệp vụ; `repository.go` chỉ chạm bảng/schema của chính module; `port.go` khai báo interface mà module export cho module khác.
- `apps/api/services/<module>/port.go`: chỗ duy nhất module khác được phép phụ thuộc. Module A gọi B qua `billing.Port`, KHÔNG import `billing/repository.go`.
- `apps/api/migrations/`: SQL migration chung 1 DB; đặt tên prefix theo module để biết bảng thuộc ai (`001_identity_users.sql`).
- `apps/api/storage/`: mount volume runtime; commit chỉ `.gitkeep`.
- `apps/admin-web/`, `apps/client-web/`: 2 Vue project riêng, build độc lập ra Docker image nginx serve tĩnh.
- `infra/compose/dev.yml`: dev local 1 lệnh (api + db + redis + FE). `infra/compose/prod.yml`: source of truth production.
- `docs/architecture.md`: bắt buộc — liệt kê module, quan hệ gọi nhau qua port nào, bảng mỗi module sở hữu. Đây là thứ giữ multi-service không loạn.
- `docs/server-registration.md`: deploy path, scale profile, domain, port registry, DB/Redis mode, networks, resource limit.

## Worker — thêm khi nào

Không có worker mặc định. Khi 1 module cần background (consume queue, scheduled sync):

1. Thêm `apps/api/cmd/worker/main.go` — wiring + signal handler, mount worker của các module cần
2. Mỗi module cần background thêm 1 file (vd `services/billing/worker.go`) gọi xuống `service.go` của chính module
3. Update `Dockerfile` build thêm target `worker`, `docker-compose.yml` thêm service `worker` cùng image khác CMD
4. Worker gọi `service.go` của module — KHÔNG copy logic, KHÔNG chạm module khác trừ qua `port.go`

## Rule

- **Docker bắt buộc** — dev cũng qua compose, không có path chạy bare metal.
- **1 binary, nhiều module** — tất cả module mount chung `cmd/api`; KHÔNG mỗi module 1 container.
- **Module chỉ phụ thuộc nhau qua `port.go`** — cấm import `repository`/`model`/bảng DB của module khác.
- **Mỗi module sở hữu bảng/schema riêng** — không join chéo; đọc dữ liệu module khác qua port của nó.
- Business logic ở `service.go`, KHÔNG ở `handler.go` hay `main.go`.
- Mỗi app có `.env.example`; `.env` thật KHÔNG commit. `storage/` chỉ `.gitkeep` + `.gitignore` có `storage/*` + `!storage/.gitkeep`.
- Có DB thì phải có `migrations/` + `scripts/migrate.sh`; binary KHÔNG tự migrate ngầm khi start. Tool migration thống nhất (`goose`/`golang-migrate`).
- Backend phải có `/healthz` (process sống) và `/readyz` (kiểm tra dependency của mọi module mount).
- Graceful shutdown ở worker BẮT BUỘC: `SIGTERM` → cancel ctx → wait → exit.
- FE gọi BE qua `VITE_API_BASE_URL`, KHÔNG hardcode URL.
- JSON log có request id; log/metrics nên gắn nhãn `module`. Thao tác admin quan trọng có audit log.
- CORS/CSRF/rate limit cấu hình theo domain thật, không wildcard production.
- Khi deploy Linux nhiều dự án: cập nhật `PORT-REGISTRY.md` + `DOMAIN-REGISTRY.md`; app/API/web mặc định `expose`, không bind host port; shared PostgreSQL/Redis với DB user + Redis prefix riêng cho scale `shared`.
- Chưa cần monorepo tool (turborepo, nx). 1 `go.mod` + `package.json` per Vue app là đủ.

## Khi nào nâng lên `structured.md`

Nâng khi có ≥ 2 tín hiệu:

- 1 module có `service.go` > ~300 dòng hoặc nhiều luồng nghiệp vụ trộn nhau
- cần test nghiệp vụ module độc lập khỏi HTTP handler / DB thật
- module bắt đầu có nhiều integration ngoài (queue, S3, payment) trộn vào `service.go`
- nhiều người sửa cùng 1 module gây xung đột vì thiếu ranh giới `domain`/`usecase`/`adapter`

## Khi nào KHÔNG nên ở template này

- nếu backend thực ra là 1 khối gắn kết, không có bounded context rõ → dùng `go-vue` cho gọn
- nếu 1 module đã cần deploy/scale/đội ngũ độc lập → tách repo riêng (`web/backend-only/go` hoặc `service/go-service`), đừng nhồi tiếp service vào đây

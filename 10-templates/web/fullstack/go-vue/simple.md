# Fullstack Simple — Go + Vue

## Khi nào dùng

- team 1–3 người, cần ship nhanh
- 1 Go API + 1 Vue admin web (+ optional client-web, optional worker)
- Deploy Linux (Docker compose hoặc compose + systemd)
- Chưa cần `internal/` hexagonal — code flat trong `apps/api/`

## Cây thư mục

```text
<product-name>/
├── docs/                              # tài liệu chung của product
│   ├── runbook.md                     # vận hành: deploy, rollback, backup/restore, log, port
│   └── server-registration.md         # project name, domain, port, network, scale profile
├── infra/                             # config triển khai dùng chung
│   ├── compose/
│   │   ├── dev.yml                    # compose tổng — dev local all-in-one
│   │   └── prod.yml                   # production source of truth nếu deploy cả product
│   ├── nginx/                         # reverse proxy production (FE + /api)
│   └── systemd/                       # (optional) unit quản lý docker compose trên Linux server
├── scripts/
│   ├── dev-up.sh                      # wrap docker compose -f infra/compose/dev.yml up
│   ├── migrate.sh                     # apply DB migration có kiểm soát
│   ├── deploy.sh                      # deploy production bằng compose/systemd wrapper
│   ├── rollback.sh                    # rollback về image/tag trước đó
│   ├── backup.sh                      # tar gz apps/*/storage + pg_dump → /var/backups/
│   └── restore.sh                     # restore từ file timestamped
├── config/                            # config mẫu chung (nếu có giá trị share giữa apps)
├── apps/                              # mỗi app tự chứa, deploy độc lập
│   ├── api/                           # Go API (có thể kèm worker binary)
│   │   ├── handler/                   # router + HTTP handler
│   │   │   └── health/                # /healthz, /readyz
│   │   ├── service/                   # business logic — share giữa api & worker
│   │   ├── repository/                # truy cập DB
│   │   ├── model/                     # struct dùng chung
│   │   ├── worker/                    # (optional) long-running loop, gọi service/
│   │   ├── job/                       # (optional) scheduled job (robfig/cron)
│   │   ├── cmd/
│   │   │   ├── api/main.go            # entrypoint HTTP — wiring + start server
│   │   │   └── worker/main.go         # (optional) entrypoint worker — wiring + signal
│   │   ├── config/                    # config mẫu (config.example.yaml)
│   │   ├── migrations/                # SQL migrations nếu API có DB
│   │   ├── storage/                   # runtime data mount volume — chỉ .gitkeep
│   │   │   └── .gitkeep
│   │   ├── Dockerfile                 # multi-stage, multi-target (api / worker)
│   │   ├── docker-compose.yml         # standalone/smoke/deploy API riêng khi cần
│   │   ├── .env.example               # template env, KHÔNG commit .env thật
│   │   ├── .gitignore                 # ignore storage/* trừ .gitkeep
│   │   ├── go.mod                     # 1 module duy nhất
│   │   ├── go.sum
│   │   └── README.md                  # cách build, run, env, storage path
│   ├── admin-web/                     # Vue app cho admin
│   │   ├── public/
│   │   ├── src/
│   │   │   ├── assets/
│   │   │   ├── components/
│   │   │   ├── pages/
│   │   │   ├── router/
│   │   │   ├── stores/
│   │   │   ├── api/                   # wrapper gọi apps/api
│   │   │   ├── App.vue
│   │   │   └── main.ts
│   │   ├── Dockerfile                 # build Vue → nginx serve static
│   │   ├── docker-compose.yml         # standalone/smoke/deploy web riêng khi cần
│   │   ├── .env.example               # VITE_API_BASE_URL...
│   │   ├── vite.config.ts
│   │   ├── package.json
│   │   └── README.md
│   └── client-web/                    # (optional) Vue app cho end-user
│       └── ...                        # cấu trúc giống admin-web/
├── README.md                          # hướng dẫn repo tổng
└── .gitignore
```

## Vai trò thư mục

- `apps/api/`: 1 Go module duy nhất. `cmd/api` là binary chính; `cmd/worker` chỉ tạo khi cần — share `service/`, `repository/`, `model/`.
- `apps/api/storage/`: mount volume runtime cho uploaded files / cache / logs persist. Commit chỉ `.gitkeep`; data thật nằm trên host hoặc named volume.
- `apps/admin-web/`, `apps/client-web/`: 2 Vue project riêng — package.json, node_modules, build độc lập. Build ra Docker image nginx serve tĩnh.
- `apps/*/docker-compose.yml`: compose app-level cho smoke test, deploy app độc lập, hoặc trường hợp chỉ có 1 app.
- `infra/compose/dev.yml`: gom tất cả apps + dependency vào 1 compose để dev local 1 lệnh.
- `infra/compose/prod.yml`: source of truth cho production khi product có nhiều app; tránh mỗi app tự định nghĩa network/DB/proxy khác nhau.
- `infra/nginx/`: reverse proxy production — route domain → admin-web (`/admin`), client-web (`/`), api (`/api`).
- `infra/systemd/`: optional, dùng để quản lý docker compose như 1 service Linux; không dùng để chạy binary Go bare metal.
- `scripts/migrate.sh`: apply migration trước deploy hoặc trong maintenance window; binary không tự migrate ngầm khi start. Tool migration phải thống nhất trong repo (`goose` hoặc `golang-migrate`).
- `scripts/deploy.sh` / `scripts/rollback.sh`: gom lệnh compose pull/up/down và ghi rõ tag image đang chạy.
- `scripts/backup.sh`: tar gz `apps/*/storage/` + `pg_dump` ra file timestamped trong `/var/backups/<product>/`. Cron daily.
- `scripts/restore.sh`: nhận tên file backup → restore ngược lại.
- `docs/server-registration.md`: ghi thông tin register lên server: deploy path, scale profile, domain, port registry, DB/Redis mode, Docker networks, resource limit.

## Worker — thêm khi nào

Không có worker trong layout mặc định. Khi cần (consume queue, scheduled sync, file watcher):

1. Thêm folder `apps/api/worker/` và/hoặc `apps/api/job/`
2. Thêm `apps/api/cmd/worker/main.go` — wiring + signal handler
3. Update `apps/api/Dockerfile` build thêm target `worker`
4. Update `apps/api/docker-compose.yml` thêm service `worker` cùng image, khác CMD
5. Worker gọi xuống `apps/api/service/` — KHÔNG copy logic

## Rule

- **Docker bắt buộc** — không có path "chạy không qua Docker". Dev cũng qua compose.
- Mỗi app có `.env.example`, `.env` thật KHÔNG commit (đã có ở `.gitignore` root).
- `storage/` chỉ chứa `.gitkeep`; `.gitignore` per-app phải có `storage/*` + `!storage/.gitkeep`.
- Nếu API có DB thì phải có `migrations/` + `scripts/migrate.sh`; không để app tự migrate ngầm khi start.
- Backend phải có `/healthz` và `/readyz`; Docker/compose nên dùng healthcheck gọi các endpoint này.
- Backup script chạy được không cần config phức tạp — đường dẫn output hardcode hoặc lấy từ env. Backup production phải có retention và restore test định kỳ.
- Graceful shutdown ở worker BẮT BUỘC: `SIGTERM` → cancel ctx → wait → exit.
- FE gọi BE qua biến env `VITE_API_BASE_URL`, KHÔNG hardcode URL.
- Log backend nên là JSON log, có request id; thao tác admin quan trọng nên có audit log tối thiểu.
- CORS/CSRF/rate limit phải cấu hình rõ theo domain thật, không để wildcard trong production.
- Khi deploy lên Linux server nhiều dự án, phải cập nhật `PORT-REGISTRY.md` và `DOMAIN-REGISTRY.md`; app/API/web mặc định dùng `expose`, không bind host port.
- Compose production dùng container name theo chuẩn nội bộ và external networks public/internal/data đúng nhu cầu.
- Scale nhỏ/vừa dùng shared PostgreSQL/Redis với DB user và Redis prefix riêng; chỉ dedicated khi project quan trọng, workload nặng hoặc cần cô lập.
- Chưa cần monorepo tool (turborepo, nx). 1 `go.mod` ở `apps/api/` + `package.json` per Vue app là đủ.
- Khi `service/` phình, nhiều integration ngoài → nâng lên `structured.md`.

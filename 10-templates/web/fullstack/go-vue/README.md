# Web Fullstack — Go + Vue

## Khi nào dùng

- Product gồm Go API + Vue web, deploy trên Linux server
- Có thể có thêm: UI admin tách khỏi UI end-user, background worker/job
- Mọi biến thể đều **bắt buộc Docker** (build, dev, deploy đều qua compose)

## Có gì trong thư mục này

- `simple.md`: 1 API + 1 Vue admin web. Worker và client-web là optional, document cách thêm nếu cần.
- `structured.md`: cùng layout nhưng backend tách `cmd/` + `internal/{app,domain,usecase,adapter}` hexagonal; dùng khi cả 2 phía phình.

## Có nên bỏ `simple.md` / `structured.md` không?

Không nên bỏ. Hai file này giải quyết 2 giai đoạn khác nhau:

- `simple.md`: dùng cho MVP, tool nội bộ nhỏ, hoặc repo mới cần chạy nhanh nhưng vẫn đúng chuẩn Docker/backup/migrate tối thiểu.
- `structured.md`: dùng làm **company default** cho product sống lâu trên Linux server, có DB thật, admin, worker/job, nhiều người cùng bảo trì.

Nếu muốn tối ưu trải nghiệm chọn template, có thể coi `structured.md` là chuẩn mặc định cho `go-vue`, còn `simple.md` là đường bootstrap. Không nên gộp thành 1 file vì sẽ làm mất ranh giới giữa "đủ dùng" và "production dài hạn".

## Quy ước cố định cho go-vue

1. **Docker bắt buộc** — không có template "không Docker" cho go-vue. Mỗi app có `Dockerfile` + `docker-compose.yml` riêng.
2. **Layout `apps/`** — mỗi app tự chứa (Dockerfile, compose, env, config, storage, README).
3. **FE tách 2** — admin web và client web là 2 app riêng (`apps/admin-web/` + `apps/client-web/`). Nếu MVP chỉ cần admin → có mỗi `admin-web`.
4. **Worker là optional** — khi cần, thêm `cmd/worker/main.go` vào `apps/api/`, share `service/` (1 Go module, 2 binary).
5. **Backup mandatory** — `scripts/backup.sh` + `restore.sh` cho `apps/*/storage/` + DB dump.
6. **Compose rõ nguồn sự thật** — `infra/compose/dev.yml` cho dev all-in-one; `infra/compose/prod.yml` là source of truth cho production khi product có nhiều app; `apps/<name>/docker-compose.yml` chỉ dùng cho app standalone/smoke/deploy tách rời.
7. **Production baseline bắt buộc** — health/readiness, migrations script, backup restore drill, logging/metrics, runbook vận hành.

## Production baseline cho server công ty

Mọi repo `go-vue` chạy production nên có tối thiểu:

- `docs/runbook.md`: start/stop, deploy, rollback, backup/restore, log path, port, domain, cron.
- `scripts/migrate.sh`: apply migration có kiểm soát, không để binary tự migrate ngầm khi start.
- `scripts/deploy.sh` + `scripts/rollback.sh`: wrapper compose/systemd để deploy có quy trình.
- Health endpoint: `/healthz` cho liveness, `/readyz` cho readiness DB/queue.
- Structured log: JSON log ở backend, request id xuyên suốt API/worker.
- Metrics endpoint: `/metrics` hoặc exporter tương đương nếu hệ thống có Prometheus.
- Security baseline: không commit secret, phân quyền DB user, CORS/CSRF rõ, rate limit cho API public/admin, audit log cho thao tác admin quan trọng.
- Backup discipline: backup có retention, restore test định kỳ, DB dump và `apps/*/storage/` cùng timestamp.
- Migration tool phải thống nhất trong repo, ví dụ `goose` hoặc `golang-migrate`, và được gọi qua `scripts/migrate.sh`.

## Khi register project lên Linux server nhiều dự án

Áp dụng thêm [`PROJECT_REGISTRATION_CHECKLIST.md`](../../../../02-checklists/PROJECT_REGISTRATION_CHECKLIST.md).

Requirement bắt buộc:

- Project nằm ở path chuẩn nội bộ, ví dụ `/opt/apps/<project-name>`.
- Container name theo chuẩn `<prefix>-<project>-<service>` hoặc `<project>-<service>`.
- Compose project name theo chuẩn `<prefix>-<project>` hoặc `<project>`.
- App/API/web mặc định dùng `expose`, không bind host port.
- Public traffic chỉ qua Caddy/Nginx port `80/443`.
- Không public PostgreSQL `5432` hoặc Redis `6379`.
- Cập nhật `PORT-REGISTRY.md` để tránh trùng port giữa nhiều dự án.
- Cập nhật `DOMAIN-REGISTRY.md` nếu có domain/subdomain.
- Dùng Docker networks chuẩn theo server: public, internal, data.
- Dùng shared PostgreSQL/Redis cho scale nhỏ-vừa; chỉ dedicated khi project quan trọng, workload nặng hoặc cần cô lập.
- Mỗi container có `mem_limit`, `cpus`, `healthcheck`.
- Log mount về log path tập trung của server.
- Backup nằm dưới backup path tập trung của server.

Scale profile:

| Profile | Khi dùng | Infra mặc định |
|---|---|---|
| `shared` | nhiều project nhỏ/vừa trên cùng server | shared PostgreSQL/Redis, không bind host port app |
| `dedicated` | project quan trọng hoặc worker/DB vừa-nặng | cân nhắc dedicated DB/Redis trong cùng server |
| `isolated` | production lớn, SLA cao, dữ liệu cần cô lập | tách app/proxy, DB/cache, monitoring/backup sang server riêng |

## Runtime model cho `api` + `worker`

Khi `apps/api/` có cả `cmd/api` và `cmd/worker` (1 module, 2 binary), có 3 cách chạy. Build và Docker image gần như **không nặng hơn** (cùng source, multi-target, 1 image chạy khác `CMD`); khác biệt chỉ ở **runtime RAM** và **isolation**:

| Cách chạy | RAM | Isolation | Scale riêng | Hợp scale profile |
|---|---|---|---|---|
| 2 container (`api` + `worker` riêng) | tốn thêm ~1 process (vài chục MB) | tốt — worker crash không sập API | có | `dedicated`, `isolated` |
| 1 container, 2 process | trung bình | khá | hạn chế | `shared` muốn tiết kiệm RAM |
| 1 process, worker = goroutine | nhẹ nhất | kém — worker panic kéo sập API | không | MVP rất nhỏ |

Khuyến nghị:

- `shared` (nhiều project/server, shared DB/Redis): cân nhắc 1 container 2 process để tiết kiệm RAM; chỉ tách khi worker đủ nặng.
- `dedicated`/`isolated` (workload nặng, cần cô lập): tách 2 container để có isolation + scale riêng.
- Dù chạy cách nào, **business logic vẫn ở `service/`/`usecase/`** dùng chung — chọn runtime model là quyết định deploy, KHÔNG đổi cấu trúc code.
- Ranh giới khi nào nên tách hẳn thành service riêng: xem [`STRUCTURE_STANDARD_CORE.md`](../../../../00-core/STRUCTURE_STANDARD_CORE.md) mục "Ranh giới Service ↔ Web".

## Ghi chú

- Template này gộp luôn case "fullstack + worker" cũ — worker chỉ là 1 binary phụ trong cùng Go module, không cần tách app_shape riêng.
- Mỗi `apps/<name>/storage/` mount volume runtime, **không commit data thật**, chỉ commit `.gitkeep` + `.gitignore` rule.
- Nếu worker thật sự độc lập (khác team, khác cadence, không share `service/`) → tách thành repo riêng dùng `service/go-service`, KHÔNG nhét vào đây.

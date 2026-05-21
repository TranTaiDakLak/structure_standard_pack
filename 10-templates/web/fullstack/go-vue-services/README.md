# Web Fullstack — Go + Vue (multi-service / modular monolith)

## Khi nào dùng

- Product gồm Go backend + Vue web, deploy trên Linux server
- Backend tách tự nhiên thành **nhiều bounded context** (ví dụ `identity`, `billing`, `catalog`, `notification`) nhưng vẫn muốn **1 product / 1 repo / 1 deployable**
- Muốn ranh giới module rõ ngay từ trong code để nhiều người làm song song, dễ tách repo về sau — nhưng **chưa** muốn chịu chi phí vận hành microservices (service discovery, network call, distributed tracing, nhiều pipeline)
- Mọi biến thể đều **bắt buộc Docker** (build, dev, deploy đều qua compose)

## go-vue vs go-vue-services — chọn cái nào?

| Tín hiệu | Dùng `go-vue` | Dùng `go-vue-services` |
|---|---|---|
| Backend là 1 khối nghiệp vụ gắn kết | ✅ | |
| Backend chia ≥ 3 bounded context khá độc lập | | ✅ |
| Nhiều người/đội làm song song trên các vùng nghiệp vụ khác nhau | | ✅ |
| Muốn dễ tách 1 vùng ra repo riêng sau này | | ✅ |
| Đã cần service riêng deploy/scale độc lập, network call giữa service | ❌ (ngoài scope) | ❌ (ngoài scope) |

`go-vue-services` là **nấc giữa** giữa monolith gắn kết (`go-vue`) và microservices (ngoài scope của pack). Nếu một module thật sự cần deploy/scale/đội ngũ độc lập → tách thành repo riêng dùng `web/backend-only/go` hoặc `service/go-service`, **không** biến repo này thành cụm microservices.

## Có gì trong thư mục này

- `simple.md`: backend 1 binary, mỗi service module phẳng (`handler` + `service` + `repository` + `model` trong module), share `platform/`. Dùng khi đã biết các bounded context nhưng mỗi cái còn nhỏ.
- `structured.md`: mỗi service module tách hexagonal (`domain` + `usecase` + `adapter`), composition root ở `internal/app`, cross-cutting ở `internal/platform`, optional `cmd/worker`. Dùng khi module phình và cần test nghiệp vụ độc lập khỏi HTTP/worker.

## Có nên bỏ `simple.md` / `structured.md` không?

Không nên bỏ. Hai file giải quyết 2 giai đoạn:

- `simple.md`: vừa nhận ra product có nhiều bounded context, muốn tách module sớm nhưng mỗi module còn mỏng.
- `structured.md`: **company default** cho product multi-service sống lâu, mỗi module có nghiệp vụ thật, nhiều integration ngoài, nhiều người bảo trì.

Không gộp thành 1 file vì sẽ mất ranh giới giữa "đã tách module nhưng còn phẳng" và "module đã hexagonal hóa".

## Quy ước cố định cho go-vue-services

Kế thừa toàn bộ quy ước của `go-vue` (Docker bắt buộc, layout `apps/`, FE tách admin/client, backup mandatory, production baseline), cộng thêm các quy ước riêng cho multi-service:

1. **1 deployable mặc định** — tất cả service module mount chung vào `cmd/api`. Backend là 1 Go module, 1 process. KHÔNG mỗi module 1 container.
2. **Mỗi service module = 1 bounded context** — `internal/services/<name>/` (structured) hoặc `apps/api/services/<name>/` (simple). Tên module theo nghiệp vụ, không theo tầng kỹ thuật.
3. **Module chỉ giao tiếp qua usecase interface** — module A muốn dùng module B thì gọi qua interface do B export ở `usecase`, KHÔNG import thẳng `repository`/`domain`/bảng DB của B.
4. **Cross-cutting ở `platform/`, wiring ở `app/`** — logger, metrics, db pool, http server, config dùng chung; composition root `internal/app` quyết định module nào được mount.
5. **Worker là optional binary** — khi cần background cho 1 hoặc nhiều module, thêm `cmd/worker/main.go`, share `usecase` của module, KHÔNG copy logic. 1 module, 2 binary.
6. **Mỗi module tự khai báo điểm gắn vào hệ thống** — routes/worker/job của module gom trong `module.go` (structured) để `app/` mount gọn, tránh `main.go` phình.
7. **DB dùng chung, schema tách theo module** — 1 database, mỗi module sở hữu bảng/schema riêng; KHÔNG join chéo bảng của module khác, đọc dữ liệu module khác qua usecase của nó.

## Khi nào tách 1 module ra repo riêng

Tách khi xuất hiện ≥ 2 tín hiệu:

- module cần deploy/scale theo nhịp riêng (release cadence khác hẳn)
- đội owner khác hoàn toàn, ít đụng code module khác
- module bắt đầu cần resource isolation (DB/CPU/RAM riêng) ở mức `dedicated`/`isolated`
- giao tiếp với module khác đã thưa và rõ ràng qua vài interface ổn định

Khi tách: dùng `web/backend-only/go` (nếu là API) hoặc `service/go-service` (nếu là worker thuần). Giữ lại interface cũ làm contract HTTP/queue. **Đây là lúc rời khỏi scope của template này, không phải lúc thêm service thứ N vào cùng repo cho tới khi thành cụm microservices không quản nổi.**

## Production baseline cho server công ty

Giống `go-vue` (xem template `go-vue` để biết chi tiết): `docs/runbook.md`, `scripts/migrate.sh`, `scripts/deploy.sh` + `rollback.sh`, `/healthz` + `/readyz`, JSON log có request id, `/metrics` nếu có Prometheus, security baseline, backup discipline, migration tool thống nhất (`goose`/`golang-migrate`).

Thêm cho multi-service:

- `/readyz` kiểm tra dependency của **mọi module được mount** (DB, queue, external) — không chỉ DB chung.
- Log/metrics gắn nhãn `module` để truy vết lỗi theo bounded context.
- `docs/architecture.md` ghi rõ danh sách module, ai gọi ai (qua usecase nào), và bảng/schema mỗi module sở hữu.

## Khi register project lên Linux server nhiều dự án

Áp dụng thêm [`PROJECT_REGISTRATION_CHECKLIST.md`](../../../../02-checklists/PROJECT_REGISTRATION_CHECKLIST.md). Yêu cầu giống `go-vue`: path chuẩn, container name chuẩn, `expose` thay vì bind host port, public chỉ qua Caddy/Nginx 80/443, không public PostgreSQL/Redis, cập nhật `PORT-REGISTRY.md` + `DOMAIN-REGISTRY.md`, Docker networks public/internal/data, `mem_limit`/`cpus`/`healthcheck`, log/backup về path tập trung.

Scale profile:

| Profile | Khi dùng | Infra mặc định |
|---|---|---|
| `shared` | nhiều project nhỏ/vừa trên cùng server | shared PostgreSQL/Redis, không bind host port app |
| `dedicated` | product multi-service quan trọng, vài module nặng | cân nhắc dedicated DB/Redis trong cùng server |
| `isolated` | production lớn, SLA cao, dữ liệu cần cô lập | tách app/proxy, DB/cache, monitoring/backup sang server riêng; thường cũng là lúc cân nhắc tách module nặng ra repo riêng |

## Ghi chú

- 1 deployable nhiều module **không phải** microservices — đừng thêm network call giữa các module trong cùng process.
- Mỗi `apps/api/storage/` mount volume runtime, **không commit data thật**, chỉ `.gitkeep` + `.gitignore` rule.
- Ranh giới Service ↔ Web vẫn áp dụng: xem [`STRUCTURE_STANDARD_CORE.md`](../../../../00-core/STRUCTURE_STANDARD_CORE.md) mục "Ranh giới Service ↔ Web".

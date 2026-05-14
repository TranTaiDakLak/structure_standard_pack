# Project Registration Checklist

> Checklist đăng ký project trước khi deploy lên Linux server nhiều dự án.
> File này chỉ chứa requirement trung tính, không gắn với tên công ty hay hạ tầng riêng.

## 1. Khi nào dùng

Dùng checklist này khi:

- tạo project mới có deploy lên Linux server
- thêm domain/subdomain mới
- thêm service mới trong project đã có
- chuyển project từ server/dev cũ sang một server chuẩn
- dự án bắt đầu có DB, Redis, worker, upload file hoặc backup thật

Mục tiêu là tránh trùng port, loạn domain, public nhầm DB/cache, thiếu backup/log/healthcheck.

---

## 2. Thông tin bắt buộc khi register project

| Nhóm | Field | Ví dụ | Bắt buộc |
|---|---|---|---|
| Identity | `project_name` | `mail-app` | Có |
| Identity | `owner` | `team-product` | Có |
| Identity | `environment` | `production`, `staging` | Có |
| Server | `server_root` | `/opt/apps/mail-app` | Có |
| Server | `scale_profile` | `S1`, `S2`, `S3` | Có |
| Domain | `domains` | `mail.example.com` | Nếu public |
| Service | `services` | `api`, `web`, `worker` | Có |
| Network | `networks` | `public`, `internal`, `data` | Có |
| Port | `container_ports` | `api:8080`, `web:80` | Có |
| Port | `host_ports` | `none`, `127.0.0.1:18100` | Có |
| Database | `db_mode` | `none`, `shared`, `dedicated`, `external` | Có |
| Cache/Queue | `redis_mode` | `none`, `shared`, `dedicated`, `external` | Có |
| Storage | `storage_paths` | `/app/storage/uploads` | Nếu có file runtime |
| Backup | `backup_scope` | `db`, `storage`, `config`, `compose` | Có |
| Resource | `resource_limits` | `api: 512m/0.50 cpu` | Có |
| Ops | `healthcheck` | `/healthz`, `/readyz` | Có |
| Ops | `log_path` | `/opt/logs/apps/mail-app` | Có |
| Security | `secret_source` | `.env on server`, vault | Có |

---

## 3. Scale profile

### `S1` — nhiều project nhỏ/vừa trên một server

Dùng khi:

- internal tool, MVP, admin portal, landing/API nhỏ
- traffic thấp-vừa
- dữ liệu không cần cô lập mạnh
- nhiều project cùng chạy trên một server

Requirement:

- app nằm ở server root chuẩn, ví dụ `/opt/apps/<project-name>`
- dùng shared PostgreSQL nếu cần DB
- dùng shared Redis nếu cần cache/queue
- mỗi project có database/user riêng
- Redis phải có key prefix riêng: `<project-name>:`
- app/web/API không bind host port; dùng `expose`
- public traffic chỉ qua Caddy/Nginx port `80/443`
- có `mem_limit`, `cpus`, healthcheck, backup, log path

### `S2` — project quan trọng hoặc workload vừa/nặng

Dùng khi:

- project có dữ liệu quan trọng hơn
- worker/job nặng
- DB query nhiều
- upload file lớn hoặc nhiều
- cần cô lập tài nguyên để không ảnh hưởng project khác

Requirement:

- vẫn nằm trong server root chuẩn, ví dụ `/opt/apps/<project-name>`
- cân nhắc dedicated PostgreSQL/Redis trong cùng server
- tách service `api`, `worker`, `scheduler` rõ trong compose
- resource limit cao hơn và có capacity note trong `README.md`
- backup/restore phải test trước release lớn
- nếu có public API thì rate limit qua proxy hoặc app
- có dashboard/monitoring hoặc tối thiểu healthcheck toàn server

### `S3` — production lớn hoặc cần cô lập mạnh

Dùng khi:

- app quan trọng, SLA cao
- PostgreSQL/Redis nặng
- RAM thường xuyên cao, disk IO cao
- backup/restore bắt đầu chậm
- một project có rủi ro kéo ảnh hưởng project khác

Requirement:

- cân nhắc tách server: app/proxy, DB/cache, monitoring/backup
- DB/Redis có server hoặc managed service riêng
- backup offsite bắt buộc
- deploy/rollback phải có runbook rõ
- healthcheck domain, app, DB/cache, queue phải được giám sát
- không dùng shared infra nếu dữ liệu hoặc workload cần cô lập

---

## 4. Port và domain requirement

Rule mặc định:

- Không bind host port cho app/API/web nếu không cần.
- Dùng `expose` trong compose để proxy route qua Docker network.
- Chỉ public port `80/443` cho Caddy/Nginx và SSH port quản trị.
- Không public PostgreSQL `5432` hoặc Redis `6379`.
- Nếu cần host port để debug, bind localhost: `127.0.0.1:<port>:<container-port>`.

Khi register project phải cập nhật:

- `PORT-REGISTRY.md`
- `DOMAIN-REGISTRY.md`

Mẫu port registry:

```md
| Project | Service | Host Port | Container Port | Public | Note |
|---|---|---:|---:|---|---|
| mail-app | api | none | 8080 | Via proxy | Docker network |
| mail-app | web | none | 80 | Via proxy | Docker network |
```

Mẫu domain registry:

```md
| Domain | Project | Target Container | Port | Proxy File | SSL |
|---|---|---|---:|---|---|
| mail.example.com | mail-app | mail-app-api | 8080 | sites/mail-app.conf | Auto |
```

---

## 5. Compose requirement

Mỗi project production phải có:

- compose project name rõ ràng, ví dụ `name: mail-app`
- `container_name` theo chuẩn nội bộ, ví dụ `<project>-<service>` hoặc `<prefix>-<project>-<service>`
- external networks đúng nhu cầu:
  - public network cho proxy route vào API/web
  - internal network cho API/worker nội bộ
  - data network cho DB/Redis
- `expose` thay vì `ports` với service app/web/API
- `mem_limit` và `cpus`
- `healthcheck`
- mount log vào log path tập trung của server
- mount storage vào `/app/storage` nếu app có runtime file

Không được:

- dùng `ports: "5432:5432"` hoặc `ports: "6379:6379"`
- dùng container name tùy tiện không theo chuẩn nội bộ
- dùng chung DB admin user cho nhiều project
- để `change_me` trong `.env` production

---

## 6. Registration output

Khi AI hoặc người tạo project mới, output tối thiểu phải có:

1. `Project classification`
2. `Scale profile`
3. `Server registration`
4. `Port registry entries`
5. `Domain registry entries`
6. `Compose/network requirements`
7. `Database/Redis decision`
8. `Backup/log/healthcheck requirements`
9. `Deploy validation checklist`

---

## 7. Validate trước deploy

- [ ] Project nằm đúng server root chuẩn.
- [ ] Có `docker-compose.yml`, `.env`, `.env.example`, `config/`, `storage/`, `README.md`.
- [ ] Project đã có entry trong `PORT-REGISTRY.md`.
- [ ] Domain public đã có entry trong `DOMAIN-REGISTRY.md`.
- [ ] Không bind public port app/API/web nếu không có lý do.
- [ ] Không public DB/Redis.
- [ ] Container name đúng chuẩn nội bộ.
- [ ] Compose dùng external networks public/internal/data đúng nhu cầu.
- [ ] DB shared có database/user riêng.
- [ ] Redis shared có key prefix riêng.
- [ ] Có `mem_limit` và `cpus`.
- [ ] Có healthcheck.
- [ ] Có log path tập trung.
- [ ] Có backup scope rõ.
- [ ] Không còn secret placeholder như `change_me`.
- [ ] `docker compose config` chạy OK.


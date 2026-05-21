# Structure Standard Core

> Chuẩn lõi về cấu trúc thư mục cho dự án nội bộ.
> Bản này ưu tiên **đúng ngữ cảnh, đúng stack, đúng độ phức tạp**, thay vì ép mọi repo giống hệt nhau.

## 1. Nguyên tắc nền

1. Không khóa cứng 1 stack mặc định trong chuẩn gốc.
2. Chuẩn hóa theo **ý nghĩa thư mục**, không ép mọi ngôn ngữ giống nhau về hình thức.
3. Tổ chức pack theo 3 lớp:
   - `delivery_type`
   - `app_shape`
   - `stack`
4. Mặc định dùng mode **simple**.
5. Chỉ nâng lên **structured** khi có dấu hiệu rõ ràng.
6. Không mặc định monorepo.
7. Không mặc định contracts-first, codegen, packages/libs, ADR, CODEOWNERS.
8. Không ép tạo hàng loạt folder rỗng chỉ để “đẹp chuẩn”.
9. Chỉ support những gì pack hiện tại đã có template thật.

---

## 2. Schema phân loại chuẩn

### Trục 1 — `delivery_type`
- `web`
- `desktop`
- `browser-extension`
- `service`

### Trục 2 — `app_shape`
- `backend-only`
- `frontend-only`
- `fullstack`
- `desktop-app`
- `extension`
- `service`

### Stack support hiện tại

#### Web backend
- `go`
- `dotnet`
- `node-express`
- `python-fastapi`

#### Web frontend
- `vue-vite`
- `nuxt`
- `react-vite`

#### Desktop
- `wails-go-vue`
- `dotnet-wpf`
- `dotnet-winform`

#### Browser extension
- `js`

#### Service (background / worker / Windows Service)
- `dotnet-worker-service`
- `go-service`

### Ngoài scope hiện tại
- mobile
- electron
- flutter
- django fullstack
- laravel
- monorepo enterprise
- contracts/codegen mặc định
- shared packages/libs governance mức enterprise
- K8s / Terraform standards
- microservices governance
- CQRS / event sourcing playbook

Nếu case nằm ngoài scope:
- vẫn áp semantic rules ở section 5
- không tự bịa template "na ná" ngoài pack
- nếu cần dùng lâu dài, tạo template mới thành file riêng và thêm vào pack

### Ranh giới Service ↔ Web (app vừa chạy nền vừa có HTTP)

Một app có thể vừa có worker chạy nền, vừa expose HTTP. KHÔNG chọn shape theo "có HTTP hay không", mà theo **mục đích chính + đối tượng phục vụ**.

Câu hỏi quyết định: *Xóa phần HTTP (hoặc worker) đi thì app còn lý do tồn tại không?*

| Trường hợp | Chủ thể chính | HTTP surface | app_shape / template |
|---|---|---|---|
| Web product cần chạy việc nền | **web** | API public + UI cho user | `web/fullstack/<stack>` — worker là binary phụ, share `service/` |
| Daemon cần health/metrics/admin nhỏ | **service** | chỉ nội bộ (healthz, metrics, trigger, dashboard mini) | `service/<stack>` — thêm HTTP nhẹ, **giữ nguyên shape** |
| Vừa web thật vừa service nặng, ít share | **cả 2** | 2 mặt riêng biệt | **2 repo riêng** theo 2 template |

Rule:

- "Service tiện thể có 1 trang web" KHÔNG biến nó thành `web`. "Web tiện thể có worker" KHÔNG biến nó thành `service`.
- Chỉ tách 2 repo khi worker thật sự độc lập: khác team owner, khác nhịp release, hầu như không share `service/`/`usecase/`.
- Khi gộp (web + worker chung repo): dùng 1 module nhiều binary (`cmd/api` + `cmd/worker`), share business logic — xem runtime model theo scale profile ở README của template tương ứng.

---

## 3. Input bắt buộc trước khi chọn template

- `delivery_type`
- `app_shape`
- `backend_stack` hoặc `frontend_stack` hoặc `desktop_stack` hoặc `extension_stack` hoặc `service_stack`
- `team_size`
- `lifetime`
- `deploy_target`
- `current_apps`
- `current_services`
- `shared_code_real`

Nếu deploy lên Linux server nhiều dự án, cần thêm nhóm input vận hành:

- `project_name`
- `server_root` — ví dụ `/opt/apps/<project-name>` hoặc path chuẩn nội bộ
- `scale_profile` — `shared`, `dedicated`, `isolated`
- `domains`
- `services`
- `container_ports`
- `host_ports`
- `db_mode` — `none`, `shared`, `dedicated`, `external`
- `redis_mode` — `none`, `shared`, `dedicated`, `external`
- `resource_limits`
- `backup_scope`

## Gợi ý giá trị

### `team_size`
- `1`
- `2-3`
- `4-5`
- `6+`

### `lifetime`
- `short`
- `medium`
- `long`

### `deploy_target`
- `docker`
- `linux-daemon`
- `iis`
- `windows-service`
- `static-hosting`
- `desktop-installer`
- `browser-store`
- `other`

---

## 4. Chọn mode

## Mode A — Simple

Dùng khi:

- team nhỏ
- MVP / demo / internal tool
- 1 app hoặc 1 service
- chưa có nhu cầu tách lớp sâu
- ưu tiên tốc độ triển khai và dễ hiểu

Đặc điểm:

- cấu trúc phẳng hơn
- ít layer hơn
- không tạo thư mục “để dành”
- repo mở ra vài phút là hiểu

## Mode B — Structured

Dùng khi:

- dự án sống dài hơn
- số module tăng
- nhiều người cùng sửa
- logic nghiệp vụ bắt đầu phình
- integration, state, UI, business bắt đầu lẫn nhau

Đặc điểm:

- tách lớp rõ hơn
- vẫn gọn, chưa enterprise
- mục tiêu là dễ tìm code, không phải “trông cao cấp”

## Quy tắc chọn

Mặc định: **Simple**

Chuyển sang **Structured** khi có ít nhất 2 tín hiệu:

- dự án dự kiến > 6 tháng
- số file nghiệp vụ tăng nhanh
- nhiều người cùng chạm vào một vùng code
- khó tìm chỗ đặt logic mới
- business logic và integration bắt đầu lẫn vào nhau
- review code bắt đầu khó vì thiếu ranh giới rõ

---

## 5. Semantic folder rules

Các vùng ý nghĩa sau phải rõ trong repo, bất kể stack nào.

| Vùng | Ý nghĩa |
|---|---|
| `docs/` | tài liệu markdown, flow, ghi chú kỹ thuật |
| `infra/` | docker, iis, nginx, systemd, deployment config, installer (chọn sub-folder theo deploy_target) |
| `scripts/` | script hỗ trợ run/dev/build/migrate |
| `config/` | file mẫu cấu hình, không chứa secret thật |
| `build/` | output, artifact, luôn gitignore |
| `tests/` | test nằm ngoài source chính |
| `sidecar/` | binary prebuilt đi kèm app (helper exe, local API service, updater) — xem quy ước bên dưới |
| vùng source chính | code chạy production |
| entrypoint | nơi app khởi động |
| business logic | nơi chứa nghiệp vụ |
| integration/adapters | db, http client, external systems |

### Root tối thiểu khuyến nghị

```text
<repo-root>/
├── docs/
├── README.md
└── .gitignore
```

### Root thường gặp khi dự án chạy thật

```text
<repo-root>/
├── docs/
├── infra/
├── scripts/
├── config/
├── tests/
├── README.md
└── .gitignore
```

### Không bắt buộc nếu chưa cần

- `.vscode/`
- `.github/`
- `tools/`
- `sidecar/`

### Quy ước `infra/` theo `deploy_target`

Sub-folder dưới `infra/` chọn theo môi trường deploy thật, không phải bê hết:

| deploy_target | Sub-folder thường có | Ghi chú |
|---|---|---|
| `docker` | `infra/docker/` | Dockerfile (multi-stage), docker-compose.yml |
| `linux-daemon` | `infra/systemd/`, `infra/nginx/` | unit file `*.service`, reverse proxy config |
| `iis` | `infra/iis/` | web.config, applicationHost mẫu |
| `windows-service` | `infra/service-install/` | `sc.exe` script / `kardianos/service` install script |
| `static-hosting` | (thường không cần `infra/`) | hosting config nằm ở provider |
| `desktop-installer` | `infra/installer/` | Inno Setup / WiX / MSIX config |

Rule:

- Mỗi binary có unit file `systemd/` riêng (api.service, worker.service), không gom 1 unit cho nhiều process.
- Dockerfile per-binary nếu khác base image hoặc cần CMD khác; gom 1 Dockerfile multi-target nếu chung base.
- Không commit secret thật vào `infra/`; dùng `*.example.*` hoặc `.env.example`.

### Quy ước `sidecar/`

Dùng khi app cần ship kèm một hoặc nhiều binary prebuilt chạy ngầm (helper exe, API service local, updater, native tool). Binary trong `sidecar/` là **artifact**, không phải source code.

Cấu trúc khuyến nghị:

```text
sidecar/
└── <service-name>/
    ├── README.md              # mục đích, version, cách start/stop, port (nếu có)
    ├── <service-name>.exe     # binary prebuilt
    └── config.example.json    # (optional) cấu hình mẫu cho sidecar
```

Rule:

- Mỗi binary nằm trong sub-folder riêng theo tên service, không xả thẳng ra root `sidecar/`.
- BẮT BUỘC có `README.md`: mục đích, version, cách start/stop thủ công, port (nếu mở socket), log path.
- Nếu binary có source code → source phải ở repo riêng; commit hash / version build ghi trong `README.md`.
- Binary > 20MB → dùng Git LFS hoặc script fetch khi build, KHÔNG commit trực tiếp.
- Installer (trong `infra/installer/`) phải copy `sidecar/` vào đúng vị trí runtime của app.
- KHÔNG nhầm với `build/` (output của chính repo này) hay `infra/installer/` (config triển khai).

---

## 6. Mapping theo stack

### Web backend — Go
- simple: `main.go`, `handler/`, `service/`, `repository/`, `model/`
- structured: `cmd/`, `internal/`, `migrations/`

### Web backend — .NET
- simple: `src/<ServiceName>/Controllers`, `Services`, `Repositories`
- structured: `.Api`, `.Application`, `.Domain`, `.Infrastructure`

### Web backend — Node.js + Express
- simple: `src/routes`, `src/controllers`, `src/services`, `src/repositories`
- structured: `src/modules`, `src/platform`, `src/app`, `src/shared`

### Web backend — Python + FastAPI
- simple: `app/api`, `app/services`, `app/repositories`, `app/models`
- structured: `app/modules`, `app/core`, `app/adapters`, `app/schemas`

### Web frontend — Vue/Vite
- simple: `src/components`, `pages`, `router`, `stores`, `api`
- structured: thêm `features`, `composables`, `services`, `types`

### Web frontend — Nuxt
- simple: giữ `pages`, `components`, `layouts`, `server`
- structured: thêm `features`, `composables`, `types`

### Web frontend — React/Vite
- simple: `src/components`, `pages`, `routes`, `services`
- structured: thêm `features`, `hooks`, `lib`, `types`

### Web fullstack — Go + Vue
- simple: `apps/{api,admin-web}/` (+ optional `apps/client-web/`, optional `apps/api/cmd/worker/`)
- structured: `apps/api/{cmd,internal/{app,domain,usecase,adapter/{http,repository,messaging,external},worker,job,config},migrations}` + `apps/{admin-web,client-web}/`
- **Bắt buộc với go-vue**:
  - Docker — mỗi app có `Dockerfile` + app-level `docker-compose.yml`; thêm `infra/compose/dev.yml` cho dev all-in-one và `infra/compose/prod.yml` làm production source of truth khi product có nhiều app
  - Backup — `scripts/{backup,restore}.sh` cho `apps/*/storage/` + DB dump
  - FE tách admin và client thành 2 app riêng dưới `apps/` (nếu sản phẩm có cả 2 mặt)
  - Worker là optional binary trong cùng `apps/api/` Go module, không tách app riêng
  - Production baseline — health/readiness, migration script, deploy/rollback script, runbook, logging/metrics, security notes

### Web fullstack — Go + Vue (multi-service / modular monolith)
- simple: `apps/api/{cmd,platform,services/<module>/{handler,service,repository,model,port},migrations}` + `apps/{admin-web,client-web}/`
- structured: `apps/api/{cmd,internal/{app,platform,shared,services/<module>/{domain,usecase,adapter/{http,repository,external,messaging}}},migrations,tests}` + `apps/{admin-web,client-web}/`
- Kế thừa toàn bộ ràng buộc của go-vue (Docker, apps/ layout, FE tách admin/client, backup, production baseline) cộng thêm:
  - 1 deployable nhiều service module — KHÔNG mỗi module 1 container, KHÔNG network call nội bộ giữa module (gọi nhau in-process qua usecase port)
  - mỗi module = 1 bounded context, sở hữu bảng/schema riêng, cấm join chéo
  - module chỉ phụ thuộc nhau qua usecase port; cấm import repository/domain của module khác
  - khi 1 module cần deploy/scale/đội độc lập → tách repo riêng (`web/backend-only/go` hoặc `service/go-service`), không nhồi thành microservices trong 1 repo

### Web fullstack — .NET + Vue / Node + React
- .NET + Vue: không dùng Docker — deploy qua IIS/Windows hoặc Kestrel + reverse proxy (nginx).
- Node + React: không bị ràng buộc Docker bắt buộc (static/Node host hoặc Docker).

### Desktop — Wails + Go + Vue
- simple: `main.go`, `app.go`, `backend/`, `frontend/`
- structured: `cmd/`, `internal/`, `frontend/src/features`

### Desktop — .NET WPF
- simple: `Views`, `ViewModels`, `Services`, `Models`
- structured: `.App`, `.Application`, `.Domain`, `.Infrastructure`

### Desktop — .NET WinForms
- simple: `Forms`, `Presenters`, `Services`, `Models` (pattern MVP)
- structured: `.App` (Forms + Presenters + Views interface), `.Application`, `.Domain`, `.Infrastructure`

### Browser extension — JS
- simple: `src/background`, `src/content`, `src/popup`, `src/options`
- structured: thêm `src/features`, `src/shared`, `src/platform`

### Service — .NET Worker Service
- simple: `src/<ServiceName>/Workers`, `Jobs`, `Services`, `Models`
- structured: `.Host` (Workers + Jobs + Program.cs), `.Application`, `.Domain`, `.Infrastructure`

### Service — Go
- simple: `worker/`, `job/`, `service/`, `repository/`, `model/`, `main.go` ở root
- structured: `cmd/<service>/main.go`, `internal/{app,worker,job,domain,usecase,adapter,config}`

---

## 7. Naming rules

| Đối tượng | Quy tắc | Ví dụ |
|---|---|---|
| Repo | `kebab-case` | `billing-service` |
| Folder cấu trúc | `lowercase-kebab` | `api-client`, `user-profile` |
| File TS/JS | `kebab-case.ts/js` | `auth-service.ts` |
| File Go | `snake_case.go` | `user_repo.go` |
| File C# | `PascalCase.cs` | `UserService.cs` |
| Vue component | `PascalCase.vue` | `UserCard.vue` |
| React component | `PascalCase.tsx` | `UserCard.tsx` |
| Python file | `snake_case.py` | `user_service.py` |
| Config mẫu | `*.example.*` hoặc `.env.example` | `appsettings.example.json` |

### Nguyên tắc naming

- cùng một cấp thư mục, tránh trộn lung tung nhiều style
- folder cấu trúc repo nên lowercase
- folder code theo idiom của ngôn ngữ
- tên phải mô tả mục đích, không dùng `misc`, `temp-final`, `common2`

---

## 8. Anti-patterns

1. Có `utils/` ở root rồi nhét đủ thứ vào.
2. Business logic nằm trong entrypoint.
3. `Config/` viết hoa, lẫn lộn với `config/`.
4. Commit `build/`, `dist/`, `bin/`, `obj/`, `node_modules/`, `.venv/`.
5. Nhét secret thật vào repo.
6. Tạo quá nhiều folder trống ngay từ đầu.
7. Tách lớp quá sớm khi dự án còn rất nhỏ.
8. Ngược lại, để mọi thứ dồn vào 1 chỗ khi repo đã bắt đầu phình.
9. Dùng structured nhưng flow thật vẫn dồn về 1 file hoặc 1 thư mục.
10. Chọn template ngoài scope rồi cố vá tay.
11. Chia pack chỉ theo ngôn ngữ, làm mất ngữ cảnh của app shape.
12. Nhét hierarchy vào tên file thay vì dùng thư mục.

---

## 9. Migration principles

Khi repo cũ đang lộn xộn, migrate theo thứ tự:

1. chuẩn hóa naming
2. tách docs / infra / scripts / config ra đúng chỗ
3. xác định entrypoint
4. gom business logic về vùng rõ ràng
5. tách integration khỏi business logic
6. nếu cần mới nâng từ Simple lên Structured

### Quy tắc migrate

- mỗi đợt đổi cấu trúc nên làm nhỏ
- ưu tiên rename/move trước khi rewrite logic
- không đổi quá nhiều thứ trong 1 PR
- không “nâng cấp kiến trúc” chỉ vì thích đẹp

---

## 10. Output format cho AI

Khi AI áp chuẩn này cho một dự án, phải output theo thứ tự:

1. `Task checklist`
2. `Input classification`
3. `Chosen mode`
4. `Recommended structure`
5. `Folder responsibilities`
6. `Operational baseline`
7. `Migration notes`
8. `Assumptions`
9. `Risks`

Nếu deploy lên Linux server nhiều dự án, output thêm:

10. `Server registration`
11. `Scale profile`
12. `Port registry entries`
13. `Domain registry entries`
14. `Compose/network requirements`
15. `Backup/log/healthcheck requirements`

## 11. Quality gate cho template trong pack

Một template chỉ được xem là đủ dùng lâu dài khi có đủ các nhóm sau:

- **Scope rõ**: nói rõ khi nào dùng, khi nào không dùng, và dấu hiệu cần nâng từ `simple` lên `structured`.
- **Entrypoint rõ**: người đọc nhìn cây thư mục là biết app/service bắt đầu ở đâu.
- **Business boundary rõ**: business logic không nằm trong entrypoint, UI handler, worker loop, hoặc integration adapter.
- **Config/secret rõ**: có chỗ cho config mẫu, không commit secret thật, có quy ước `.env.example` hoặc file example tương đương.
- **Test path rõ**: có nơi đặt unit/integration/e2e test phù hợp với stack và không trộn test vào source chính khi không cần.
- **Deploy/ops rõ**: nếu có deploy thật thì phải có vị trí `infra/`, `scripts/`, healthcheck hoặc hướng dẫn vận hành tương ứng.
- **Upgrade path rõ**: template `simple` phải chỉ ra ngưỡng chuyển sang `structured`; template `structured` phải nhắc không chia nhỏ quá mức.

Khi sửa một template:

- không chỉ sửa cây thư mục, phải sửa luôn phần vai trò và rule liên quan
- không thêm folder mới nếu chưa có trách nhiệm rõ
- không thêm thuật ngữ enterprise nếu pack chưa cung cấp governance đi kèm
- ưu tiên rule trung tính, không phụ thuộc hạ tầng riêng của một công ty

## Metadata

- Version: `3.0-refactored`
- Owner: `https://github.com/TranTaiDakLak/`
- Maintainer: `Engineering / Architecture`

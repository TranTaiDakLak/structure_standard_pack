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
| `infra/` | docker, iis, nginx, deployment config, installer |
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
6. `Migration notes`
7. `Assumptions`
8. `Risks`

## Metadata

- Version: `3.0-refactored`
- Owner: `https://github.com/TranTaiDakLak/`
- Maintainer: `Engineering / Architecture`

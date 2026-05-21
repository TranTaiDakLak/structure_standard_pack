# Template Library Index

> File này dùng cùng với `STRUCTURE_STANDARD_CORE.md`.

## 1. Cách đọc index

Chọn lần lượt theo 3 tầng:

1. `delivery_type`
2. `app_shape`
3. `stack`

Sau đó chọn mode:
- `simple`
- `structured`

---

## 2. Mapping nhanh

| delivery_type | app_shape | stack | Modes có sẵn | Đường dẫn |
|---|---|---|---|---|
| web | backend-only | go | simple, structured | `10-templates/web/backend-only/go/` |
| web | backend-only | dotnet | simple, structured | `10-templates/web/backend-only/dotnet/` |
| web | backend-only | node-express | simple, structured | `10-templates/web/backend-only/node-express/` |
| web | backend-only | python-fastapi | simple, structured | `10-templates/web/backend-only/python-fastapi/` |
| web | frontend-only | vue-vite | simple, structured | `10-templates/web/frontend-only/vue-vite/` |
| web | frontend-only | nuxt | simple, structured | `10-templates/web/frontend-only/nuxt/` |
| web | frontend-only | react-vite | simple, structured | `10-templates/web/frontend-only/react-vite/` |
| web | fullstack | go-vue | simple, structured | `10-templates/web/fullstack/go-vue/` |
| web | fullstack | go-vue-services | simple, structured | `10-templates/web/fullstack/go-vue-services/` |
| web | fullstack | dotnet-vue | simple, structured | `10-templates/web/fullstack/dotnet-vue/` |
| web | fullstack | node-react | simple, structured | `10-templates/web/fullstack/node-react/` |
| desktop | desktop-app | wails-go-vue | simple, structured | `10-templates/desktop/wails-go-vue/` |
| desktop | desktop-app | dotnet-wpf | simple, structured | `10-templates/desktop/dotnet-wpf/` |
| desktop | desktop-app | dotnet-winform | simple, structured | `10-templates/desktop/dotnet-winform/` |
| browser-extension | extension | js | simple, structured | `10-templates/browser-extension/js/` |
| service | service | dotnet-worker-service | simple, structured | `10-templates/service/dotnet-worker-service/` |
| service | service | go-service | simple, structured | `10-templates/service/go-service/` |

---

## 3. Quy tắc chọn mode

### Chọn `simple` khi:

- cần ship nhanh
- team nhỏ
- repo mới dựng
- feature ít
- chưa có dấu hiệu phình cấu trúc

### Chọn `structured` khi:

- logic bắt đầu phân lớp tự nhiên
- nhiều người sửa cùng vùng
- cần boundary rõ
- simple bắt đầu khó tìm code

### Ràng buộc riêng cho `web/fullstack/go-vue`

- **Docker bắt buộc** — không có path bypass. Mỗi app trong `apps/` có Dockerfile + app-level docker-compose.yml. Dev local qua `infra/compose/dev.yml`; production nhiều app dùng `infra/compose/prod.yml` làm source of truth.
- **FE tách admin và client** — `apps/admin-web/` + (optional) `apps/client-web/`.
- **Worker là optional binary**, không phải app_shape riêng. Khi cần, thêm `apps/api/cmd/worker/main.go` + `apps/api/worker/`, share `service/` (1 Go module, 2 binary). Nếu worker thật sự độc lập (khác team, khác cadence, không share `service/`) → tách thành repo `service/go-service` riêng.
- **Backup mandatory** — `scripts/{backup,restore}.sh` cho `apps/*/storage/` + DB dump.
- **Production baseline mandatory** — health/readiness, migration script, deploy/rollback script, runbook, logging/metrics, security notes.

### Ràng buộc riêng cho `web/fullstack/go-vue-services`

- Kế thừa toàn bộ ràng buộc của `go-vue` ở trên.
- **1 deployable, nhiều service module** — mỗi `services/<module>/` là 1 bounded context; KHÔNG mỗi module 1 container, KHÔNG network call nội bộ giữa module (gọi nhau in-process qua usecase port).
- **Module sở hữu bảng/schema riêng** — cấm join chéo, cấm import repository/domain của module khác; đọc dữ liệu module khác qua usecase port của nó.
- **`internal/app` là composition root** — thêm/bớt module chỉ sửa ở đây + folder module.
- Khi 1 module cần deploy/scale/đội ngũ độc lập → tách repo riêng (`web/backend-only/go` hoặc `service/go-service`), không biến thành cụm microservices trong 1 repo.
- Chọn `go-vue` khi backend là 1 khối gắn kết; chọn `go-vue-services` khi backend chia ≥ 3 bounded context khá độc lập.

---

## 4. Quy tắc dùng template

- luôn đọc `README.md` trong từng stack folder trước
- sau đó chọn `simple.md` hoặc `structured.md`
- không tự bịa template ngoài pack khi chưa có lý do rõ
- nếu template được dùng cho production, kiểm tra thêm config/secret, test, deploy, backup/log/healthcheck theo mức độ rủi ro của stack
- nếu sửa hoặc thêm template, cập nhật đồng thời file index này và `STRUCTURE_STANDARD_CORE.md`

## 5. Invariants của thư viện template

Thư viện template phải giữ các invariant sau:

- mỗi dòng mapping ở section 2 phải có folder thật trong `10-templates/`
- mỗi folder stack phải có đủ `README.md`, `simple.md`, `structured.md`
- `README.md` của stack mô tả scope và cách chọn mode, không chỉ liệt kê file
- `simple.md` phải có đường nâng cấp sang `structured.md`
- `structured.md` phải có rule chống over-engineering
- stack nào có ràng buộc riêng phải được nhắc ở cả `STRUCTURE_STANDARD_CORE.md` và index này

## Metadata

- Version: `3.0-refactored`
- Owner: `https://github.com/TranTaiDakLak/`

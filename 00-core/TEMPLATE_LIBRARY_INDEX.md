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

---

## 4. Quy tắc dùng template

- luôn đọc `README.md` trong từng stack folder trước
- sau đó chọn `simple.md` hoặc `structured.md`
- không tự bịa template ngoài pack khi chưa có lý do rõ

## Metadata

- Version: `3.0-refactored`
- Owner: `https://github.com/TranTaiDakLak/`

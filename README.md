# Structure Standard Pack — v3 Refactored

Bộ chuẩn cấu trúc thư mục cho dự án nội bộ. Tách pack theo **core / prompts / checklists / templates**, chuẩn hóa theo 3 trục `delivery_type → app_shape → stack`, có đường nâng cấp từ `simple` lên `structured`.

## Cấu trúc pack

```text
structure_standard_pack/
├── 00-core/          # chuẩn lõi: nguyên tắc + schema + index template
├── 01-prompts/       # prompt dùng với AI để áp chuẩn vào dự án
├── 02-checklists/    # checklist migrate repo cũ sang chuẩn mới
├── 10-templates/     # thư viện template thật, phân theo delivery/shape/stack
└── README.md         # file này — tổng quan pack
```

## Vì sao dùng prefix số?

Prefix số giúp pack có thứ tự đọc ổn định và còn chỗ mở rộng:

- `00-core/`: nền tảng bắt buộc đọc trước.
- `01-prompts/`: prompt hỗ trợ AI áp chuẩn.
- `02-checklists/`: checklist dùng khi migrate/register/deploy.
- `10-templates/`: thư viện template thật. Đặt ở `10` để tách khỏi phần policy/checklist phía trước và chừa `03-09` cho các chuẩn bổ sung sau này.

Không nên đổi `10-templates` thành tên không đánh số nếu pack còn tiếp tục lớn lên, vì prefix giúp người dùng nhìn tree là biết luồng đọc.

## Mục đích từng thư mục

### `00-core/` — chuẩn lõi (đọc trước tiên)

- [`STRUCTURE_STANDARD_CORE.md`](00-core/STRUCTURE_STANDARD_CORE.md) — nguyên tắc nền, schema phân loại (delivery_type / app_shape / stack), quy tắc chọn mode simple vs structured, semantic folder rules (gồm quy ước `sidecar/`), naming rules, anti-patterns, migration principles.
- [`TEMPLATE_LIBRARY_INDEX.md`](00-core/TEMPLATE_LIBRARY_INDEX.md) — bảng mapping input → đường dẫn template trong `10-templates/`, kèm quy tắc chọn `simple` hay `structured`.

### `01-prompts/` — prompt dùng với AI

- [`PROMPT_CLAUDE_CODE_STRUCTURE_SELECTION.md`](01-prompts/PROMPT_CLAUDE_CODE_STRUCTURE_SELECTION.md) — prompt đóng vai Principal Software Architect: nhận mô tả dự án → chọn delivery_type / app_shape / stack / mode → xuất cây thư mục chuẩn + vai trò folder + migration notes.

### `02-checklists/` — checklist migrate

- [`MIGRATION_CHECKLIST.md`](02-checklists/MIGRATION_CHECKLIST.md) — checklist 7 bước để chuyển repo cũ lộn xộn sang chuẩn mới mà không over-engineer.
- [`PROJECT_REGISTRATION_CHECKLIST.md`](02-checklists/PROJECT_REGISTRATION_CHECKLIST.md) — checklist đăng ký project lên Linux server nhiều dự án: scale profile, port/domain registry, Docker network, shared DB/Redis, backup/log/healthcheck.

### `10-templates/` — thư viện template thật

Phân theo 3 tầng: `<delivery_type>/<app_shape>/<stack>/` — mỗi stack có `README.md` + `simple.md` + `structured.md` (tree folder kèm comment tiếng Việt inline).

Tra đầy đủ mapping tại [`00-core/TEMPLATE_LIBRARY_INDEX.md`](00-core/TEMPLATE_LIBRARY_INDEX.md).

## Scope hiện support

| delivery_type | app_shape | stack |
|---|---|---|
| web | backend-only | go, dotnet (ASP.NET Core), node-express, python-fastapi |
| web | frontend-only | vue-vite, nuxt, react-vite |
| web | fullstack | go-vue, go-vue-services, dotnet-vue, node-react |
| desktop | desktop-app | wails-go-vue, dotnet-wpf, dotnet-winform |
| browser-extension | extension | js |
| service | service | dotnet-worker-service, go-service |

Convention phụ: `sidecar/` (optional) cho binary prebuilt đi kèm app — xem [`STRUCTURE_STANDARD_CORE.md`](00-core/STRUCTURE_STANDARD_CORE.md) section 5.

Ràng buộc stack-specific: **`web/fullstack/go-vue` bắt buộc Docker + apps/ layout + backup script + tách admin/client FE + production baseline**. **`web/fullstack/go-vue-services`** kế thừa toàn bộ ràng buộc go-vue và thêm: 1 deployable nhiều service module (modular monolith), mỗi module là 1 bounded context giao tiếp in-process qua usecase port — không phải microservices. Xem [`STRUCTURE_STANDARD_CORE.md`](00-core/STRUCTURE_STANDARD_CORE.md) section 6 và [`TEMPLATE_LIBRARY_INDEX.md`](00-core/TEMPLATE_LIBRARY_INDEX.md).

Tài liệu hạ tầng riêng, script deploy riêng, hoặc chuẩn server có thông tin nội bộ không nên được link từ README public của pack. Nếu cần dùng nội bộ, giữ riêng và chỉ trích xuất requirement trung tính vào checklist.

## Cách dùng nhanh

1. Đọc [`00-core/STRUCTURE_STANDARD_CORE.md`](00-core/STRUCTURE_STANDARD_CORE.md) để hiểu nguyên tắc và schema.
2. Đọc [`00-core/TEMPLATE_LIBRARY_INDEX.md`](00-core/TEMPLATE_LIBRARY_INDEX.md) để tra template theo input dự án.
3. Mở template tương ứng trong [`10-templates/`](10-templates/), chọn `simple.md` hoặc `structured.md`.
4. Nếu deploy lên Linux server nhiều dự án, làm thêm [`02-checklists/PROJECT_REGISTRATION_CHECKLIST.md`](02-checklists/PROJECT_REGISTRATION_CHECKLIST.md) để đăng ký project, port, domain, network, DB/Redis, backup/log.
5. Nếu là repo cũ, làm theo [`02-checklists/MIGRATION_CHECKLIST.md`](02-checklists/MIGRATION_CHECKLIST.md).
6. Nếu dùng Claude Code / AI, đưa kèm [`01-prompts/PROMPT_CLAUDE_CODE_STRUCTURE_SELECTION.md`](01-prompts/PROMPT_CLAUDE_CODE_STRUCTURE_SELECTION.md) + 2 file core ở bước 1–2.

## Nguyên tắc nền (tóm tắt)

- Mặc định dùng mode **simple**. Chỉ nâng lên **structured** khi có ít nhất 2 tín hiệu rõ (team nhiều người, dự án > 6 tháng, code bắt đầu khó tìm...).
- Không khóa cứng 1 stack mặc định trong chuẩn gốc.
- Không mặc định monorepo, contracts-first, codegen, CODEOWNERS.
- Chỉ support những gì pack hiện tại đã có template thật; case ngoài scope thì dùng semantic rules từ core, không tự bịa "na ná".
- Với `web/fullstack/go-vue`, nếu là server/product đi dài thì ưu tiên `structured.md`; `simple.md` là simple code layout, không phải simple operations.

## Quality gate khi sửa pack

Mỗi thay đổi nên giữ 5 điều kiện sau:

- `README.md`, `STRUCTURE_STANDARD_CORE.md`, `TEMPLATE_LIBRARY_INDEX.md` cùng version và cùng scope support.
- Mỗi stack trong `10-templates/` có đủ `README.md`, `simple.md`, `structured.md`.
- Mỗi template `simple/structured` có đủ: khi nào dùng, cây thư mục, vai trò thư mục, rule, baseline test/config/security/deploy phù hợp stack.
- Link Markdown nội bộ phải trỏ tới file có thật; không để hướng dẫn trỏ sang tài liệu nội bộ không nằm trong pack.
- Nếu thêm stack mới, phải cập nhật đồng thời core, index, README root, và template folder tương ứng.

## Metadata

- Version: `3.0-refactored`
- Owner: [TranTaiDakLak](https://github.com/TranTaiDakLak/)
- Maintainer: Engineering / Architecture

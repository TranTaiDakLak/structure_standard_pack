# Structure Standard Pack — v1 Refactored

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

## Mục đích từng thư mục

### `00-core/` — chuẩn lõi (đọc trước tiên)

- [`STRUCTURE_STANDARD_CORE.md`](00-core/STRUCTURE_STANDARD_CORE.md) — nguyên tắc nền, schema phân loại (delivery_type / app_shape / stack), quy tắc chọn mode simple vs structured, semantic folder rules (gồm quy ước `sidecar/`), naming rules, anti-patterns, migration principles.
- [`TEMPLATE_LIBRARY_INDEX.md`](00-core/TEMPLATE_LIBRARY_INDEX.md) — bảng mapping input → đường dẫn template trong `10-templates/`, kèm quy tắc chọn `simple` hay `structured`.

### `01-prompts/` — prompt dùng với AI

- [`PROMPT_CLAUDE_CODE_STRUCTURE_SELECTION.md`](01-prompts/PROMPT_CLAUDE_CODE_STRUCTURE_SELECTION.md) — prompt đóng vai Principal Software Architect: nhận mô tả dự án → chọn delivery_type / app_shape / stack / mode → xuất cây thư mục chuẩn + vai trò folder + migration notes.

### `02-checklists/` — checklist migrate

- [`MIGRATION_CHECKLIST.md`](02-checklists/MIGRATION_CHECKLIST.md) — checklist 7 bước để chuyển repo cũ lộn xộn sang chuẩn mới mà không over-engineer.

### `10-templates/` — thư viện template thật

Phân theo 3 tầng: `<delivery_type>/<app_shape>/<stack>/` — mỗi stack có `README.md` + `simple.md` + `structured.md` (tree folder kèm comment tiếng Việt inline).

Tra đầy đủ mapping tại [`00-core/TEMPLATE_LIBRARY_INDEX.md`](00-core/TEMPLATE_LIBRARY_INDEX.md).

## Scope hiện support

| delivery_type | app_shape | stack |
|---|---|---|
| web | backend-only | go, dotnet (ASP.NET Core), node-express, python-fastapi |
| web | frontend-only | vue-vite, nuxt, react-vite |
| web | fullstack | go-vue, dotnet-vue, node-react |
| desktop | desktop-app | wails-go-vue, dotnet-wpf, dotnet-winform |
| browser-extension | extension | js |
| service | service | dotnet-worker-service, go-service |

Convention phụ: `sidecar/` (optional) cho binary prebuilt đi kèm app — xem [`STRUCTURE_STANDARD_CORE.md`](00-core/STRUCTURE_STANDARD_CORE.md) section 5.

## Cách dùng nhanh

1. Đọc [`00-core/STRUCTURE_STANDARD_CORE.md`](00-core/STRUCTURE_STANDARD_CORE.md) để hiểu nguyên tắc và schema.
2. Đọc [`00-core/TEMPLATE_LIBRARY_INDEX.md`](00-core/TEMPLATE_LIBRARY_INDEX.md) để tra template theo input dự án.
3. Mở template tương ứng trong [`10-templates/`](10-templates/), chọn `simple.md` hoặc `structured.md`.
4. Nếu là repo cũ, làm theo [`02-checklists/MIGRATION_CHECKLIST.md`](02-checklists/MIGRATION_CHECKLIST.md).
5. Nếu dùng Claude Code / AI, đưa kèm [`01-prompts/PROMPT_CLAUDE_CODE_STRUCTURE_SELECTION.md`](01-prompts/PROMPT_CLAUDE_CODE_STRUCTURE_SELECTION.md) + 2 file core ở bước 1–2.

## Nguyên tắc nền (tóm tắt)

- Mặc định dùng mode **simple**. Chỉ nâng lên **structured** khi có ít nhất 2 tín hiệu rõ (team nhiều người, dự án > 6 tháng, code bắt đầu khó tìm...).
- Không khóa cứng 1 stack mặc định trong chuẩn gốc.
- Không mặc định monorepo, contracts-first, codegen, CODEOWNERS.
- Chỉ support những gì pack hiện tại đã có template thật; case ngoài scope thì dùng semantic rules từ core, không tự bịa "na ná".

## Metadata

- Version: `1.0-refactored`
- Owner: [TranTaiDakLak](https://github.com/TranTaiDakLak/)
- Maintainer: Engineering / Architecture

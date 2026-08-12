# Structure Standard Pack

Bộ chuẩn cho dự án nội bộ, hai trục:

- **Cấu trúc** — code đặt ở folder nào. Chuẩn hóa theo 3 tầng `delivery_type → app_shape → stack`, có đường nâng cấp từ `simple` lên `structured`.
- **Hành vi** — code phải hành xử thế nào. Hiện có [API Response Contract](03-standards/API_RESPONSE_CONTRACT.md) (`api-1`): mọi dự án trả response cùng một hình dạng.

Pack tách theo **core / prompts / checklists / standards / templates**. Đổi gì qua từng version, và dự án đang chạy cần làm gì: [`CHANGELOG.md`](CHANGELOG.md).

## Cấu trúc pack

```text
structure_standard_pack/
├── 00-core/          # chuẩn lõi: nguyên tắc + schema + index template
├── 01-prompts/       # prompt dùng với AI để áp chuẩn vào dự án
├── 02-checklists/    # checklist migrate repo cũ sang chuẩn mới
├── 03-standards/     # chuẩn cross-cutting: code phải hành xử thế nào (API response contract...)
├── 10-templates/     # thư viện template thật, phân theo delivery/shape/stack
├── CHANGELOG.md      # pack đổi gì qua từng version, và dự án đang chạy cần làm gì
└── README.md         # file này — tổng quan pack
```

Hai trục đừng nhầm: `10-templates/` trả lời **"code đặt ở folder nào"**, `03-standards/` trả lời **"code phải hành xử thế nào"**.

## Vì sao dùng prefix số?

Prefix số giúp pack có thứ tự đọc ổn định và còn chỗ mở rộng:

- `00-core/`: nền tảng bắt buộc đọc trước.
- `01-prompts/`: prompt hỗ trợ AI áp chuẩn.
- `02-checklists/`: checklist dùng khi migrate/register/deploy.
- `03-standards/`: chuẩn cross-cutting áp cho mọi stack, không phụ thuộc cây thư mục.
- `10-templates/`: thư viện template thật. Đặt ở `10` để tách khỏi phần policy/standard/checklist phía trước và chừa `04-09` cho các chuẩn bổ sung sau này.

Không nên đổi `10-templates` thành tên không đánh số nếu pack còn tiếp tục lớn lên, vì prefix giúp người dùng nhìn tree là biết luồng đọc.

## Mục đích từng thư mục

### `00-core/` — chuẩn lõi (đọc trước tiên)

- [`STRUCTURE_STANDARD_CORE.md`](00-core/STRUCTURE_STANDARD_CORE.md) — nguyên tắc nền, schema phân loại (delivery_type / app_shape / stack), quy tắc chọn mode simple vs structured, semantic folder rules (gồm quy ước `sidecar/`), naming rules, anti-patterns, migration principles.
- [`TEMPLATE_LIBRARY_INDEX.md`](00-core/TEMPLATE_LIBRARY_INDEX.md) — bảng mapping input → đường dẫn template trong `10-templates/`, kèm quy tắc chọn `simple` hay `structured`.

### `01-prompts/` — prompt dùng với AI

- [`PROMPT_CLAUDE_CODE_STRUCTURE_SELECTION.md`](01-prompts/PROMPT_CLAUDE_CODE_STRUCTURE_SELECTION.md) — prompt đóng vai Principal Software Architect: nhận mô tả dự án → chọn delivery_type / app_shape / stack / mode → xuất cây thư mục chuẩn + vai trò folder + migration notes.

### `02-checklists/` — checklist migrate

- [`MIGRATION_CHECKLIST.md`](02-checklists/MIGRATION_CHECKLIST.md) — checklist 8 bước để chuyển repo cũ lộn xộn sang chuẩn mới mà không over-engineer.
- [`PROJECT_REGISTRATION_CHECKLIST.md`](02-checklists/PROJECT_REGISTRATION_CHECKLIST.md) — checklist đăng ký project lên Linux server nhiều dự án: scale profile, port/domain registry, Docker network, shared DB/Redis, backup/log/healthcheck.

### `03-standards/` — chuẩn cross-cutting

Luật áp cho mọi dự án bất kể stack. Tra danh mục và bảng "chuẩn nào áp cho stack nào" tại [`03-standards/README.md`](03-standards/README.md).

- [`API_RESPONSE_CONTRACT.md`](03-standards/API_RESPONSE_CONTRACT.md) — hình dạng response thống nhất cho mọi API: thành công trả gì, thất bại trả gì, bảng mã lỗi, map tình huống sang HTTP status, phân trang, validation, wire conventions, versioning, anti-patterns. Phần normative.
- [`API_RESPONSE_CONTRACT_APPENDIX.md`](03-standards/API_RESPONSE_CONTRACT_APPENDIX.md) — edge case (file, streaming, redirect, batch, job 202, health, webhook, panic, timeout, rate limit), transport không dùng HTTP (Wails bridge, extension messaging, worker), nơi đặt code theo từng stack, bẫy framework, api client mẫu cho FE (section 5), đánh đổi và rủi ro.
- [`snippets/go-api-contract.md`](03-standards/snippets/go-api-contract.md) — code Go tham chiếu (stdlib + `chi`) để áp `api-1`: bảng 17 mã + `AppError`, response writer, request id + recover middleware, wiring 404/405, bẫy DTO (`time.Time` lệch múi giờ, `int64` id), `Optional[T]` cho PATCH null-vs-absent, validator xuất tên field trên wire. Snippet để copy vào folder mà template chỉ định, không phải package chung để import.

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
4. Nếu dự án có API, đọc [`03-standards/API_RESPONSE_CONTRACT.md`](03-standards/API_RESPONSE_CONTRACT.md) — 8 luật Tier 0 là bắt buộc kể cả với MVP một người.
5. Nếu deploy lên Linux server nhiều dự án, làm thêm [`02-checklists/PROJECT_REGISTRATION_CHECKLIST.md`](02-checklists/PROJECT_REGISTRATION_CHECKLIST.md) để đăng ký project, port, domain, network, DB/Redis, backup/log.
6. Nếu là repo cũ, làm theo [`02-checklists/MIGRATION_CHECKLIST.md`](02-checklists/MIGRATION_CHECKLIST.md).
7. Nếu dùng Claude Code / AI, đưa kèm [`01-prompts/PROMPT_CLAUDE_CODE_STRUCTURE_SELECTION.md`](01-prompts/PROMPT_CLAUDE_CODE_STRUCTURE_SELECTION.md) + 2 file core ở bước 1–2 + file chuẩn ở bước 4 nếu dự án có API.

## Nguyên tắc nền (tóm tắt)

- Mặc định dùng mode **simple**. Chỉ nâng lên **structured** khi có ít nhất 2 tín hiệu rõ (team nhiều người, dự án > 6 tháng, code bắt đầu khó tìm...).
- Không khóa cứng 1 stack mặc định trong chuẩn gốc.
- Không mặc định monorepo, contracts-first, codegen, CODEOWNERS.
- Chỉ support những gì pack hiện tại đã có template thật; case ngoài scope thì dùng semantic rules từ core, không tự bịa "na ná".
- Với `web/fullstack/go-vue`, nếu là server/product đi dài thì ưu tiên `structured.md`; `simple.md` là simple code layout, không phải simple operations.
- Dự án có API phải theo chuẩn response chung ở `03-standards/`: một hình dạng thành công, một hình dạng thất bại, một bảng mã lỗi — không mỗi dự án một kiểu.

## Quality gate khi sửa pack

Mỗi thay đổi nên giữ 7 điều kiện sau:

- `README.md`, `STRUCTURE_STANDARD_CORE.md`, `TEMPLATE_LIBRARY_INDEX.md` cùng version và cùng scope support.
- Mỗi stack trong `10-templates/` có đủ `README.md`, `simple.md`, `structured.md`.
- Mỗi template `simple/structured` có đủ: khi nào dùng, cây thư mục, vai trò thư mục, rule, baseline test/config/security/deploy phù hợp stack.
- Link Markdown nội bộ phải trỏ tới file có thật; không để hướng dẫn trỏ sang tài liệu nội bộ không nằm trong pack.
- Nếu thêm stack mới, phải cập nhật đồng thời core, index, README root, và template folder tương ứng.
- Nếu thêm hoặc sửa chuẩn trong `03-standards/`, phải cập nhật đồng thời `03-standards/README.md`, core, index, README root, và các template bị ràng buộc. Template chỉ **trỏ** về chuẩn, không nhân bản nội dung chuẩn.
- Mỗi lần bump version phải có entry trong `CHANGELOG.md`, kèm mục "dự án đang chạy cần làm gì" — pack được tiêu thụ qua raw URL nên người dùng không thấy diff.

## Metadata

- Version: `3.1`
- Owner: [TranTaiDakLak](https://github.com/TranTaiDakLak/)
- Maintainer: Engineering / Architecture

# Standards

> Chuẩn cross-cutting: luật áp cho **mọi** dự án bất kể delivery_type / app_shape / stack.
> Khác với `10-templates/` (trả lời "code đặt ở folder nào"), thư mục này trả lời "code phải hành xử thế nào".

## 1. Danh mục chuẩn

| Chuẩn | File | Phạm vi | Version |
|---|---|---|---|
| API Response Contract | [`API_RESPONSE_CONTRACT.md`](API_RESPONSE_CONTRACT.md) | Mọi stack có HTTP surface; consumer-only cho FE/desktop/extension | `api-1` |
| ↳ Appendix | [`API_RESPONSE_CONTRACT_APPENDIX.md`](API_RESPONSE_CONTRACT_APPENDIX.md) | Edge case, transport không HTTP, nơi đặt code theo stack, bẫy framework, api client mẫu cho FE (section 5) | `api-1` |
| ↳ Snippet Go | [`snippets/go-api-contract.md`](snippets/go-api-contract.md) | Code Go tham chiếu (stdlib + `chi`): bảng 17 mã, writer, middleware, wiring 404/405, bẫy DTO, `Optional[T]` cho PATCH, validator. Copy vào folder template chỉ định — không phải package chung | `api-1` |

## 2. Chuẩn nào áp cho stack nào

| Stack | API Response Contract |
|---|---|
| `web/backend-only/{go,dotnet,node-express,python-fastapi}` | Toàn bộ |
| `web/fullstack/{go-vue,go-vue-services,dotnet-vue,node-react}` | Toàn bộ |
| `web/frontend-only/{vue-vite,nuxt,react-vite}` | Vai consumer — api client + type mirror. Nuxt: nghĩa vụ producer chỉ áp cho `server/api/**` (machine-facing); trang SSR cho browser không áp envelope — xem appendix section 2.6 |
| `service/{go-service,dotnet-worker-service}` | `/healthz` + `/readyz` bắt buộc khi service có HTTP surface; `error.code` + `retryable` làm căn cứ retry/backoff ở mọi trường hợp |
| `desktop/wails-go-vue` | Phần error qua Wails bridge |
| `desktop/{dotnet-wpf,dotnet-winform}` | Vai consumer |
| `browser-extension/js` | Vai consumer + messaging error |

## 3. Cách dùng

1. Chọn template theo [`../00-core/TEMPLATE_LIBRARY_INDEX.md`](../00-core/TEMPLATE_LIBRARY_INDEX.md) trước — nó quyết định cây thư mục.
2. Đọc chuẩn trong thư mục này để biết code trong cây đó phải hành xử thế nào.
3. Mỗi template trong `10-templates/` có mục `## API response contract` (hoặc dòng tương ứng trong `## Rule`) trỏ về đây kèm đường dẫn cụ thể cho stack đó — không nhân bản nội dung chuẩn vào template.

## 4. Quy tắc khi thêm hoặc sửa chuẩn ở đây

- Chuẩn cross-cutting mới phải áp được cho **nhiều** delivery_type. Thứ chỉ đúng với một stack thuộc về `10-templates/<stack>/`, không thuộc về đây.
- Mỗi chuẩn tách **normative** (chỉ luật, đọc là làm được) và **appendix** (edge case, đánh đổi, per-stack). File normative phải đứng độc lập được.
- Chuẩn phải nêu rõ tier bắt buộc: cái gì bắt buộc với mọi repo, cái gì bắt buộc khi điều kiện xảy ra, cái gì được phép hoãn và hoãn theo **điều kiện** nào. Tier không được gate bằng mode thư mục (`simple`/`structured`) — mode nói về khối lượng code, không nói về nghĩa vụ của chuẩn. Không có tier thì solo MVP sẽ bỏ qua cả chuẩn.
- Chuẩn phải có phần **đánh đổi và rủi ro đã biết**, viết thật, không giấu.
- Chuẩn có version riêng (`api-1`) tách khỏi version pack. Đổi frame của chuẩn thì bump version chuẩn; sửa chữ nghĩa thì không.
- Thêm chuẩn mới phải cập nhật đồng thời: file này, [`../README.md`](../README.md), [`../00-core/STRUCTURE_STANDARD_CORE.md`](../00-core/STRUCTURE_STANDARD_CORE.md), [`../00-core/TEMPLATE_LIBRARY_INDEX.md`](../00-core/TEMPLATE_LIBRARY_INDEX.md), và các template bị ràng buộc.

## Metadata

- Pack version: `3.1`
- Owner: `https://github.com/TranTaiDakLak/`
- Maintainer: `Engineering / Architecture`

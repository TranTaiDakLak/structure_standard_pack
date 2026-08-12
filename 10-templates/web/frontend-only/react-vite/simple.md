# React + Vite Frontend Simple

## Khi nào dùng

- web app React nhỏ-vừa
- team nhỏ
- ưu tiên ship nhanh

## Cây thư mục

```text
<app-name>/
├── docs/                 # tài liệu, flow, ghi chú kỹ thuật
├── infra/                # docker, nginx, deployment config
├── scripts/              # script build/preview/deploy
├── config/               # file cấu hình mẫu, .env.example
├── tests/                # test nằm ngoài source chính
├── public/               # tài nguyên tĩnh phục vụ thẳng
├── src/                  # vùng source chính
│   ├── assets/           # ảnh/font/css import qua bundler
│   ├── components/       # component UI dùng lại
│   ├── pages/            # page-level screen (map với route)
│   ├── routes/           # khai báo route (react-router)
│   ├── services/         # wrapper gọi API/SDK ngoài
│   ├── App.tsx           # root component
│   └── main.tsx          # entrypoint (createRoot, render App)
├── index.html            # HTML gốc Vite dùng
├── vite.config.ts        # cấu hình Vite (alias, plugin)
├── package.json          # dependency + script
├── README.md             # hướng dẫn repo
└── .gitignore            # bỏ qua node_modules/, dist/
```

## Vai trò thư mục

- `components/`: reusable UI
- `pages/`: page screens
- `routes/`: route setup
- `services/`: API/integration wrappers

## Rule

- page không nên ôm toàn bộ logic app
- `services/` không ôm state UI
- nếu feature tăng nhanh, nâng lên structured
- Config dùng `.env.example` và biến `VITE_*`; không hardcode endpoint production.
- Test tối thiểu component/logic quan trọng và e2e cho flow chính.
- `dist/`, coverage, cache, node_modules phải gitignore.

## API response contract

Consumer của [`03-standards/API_RESPONSE_CONTRACT.md`](../../../../03-standards/API_RESPONSE_CONTRACT.md), contract `api-1`.

- Api client duy nhất: `src/services/client.ts` — mọi request đi qua đây, cấm gọi `fetch`/`axios` rải rác trong component.
- Gọi thẳng API bên thứ 3 (map, payment SDK, analytics) thì API đó KHÔNG tuân `api-1`: mỗi upstream một adapter riêng trong `src/services/`, cấm dùng chung `client.ts` và cấm duck-type `body.error` — [appendix](../../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md) section 2.5.
- Type mirror: `src/services/types.ts` — union `ErrorCode` khai lại đúng bảng code của backend, để TS bắt được `switch` thiếu case. Nguồn là `docs/api-contract.md` của repo backend (section 11 của chuẩn), không hỏi miệng; backend thêm code mới thì cập nhật mirror theo file đó.
- `204` phải map thành `undefined` TRƯỚC khi thử `res.json()` — gọi `.json()` trên body rỗng sẽ throw. Timeout riêng bằng `AbortController`, ngắn hơn timeout của reverse proxy. Dùng TanStack Query thì `ApiError` là error type duy nhất ném ra, retry chỉ khi `error.retryable === true`.
- `retryable: true` nói về lỗi, không phải giấy phép phát lại request: interceptor **chỉ** tự retry method idempotent (GET/PUT/DELETE); POST tạo mới phải để user tự bấm lại hoặc gửi kèm `Idempotency-Key` — tự retry POST khi gặp `timeout`/`dependency_failed` là cách tạo đơn trùng, section 6.1b của chuẩn.
- File download và stream dùng `download()` / `stream()` ở [appendix](../../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md) section 5, đặt cạnh `api()` trong cùng file client. `download()` phải kiểm `res.ok` TRƯỚC khi đọc blob, nếu không sẽ lưu ra file chứa JSON lỗi.
- SSE/NDJSON dưới `/api/v1` consume bằng `fetch` + `ReadableStream` (hàm `stream()`), CẤM `EventSource` — nó không gắn được header `Authorization` (buộc nhét token vào query, rò vào access log) và không đọc được body lỗi.
- Client phải TOLERANT: gặp `error.code` lạ hoặc enum value lạ thì fallback theo status class, không crash. Backend thêm code mới là non-breaking, và điều đó chỉ đúng nếu client chịu được.
- Thứ tự fallback message hiển thị là **cố định** — dùng đúng chuỗi ở section 9.2 của chuẩn, không tự chế thứ tự khác.
- Cấm parse `error.message` để đoán loại lỗi — luôn `switch` trên `error.code`.

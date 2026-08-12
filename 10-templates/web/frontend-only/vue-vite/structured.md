# Vue + Vite Frontend Structured

## Khi nào dùng

- app Vue bắt đầu nhiều feature
- nhiều người sửa cùng lúc

## Cây thư mục

```text
<app-name>/
├── docs/                       # tài liệu, flow, ghi chú kỹ thuật
├── infra/                      # docker, nginx, deployment config
├── scripts/                    # script build/preview/deploy
├── config/                     # file cấu hình mẫu
├── tests/                      # test nằm ngoài source chính
│   ├── unit/                   # unit test (Vitest)
│   └── e2e/                    # end-to-end (Playwright/Cypress)
├── public/                     # tài nguyên tĩnh
├── src/                        # vùng source chính
│   ├── assets/                 # ảnh/font/css import qua bundler
│   ├── components/             # component generic dùng chung nhiều feature
│   ├── composables/            # composable/hook dùng chung (useXxx)
│   ├── features/               # logic theo domain — mỗi feature độc lập
│   │   ├── auth/               # feature auth (component + store + api)
│   │   ├── dashboard/          # feature dashboard
│   │   └── settings/           # feature settings
│   ├── layouts/                # layout bọc page (default, auth, admin...)
│   ├── pages/                  # page mỏng — compose từ feature
│   ├── router/                 # khai báo vue-router
│   ├── stores/                 # Pinia store dùng toàn cục
│   ├── services/               # tích hợp ngoài (API client, SDK wrapper)
│   ├── types/                  # TypeScript type dùng chung
│   ├── utils/                  # helper thuần (không state) — rất tiết chế
│   ├── App.vue                 # root component
│   └── main.ts                 # entrypoint
├── index.html                  # HTML gốc Vite dùng
├── vite.config.ts              # cấu hình Vite
├── package.json                # dependency + script
├── README.md                   # hướng dẫn repo
└── .gitignore
```

## Vai trò thư mục

- `features/`: logic theo domain
- `components/`: generic shared UI
- `composables/`: hooks/composables dùng chung
- `services/`: integrations
- `pages/`: mỏng, compose từ feature

## Rule

- đừng để `pages/` thành nơi chứa đủ thứ
- `components/` root không chứa component chỉ dùng cho 1 feature
- `utils/` phải rất tiết chế
- Feature nên chứa component/composable/api riêng nếu chỉ dùng trong feature đó.
- Config dùng `.env.example` và biến `VITE_*`; không hardcode endpoint production.
- Unit test feature/composable/store; e2e cho flow chính; build output luôn gitignore.
- Nếu feature layer chỉ là folder rỗng, quay lại `simple.md`.

## API response contract

Consumer của [`03-standards/API_RESPONSE_CONTRACT.md`](../../../../03-standards/API_RESPONSE_CONTRACT.md), contract `api-1`.

- Api client duy nhất: `src/services/http.ts` — mọi request đi qua đây, cấm gọi `fetch`/`axios` rải rác trong component; api riêng của feature bọc lại client này, không tự mở kết nối.
- Gọi thẳng API bên thứ 3 (map, payment SDK, analytics) thì API đó KHÔNG tuân `api-1`: mỗi upstream một adapter riêng trong `src/services/`, cấm dùng chung `http.ts` và cấm duck-type `body.error` — [appendix](../../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md) section 2.5.
- Type mirror: `src/types/api.ts` — union `ErrorCode` khai lại đúng bảng code của backend, để TS bắt được `switch` thiếu case. Nguồn là `docs/api-contract.md` của repo backend (section 11 của chuẩn), không hỏi miệng; backend thêm code mới thì cập nhật mirror theo file đó.
- `204` phải map thành `undefined` TRƯỚC khi thử `res.json()` — gọi `.json()` trên body rỗng sẽ throw. Timeout riêng bằng `AbortController`, ngắn hơn timeout của reverse proxy.
- File download và stream dùng `download()` / `stream()` ở [appendix](../../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md) section 5, đặt cạnh `api()` trong cùng file client. `download()` phải kiểm `res.ok` TRƯỚC khi đọc blob, nếu không sẽ lưu ra file chứa JSON lỗi.
- SSE/NDJSON dưới `/api/v1` consume bằng `fetch` + `ReadableStream` (hàm `stream()`), CẤM `EventSource` — nó không gắn được header `Authorization` (buộc nhét token vào query, rò vào access log) và không đọc được body lỗi.
- Client phải TOLERANT: gặp `error.code` lạ hoặc enum value lạ thì fallback theo status class, không crash. Backend thêm code mới là non-breaking, và điều đó chỉ đúng nếu client chịu được.
- Thứ tự fallback message hiển thị là **cố định** — dùng đúng chuỗi ở section 9.2 của chuẩn, không tự chế thứ tự khác.
- Cấm parse `error.message` để đoán loại lỗi — luôn `switch` trên `error.code`.

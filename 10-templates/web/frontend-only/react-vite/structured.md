# React + Vite Frontend Structured

## Khi nào dùng

- app React nhiều feature hơn
- cần tách state, hooks, feature logic

## Cây thư mục

```text
<app-name>/
├── docs/                   # tài liệu, flow, ghi chú kỹ thuật
├── infra/                  # docker, nginx, deployment config
├── scripts/                # script build/preview/deploy
├── config/                 # file cấu hình mẫu
├── tests/                  # test nằm ngoài source chính
│   ├── unit/               # unit test (Vitest)
│   └── e2e/                # end-to-end (Playwright)
├── public/                 # tài nguyên tĩnh
├── src/                    # vùng source chính
│   ├── assets/             # ảnh/font/css import qua bundler
│   ├── components/         # component generic dùng chung nhiều feature
│   ├── features/           # logic theo domain
│   │   ├── auth/           # feature auth (component + hook + api)
│   │   ├── dashboard/      # feature dashboard
│   │   └── settings/       # feature settings
│   ├── hooks/              # hook dùng chung (useXxx)
│   ├── lib/                # infra/helper (client, logger...) — tiết chế
│   ├── pages/              # page mỏng — compose từ feature
│   ├── routes/             # khai báo route
│   ├── services/           # API client, SDK wrapper
│   ├── types/              # TypeScript type dùng chung
│   ├── App.tsx             # root component
│   └── main.tsx            # entrypoint
├── index.html              # HTML gốc Vite dùng
├── vite.config.ts          # cấu hình Vite
├── package.json            # dependency + script
├── README.md               # hướng dẫn repo
└── .gitignore
```

## Vai trò thư mục

- `features/`: domain logic
- `hooks/`: shared hooks
- `lib/`: infra/helper rất tiết chế
- `pages/`: compose từ features

## Rule

- tránh để `components/` và `features/` chồng chéo
- không biến `lib/` thành thùng rác
- structured là để code dễ tìm hơn
- Feature nên chứa component/hook/api riêng nếu chỉ dùng trong feature đó.
- Config dùng `.env.example` và biến `VITE_*`; không hardcode endpoint production.
- Unit test feature/hook; e2e cho flow chính; build output luôn gitignore.
- Nếu feature layer chỉ là folder rỗng, quay lại `simple.md`.

## API response contract

Consumer của [`03-standards/API_RESPONSE_CONTRACT.md`](../../../../03-standards/API_RESPONSE_CONTRACT.md), contract `api-1`.

- Api client duy nhất: `src/services/http.ts` — mọi request đi qua đây, cấm gọi `fetch`/`axios` rải rác trong component; api riêng của feature bọc lại client này, không tự mở kết nối.
- Gọi thẳng API bên thứ 3 (map, payment SDK, analytics) thì API đó KHÔNG tuân `api-1`: mỗi upstream một adapter riêng trong `src/services/`, cấm dùng chung `http.ts` và cấm duck-type `body.error` — [appendix](../../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md) section 2.5.
- Type mirror: `src/types/api.ts` — union `ErrorCode` khai lại đúng bảng code của backend, để TS bắt được `switch` thiếu case. Nguồn là `docs/api-contract.md` của repo backend (section 11 của chuẩn), không hỏi miệng; backend thêm code mới thì cập nhật mirror theo file đó.
- `204` phải map thành `undefined` TRƯỚC khi thử `res.json()` — gọi `.json()` trên body rỗng sẽ throw. Timeout riêng bằng `AbortController`, ngắn hơn timeout của reverse proxy. Dùng TanStack Query thì `ApiError` là error type duy nhất ném ra, retry chỉ khi `error.retryable === true`.
- File download và stream dùng `download()` / `stream()` ở [appendix](../../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md) section 5, đặt cạnh `api()` trong cùng file client. `download()` phải kiểm `res.ok` TRƯỚC khi đọc blob, nếu không sẽ lưu ra file chứa JSON lỗi.
- SSE/NDJSON dưới `/api/v1` consume bằng `fetch` + `ReadableStream` (hàm `stream()`), CẤM `EventSource` — nó không gắn được header `Authorization` (buộc nhét token vào query, rò vào access log) và không đọc được body lỗi.
- Client phải TOLERANT: gặp `error.code` lạ hoặc enum value lạ thì fallback theo status class, không crash. Backend thêm code mới là non-breaking, và điều đó chỉ đúng nếu client chịu được.
- Thứ tự fallback message hiển thị là **cố định** — dùng đúng chuỗi ở section 9.2 của chuẩn, không tự chế thứ tự khác.
- Cấm parse `error.message` để đoán loại lỗi — luôn `switch` trên `error.code`.

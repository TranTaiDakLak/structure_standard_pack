# Nuxt Frontend Simple

## Khi nào dùng

- landing/marketing site
- Nuxt app nhỏ-vừa

## Cây thư mục

```text
<app-name>/
├── docs/                 # tài liệu, flow, ghi chú kỹ thuật
├── infra/                # docker, deployment config
├── scripts/              # script build/preview/deploy
├── config/               # file cấu hình mẫu, .env.example
├── tests/                # test nằm ngoài source chính
├── assets/               # ảnh/font/scss import qua bundler
├── components/           # component UI dùng lại (auto-import)
├── composables/          # composable dùng chung (useXxx — auto-import), gồm api client
├── layouts/              # layout Nuxt (default, auth, admin...)
├── pages/                # page Nuxt — file = route (file-based routing)
├── public/               # tài nguyên tĩnh phục vụ thẳng
├── server/               # route server / BFF nhẹ (Nitro)
│   ├── api/              # endpoint machine-facing — áp đủ contract api-1
│   └── utils/            # response writer + bảng error code (auto-import)
├── app.vue               # root component Nuxt
├── nuxt.config.ts        # cấu hình Nuxt (module, runtimeConfig)
├── package.json          # dependency + script
├── README.md             # hướng dẫn repo
└── .gitignore            # bỏ qua .nuxt/, .output/, node_modules/
```

## Vai trò thư mục

- `pages/`: route pages
- `components/`: reusable UI
- `composables/`: composable dùng chung, gồm api client (`useApi.ts`)
- `layouts/`: layouts
- `server/`: server routes/BFF nhẹ — `server/api/` là producer machine-facing, `server/utils/` chứa response writer + bảng error code
- `assets/`: styles, images

## Rule

- giữ convention Nuxt
- chưa cần `features/` nếu app còn nhỏ
- Runtime config dùng Nuxt config/env, không hardcode API URL production.
- Test tối thiểu composable/component quan trọng và e2e cho flow chính.
- Nếu `pages/` bắt đầu chứa nhiều logic nghiệp vụ, nâng sang `structured.md`.

## API response contract

Consumer của [`03-standards/API_RESPONSE_CONTRACT.md`](../../../../03-standards/API_RESPONSE_CONTRACT.md), contract `api-1`. Nuxt vừa consumer vừa producer — ranh giới ở [appendix section 2.6](../../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md).

- Api client duy nhất: `composables/useApi.ts` — wrapper quanh `$fetch`, mọi request đi qua đây, cấm gọi `fetch`/`axios` rải rác trong component.
- Type mirror: khai ngay trong `composables/useApi.ts` ở mode này — union `ErrorCode` khai lại đúng bảng code của backend, để TS bắt được `switch` thiếu case. Khi type bắt đầu nhiều thì tách ra `types/api.ts`, và đó cũng là lúc cân nhắc lên `structured.md`.
- Nghĩa vụ producer chỉ áp cho `server/api/**` — machine-facing, tuân đủ contract: error envelope, status đúng sự thật, `X-Request-Id`. Nơi đặt: writer + bảng code ở `server/utils/response.ts` (auto-import của Nitro), override lỗi mặc định ở `nitro.errorHandler` trong `nuxt.config.ts`.
- Trang SSR là browser-facing, KHÔNG áp envelope: `pages/products/[slug].vue` khi backend trả 404 phải render trang 404 HTML cho SEO, không trả JSON. Vẫn bắt buộc giữ đúng status code (404 thật, không phải 200) và in `trace_id` lên trang để user gửi support.
- Trusted upstream passthrough: upstream cũng khai `contract: api-1` thì `server/api/**` forward NGUYÊN VĂN status + object `error` cho mọi 4xx. Bọc lại thành `dependency_failed` là mất `code` mà FE cần rẽ nhánh, mất `details[]` mà form cần, và biến lỗi nghiệp vụ thành báo động vận hành giả. Chỉ 5xx và timeout của upstream mới bọc thành `502 dependency_failed`. Upstream không khai `api-1` đi qua adapter riêng (appendix section 2.5). Khai upstream nào là trusted trong `docs/api-contract.md`.
- Phía client, `$fetch` ném `FetchError` với body nằm ở `err.data` — unwrap ở đúng một chỗ trong composable, không rải `err.data?.error` khắp page.
- Client phải TOLERANT: gặp `error.code` lạ hoặc enum value lạ thì fallback theo status class, không crash. Backend thêm code mới là non-breaking, và điều đó chỉ đúng nếu client chịu được.
- Thứ tự fallback message hiển thị là **cố định** — dùng đúng chuỗi ở section 9.2 của chuẩn, không tự chế thứ tự khác.
- Cấm parse `error.message` để đoán loại lỗi — luôn `switch` trên `error.code`.

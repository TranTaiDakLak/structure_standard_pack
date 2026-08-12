# Web Frontend — Nuxt

## Khi nào dùng

- Landing page, marketing site, Nuxt app SSR/SSG
- structured khi page và feature tăng

## Có gì trong thư mục này

- `simple.md`: dùng khi cần ship nhanh, repo còn gọn
- `structured.md`: dùng khi code bắt đầu phình và cần boundary rõ hơn

## Baseline lâu dài

- Page nên mỏng; logic tái dùng nằm trong composables/features/services.
- Runtime config phải đi qua Nuxt config/env, không hardcode API URL production.
- Nếu bật SSR/server routes, phải ghi rõ boundary giữa frontend UI và server API nhỏ.
- Test tối thiểu component/composable quan trọng và e2e cho flow chính.
- Deploy target static/SSR/container phải được phản ánh trong `infra/` và README của repo.
- Vai consumer của [`03-standards/API_RESPONSE_CONTRACT.md`](../../../../03-standards/API_RESPONSE_CONTRACT.md) (contract `api-1`): một api client duy nhất + type mirror `ErrorCode`, client tolerant với code/enum lạ.
- Nghĩa vụ producer chỉ áp cho `server/api/**` (Nitro, machine-facing): error envelope, status code, `X-Request-Id`. Trang SSR là browser-facing nên không áp envelope — chỉ giữ đúng status code và in `trace_id` lên trang. Ranh giới ở [appendix section 2.6](../../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md).
- Upstream cũng khai `contract: api-1` thì forward nguyên văn status + object `error` cho mọi 4xx; chỉ 5xx và timeout mới bọc thành `502 dependency_failed`.
- Nơi đặt code producer: writer + bảng code ở `server/utils/response.ts` (auto-import của Nitro), override lỗi ở `nitro.errorHandler`.

## Ghi chú

- Phải tôn trọng convention của Nuxt, không cố bẻ cho giống stack khác.

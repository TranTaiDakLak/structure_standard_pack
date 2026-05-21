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

## Ghi chú

- Phải tôn trọng convention của Nuxt, không cố bẻ cho giống stack khác.

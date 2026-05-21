# Web Frontend — Vue + Vite

## Khi nào dùng

- Admin portal, dashboard, web app nhỏ-vừa
- structured khi feature tăng

## Có gì trong thư mục này

- `simple.md`: dùng khi cần ship nhanh, repo còn gọn
- `structured.md`: dùng khi code bắt đầu phình và cần boundary rõ hơn

## Baseline lâu dài

- Page/View mỏng; feature logic, composables, API client và store có ranh giới rõ.
- Config dùng `.env.example` và biến `VITE_*`; không hardcode endpoint production.
- Test tối thiểu component/composable/store quan trọng và e2e cho flow chính.
- Build output `dist/` luôn gitignore; deploy target static/nginx/container phải rõ.
- Khi `components/` hoặc `stores/` bắt đầu chứa nhiều domain khác nhau, chuyển sang `structured.md`.

## Ghi chú

- Vue/Vite thường là frontend web mặc định gọn, nhanh, dễ mở rộng.

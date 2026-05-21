# Web Frontend — React + Vite

## Khi nào dùng

- Dashboard, admin panel, client app React gọn
- structured khi feature nhiều và state bắt đầu phình

## Có gì trong thư mục này

- `simple.md`: dùng khi cần ship nhanh, repo còn gọn
- `structured.md`: dùng khi code bắt đầu phình và cần boundary rõ hơn

## Baseline lâu dài

- Page mỏng; feature logic, API client, state và shared component có ranh giới rõ.
- Config dùng `.env.example` và biến `VITE_*`; không hardcode endpoint production.
- Test tối thiểu component/logic quan trọng và e2e cho flow chính.
- Build output `dist/` luôn gitignore; deploy target static/nginx/container phải rõ.
- Khi `components/` thành thùng rác hoặc state rải khắp page, chuyển sang `structured.md`.

## Ghi chú

- Pack này không khóa cứng Redux/Zustand/React Query; chỉ chuẩn hóa cấu trúc thư mục.

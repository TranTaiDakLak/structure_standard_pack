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
- Vai consumer của [`03-standards/API_RESPONSE_CONTRACT.md`](../../../../03-standards/API_RESPONSE_CONTRACT.md) (contract `api-1`): một api client duy nhất, type mirror bảng `ErrorCode`, và client phải tolerant với code/enum lạ.
- SSE dưới `/api/v1` consume bằng `fetch` + `ReadableStream`, không dùng `EventSource`; API bên thứ 3 gọi thẳng từ FE cần adapter riêng vì không tuân `api-1` — [appendix](../../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md) section 2.5 và 5.

## Ghi chú

- Vue/Vite thường là frontend web mặc định gọn, nhanh, dễ mở rộng.

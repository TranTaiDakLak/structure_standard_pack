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
- Vai consumer của [`03-standards/API_RESPONSE_CONTRACT.md`](../../../../03-standards/API_RESPONSE_CONTRACT.md) (contract `api-1`): một api client duy nhất, type mirror bảng `ErrorCode`, client tolerant với code/enum lạ; dùng TanStack Query thì retry theo `error.retryable`.
- SSE dưới `/api/v1` consume bằng `fetch` + `ReadableStream`, không dùng `EventSource`; API bên thứ 3 gọi thẳng từ FE cần adapter riêng vì không tuân `api-1` — [appendix](../../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md) section 2.5 và 5.

## Ghi chú

- Pack này không khóa cứng Redux/Zustand/React Query; chỉ chuẩn hóa cấu trúc thư mục.

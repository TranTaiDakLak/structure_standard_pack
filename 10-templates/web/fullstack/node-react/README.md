# Web Fullstack — Node.js + React

## Khi nào dùng

- Combo phổ biến cho internal tools hoặc SaaS nhỏ-vừa
- simple để setup nhanh, structured khi modules tăng

## Có gì trong thư mục này

- `simple.md`: dùng khi cần ship nhanh, repo còn gọn
- `structured.md`: dùng khi code bắt đầu phình và cần boundary rõ hơn

## Baseline lâu dài

- Backend và frontend có README/env/build/test riêng; root README mô tả cách chạy cả product.
- Express API không chứa UI concern; React không hardcode API URL production.
- Config mẫu, migration, healthcheck, logging, validation, và deploy target phải rõ nếu chạy production.
- Test tối thiểu unit cho service/module backend, integration cho API/DB, và e2e cho flow UI chính.
- Nếu cần workspace tool, chỉ thêm khi nhiều package thật sự cần share/build chung.
- Express API bị ràng buộc **toàn bộ** [`03-standards/API_RESPONSE_CONTRACT.md`](../../../../03-standards/API_RESPONSE_CONTRACT.md) (contract `api-1`): error middleware 4 tham số đăng ký cuối cùng, catch-all 404 cũng trả JSON envelope.
- React ở vai consumer: api client + type mirror union `ErrorCode` dùng chung đúng tên code với backend, không tự chế shape lỗi riêng.

## Ghi chú

- Phần backend theo Express, frontend theo React + Vite.

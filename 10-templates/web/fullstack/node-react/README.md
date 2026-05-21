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

## Ghi chú

- Phần backend theo Express, frontend theo React + Vite.

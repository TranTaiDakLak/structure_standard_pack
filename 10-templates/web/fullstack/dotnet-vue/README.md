# Web Fullstack — ASP.NET Core + Vue

## Khi nào dùng

- Combo rất hợp nội bộ công ty
- simple cho đa số dự án nhỏ-vừa
- structured khi backend và frontend cùng phình

## Có gì trong thư mục này

- `simple.md`: dùng khi cần ship nhanh, repo còn gọn
- `structured.md`: dùng khi code bắt đầu phình và cần boundary rõ hơn

## Baseline lâu dài

- Backend và frontend có README/env/build/test riêng; root README mô tả cách chạy cả product.
- API không chứa UI logic; Vue không hardcode API URL production.
- Config mẫu, migration, healthcheck, logging, và deploy target phải rõ nếu chạy production.
- Test tối thiểu unit cho nghiệp vụ backend, integration cho API/DB, và e2e cho flow UI chính.
- Deploy có thể là IIS/static hosting hoặc Kestrel + reverse proxy (nginx); template chỉ chứa cấu hình hạ tầng thật đang dùng.

## Ghi chú

- Có thể chạy IIS hoặc Kestrel + reverse proxy (nginx).

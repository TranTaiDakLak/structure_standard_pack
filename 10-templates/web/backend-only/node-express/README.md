# Web Backend — Node.js + Express

## Khi nào dùng

- API JS/TS nhẹ, tốc độ dựng nhanh
- Hợp tool nội bộ, admin API, BFF nhỏ-vừa
- structured khi module tăng

## Có gì trong thư mục này

- `simple.md`: dùng khi cần ship nhanh, repo còn gọn
- `structured.md`: dùng khi code bắt đầu phình và cần boundary rõ hơn

## Baseline lâu dài

- Route/controller mỏng; nghiệp vụ nằm trong services/modules, data access nằm trong repositories/platform.
- Khuyến nghị TypeScript cho dự án sống lâu; nếu dùng JS thuần, cần lint/test nghiêm hơn.
- Config mẫu dùng `.env.example`; secret production không nằm trong repo.
- API production nên có healthcheck, request id, error middleware, validation, và migration script nếu có DB.
- Test tối thiểu service/module unit test và integration test cho route quan trọng.

## Ghi chú

- Pack này ghi theo JS/Node tổng quát.
- Có thể dùng TS nhưng giữ cùng logic thư mục.

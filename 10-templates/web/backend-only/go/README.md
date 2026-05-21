# Web Backend — Go

## Khi nào dùng

- API hoặc worker Go
- Hợp với môi trường chạy gọn, rõ, nhanh
- Structured dùng khi domain và integration bắt đầu lớn

## Có gì trong thư mục này

- `simple.md`: dùng khi cần ship nhanh, repo còn gọn
- `structured.md`: dùng khi code bắt đầu phình và cần boundary rõ hơn

## Baseline lâu dài

- `main.go`/`cmd/*` chỉ wiring; handler không chứa business logic dài.
- Config mẫu không chứa secret; env/file config phải rõ cho dev và production.
- API production nên có `/healthz`, `/readyz`, structured log, migration script nếu có DB.
- Test tối thiểu unit cho service/usecase và integration cho repository/HTTP quan trọng.
- Khi nhiều integration hoặc nhiều binary, chuyển sang `structured.md` thay vì phình root package.

## Ghi chú

- Go simple dùng `main.go` ở root.
- Go structured dùng `cmd/` + `internal/`.

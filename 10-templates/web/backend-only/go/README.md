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
- Stack này sinh HTTP API nên bị ràng buộc **toàn bộ** [`03-standards/API_RESPONSE_CONTRACT.md`](../../../../03-standards/API_RESPONSE_CONTRACT.md) (contract `api-1`): Tier 0 bắt buộc kể cả `simple.md`; Tier 1 và Tier 2 áp theo **điều kiện của dự án** (có consumer ngoài repo, endpoint đụng tiền, collection lớn dần...), không theo mode thư mục — chọn `simple` không miễn Tier 2, xem section 3 của chuẩn.
- Server đặt `retryable` theo **bản chất lỗi** (tạm thời hay không), KHÔNG hàm ý client được phát lại request: `timeout`/`dependency_failed` vẫn `true` dù việc có thể đã chạy xong ở server, nên POST không idempotent chỉ được client tự động phát lại khi có `Idempotency-Key` — section 6.1b + Tier 2 của chuẩn.
- Khi nhiều integration hoặc nhiều binary, chuyển sang `structured.md` thay vì phình root package.

## Ghi chú

- Go simple dùng `main.go` ở root.
- Go structured dùng `cmd/` + `internal/`.

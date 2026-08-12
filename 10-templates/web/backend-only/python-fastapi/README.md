# Web Backend — Python + FastAPI

## Khi nào dùng

- API Python gọn, nhanh, hợp service nội bộ hoặc automation backend
- structured khi module nhiều và integration tăng

## Có gì trong thư mục này

- `simple.md`: dùng khi cần ship nhanh, repo còn gọn
- `structured.md`: dùng khi code bắt đầu phình và cần boundary rõ hơn

## Baseline lâu dài

- Router mỏng; nghiệp vụ nằm trong services/usecases, persistence nằm trong repositories/adapters.
- Config mẫu dùng `.env.example` hoặc settings example; secret thật không nằm trong repo.
- API production nên có health/readiness, logging chuẩn, validation schema rõ, migration script nếu có DB.
- Stack này sinh HTTP API nên bị ràng buộc **toàn bộ** [`03-standards/API_RESPONSE_CONTRACT.md`](../../../../03-standards/API_RESPONSE_CONTRACT.md) (contract `api-1`): Tier 0 bắt buộc kể cả `simple.md`; Tier 1 và Tier 2 áp theo **điều kiện của dự án** (có consumer ngoài repo, endpoint đụng tiền, collection lớn dần...), không theo mode thư mục — chọn `simple` không miễn Tier 2, xem section 3 của chuẩn.
- Server đặt `retryable` theo **bản chất lỗi** (tạm thời hay không), KHÔNG hàm ý client được phát lại request: `timeout`/`dependency_failed` vẫn `true` dù việc có thể đã chạy xong ở server, nên POST không idempotent chỉ được client tự động phát lại khi có `Idempotency-Key` — section 6.1b + Tier 2 của chuẩn.
- Riêng FastAPI: phải override `HTTPException` và `RequestValidationError` ngay từ đầu, mặc định `{"detail": ...}` là sai contract — xem section `API response contract` trong `simple.md`/`structured.md`.
- Test tối thiểu unit cho service và integration cho route/repository quan trọng.
- Dependency và formatter/linter nên cố định sớm để tránh drift giữa máy dev và CI.

## Ghi chú

- Có thể dùng Pydantic, SQLAlchemy, Alembic; pack không khóa cứng lib cụ thể.

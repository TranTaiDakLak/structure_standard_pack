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
- Test tối thiểu unit cho service và integration cho route/repository quan trọng.
- Dependency và formatter/linter nên cố định sớm để tránh drift giữa máy dev và CI.

## Ghi chú

- Có thể dùng Pydantic, SQLAlchemy, Alembic; pack không khóa cứng lib cụ thể.

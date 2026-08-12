# Python + FastAPI Backend Simple

## Khi nào dùng

- service Python nhỏ-vừa
- team nhỏ
- cần tốc độ dựng nhanh

## Cây thư mục

```text
<service-name>/
├── docs/                 # tài liệu, flow, ghi chú kỹ thuật
├── infra/                # docker, nginx, deployment config
├── scripts/              # script build/run/migrate (alembic...)
├── config/               # file cấu hình mẫu, .env.example
├── tests/                # test nằm ngoài source chính
├── app/                  # vùng source chính (Python package)
│   ├── api/              # router FastAPI, khai báo endpoint
│   ├── services/         # business logic
│   ├── repositories/     # truy cập DB (SQLAlchemy/Tortoise)
│   ├── models/           # ORM model / domain entity
│   ├── schemas/          # Pydantic schema cho request/response
│   └── main.py           # entrypoint (tạo FastAPI app, include router)
├── requirements.txt      # dependency (hoặc pyproject.toml nếu dùng Poetry)
├── README.md             # hướng dẫn repo
└── .gitignore            # bỏ qua __pycache__/, .venv/, .env
```

## Vai trò thư mục

- `api/`: routers/endpoints
- `services/`: business logic
- `repositories/`: persistence
- `models/`: ORM/domain models đơn giản
- `schemas/`: request/response schema

## Rule

- `main.py` chỉ boot app
- không nhét business logic vào router
- schema không thay cho business rule
- Config mẫu dùng `.env.example` hoặc settings example; `.env` thật phải gitignore.
- API production nên có healthcheck, logging chuẩn, exception handler và validation rõ.
- Nếu có DB, Alembic/migration script phải có path rõ trong `scripts/` hoặc README.

## API response contract

Theo [`03-standards/API_RESPONSE_CONTRACT.md`](../../../../03-standards/API_RESPONSE_CONTRACT.md), contract `api-1` — không tự chế hình dạng response, không tự chế bảng mã lỗi.

- Response helper + writer: `app/schemas/common.py` — `ApiError`, `ErrorDetail`, `Page`, cộng helper dựng `JSONResponse` lỗi; router không tự ráp dict lỗi.
- File khai báo error code, đúng 1 file cho cả repo: `app/schemas/errors.py` — `StrEnum`.
- Exception/recover middleware toàn cục: handler viết cùng `app/schemas/common.py` hoặc `app/api/`, đăng ký ở `app/main.py`.
- Bẫy phải chặn ngay từ commit đầu: FastAPI mặc định trả `{"detail": ...}` cho **cả** `HTTPException` **lẫn** `RequestValidationError` — sai hình dạng ngay từ hộp. Bắt buộc override cả hai, cộng handler cho `Exception` để 500 không lộ `str(e)`:

  ```python
  # PHẢI là HTTPException của Starlette, KHÔNG phải của fastapi: 404 route lạ và 405 sai
  # method do chính Starlette raise, và fastapi.HTTPException không nằm trên MRO của nó
  # -> đăng ký bản fastapi là bỏ sót đúng hai ca đó, chúng vẫn trả {"detail": "Not Found"}.
  # Đăng ký bản Starlette phủ cả hai, vì fastapi.HTTPException kế thừa từ nó.
  from starlette.exceptions import HTTPException as StarletteHTTPException

  app.add_exception_handler(StarletteHTTPException, http_error_handler)        # -> envelope {"error": {...}}
  app.add_exception_handler(RequestValidationError, validation_error_handler)  # -> 422 validation_failed + details[]
  app.add_exception_handler(Exception, unhandled_error_handler)                # -> 500 internal, message cố định
  ```
- `docs/api-contract.md`: contract version, bảng domain error code, danh sách endpoint ngoại lệ.

Không đẻ layer mới cho việc này — mọi path trên đều nằm trong cây thư mục ở trên.

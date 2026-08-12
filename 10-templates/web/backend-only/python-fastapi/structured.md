# Python + FastAPI Backend Structured

## Khi nào dùng

- service Python sống lâu hơn
- nhiều module hơn
- cần boundary rõ

## Cây thư mục

```text
<service-name>/
├── docs/                       # tài liệu, flow, ghi chú kỹ thuật
├── infra/                      # docker, nginx, deployment config
├── scripts/                    # script build/run/migrate
├── config/                     # file cấu hình mẫu
├── tests/                      # test nằm ngoài source chính
│   ├── integration/            # test tích hợp (DB thật, HTTP thật)
│   └── e2e/                    # end-to-end qua API công khai
├── app/                        # vùng source chính (Python package)
│   ├── core/                   # config, bootstrap, logging, DI container
│   ├── modules/                # mỗi module 1 domain (có router + service + repo)
│   │   ├── auth/               # ví dụ module auth
│   │   ├── users/              # ví dụ module users
│   │   └── billing/            # ví dụ module billing
│   ├── adapters/               # cổng ra ngoài (hexagonal)
│   │   ├── http/               # middleware HTTP, exception handler
│   │   ├── repository/         # implement persistence (SQLAlchemy...)
│   │   └── external/           # client gọi service ngoài
│   ├── schemas/                # Pydantic schema dùng chung
│   └── main.py                 # entrypoint (FastAPI app + wiring)
├── requirements.txt            # dependency (hoặc pyproject.toml)
├── README.md                   # hướng dẫn repo
└── .gitignore
```

## Vai trò thư mục

- `core/`: config, bootstrap, common runtime
- `modules/`: domain/use cases theo module
- `adapters/`: http/repository/external
- `schemas/`: API schema

## Rule

- module không bị rò framework khắp nơi
- adapter không ôm logic nghiệp vụ
- đừng biến `core/` thành thùng rác
- Service/usecase không import trực tiếp ORM/session nếu đã có repository boundary.
- Unit test service/usecase; integration test route/repository quan trọng.
- API production nên có health/readiness, logging, exception handler, migration script và secret loading rõ.
- Nếu module/layer còn giả tạo, quay lại `simple.md`.

## API response contract

Theo [`03-standards/API_RESPONSE_CONTRACT.md`](../../../../03-standards/API_RESPONSE_CONTRACT.md), contract `api-1` — không tự chế hình dạng response, không tự chế bảng mã lỗi.

- Response helper + writer: `app/schemas/common.py` — `ApiError`, `ErrorDetail`, `Page`, cộng helper dựng `JSONResponse` lỗi; mọi module dùng lại, cấm mỗi module tự viết writer riêng.
- File khai báo error code, đúng 1 file cho cả repo: `app/schemas/errors.py` — `StrEnum`. Domain code dạng `<module>.<reason>`, `<module>` khớp đúng tên folder trong `app/modules/`.
- Exception/recover middleware toàn cục: `app/adapters/http/` (chỗ cây thư mục đã ghi "exception handler"), wiring vào app ở `app/main.py`. Module trả typed error, chỉ adapter HTTP mới map sang status + code.
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

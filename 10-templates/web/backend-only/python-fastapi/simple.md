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

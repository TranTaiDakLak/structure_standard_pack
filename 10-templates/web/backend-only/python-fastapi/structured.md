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

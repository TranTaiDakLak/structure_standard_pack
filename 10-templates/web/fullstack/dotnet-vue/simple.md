# Fullstack Simple — ASP.NET Core + Vue

## Khi nào dùng

- team 1–5 người
- 1 API + 1 frontend Vue

## Cây thư mục

```text
<product-name>/
├── docs/                                   # tài liệu chung của product
├── infra/                                  # deployment config chung
│   ├── iis/                                # cấu hình IIS (nếu host Windows)
│   └── nginx/                              # reverse proxy cho BE + FE
├── scripts/                                # script dev/build/deploy chung
├── config/                                 # file cấu hình mẫu
├── backend/                                # ASP.NET Core — như dotnet/simple
│   ├── src/                                # vùng source chính
│   │   └── <ServiceName>/                  # project API duy nhất
│   │       ├── Controllers/                # API endpoint
│   │       ├── Services/                   # business logic
│   │       ├── Repositories/               # truy cập DB
│   │       ├── Models/                     # entity/model đơn giản
│   │       ├── Dtos/                       # request/response DTO
│   │       ├── Program.cs                  # entrypoint + DI
│   │       └── appsettings.json            # cấu hình runtime
│   ├── tests/                              # test BE
│   ├── <ServiceName>.sln                   # solution
│   └── Directory.Build.props               # thiết lập build chung
├── frontend/                               # Vue app — như vue-vite/simple
│   ├── public/                             # tài nguyên tĩnh
│   ├── src/                                # source Vue
│   │   ├── assets/                         # ảnh/font/css
│   │   ├── components/                     # component UI dùng lại
│   │   ├── pages/                          # page screen
│   │   ├── router/                         # vue-router
│   │   ├── stores/                         # Pinia store
│   │   ├── api/                            # wrapper gọi backend .NET
│   │   ├── App.vue                         # root component
│   │   └── main.ts                         # entrypoint
│   ├── vite.config.ts                      # cấu hình Vite
│   └── package.json                        # dependency FE
├── README.md                               # hướng dẫn repo
└── .gitignore
```

## Vai trò thư mục

- `backend/`: ASP.NET Core service
- `frontend/`: Vue app
- `infra/`: IIS, nginx

## Rule

- fullstack small cho phép `backend/` + `frontend/`
- chưa cần contracts/codegen mặc định
- Backend có config/secret/migration/healthcheck rõ nếu chạy production.
- Frontend dùng env cho API base URL, không hardcode endpoint production.
- Root README phải có lệnh run/build/test cho cả backend và frontend.
- Nếu một phía phình trước, có thể nâng riêng phía đó sang cấu trúc structured.

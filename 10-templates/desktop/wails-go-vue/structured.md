# Desktop Structured — Wails + Go + Vue

## Khi nào dùng

- app Wails đã nhiều feature
- cần phân lớp rõ hơn giữa Go backend và FE

## Cây thư mục

```text
<app-name>/
├── docs/                       # tài liệu, flow, ghi chú kỹ thuật
├── infra/                      # deployment config
│   └── installer/              # script tạo installer (NSIS, Inno Setup...)
├── scripts/                    # script build/run/package
├── config/                     # file cấu hình mẫu
├── build/                      # output Wails build — gitignore
├── sidecar/                    # (optional) binary prebuilt ship kèm app
│   └── <service-name>/         # mỗi sidecar 1 sub-folder
│       ├── README.md           # mục đích, version, start/stop, port
│       └── <service-name>.exe  # binary (API local, updater, helper)
├── cmd/                        # entrypoint Wails
│   └── app/                    # binary desktop app
│       └── main.go             # wails.Run + bind bridge
├── internal/                   # code private Go
│   ├── app/                    # bootstrap, DI, bridge expose ra FE
│   ├── domain/                 # entity + rule cốt lõi
│   ├── usecase/                # application flow
│   └── adapter/                # cổng ra ngoài
│       ├── repository/         # persistence (file/DB local)
│       └── external/           # client gọi API ngoài
├── frontend/                   # Vue app trong webview
│   ├── src/                    # source Vue
│   │   ├── components/         # component generic
│   │   ├── composables/        # composable dùng chung
│   │   ├── features/           # logic theo domain
│   │   ├── services/           # wrapper gọi bridge Wails
│   │   └── main.ts             # entrypoint
│   ├── vite.config.ts          # cấu hình Vite
│   └── package.json            # dependency FE
├── tests/                      # test nằm ngoài source chính
│   ├── go/                     # unit/integration test backend Go
│   └── frontend/               # component/e2e test Vue
├── wails.json                  # cấu hình Wails
├── go.mod                      # khai báo module Go
├── go.sum                      # checksum dependency
├── README.md                   # hướng dẫn repo
└── .gitignore                  # bỏ qua build/, node_modules/, dist/
```

## Vai trò thư mục

- `cmd/`: Wails entry
- `internal/`: Go business/integration
- `frontend/src/features`: FE logic theo domain
- `infra/installer`: installer config

## Rule

- structured để tránh Go và FE dính chặt vào nhau
- không cần kéo sang monorepo
- `cmd/` chỉ bootstrap app; logic Go nằm trong `internal/`.
- Frontend feature không gọi trực tiếp chi tiết integration Go nếu chưa qua boundary rõ.
- Build/release phải ghi rõ target OS, artifact, version, và installer config.
- Nếu cấu trúc nhiều layer nhưng app vẫn nhỏ, hạ bớt về `simple.md`.

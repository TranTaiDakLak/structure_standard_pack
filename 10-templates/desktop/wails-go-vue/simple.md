# Desktop Simple — Wails + Go + Vue

## Khi nào dùng

- app desktop nhỏ-vừa
- team nhỏ
- cần ship nhanh

## Cây thư mục

```text
<app-name>/
├── docs/                 # tài liệu, flow, ghi chú kỹ thuật
├── infra/                # installer config, deployment
├── scripts/              # script build/run/package
├── config/               # file cấu hình mẫu
├── build/                # output Wails build — gitignore
├── sidecar/              # (optional) binary prebuilt ship kèm app
│   └── <service-name>/   # mỗi sidecar 1 sub-folder
│       ├── README.md     # mục đích, version, start/stop, port
│       └── <service-name>.exe  # binary (API local, updater, helper)
├── backend/              # Go business logic phía Wails
│   ├── service/          # business logic gọi bởi app.go
│   ├── repository/       # truy cập data (file, DB local, API)
│   └── model/            # struct dùng chung BE
├── frontend/             # Vue app hiển thị trong Wails webview
│   ├── src/              # source Vue
│   │   ├── components/   # component UI dùng lại
│   │   ├── pages/        # page screen
│   │   ├── services/     # wrapper gọi bridge Wails (window.go.*)
│   │   └── main.ts       # entrypoint
│   ├── vite.config.ts    # cấu hình Vite
│   └── package.json      # dependency FE
├── main.go               # entrypoint Wails (wails.Run)
├── app.go                # bridge FE ↔ Go (method expose ra JS)
├── wails.json            # cấu hình Wails (tên app, build...)
├── go.mod                # khai báo module Go
├── go.sum                # checksum dependency
└── README.md             # hướng dẫn repo
```

## Vai trò thư mục

- `main.go`: entry Wails
- `app.go`: bridge giữa FE và Go
- `backend/`: Go business logic
- `frontend/`: Vue app

## Rule

- FE không gọi lung tung ngoài bridge
- `build/` luôn gitignore
- chưa cần internal/cmd nếu app còn nhỏ

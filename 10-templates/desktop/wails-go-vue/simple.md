# Desktop Simple — Wails + Go + Vue

## Khi nào dùng

- app desktop nhỏ-vừa
- team nhỏ
- cần ship nhanh

## Cây thư mục

```text
<app-name>/
├── docs/                       # tài liệu, flow, ghi chú kỹ thuật
├── infra/                      # installer config, deployment
├── scripts/                    # script build/run/package
├── config/                     # file cấu hình mẫu
├── build/                      # output Wails build — gitignore
├── sidecar/                    # (optional) binary prebuilt ship kèm app
│   └── <service-name>/         # mỗi sidecar 1 sub-folder
│       ├── README.md           # mục đích, version, start/stop, port
│       └── <service-name>.exe  # binary (API local, updater, helper)
├── backend/                    # Go business logic phía Wails
│   ├── service/                # business logic gọi bởi app.go
│   ├── repository/             # truy cập data (file, DB local, API)
│   └── model/                  # struct dùng chung BE + errors.go (bảng error code)
├── frontend/                   # Vue app hiển thị trong Wails webview
│   ├── src/                    # source Vue
│   │   ├── components/         # component UI dùng lại
│   │   ├── pages/              # page screen
│   │   ├── services/           # wrapper gọi bridge Wails (window.go.*)
│   │   └── main.ts             # entrypoint
│   ├── vite.config.ts          # cấu hình Vite
│   └── package.json            # dependency FE
├── main.go                     # entrypoint Wails (wails.Run)
├── app.go                      # bridge FE ↔ Go (method expose ra JS)
├── wails.json                  # cấu hình Wails (tên app, build...)
├── go.mod                      # khai báo module Go
├── go.sum                      # checksum dependency
├── README.md                   # hướng dẫn repo
└── .gitignore                  # bỏ qua build/, node_modules/, dist/
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
- Backend Go phải test được độc lập khỏi webview.
- Frontend dùng env/config rõ cho API/binding mode, không hardcode path máy dev.
- Build/package desktop phải ghi rõ target OS, artifact, version và installer config nếu có.
- Secret/token runtime không nhúng vào frontend bundle; dùng config hoặc OS credential store nếu cần.
- Nếu backend hoặc frontend bắt đầu có nhiều feature, nâng phần đó sang `structured.md`.

## API response contract

Theo [`03-standards/API_RESPONSE_CONTRACT.md`](../../../03-standards/API_RESPONSE_CONTRACT.md), contract `api-1`. Bridge Wails không phải HTTP nên áp **phần error, không áp phần success**:

- Success: bound method trả giá trị trần theo idiom Go, không bọc envelope.
- Error: Wails ép Go `error` thành Promise reject với một **chuỗi trần**, mất sạch `code`, `retryable`, `user_message`. Vì thế bound method trả `(T, *AppError)` và bridge wrapper serialize error thành đúng object `{code, message, trace_id, retryable, user_message?}` như HTTP.
- Nơi đặt bridge wrapper: `app.go`
- `backend/model/errors.go`: nơi khai `Code` + `AppError` + `Retryable()`, đúng 1 file cho cả repo (chuẩn section 6.4). Đặt cùng tầng business logic Go để `backend/service/` và `backend/repository/` rẽ nhánh theo `code` mà không phải import ngược lên `app.go`.
- Code Go tham chiếu: [`03-standards/snippets/go-api-contract.md`](../../../03-standards/snippets/go-api-contract.md) — Wails **chỉ cần section 1** (bảng 17 mã + `AppError` + `Retryable()`) copy vào `backend/model/errors.go`; bridge không phải HTTP nên bỏ writer/middleware ở section 2–4.
- `recover()` trong wrapper map sang `internal` + `trace_id`, ghi `trace_id` vào log file local — để support desktop có cùng khoá join với log server.
- FE trong `frontend/src/services/` bắt `ApiError` cùng một union `ErrorCode` với API server nếu app có gọi API ngoài.
- Sidecar trong `sidecar/` là một **upstream không tuân `api-1`**: mỗi sidecar một adapter riêng trong `backend/repository/` theo [appendix section 2.5](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md). Nó có thêm một trạng thái mà API mạng không có — **chưa chạy** — map thành `dependency_failed` + `message` nói rõ sidecar nào; cấm để `ECONNREFUSED` rò lên UI.
- Lỗi chỉ tồn tại ở máy người dùng (mất mạng, user huỷ, ghi SQLite local hỏng) dùng nhóm `client.*` ở [appendix section 2.0b](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md): `client.offline` (`retryable: true`), `client.cancelled`, `client.storage_failed`. Không mượn `unavailable`/`timeout` — hai mã đó nghĩa là server có vấn đề, làm hỏng dashboard và nhắm sai retry.
- Bound method đẩy thay đổi local lên server (`SyncNow`) **bắt buộc gửi `Idempotency-Key`** theo Tier 2: client tự retry POST, server không kiểm soát được nhịp retry của client.
- `SyncNow` retry sau khi mất mạng **KHÔNG** được tự phát lại POST tạo mới nếu request không có `Idempotency-Key` — `retryable: true` nói lỗi có tính tạm thời, không phải giấy phép phát lại request (chuẩn section 6.1b); không có key thì chờ người dùng bấm lại.

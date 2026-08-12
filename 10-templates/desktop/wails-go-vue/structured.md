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
│   ├── domain/                 # entity + rule cốt lõi + errors.go (bảng error code)
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

## API response contract

Theo [`03-standards/API_RESPONSE_CONTRACT.md`](../../../03-standards/API_RESPONSE_CONTRACT.md), contract `api-1`. Bridge Wails không phải HTTP nên áp **phần error, không áp phần success**:

- Success: bound method trả giá trị trần theo idiom Go, không bọc envelope.
- Error: Wails ép Go `error` thành Promise reject với một **chuỗi trần**, mất sạch `code`, `retryable`, `user_message`. Vì thế bound method trả `(T, *AppError)` và bridge wrapper serialize error thành đúng object `{code, message, trace_id, retryable, user_message?}` như HTTP.
- Nơi đặt bridge wrapper: `internal/app/`
- `internal/domain/errors.go`: nơi khai `Code` + `AppError` + `Retryable()`, đúng 1 file cho cả repo (chuẩn section 6.4). Đặt ở `domain` để `internal/usecase/` và `internal/adapter/` rẽ nhánh theo `code` mà không phải import ngược vào `internal/app/`.
- Code Go tham chiếu: [`03-standards/snippets/go-api-contract.md`](../../../03-standards/snippets/go-api-contract.md) — Wails **chỉ cần section 1** (bảng 17 mã + `AppError` + `Retryable()`) copy vào `internal/domain/errors.go`; bridge không phải HTTP nên bỏ writer/middleware ở section 2–4.
- `recover()` trong wrapper map sang `internal` + `trace_id`, ghi `trace_id` vào log file local — để support desktop có cùng khoá join với log server.
- FE trong `frontend/src/services/` bắt `ApiError` cùng một union `ErrorCode` với API server nếu app có gọi API ngoài.
- Sidecar trong `sidecar/` là một **upstream không tuân `api-1`**: mỗi sidecar một adapter riêng trong `internal/adapter/external/` theo [appendix section 2.5](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md). Nó có thêm một trạng thái mà API mạng không có — **chưa chạy** — map thành `dependency_failed` + `message` nói rõ sidecar nào; cấm để `ECONNREFUSED` rò lên UI.
- Lỗi chỉ tồn tại ở máy người dùng (mất mạng, user huỷ, ghi SQLite local hỏng) dùng nhóm `client.*` ở [appendix section 2.0b](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md): `client.offline` (`retryable: true`), `client.cancelled`, `client.storage_failed`. Không mượn `unavailable`/`timeout` — hai mã đó nghĩa là server có vấn đề, làm hỏng dashboard và nhắm sai retry.
- Bound method đẩy thay đổi local lên server (`SyncNow`) **bắt buộc gửi `Idempotency-Key`** theo Tier 2: client tự retry POST, server không kiểm soát được nhịp retry của client.
- `SyncNow` retry sau khi mất mạng **KHÔNG** được tự phát lại POST tạo mới nếu request không có `Idempotency-Key` — `retryable: true` nói lỗi có tính tạm thời, không phải giấy phép phát lại request (chuẩn section 6.1b); không có key thì chờ người dùng bấm lại.

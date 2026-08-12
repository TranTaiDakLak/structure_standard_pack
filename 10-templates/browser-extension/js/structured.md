# Browser Extension JS Structured

## Khi nào dùng

- extension có nhiều feature
- cần chia rõ shared/platform/feature logic

## Cây thư mục

```text
<extension-name>/
├── docs/                       # tài liệu, flow, ghi chú kỹ thuật
├── infra/                      # config build/package
├── scripts/                    # script build/watch/package
├── config/                     # file cấu hình mẫu
├── build/                      # output bundle — gitignore
├── src/                        # vùng source chính
│   ├── background/             # service worker MV3 — chỉ dispatch event
│   ├── content/                # content script — bridge trang ↔ extension
│   ├── popup/                  # UI popup
│   ├── options/                # trang Options
│   ├── features/               # logic theo domain — mỗi feature độc lập
│   │   ├── auth/               # ví dụ feature auth
│   │   ├── collector/          # ví dụ feature thu thập dữ liệu
│   │   └── sync/               # ví dụ feature đồng bộ server
│   ├── platform/               # wrapper browser API
│   │   ├── storage/            # wrap chrome.storage
│   │   ├── messaging/          # wrap chrome.runtime.sendMessage — 3 cơ chế hỏng
│   │   ├── api-client/         # client cho API nội bộ khai api-1
│   │   ├── providers/          # adapter cho upstream KHÔNG tuân api-1 (zalo.js, graph.js)
│   │   └── browser-api/        # wrap các API khác (tabs, alarms...)
│   ├── shared/                 # helper dùng chung — rất tiết chế
│   ├── assets/                 # icon, ảnh, css
│   └── manifest.json           # manifest extension (MV3)
├── tests/                      # test nằm ngoài source chính
│   ├── unit/                   # unit test feature + platform
│   └── integration/            # integration test (mock browser API)
├── package.json                # dependency + script build
├── README.md                   # hướng dẫn repo
└── .gitignore
```

## Vai trò thư mục

- `features/`: feature logic
- `platform/`: browser APIs, messaging, storage wrappers
- `platform/api-client/`: client duy nhất cho API nội bộ khai `api-1` + union `ErrorCode`
- `platform/providers/`: mỗi upstream không tuân `api-1` một file adapter riêng (Zalo OA, Facebook Graph)
- `shared/`: shared helpers rất tiết chế

## Rule

- không để `shared/` thành thùng rác
- browser API wrappers nên đi qua `platform/`
- feature logic không rải khắp background/content/popup
- Permission trong `manifest.json` phải review theo từng feature, không xin quyền rộng mặc định.
- Test unit cho `features/` và `platform/`; integration test message flow quan trọng.
- Build/package artifact phải tái lập được, không commit output extension store.

## API response contract

Consumer của [`03-standards/API_RESPONSE_CONTRACT.md`](../../../03-standards/API_RESPONSE_CONTRACT.md), contract `api-1`.

- `src/platform/api-client/` là client duy nhất cho **API nội bộ khai `api-1`** — background/content/popup và mọi `features/` gọi qua đó, cấm `fetch` rải rác.
- **Upstream không tuân chuẩn đi đường riêng.** Mỗi provider một adapter trong `src/platform/providers/` (`zalo.js` → `normalizeZaloError`, `graph.js` → `normalizeGraphError`): quyết định thành công/thất bại theo luật CỦA HỌ trước — Zalo OA trả `200` kèm `{"error": -216}` khi thất bại — rồi mới normalize về error object của mình. Luật đầy đủ: [appendix section 2.5](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md).
- **Cấm dùng chung một api client cho cả 3 loại upstream** (nội bộ, Zalo OA, Graph). **Cấm duck-type `body.error`** — Graph cũng có key `error`, nhưng `error.code` của họ là SỐ chứ không phải chuỗi.
- **`src/platform/messaging/` bắt đủ 3 cơ chế hỏng** (bảng ở appendix section 2.2): throw đồng bộ / promise reject khi context invalidated → `ext.context_invalidated`; `chrome.runtime.lastError` + `response === undefined` → `ext.no_receiver`; `port.onDisconnect` → `ext.port_disconnected`.
- Union `ErrorCode` ở `src/shared/` gồm 4 nhóm: code của API nội bộ, code `ext.*` của transport, code `client.*` của lỗi phía máy người dùng, và code đã normalize từ provider ngoài. Popup, content script và `features/` `switch` trên đúng union này.
- Lỗi chỉ tồn tại ở máy người dùng (mất mạng, user đóng tab, ghi `chrome.storage` hỏng) dùng nhóm `client.*` ở [appendix section 2.0b](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md): `client.offline` (`retryable: true`), `client.cancelled`, `client.storage_failed`. Không mượn `unavailable`/`timeout` — hai mã đó nghĩa là server có vấn đề.
- Client phải TOLERANT: gặp `code` lạ thì fallback theo status class, không crash.
- `error.trace_id` ghi vào log của extension để đối chiếu với log server.

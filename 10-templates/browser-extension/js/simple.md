# Browser Extension JS Simple

## Khi nào dùng

- extension nhỏ-vừa
- popup, content script, background script cơ bản

## Cây thư mục

```text
<extension-name>/
├── docs/                 # tài liệu, flow, ghi chú kỹ thuật
├── infra/                # config build/package (zip, CRX)
├── scripts/              # script build/watch/package
├── config/               # file cấu hình mẫu
├── build/                # output bundle — gitignore
├── src/                  # vùng source chính
│   ├── background/       # service worker / background script (MV3)
│   ├── content/          # content script inject vào trang web
│   ├── popup/            # UI popup khi click icon extension
│   ├── options/          # trang Options của extension
│   ├── assets/           # icon, ảnh, css tĩnh
│   ├── providers/        # adapter cho upstream KHÔNG tuân api-1 (zalo.js, graph.js)
│   ├── api-client.js     # client cho API nội bộ api-1 + wrapper messaging + ErrorCode
│   └── manifest.json     # manifest extension (MV3)
├── tests/                # test nằm ngoài source chính
├── package.json          # dependency + script build
├── README.md             # hướng dẫn repo
└── .gitignore            # bỏ qua node_modules/, build/, *.zip
```

## Vai trò thư mục

- `background/`: background service worker / background script
- `content/`: content scripts
- `popup/`: popup UI
- `options/`: options page
- `providers/`: mỗi upstream không tuân `api-1` một file adapter riêng (Zalo OA, Facebook Graph)
- `api-client.js`: client cho API nội bộ `api-1`, wrapper messaging, và union `ErrorCode`
- `manifest.json`: extension manifest

## Rule

- không nhét toàn bộ logic vào content script
- `build/` là output bundle, phải gitignore
- popup logic và background logic tách rõ
- `manifest.json` phải xin quyền tối thiểu; quyền mới phải gắn với feature thật.
- Message contract giữa popup/content/background phải có test hoặc ghi chú rõ.
- Config endpoint/key public đặt trong config mẫu; secret thật không nhúng vào bundle extension.
- Script package/deploy store phải tạo artifact từ source sạch, không dùng file build thủ công.
- Nếu bắt đầu nhiều feature, thêm `features/` và nâng sang `structured.md`.

## API response contract

Consumer của [`03-standards/API_RESPONSE_CONTRACT.md`](../../../03-standards/API_RESPONSE_CONTRACT.md), contract `api-1`.

- `src/api-client.js` là client duy nhất cho **API nội bộ khai `api-1`** — background/content/popup gọi qua đó, cấm `fetch` rải rác.
- **Upstream không tuân chuẩn đi đường riêng.** Mỗi provider một adapter trong `src/providers/` (`zalo.js` → `normalizeZaloError`, `graph.js` → `normalizeGraphError`): quyết định thành công/thất bại theo luật CỦA HỌ trước — Zalo OA trả `200` kèm `{"error": -216}` khi thất bại — rồi mới normalize về error object của mình. Luật đầy đủ: [appendix section 2.5](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md).
- **Cấm dùng chung một api client cho cả 3 loại upstream** (nội bộ, Zalo OA, Graph). **Cấm duck-type `body.error`** — Graph cũng có key `error`, nhưng `error.code` của họ là SỐ chứ không phải chuỗi.
- **Messaging hỏng theo 3 cơ chế, `src/api-client.js` bắt đủ cả ba** (bảng ở appendix section 2.2): throw đồng bộ / promise reject khi context invalidated → `ext.context_invalidated`; `chrome.runtime.lastError` + `response === undefined` → `ext.no_receiver`; `port.onDisconnect` → `ext.port_disconnected`.
- Union `ErrorCode` khai cùng chỗ, gồm 4 nhóm: code của API nội bộ, code `ext.*` của transport, code `client.*` của lỗi phía máy người dùng, và code đã normalize từ provider ngoài. Popup và content script `switch` trên đúng union này.
- Lỗi chỉ tồn tại ở máy người dùng (mất mạng, user đóng tab, ghi `chrome.storage` hỏng) dùng nhóm `client.*` ở [appendix section 2.0b](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md): `client.offline` (`retryable: true`), `client.cancelled`, `client.storage_failed`. Không mượn `unavailable`/`timeout` — hai mã đó nghĩa là server có vấn đề.
- Client phải TOLERANT: gặp `code` lạ thì fallback theo status class, không crash.
- `error.trace_id` ghi vào log của extension để đối chiếu với log server.

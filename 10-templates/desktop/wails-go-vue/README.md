# Desktop — Wails + Go + Vue

## Khi nào dùng

- App desktop dùng Go backend + webview frontend
- structured khi app lớn hơn và cần boundary rõ giữa Go/FE

## Có gì trong thư mục này

- `simple.md`: dùng khi cần ship nhanh, repo còn gọn
- `structured.md`: dùng khi code bắt đầu phình và cần boundary rõ hơn

## Baseline lâu dài

- Backend Go và frontend Vue phải có boundary rõ; frontend không gọi trực tiếp logic không được expose qua binding/API.
- Build/release phải ghi rõ target OS, artifact output, version, và cách ký/cài nếu production cần.
- Config mẫu và runtime data tách khỏi source; không commit file local của người dùng.
- Nếu có sidecar binary, dùng quy ước `sidecar/` trong core và ghi version/source rõ.
- Test backend Go độc lập với webview; test frontend như Vue/Vite bình thường.
- Bridge Wails áp **phần error** của [`03-standards/API_RESPONSE_CONTRACT.md`](../../../03-standards/API_RESPONSE_CONTRACT.md) (contract `api-1`): bound method trả typed error, wrapper serialize thành `{code, message, trace_id, retryable}`; success vẫn trả giá trị trần theo idiom Go, không bọc envelope.
- Sidecar là một **upstream không tuân `api-1`**: mỗi sidecar một adapter riêng ([appendix section 2.5](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md)), và trạng thái **chưa chạy** map thành `dependency_failed` + `message` chỉ đích danh sidecar; không để `ECONNREFUSED` rò lên UI.
- Lỗi phía máy người dùng (offline, user huỷ, ghi storage local hỏng) dùng nhóm `client.*` ([appendix section 2.0b](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md)), không mượn `unavailable`/`timeout`; POST đẩy thay đổi local lên server bắt buộc `Idempotency-Key`.

## Ghi chú

- Wails có đặc thù riêng: phải giữ ranh giới rõ giữa backend và frontend.

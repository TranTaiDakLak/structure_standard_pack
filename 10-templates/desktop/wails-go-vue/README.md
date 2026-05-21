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

## Ghi chú

- Wails có đặc thù riêng: phải giữ ranh giới rõ giữa backend và frontend.

# Browser Extension — JavaScript

## Khi nào dùng

- Extension viết bằng JS thuần hoặc JS + bundler nhẹ
- structured khi extension có nhiều feature, nhiều page nội bộ

## Có gì trong thư mục này

- `simple.md`: dùng khi cần ship nhanh, repo còn gọn
- `structured.md`: dùng khi code bắt đầu phình và cần boundary rõ hơn

## Baseline lâu dài

- Manifest permission phải tối thiểu, không xin quyền rộng nếu feature chưa dùng.
- Background, content, popup, options phải có ranh giới trách nhiệm; không dồn toàn bộ logic vào content script.
- Build/package script phải tạo artifact tái lập được và không commit `build/`, zip, CRX.
- Config endpoint/key public phải đi qua config mẫu; secret thật không nằm trong source extension.
- Test tối thiểu wrapper browser API và message contract giữa background/content/popup.
- Stack này ở vai **consumer** của [`03-standards/API_RESPONSE_CONTRACT.md`](../../../03-standards/API_RESPONSE_CONTRACT.md) (contract `api-1`): một client cho API nội bộ, một adapter riêng cho mỗi upstream không tuân chuẩn (Zalo OA, Facebook Graph — [appendix section 2.5](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md)), client phải tolerant với code lạ.
- Messaging nội bộ hỏng theo **3 cơ chế** (context invalidated, không có receiver, port đóng) — wrap đủ cả ba về cùng error object với lỗi API; union `ErrorCode` gộp code nội bộ, code `ext.*`, code `client.*` (offline, user huỷ, ghi storage hỏng — [appendix section 2.0b](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md)), và code đã normalize từ provider ngoài.

## Ghi chú

- Scope là browser extension kiểu Chrome/Edge/Chromium.

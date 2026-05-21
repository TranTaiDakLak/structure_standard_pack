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

## Ghi chú

- Scope là browser extension kiểu Chrome/Edge/Chromium.

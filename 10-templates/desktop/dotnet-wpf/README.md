# Desktop — .NET WPF

## Khi nào dùng

- App desktop WPF nội bộ hoặc sản phẩm Windows
- simple theo MVVM nhẹ, structured khi domain lớn hơn

## Có gì trong thư mục này

- `simple.md`: dùng khi cần ship nhanh, repo còn gọn
- `structured.md`: dùng khi code bắt đầu phình và cần boundary rõ hơn

## Baseline lâu dài

- View chỉ render/bind; nghiệp vụ nằm ở ViewModel/Application/Service tùy mode.
- Config mẫu, installer, sidecar và output publish phải tách rõ khỏi source.
- Test ưu tiên ViewModel, service, domain; không phụ thuộc UI automation nếu chưa cần.
- Resource, theme, icon, localization nên ở vị trí rõ để tránh rải trong từng View.
- Khi nhiều module nghiệp vụ hoặc nhiều integration ngoài, chuyển sang `structured.md`.

## Ghi chú

- Không bắt buộc framework MVVM cụ thể; pack chỉ chuẩn hóa cấu trúc.

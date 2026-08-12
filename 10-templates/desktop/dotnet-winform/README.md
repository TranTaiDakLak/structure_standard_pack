# Desktop — .NET WinForms

## Khi nào dùng

- App desktop WinForms cho tool nội bộ, line-of-business Windows
- Dự án cần ship nhanh trên máy Windows, không cần cross-platform
- Team đã quen paradigm event-driven + Form Designer
- simple dùng pattern MVP/Controller nhẹ, structured khi domain lớn hơn

## Có gì trong thư mục này

- `simple.md`: dùng khi repo còn gọn, 1 project duy nhất
- `structured.md`: dùng khi code phình, cần tách domain / application / infrastructure

## Baseline lâu dài

- UI event handler chỉ forward sang Presenter/Service; không chứa nghiệp vụ dài.
- Config mẫu, installer script, sidecar binary nếu có phải tách khỏi source chính.
- Test tập trung vào Presenter/Application/Domain; không cố test Form Designer trực tiếp nếu không cần.
- Release desktop phải có version, publish output, installer config, và ghi chú rollback/cài lại.
- App ở **vai consumer** của [`03-standards/API_RESPONSE_CONTRACT.md`](../../../03-standards/API_RESPONSE_CONTRACT.md) (contract `api-1`): gọi API ngoài thì parse `error.code` / `retryable` / `trace_id` tại đúng một chỗ, không rải `HttpClient` trong `Forms`/`Presenters`.
- Mỗi upstream **không tuân `api-1`** (partner API, sidecar exe, hệ thống nội bộ cũ) có adapter riêng đặt cùng chỗ với http client — `Services/` ở simple, `.Infrastructure` ở structured ([appendix section 2.5](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md)); lỗi phía máy người dùng (offline, user huỷ) dùng nhóm `client.*` ([appendix section 2.0b](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md)), không mượn `unavailable`/`timeout`.
- Nếu Form tăng nhanh hoặc nhiều integration ngoài, nâng sang `structured.md` trước khi logic dính vào UI.

## Ghi chú

- WinForms không có data-binding 2 chiều mạnh như WPF → pattern khuyến nghị là **MVP** (Model–View–Presenter) hoặc Controller-style, KHÔNG gọi là MVVM.
- Business logic KHÔNG được nằm trong file `*.Designer.cs` hoặc trong handler sự kiện của Form.
- Nếu dự án đang có WPF và WinForm song song → chọn đúng template theo project thật, không trộn cấu trúc.

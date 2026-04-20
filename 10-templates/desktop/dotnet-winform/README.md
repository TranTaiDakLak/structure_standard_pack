# Desktop — .NET WinForms

## Khi nào dùng

- App desktop WinForms cho tool nội bộ, line-of-business Windows
- Dự án cần ship nhanh trên máy Windows, không cần cross-platform
- Team đã quen paradigm event-driven + Form Designer
- simple dùng pattern MVP/Controller nhẹ, structured khi domain lớn hơn

## Có gì trong thư mục này

- `simple.md`: dùng khi repo còn gọn, 1 project duy nhất
- `structured.md`: dùng khi code phình, cần tách domain / application / infrastructure

## Ghi chú

- WinForms không có data-binding 2 chiều mạnh như WPF → pattern khuyến nghị là **MVP** (Model–View–Presenter) hoặc Controller-style, KHÔNG gọi là MVVM.
- Business logic KHÔNG được nằm trong file `*.Designer.cs` hoặc trong handler sự kiện của Form.
- Nếu dự án đang có WPF và WinForm song song → chọn đúng template theo project thật, không trộn cấu trúc.

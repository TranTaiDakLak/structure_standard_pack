# Service — .NET Worker Service

## Khi nào dùng

- Dự án chạy ngầm, không có UI, không phục vụ HTTP công khai (nếu có API thì là endpoint quản trị nội bộ)
- Cover được 3 use case phổ biến trong 1 cấu trúc:
  - Cài đặt **Windows Service** (auto-start khi boot, quản lý qua SCM)
  - Chạy **background worker** trong Docker/Linux daemon
  - Chạy **scheduled job** (Quartz.NET / Hangfire, hoặc exe kích bởi Task Scheduler)
- Team .NET muốn tận dụng `IHostedService` / `BackgroundService` chuẩn MS

## Có gì trong thư mục này

- `simple.md`: dùng khi 1–2 worker, 1 project duy nhất
- `structured.md`: dùng khi có nhiều worker, domain lớn, cần tách lớp

## Ghi chú

- Với .NET Framework cũ (4.x) dùng `ServiceBase`: vẫn theo cấu trúc simple nhưng thay `Workers/` bằng class kế thừa `ServiceBase` và project `OutputType=WinExe`. Pack không ship template riêng cho legacy — tận dụng simple + ghi chú trong repo.
- Nếu service cần expose HTTP endpoint nội bộ (health check, trigger job manual): thêm `Minimal API` trong `Program.cs`, KHÔNG tách thành delivery_type `web`.
- Deploy target khuyến nghị: `windows-service` (sc.exe / nssm), `docker` (Linux container), hoặc exe đơn thuần kích bởi Task Scheduler.

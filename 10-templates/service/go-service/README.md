# Service — Go

## Khi nào dùng

- Service Go chạy ngầm, không có UI, không phục vụ HTTP công khai (nếu có endpoint thì là admin/health nội bộ)
- Cover 3 use case phổ biến trong 1 cấu trúc:
  - **Daemon / long-running worker** (consume queue, watch file, sync định kỳ)
  - **Windows Service** (dùng `kardianos/service` hoặc `golang.org/x/sys/windows/svc`)
  - **Scheduled job** (robfig/cron hoặc exe kích bởi Task Scheduler / cron)
- Team thích Go vì single binary, memory footprint nhỏ, dễ deploy trên container / Linux

## Có gì trong thư mục này

- `simple.md`: dùng khi 1–2 worker, code còn gọn
- `structured.md`: dùng khi nhiều worker / job + tích hợp ngoài đa dạng

## Baseline lâu dài

- Worker/job chỉ orchestrate; business logic nằm trong `service/` hoặc `internal/usecase/`.
- Mọi loop dài phải dùng `context.Context`, signal handling, logging chuẩn, retry/backoff rõ.
- Config mẫu không chứa secret; runtime config nạp từ file/env theo deploy target.
- Deploy phải nói rõ chạy bằng Docker, systemd, Windows Service, cron, hay Task Scheduler.
- Health/metrics/admin endpoint nếu có chỉ là nội bộ và không làm đổi `delivery_type`.

## Ghi chú

- Với Go, chạy console ngầm hay Windows Service đều cùng 1 binary — khác nhau ở bootstrap. Template ưu tiên chạy console + script cài service ở `infra/service-install/`.
- Deploy target khuyến nghị: `docker`, `linux-daemon` (systemd), `windows-service`, hoặc exe kích bởi cron/Task Scheduler.
- Nếu service cần expose endpoint nội bộ (health, metrics, trigger manual) → thêm HTTP nhẹ (net/http hoặc Chi) trong `main.go`, KHÔNG chuyển sang delivery_type `web`.
- Quyết định "service hay web" khi app vừa chạy nền vừa có HTTP: xem [`STRUCTURE_STANDARD_CORE.md`](../../../00-core/STRUCTURE_STANDARD_CORE.md) mục "Ranh giới Service ↔ Web". Tóm tắt: chủ thể chính là **việc chạy nền** + HTTP chỉ nội bộ → ở lại `service`; chủ thể chính là **API/UI phục vụ user** + worker chỉ phụ trợ → dùng `web/fullstack/<stack>`; cả hai đều nặng và ít share → tách 2 repo.

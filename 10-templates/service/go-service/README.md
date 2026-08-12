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
- Ràng buộc bởi [`03-standards/API_RESPONSE_CONTRACT.md`](../../../03-standards/API_RESPONSE_CONTRACT.md) (contract `api-1`) ở 2 mức. **Mức 1 — hợp đồng với hệ thống chạy service:** `/healthz` + `/readyz` đúng hình dạng khi service có HTTP surface; job chạy một lần rồi thoát thì miễn HTTP và lấy **exit code** làm contract (`0` = mọi bước bắt buộc hoàn tất, `!= 0` = cần người can thiệp), danh sách bước bắt buộc / best-effort khai trong `docs/` — [appendix](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md) section 2.3. `trace_id` của lần chạy là `run_id` sinh lúc bắt đầu, persist cùng bản ghi (appendix section 2.0).
- **Mức 2 — vai consumer, áp cho mọi service:** gọi API tự khai `contract: api-1` thì đọc `error.code` + `error.retryable`; gọi API bên thứ 3 thì **HTTP status code + header `Retry-After` mới là nguồn sự thật**, mỗi upstream một adapter riêng map về taxonomy của mình — [appendix](../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md) section 2.5.
- Endpoint HTTP nội bộ nếu có (admin, trigger) thì áp toàn bộ contract như một API thật.

## Ghi chú

- Với Go, chạy console ngầm hay Windows Service đều cùng 1 binary — khác nhau ở bootstrap. Template ưu tiên chạy console + script cài service ở `infra/service-install/`.
- Deploy target khuyến nghị: `docker`, `linux-daemon` (systemd), `windows-service`, hoặc exe kích bởi cron/Task Scheduler.
- Nếu service cần expose endpoint nội bộ (health, metrics, trigger manual) → thêm HTTP nhẹ (net/http hoặc Chi) trong `main.go`, KHÔNG chuyển sang delivery_type `web`.
- Quyết định "service hay web" khi app vừa chạy nền vừa có HTTP: xem [`STRUCTURE_STANDARD_CORE.md`](../../../00-core/STRUCTURE_STANDARD_CORE.md) mục "Ranh giới Service ↔ Web". Tóm tắt: chủ thể chính là **việc chạy nền** + HTTP chỉ nội bộ → ở lại `service`; chủ thể chính là **API/UI phục vụ user** + worker chỉ phụ trợ → dùng `web/fullstack/<stack>`; cả hai đều nặng và ít share → tách 2 repo.

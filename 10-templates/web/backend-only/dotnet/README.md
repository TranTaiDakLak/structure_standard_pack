# Web Backend — ASP.NET Core

## Khi nào dùng

- API .NET cho internal tool hoặc sản phẩm web
- simple cho repo nhỏ
- structured khi bắt đầu cần phân lớp rõ hơn

## Có gì trong thư mục này

- `simple.md`: dùng khi cần ship nhanh, repo còn gọn
- `structured.md`: dùng khi code bắt đầu phình và cần boundary rõ hơn

## Baseline lâu dài

- Controller mỏng; nghiệp vụ nằm trong Services/Application, data access nằm trong Repository/Infrastructure.
- Config mẫu dùng `appsettings.example.json` hoặc env mapping; không commit connection string thật.
- API production nên có health/readiness, logging có correlation id, migration path rõ.
- Stack này sinh HTTP API nên bị ràng buộc **toàn bộ** [`03-standards/API_RESPONSE_CONTRACT.md`](../../../../03-standards/API_RESPONSE_CONTRACT.md) (contract `api-1`): Tier 0 bắt buộc kể cả `simple.md`; Tier 1 và Tier 2 áp theo **điều kiện của dự án** (có consumer ngoài repo, endpoint đụng tiền, collection lớn dần...), không theo mode thư mục — chọn `simple` không miễn Tier 2, xem section 3 của chuẩn.
- Server đặt `retryable` theo **bản chất lỗi** (tạm thời hay không), KHÔNG hàm ý client được phát lại request: `timeout`/`dependency_failed` vẫn `true` dù việc có thể đã chạy xong ở server, nên POST không idempotent chỉ được client tự động phát lại khi có `Idempotency-Key` — section 6.1b + Tier 2 của chuẩn.
- Riêng .NET: `Program.cs` phải tắt `ProblemDetails` tự sinh (`SuppressModelStateInvalidFilter = true`) và set `JsonNamingPolicy.SnakeCaseLower`, xem section `API response contract` trong `simple.md`/`structured.md`.
- Test tối thiểu unit cho nghiệp vụ và integration cho DB/API quan trọng.
- Deploy target có thể là IIS, Kestrel + reverse proxy (nginx), hoặc service Linux/Windows; `infra/` chỉ chứa thứ đang dùng thật.

## Ghi chú

- Chưa mặc định Clean Architecture full enterprise.
- Structured ở đây là mức vừa phải cho team nhỏ-vừa.

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
- Test tối thiểu unit cho nghiệp vụ và integration cho DB/API quan trọng.
- Deploy target có thể là IIS, Kestrel + reverse proxy (nginx), hoặc service Linux/Windows; `infra/` chỉ chứa thứ đang dùng thật.

## Ghi chú

- Chưa mặc định Clean Architecture full enterprise.
- Structured ở đây là mức vừa phải cho team nhỏ-vừa.

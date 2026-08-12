# ASP.NET Core Backend Simple

## Khi nào dùng

- API nhỏ hoặc vừa
- team 1–3 người
- MVP hoặc internal tool

## Cây thư mục

```text
<ServiceName>/
├── docs/                               # tài liệu, flow, ghi chú kỹ thuật
├── infra/                              # iis, nginx, deployment config
│   └── iis/                            # web.config — override trang lỗi IIS/ANCM
├── scripts/                            # script build/publish/migrate
├── config/                             # appsettings.example.json, không secret thật
├── tests/                              # test nằm ngoài source chính
│   └── <ServiceName>.Tests/            # unit test project
├── src/                                # vùng source chính
│   └── <ServiceName>/                  # project ASP.NET Core duy nhất
│       ├── Controllers/                # API endpoint (Attribute Routing)
│       ├── Middlewares/                # middleware: exception handling, request id
│       ├── Services/                   # business logic
│       ├── Repositories/               # truy cập DB (EF Core / Dapper)
│       ├── Models/                     # entity / domain model đơn giản
│       ├── Dtos/                       # request/response DTO
│       │   └── Common/                 # ApiError, ErrorDetail, PagedResult<T>, ErrorCodes
│       ├── Program.cs                  # entrypoint + DI + middleware pipeline
│       └── appsettings.json            # cấu hình runtime
├── <ServiceName>.sln                   # solution file
├── Directory.Build.props               # thiết lập build dùng chung
├── README.md                           # hướng dẫn repo
└── .gitignore                          # bỏ qua bin/, obj/, *.user...
```

## Vai trò thư mục

- `Controllers/`: API endpoints
- `Middlewares/`: exception handling, request id — pipeline cross-cutting
- `Services/`: business logic
- `Repositories/`: data access
- `Models/`: entity/model đơn giản
- `Dtos/`: request/response DTO
- `Dtos/Common/`: type dùng chung cho mọi controller — `ApiError`, `ErrorDetail`, `PagedResult<T>`, `ErrorCodes.cs`
- `infra/iis/`: `web.config` — override lỗi IIS/ANCM sinh trước khi request tới app

## Rule

- controller không ôm logic dài
- `src/` và `tests/` tách riêng
- nếu flow bắt đầu rối, nâng lên structured
- Config mẫu dùng `appsettings.example.json` hoặc env mapping; secret thật không nằm trong repo.
- Nếu có DB, migration phải có script hoặc hướng dẫn rõ, không chạy ngầm ngoài kiểm soát.
- API production nên có healthcheck, logging có correlation id, validation và error response thống nhất.

## API response contract

Theo [`03-standards/API_RESPONSE_CONTRACT.md`](../../../../03-standards/API_RESPONSE_CONTRACT.md), contract `api-1` — không tự chế hình dạng response, không tự chế bảng mã lỗi.

- Response helper + writer: `src/<ServiceName>/Dtos/Common/` — `ApiError`, `ErrorDetail`, `PagedResult<T>` dùng chung; mọi controller trả lỗi và list qua đây.
- File khai báo error code, đúng 1 file cho cả repo: `src/<ServiceName>/Dtos/Common/ErrorCodes.cs` — static class, `const string`.
- Exception/recover middleware toàn cục: `src/<ServiceName>/Middlewares/ExceptionHandlingMiddleware.cs`, đăng ký sớm nhất trong pipeline ở `Program.cs`.
- `Program.cs` bắt buộc 3 khối sau (cần .NET 8+ cho `SnakeCaseLower`; .NET 6/7 phải tự viết `JsonNamingPolicy`). Thiếu khối 1 thì success trả camelCase, còn error chỉ đúng snake_case khi ghi từ middleware — lỗi trả bằng `ObjectResult` của controller (422 từ ValidationFilter) cũng camelCase. Sai im lặng: soát bằng một request lỗi đi qua middleware sẽ thấy đúng, phải gọi thử endpoint success mới lộ.

  ```csharp
  // 1. Controller MVC — BẮT BUỘC. Thiếu dòng này thì mọi response success trả camelCase.
  builder.Services.AddControllers()
      .AddJsonOptions(o => o.JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.SnakeCaseLower);

  // 2. Exception middleware ghi bằng WriteAsJsonAsync — dùng options KHÁC, phải set riêng.
  builder.Services.ConfigureHttpJsonOptions(o => o.SerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.SnakeCaseLower);

  // 3. Tắt ProblemDetails tự sinh cho lỗi model-binding, để tự trả 422 đúng contract.
  builder.Services.Configure<ApiBehaviorOptions>(o => o.SuppressModelStateInvalidFilter = true);
  ```

  Vì `[ApiController]` tự sinh RFC 7807 `ValidationProblemDetails` cho lỗi model-binding, và `System.Text.Json` mặc định camelCase.

- `user_message` và `details` phải **vắng mặt** khi rỗng chứ không gửi `null` (chuẩn section 5.4) — `System.Text.Json` mặc định ghi cả null. Đánh dấu trên **đúng hai property đó** của `ApiError`:

  ```csharp
  public sealed record ApiErrorBody(
      string Code, string Message, string TraceId, bool Retryable,
      [property: JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)] string? UserMessage,
      [property: JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)] IReadOnlyList<ErrorDetail>? Details);
  ```

  **Đừng** bật `DefaultIgnoreCondition = WhenWritingNull` toàn cục ở khối 1/2 để làm việc này: nó sửa được envelope lỗi nhưng **xóa luôn field null trong DTO success**, trong khi section 10 bắt buộc field đã khai trong schema luôn phải có mặt. Đo được: bật global thì response success mất hẳn khóa `note` thay vì trả `"note": null`.
- Khối 3 chỉ **tắt** hành vi mặc định chứ không thay thế nó — thiếu filter của mình thì model-binding fail đi thẳng vào action với model rỗng, tệ hơn trước khi áp chuẩn. Filter đặt cạnh `ExceptionHandlingMiddleware.cs` trong `src/<ServiceName>/Middlewares/`:

  ```csharp
  // 4. Đăng ký ngay trong AddControllers của khối 1.
  builder.Services.AddControllers(o =>
  {
      o.Filters.Add<ValidationFilter>();
      // ModelState key phải là tên field trên wire, không phải tên property C#
      o.ModelMetadataDetailsProviders.Add(new WireNames(JsonNamingPolicy.SnakeCaseLower));
  }); // .AddJsonOptions(...) của khối 1 nối tiếp vào đây

  sealed class WireNames(JsonNamingPolicy p) : IValidationMetadataProvider
  {
      public void CreateValidationMetadata(ValidationMetadataProviderContext c) =>
          c.ValidationMetadata.ValidationModelName =
              c.Attributes.OfType<JsonPropertyNameAttribute>().FirstOrDefault()?.Name
              ?? (c.Key.Name is null ? null : p.ConvertName(c.Key.Name));
  }

  sealed class ValidationFilter : IActionFilter
  {
      public void OnActionExecuted(ActionExecutedContext c) { }
      public void OnActionExecuting(ActionExecutingContext c)
      {
          if (c.ModelState.IsValid) return;
          var details = c.ModelState.Where(kv => kv.Value!.Errors.Count > 0).SelectMany(kv =>
              kv.Value!.Errors.Select(e => new ErrorDetail(
                  // "$." là tiền tố STJ gắn cho lỗi decode; param query/path thì đổi In cho khớp
                  Field: kv.Key.TrimStart('$', '.'), In: "body",
                  Code: "invalid_format", Message: e.ErrorMessage))).ToList();
          var decodeFailed = c.ModelState.Keys.Any(k => k.StartsWith("$."));  // section 9: decode hỏng là 400
          // ApiError.Of: trace_id lấy từ HttpContext, retryable tra từ bảng code
          c.Result = new ObjectResult(decodeFailed
              ? ApiError.Of(c.HttpContext, ErrorCodes.BadRequest, "malformed request body")
              : ApiError.Of(c.HttpContext, ErrorCodes.ValidationFailed, "request body failed validation", details))
              { StatusCode = decodeFailed ? 400 : 422 };
      }
  }
  ```

- PATCH: `System.Text.Json` để `{"ly_do": null}` và field vắng mặt cùng ra `null`, trong khi section 10 của chuẩn bắt buộc phân biệt (vắng = giữ nguyên, `null` = xóa) — bỏ qua là mất dữ liệu thật. `Optional<T>` + converter đặt cùng chỗ với `ApiError`, ở `src/<ServiceName>/Dtos/Common/`:

  ```csharp
  public readonly record struct Optional<T>(T? Value, bool IsSet);  // vắng mặt = default → IsSet false

  public sealed class OptionalConverter<T> : JsonConverter<Optional<T>>  // đăng ký qua JsonConverterFactory
  {
      public override Optional<T> Read(ref Utf8JsonReader r, Type t, JsonSerializerOptions o)
          => new(JsonSerializer.Deserialize<T>(ref r, o), true);  // converter CHỈ chạy khi key có mặt
      public override void Write(Utf8JsonWriter w, Optional<T> v, JsonSerializerOptions o)
          => JsonSerializer.Serialize(w, v.Value, o);
  }
  // DTO:   public Optional<string?> LyDo { get; init; }
  // Apply: if (dto.LyDo.IsSet) order.LyDo = dto.LyDo.Value;   // null = xóa, vắng = không đụng tới
  ```

- Deploy sau IIS: app pool recycle hoặc app chưa boot thì IIS/ANCM tự trả HTML 502.3/503 trước khi request tới app, writer trong app vô dụng — `infra/iis/web.config` phải override chúng thành JSON envelope và tự sinh `X-Request-Id` ([appendix](../../../../03-standards/API_RESPONSE_CONTRACT_APPENDIX.md) section 4, hàng reverse proxy).
- `docs/api-contract.md`: contract version, bảng domain error code, danh sách endpoint ngoại lệ.

Không đẻ layer mới cho việc này — mọi path trên đều nằm trong cây thư mục ở trên.

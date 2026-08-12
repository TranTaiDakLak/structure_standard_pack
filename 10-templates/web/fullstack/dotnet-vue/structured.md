# Fullstack Structured — ASP.NET Core + Vue

## Khi nào dùng

- product dài hạn hơn
- backend và frontend cùng nhiều module

## Cây thư mục

```text
<product-name>/
├── docs/                                   # tài liệu chung của product
├── infra/                                  # deployment config chung
│   └── iis/                                # web.config — override trang lỗi IIS/ANCM
├── scripts/                                # script dev/build/deploy chung
├── config/                                 # file cấu hình mẫu
├── backend/                                # ASP.NET Core — như dotnet/structured
│   ├── src/                                # multi-project
│   │   ├── <ServiceName>.Api/              # controller, middleware, Program.cs
│   │   ├── <ServiceName>.Application/      # use case, DTO, orchestration
│   │   ├── <ServiceName>.Domain/           # entity, rule — không phụ thuộc framework
│   │   └── <ServiceName>.Infrastructure/   # EF Core, HTTP client, adapter
│   ├── tests/                              # test BE (unit + integration)
│   ├── <ServiceName>.sln                   # solution
│   ├── Directory.Build.props               # thiết lập build chung
│   └── Directory.Packages.props            # quản lý version NuGet tập trung
├── frontend/                               # Vue app — như vue-vite/structured
│   ├── public/                             # tài nguyên tĩnh
│   ├── src/                                # source Vue
│   │   ├── components/                     # component generic
│   │   ├── composables/                    # composable dùng chung
│   │   ├── features/                       # logic theo domain
│   │   ├── pages/                          # page mỏng
│   │   ├── router/                         # vue-router
│   │   ├── services/                       # API client gọi BE
│   │   ├── stores/                         # Pinia store
│   │   ├── types/                          # TypeScript type
│   │   ├── App.vue                         # root component
│   │   └── main.ts                         # entrypoint
│   ├── tests/                              # test FE
│   ├── vite.config.ts                      # cấu hình Vite
│   └── package.json                        # dependency FE
├── README.md                               # hướng dẫn repo
└── .gitignore
```

## Vai trò thư mục

- backend structured theo .NET layered layout
- frontend structured theo Vue feature-based layout
- `infra/iis/`: `web.config` — override lỗi IIS/ANCM sinh trước khi request tới app

## Rule

- structured để dễ maintain, không phải để giả enterprise
- nếu một phía còn nhỏ, có thể chỉ nâng phía đó trước
- Backend và frontend giữ test/build pipeline riêng; root script chỉ orchestration.
- API contract ổn định nên có OpenAPI hoặc tài liệu tương đương, không bắt buộc codegen.
- Production cần healthcheck, logging, migration, secret loading và deploy target rõ.
- Nếu shared package/contracts chưa có nhu cầu thật, không thêm layer chung.

## API response contract

Theo [`03-standards/API_RESPONSE_CONTRACT.md`](../../../../03-standards/API_RESPONSE_CONTRACT.md), contract `api-1` — không tự chế hình dạng response, không tự chế bảng mã lỗi.

- Response helper + writer: type (`ApiError`, `ErrorDetail`, `PagedResult<T>`) ở `backend/src/<ServiceName>.Application/`, writer + `IActionResult` mapping ở `backend/src/<ServiceName>.Api/` — **không** đẻ project `.Contracts`
- File khai báo error code, đúng 1 file cho cả repo: `backend/src/<ServiceName>.Application/ErrorCodes.cs` — static class, `const string`; domain code `<module>.<reason>` khớp tên use case trong `.Application`
- Exception/recover middleware toàn cục: `backend/src/<ServiceName>.Api/` (middleware + `Program.cs`) — `.Domain` trả typed error, `.Api` mới map sang HTTP status
- `[ApiController]` tự sinh RFC 7807 cho lỗi model-binding và `System.Text.Json` mặc định camelCase. `Program.cs` bắt buộc 3 khối sau (cần .NET 8+ cho `SnakeCaseLower`; .NET 6/7 phải tự viết `JsonNamingPolicy`). Thiếu khối 1 thì success trả camelCase, còn error chỉ đúng snake_case khi ghi từ middleware — lỗi trả bằng `ObjectResult` của controller (422 từ ValidationFilter) cũng camelCase. Sai im lặng, và test DoD vẫn xanh nếu request lỗi cố ý đi qua middleware:

  ```csharp
  // 1. Controller MVC — BẮT BUỘC. Thiếu dòng này thì mọi response success trả camelCase.
  builder.Services.AddControllers()
      .AddJsonOptions(o => o.JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.SnakeCaseLower);

  // 2. Exception middleware ghi bằng WriteAsJsonAsync — dùng options KHÁC, phải set riêng.
  builder.Services.ConfigureHttpJsonOptions(o => o.SerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.SnakeCaseLower);

  // 3. Tắt ProblemDetails tự sinh cho lỗi model-binding, để tự trả 422 đúng contract.
  builder.Services.Configure<ApiBehaviorOptions>(o => o.SuppressModelStateInvalidFilter = true);
  ```

- `user_message` và `details` phải **vắng mặt** khi rỗng chứ không gửi `null` (chuẩn section 5.4) — `System.Text.Json` mặc định ghi cả null. Đánh dấu trên **đúng hai property đó** của `ApiError`:

  ```csharp
  public sealed record ApiErrorBody(
      string Code, string Message, string TraceId, bool Retryable,
      [property: JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)] string? UserMessage,
      [property: JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)] IReadOnlyList<ErrorDetail>? Details);
  ```

  **Đừng** bật `DefaultIgnoreCondition = WhenWritingNull` toàn cục ở khối 1/2 để làm việc này: nó sửa được envelope lỗi nhưng **xóa luôn field null trong DTO success**, trong khi section 10 bắt buộc field đã khai trong schema luôn phải có mặt. Đo được: bật global thì response success mất hẳn khóa `note` thay vì trả `"note": null`.

- Khối 3 chỉ **tắt** hành vi mặc định chứ không thay thế nó — thiếu filter của mình thì model-binding fail đi thẳng vào action với model rỗng, tệ hơn trước khi áp chuẩn. Filter thuộc tầng HTTP nên nằm ở `backend/src/<ServiceName>.Api/`:

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

  `details[].field` là tên trên wire nên FE map thẳng vào form state bằng lodash `get()`, không phải đoán từ tên property C#.
- PATCH: `System.Text.Json` để `{"ly_do": null}` và field vắng mặt cùng ra `null`, trong khi section 10 của chuẩn bắt buộc phân biệt (vắng = giữ nguyên, `null` = xóa) — bỏ qua là mất dữ liệu thật. `Optional<T>` + converter là type dùng chung nên đặt cạnh `ErrorCodes.cs` ở `backend/src/<ServiceName>.Application/`:

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
- FE Vue: api client trong `frontend/src/services/`, `frontend/src/types/api.ts` mirror union `ErrorCode` đúng tên code của backend để TS bắt được `switch` thiếu case
- `docs/api-contract.md`: contract version, bảng domain error code, danh sách endpoint ngoại lệ.

Không đẻ layer mới cho việc này — mọi path trên đều nằm trong cây thư mục ở trên.

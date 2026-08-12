# Migration Checklist

> Checklist ngắn để migrate repo cũ sang chuẩn cấu trúc mới mà không làm quá tay.

## 1. Xác định input trước

- [ ] `delivery_type` là gì
- [ ] `app_shape` là gì
- [ ] `stack` là gì
- [ ] mode hiện tại hợp với `simple` hay nên lên `structured`
- [ ] repo này có thật sự trong scope support của pack không

## 2. Audit hiện trạng

- [ ] entrypoint đang nằm ở đâu
- [ ] business logic đang nằm ở đâu
- [ ] integration đang nằm ở đâu
- [ ] docs có đang nằm lẫn trong source không
- [ ] deploy files có đang nằm sai chỗ không
- [ ] config mẫu có lẫn secret thật không
- [ ] build artifacts có đang bị commit không
- [ ] nếu repo có API: hiện đang trả response theo mấy kiểu khác nhau, mã lỗi rải rác ở đâu, có endpoint nào trả `200` kèm `success: false` không

## 3. Dọn root trước

- [ ] tạo `docs/` nếu chưa có
- [ ] tạo `infra/` nếu có deploy files
- [ ] tạo `scripts/` nếu đang có script rải rác
- [ ] tạo `config/` nếu đang có file mẫu cấu hình
- [ ] tạo `tests/` nếu test đang nằm lẫn trong source
- [ ] thêm / chuẩn hóa `.gitignore`
- [ ] tách `build/` và đảm bảo gitignore

## 4. Chuẩn hóa naming

- [ ] đổi `Config/` thành `config/`
- [ ] đổi tên folder mơ hồ
- [ ] thống nhất lowercase cho folder cấu trúc
- [ ] bỏ tên kiểu `temp-final`, `misc`, `common2`

## 5. Chọn đích migrate

### Nếu giữ `simple`
- [ ] gom code về đúng các folder simple
- [ ] giữ cấu trúc phẳng, không over-engineer
- [ ] tách business logic ra khỏi entrypoint

### Nếu lên `structured`
- [ ] tạo folder structured đích
- [ ] move trước, rewrite sau
- [ ] tách rõ entrypoint / business / integration

## 5b. Nếu repo có API — áp response contract

Làm **sau** khi cấu trúc đã ổn, không trộn vào cùng đợt move folder. Theo [`03-standards/API_RESPONSE_CONTRACT.md`](../03-standards/API_RESPONSE_CONTRACT.md).

**Trả lời câu này trước mọi thứ khác: API có consumer ngoài repo không?** Nó quyết định toàn bộ chiến lược migrate, không phải chi tiết chốt sau.

- [ ] liệt kê mọi caller thật: FE trong repo, app mobile đã phát hành, đối tác tích hợp, script/cron của khách, extension đã publish
- [ ] đối chiếu bằng access log (user-agent, client version, api key), không hỏi miệng
- [ ] không chắc → coi như **CÓ** consumer ngoài repo
- [ ] chốt nhánh: chỉ có client trong repo → **nhánh A**; có client ngoài repo → **nhánh B**

### Việc chung cho cả hai nhánh

Khối này dựng nền, chưa đổi status của endpoint nào đang chạy.

- [ ] tạo helper response + writer ở đúng folder của template (xem appendix section 3), chưa sửa handler nào
- [ ] gom toàn bộ mã lỗi đang rải rác về đúng 1 file khai báo
- [ ] đăng ký exception/recover middleware toàn cục, trả 500 với `message` cố định
- [ ] xử lý bẫy framework: `.NET` tắt auto `ProblemDetails` + set `SnakeCaseLower`; `FastAPI` override `HTTPException` và `RequestValidationError`; `Express` error middleware đăng ký cuối cùng + catch-all 404 trả JSON
- [ ] thêm `X-Request-Id` trên mọi response và nối vào log
- [ ] viết `docs/api-contract.md`: contract version, bảng domain error code, danh sách endpoint ngoại lệ

- [ ] khai `code` thành một kiểu riêng và ép writer chỉ nhận kiểu đó (`type Code string` ở Go, `StrEnum` ở Python, union `as const` ở TS, `const string` trong static class ở C#) — compiler chặn code lạ ngay lúc build, rẻ hơn mọi cơ chế khác

Nếu dự án muốn tự kiểm chặt hơn, section 13 của chuẩn gợi ý 3 test và nói rõ khi nào đáng làm. **Phạm vi nếu dùng:** nhánh A là toàn bộ API; nhánh B **chỉ `/api/v2`** — nhánh B cấm đụng `/api/v1` nên smoke test duyệt route sẽ đỏ trên v1 theo đúng thiết kế, giới hạn test theo prefix chứ không nới luật.

### Nhánh A — mọi client nằm trong repo

Đổi tại chỗ được vì client deploy cùng lúc với backend. Giữ đúng thứ tự:

- [ ] đổi từng nhóm endpoint sang shape mới, mỗi nhóm 1 commit — **không** đổi cả API trong 1 PR
- [ ] endpoint đang trả `200` kèm `success: false` phải đổi status trước, đổi body sau
- [ ] FE cập nhật api client theo đúng thứ tự: interceptor mới chịu được cả 2 shape → backend đổi → gỡ nhánh cũ ở FE

### Nhánh B — có client ngoài repo (mobile đã phát hành, đối tác)

**CẤM đổi status tại chỗ.** Dòng "đổi status trước" của nhánh A không áp ở đây: bản vá không tới được máy người dùng, đổi status là app ngoài thị trường gãy ngay. `/api/v1` giữ nguyên trạng, kể cả endpoint đang trả `200` kèm `success: false`.

- [ ] xác nhận **có kênh ép nâng cấp client cũ**: force-update trên store, chặn version cũ ngay trong app, hoặc điều khoản hợp đồng với đối tác
- [ ] xác nhận **server log được version client** mỗi request (user-agent, header version, api key theo đối tác) — chưa có thì thêm log này trước
- [ ] thiếu một trong hai → `/api/v1` sẽ sống vĩnh viễn và bạn gánh 2 API mãi mãi. Chốt chấp nhận hay không **ở đây**, trước khi dựng surface mới
- [ ] dựng `/api/v2` song song, dùng chung helper + file mã lỗi ở khối chung; `/api/v1` không đụng vào
- [ ] endpoint mới chỉ mọc ở v2, không thêm gì vào v1
- [ ] v2 áp đủ nghĩa vụ của API có consumer ngoài repo: `user_message` + OpenAPI (Tier 1), `Idempotency-Key` cho POST đụng tiền/kho (Tier 2, appendix section 1.13)
- [ ] đo traffic theo version client, có số liệu theo ngày — đây là căn cứ duy nhất để retire
- [ ] đặt mốc sunset cho `/api/v1`, ghi vào `docs/api-contract.md` và CHANGELOG
- [ ] bật header `Deprecation: true` + `Sunset: <http-date>` trên mọi response của v1, nhớ `Access-Control-Expose-Headers`
- [ ] thông báo đối tác bằng văn bản: mốc sunset, diff v1 → v2, thời hạn phải chuyển xong
- [ ] chỉ retire `/api/v1` khi **đo được** không còn traffic, không retire theo lịch suông

## 6. Chiến lược commit

- [ ] 1 commit cho rename
- [ ] 1 commit cho move folder
- [ ] 1 commit cho update imports / namespaces
- [ ] 1 commit cho cleanup
- [ ] không trộn refactor logic lớn vào cùng commit đổi cấu trúc

## 7. Validate sau migrate

- [ ] app/service build được sau khi move folder
- [ ] test hiện có chạy được hoặc có ghi chú rõ test nào chưa chạy
- [ ] README đã cập nhật entrypoint, lệnh run/build/test, cấu trúc mới
- [ ] config mẫu còn đủ nhưng không chứa secret thật
- [ ] deploy files trong `infra/` vẫn trỏ đúng path mới
- [ ] script trong `scripts/` không còn hardcode path cũ
- [ ] CI/CD hoặc hướng dẫn release đã cập nhật nếu repo có pipeline

## 8. Dấu hiệu dừng lại

Dừng migrate nếu:
- [ ] team không hiểu vì sao đang đổi
- [ ] cấu trúc mới không làm code dễ tìm hơn
- [ ] đang biến repo nhỏ thành repo enterprise
- [ ] đổi cấu trúc nhiều hơn nhu cầu thật

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

## 6. Chiến lược commit

- [ ] 1 commit cho rename
- [ ] 1 commit cho move folder
- [ ] 1 commit cho update imports / namespaces
- [ ] 1 commit cho cleanup
- [ ] không trộn refactor logic lớn vào cùng commit đổi cấu trúc

## 7. Dấu hiệu dừng lại

Dừng migrate nếu:
- [ ] team không hiểu vì sao đang đổi
- [ ] cấu trúc mới không làm code dễ tìm hơn
- [ ] đang biến repo nhỏ thành repo enterprise
- [ ] đổi cấu trúc nhiều hơn nhu cầu thật

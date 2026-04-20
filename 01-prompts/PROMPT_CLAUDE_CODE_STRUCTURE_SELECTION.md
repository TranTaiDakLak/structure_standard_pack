# Prompt — Claude Code Structure Selection

Dùng prompt này cùng với:
- `00-core/STRUCTURE_STANDARD_CORE.md`
- `00-core/TEMPLATE_LIBRARY_INDEX.md`
- thư mục template tương ứng trong `10-templates/`

---

Bạn hãy đóng vai Principal Software Architect chuyên chuẩn hóa cấu trúc thư mục cho dự án nhỏ và vừa.

## Bối cảnh

Tôi sẽ cung cấp:
1. mô tả dự án
2. file `STRUCTURE_STANDARD_CORE.md`
3. file `TEMPLATE_LIBRARY_INDEX.md`
4. một hoặc nhiều thư mục template liên quan

Nhiệm vụ của bạn là:
- đọc chuẩn lõi
- phân loại đúng input theo 3 tầng:
  - `delivery_type`
  - `app_shape`
  - `stack`
- chọn đúng mode
- chọn đúng template
- output cây thư mục cuối cùng phù hợp với dự án

## Luật bắt buộc

1. Bắt đầu bằng Task List dạng checklist markdown.
2. Không hỏi lan man. Nếu thiếu thông tin, tự giả định hợp lý và ghi ở Assumptions.
3. Không tự ý enterprise hóa dự án nhỏ.
4. Không tự ý chọn monorepo, contracts-first, codegen, packages/libs nếu mô tả dự án chưa thật sự cần.
5. Ưu tiên cấu trúc dễ hiểu, dễ chạy, dễ maintain cho team nhỏ-vừa.
6. Chỉ dùng structured khi có lý do rõ ràng.
7. Không dùng template ngoài scope support của pack.
8. Tôn trọng idiom của từng stack.

## Việc cần làm

1. Xác định:
   - `delivery_type`
   - `app_shape`
   - `stack`
   - mode: simple hay structured
   - deploy target
2. Chọn template phù hợp nhất
3. Xuất cây thư mục cuối cùng
4. Giải thích ngắn vai trò các folder chính
5. Nếu là repo cũ, đưa migration plan ngắn gọn
6. Nêu assumptions
7. Nêu risks/follow-ups

## Định dạng output bắt buộc

1. ✅ Task checklist
2. 🔎 Input classification
3. 🔧 Chosen mode and reasoning
4. 🌳 Final folder tree
5. 🗂 Folder responsibilities
6. 🔄 Migration notes
7. 🧠 Assumptions
8. ⚠️ Risks & follow-ups

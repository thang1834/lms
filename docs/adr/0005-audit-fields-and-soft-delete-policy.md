# ADR-0005: Quy Chuẩn Bắt Buộc 6 Audit Fields và Chính Sách Soft Delete Cho 100% Bảng CSDL

## Status
**Accepted** (2026-09-23)

## Context
Trong hệ thống giáo dục đào tạo và tài chính học phí, dữ liệu về lịch sử ghi danh, điểm danh, kết quả bài thi, lịch sử đóng tiền và biên bản xử lý vi phạm/sự cố có giá trị pháp lý và tính lịch sử rất cao. Nếu thực hiện xóa cứng (Hard Delete: `DELETE FROM table WHERE id = ...`), toàn bộ dấu vết trách nhiệm sẽ bị xóa sổ, không thể phục hồi dữ liệu khi nhân viên thao tác nhầm, và phá vỡ tính toàn vẹn quan hệ của cơ sở dữ liệu.

## Decision
Chúng tôi quyết định thiết lập quy chuẩn thiết kế bắt buộc cho **100% các bảng CSDL (59/59 bảng hiện tại và mọi bảng mở rộng sau này)**:
1. **6 Trường Kiểm Toán (Audit Fields):**
   ```sql
   created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
   created_by UUID,
   updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
   updated_by UUID,
   deleted_at TIMESTAMPTZ,
   deleted_by UUID
   ```
2. **Go Struct Embeded DTO:**
   Mọi struct Domain Model trong Go đều bắt buộc nhúng struct `AuditFields`:
   ```go
   type AuditFields struct {
       CreatedAt time.Time  `json:"createdAt" db:"created_at"`
       CreatedBy *uuid.UUID `json:"createdBy,omitempty" db:"created_by"`
       UpdatedAt time.Time  `json:"updatedAt" db:"updated_at"`
       UpdatedBy *uuid.UUID `json:"updatedBy,omitempty" db:"updated_by"`
       DeletedAt *time.Time `json:"deletedAt,omitempty" db:"deleted_at"`
       DeletedBy *uuid.UUID `json:"deletedBy,omitempty" db:"deleted_by"`
   }
   ```
3. **Chính Sách Soft Delete:**
   - Mọi thao tác xóa dữ liệu trên hệ thống thực chất là cập nhật `deleted_at = NOW()` và `deleted_by = current_user_id`.
   - 100% các câu truy vấn `SELECT` phải có điều kiện `WHERE deleted_at IS NULL` (hoặc thông qua Database Views / Query Scopes).
   - Chỉ Super Admin mới có quyền truy cập màn hình Thùng rác để xem lại và Khôi phục (Restore) bản ghi khi cần thiết.

## Consequences

### Thuận lợi (Pros):
- Đảm bảo tính minh bạch, lưu vết trách nhiệm pháp lý 100% các thao tác thêm, sửa, xóa trong hệ thống.
- Khôi phục dữ liệu tức thì khi người dùng vô tình bấm xóa nhầm lớp học hoặc điểm danh.
- Dữ liệu lịch sử phục vụ đắc lực cho phân tích thống kê Churn Radar và báo cáo tài chính kiểm toán.

### Hạn chế (Cons) & Biện pháp khắc phục:
- Kích thước lưu trữ CSDL tăng dần theo thời gian do không xóa bản ghi vật lý.
- Cần đánh index một phần (`Partial Index`) với điều kiện `WHERE deleted_at IS NULL` để đảm bảo tốc độ truy vấn luôn đạt tối đa.

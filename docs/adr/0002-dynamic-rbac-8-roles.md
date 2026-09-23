# ADR-0002: Phân Quyền Hạt Nhân Động Dynamic RBAC với 8 Vai Trò Chuyên Biệt

## Status
**Accepted** (2026-09-21)

## Context
Một trung tâm đào tạo đa ngành vận hành với nhiều đối tượng có phạm vi quyền hạn và trách nhiệm khác nhau: Ban giám đốc, Giáo vụ, Quản nhiệm ca học, Giảng viên, Trợ giảng, Hội đồng chấm tốt nghiệp, Học viên và Phụ huynh. Nếu hardcode quyền hạn theo chuỗi String tĩnh hoặc phân quyền thô sơ (Role-based cứng) trong mã nguồn, hệ thống sẽ không thể tùy biến quyền khi mở rộng cơ sở hoặc tạo vai trò mới mà không cần sửa code và deploy lại.

## Decision
Chúng tôi quyết định thiết kế mô hình **Dynamic RBAC (Role-Based Access Control hạt nhân động)** dựa trên 4 bảng CSDL:
1. `roles`: Lưu danh mục vai trò (`SUPER_ADMIN`, `ACADEMIC_MANAGER`, `CLASS_COORDINATOR`, `TEACHER`, `TA`, `EXAMINER`, `STUDENT`, `PARENT`).
2. `permissions`: Lưu danh mục quyền hạn nguyên tử theo format `resource:action` (ví dụ: `attendance:create`, `grades:publish`, `classes:transfer`, `system:customize`).
3. `role_permissions`: Bảng liên kết trung gian cấu hình quyền hạn cho từng vai trò mà không cần deploy lại.
4. `user_roles`: Bảng gán một hoặc nhiều vai trò cho người dùng.

Middleware phân quyền trong Backend Go-chi sẽ:
- Giải mã JWT Token để lấy `user_id`.
- Tra cứu danh sách quyền hạn đã cache tại Redis (TTL 15 phút, tự động invalidate khi admin cập nhật quyền).
- Chặn lập tức các request không có quyền hạn tương ứng với mã lỗi `403 Forbidden`.

## Consequences

### Thuận lợi (Pros):
- Cực kỳ linh hoạt: Super Admin có thể bật/tắt quyền cho bất kỳ vai trò nào trực tiếp từ màn hình `/admin/roles`.
- An toàn tuyệt đối: Ngăn chặn triệt để lỗ hổng Broken Access Control (OWASP Top 10) và IDOR.
- Phù hợp với kiến trúc nhiều chi nhánh và phân cấp quản lý đào tạo.

### Hạn chế (Cons) & Biện pháp khắc phục:
- Cần truy vấn quyền hạn nhiều lần trong mỗi request.
- Khắc phục bằng Redis Caching tập trung theo khóa `rbac:user:{user_id}:permissions`.

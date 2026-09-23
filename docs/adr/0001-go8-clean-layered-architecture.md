# ADR-0001: Áp dụng Blueprint gmhafiz/go8 và Clean Layered Architecture cho Backend

## Status
**Accepted** (2026-09-20)

## Context
Hệ thống LMS trung tâm đào tạo đòi hỏi tính ổn định cao, khả năng chịu tải tốt khi hàng nghìn học viên làm bài và nộp code cùng lúc, đồng thời phải dễ bảo trì, kiểm thử và mở rộng nghiệp vụ trong tương lai (với hơn 100+ endpoints và 24 domains). Nếu sử dụng kiến trúc nguyên khối thiếu chuẩn mực hoặc cấu trúc lộn xộn, mã nguồn sẽ nhanh chóng trở nên khó kiểm soát, khó viết unit/integration tests và dễ sinh lỗi rò rỉ dữ liệu hoặc IDOR.

## Decision
Chúng tôi quyết định chọn blueprint **`gmhafiz/go8`** làm nền tảng phát triển Backend:
1. **Framework:** Sử dụng **Go-chi** (`github.com/go-chi/chi/v5`) nhẹ nhàng, tốc độ cao, hoàn toàn tương thích chuẩn `net/http` tiêu chuẩn của Go.
2. **Kiến trúc phân tầng nghiêm ngặt (Clean Layered Architecture):**
   - **`Handler` (Tầng Giao tiếp HTTP):** Tiếp nhận request, bind và validate dữ liệu DTO, gọi UseCase và đóng gói phản hồi theo chuẩn `Envelope`.
   - **`UseCase` (Tầng Nghiệp vụ cốt lõi):** Chứa 100% logic nghiệp vụ, quy tắc kiểm tra điều kiện, giao dịch (transactions), độc lập hoàn toàn với framework web.
   - **`Repository` (Tầng Truy cập dữ liệu):** Thao tác truy vấn PostgreSQL qua `database/sql` và `sqlx`, chịu trách nhiệm ánh xạ quan hệ dữ liệu.
3. **Database Migration:** Sử dụng **Goose SQL Migrations** với các tập tin `.sql` thuần túy (`database/migrations/*.sql`), có khả năng rollback (`-- +goose Down`).
4. **Task Runner:** Sử dụng **Taskfile** (`Taskfile.yml`) chuẩn hóa mọi lệnh khởi động, migrate, sinh swagger, linting và chạy tests.

## Consequences

### Thuận lợi (Pros):
- Phân tách rõ ràng ranh giới giữa HTTP, Logic và Database, giúp việc viết Unit Tests mock Repository vô cùng đơn giản.
- Tốc độ xử lý cực nhanh nhờ tối ưu hóa của Go-chi và goroutines.
- Toàn bộ nhóm phát triển tuân thủ chung một cấu trúc thư mục đồng nhất (`internal/domain/{entity}`).

### Hạn chế (Cons) & Biện pháp khắc phục:
- Số lượng file và boilerplate code nhiều hơn so với MVC truyền thống.
- Khắc phục bằng cách chuẩn hóa các template code và DTO envelopes dùng chung trong `internal/utility/respond`.

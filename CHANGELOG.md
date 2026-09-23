# Changelog

Tất cả những thay đổi đáng chú ý của dự án **LMS Center Platform** sẽ được ghi chép trong tập tin này.  
Định dạng tuân thủ theo tiêu chuẩn [Keep a Changelog](https://keepachangelog.com/en/1.0.0/) và quy ước phiên bản [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [Unreleased]

### Planned
- Tích hợp cổng thanh toán trực tiếp VNPay & Momo song song với VietQR.
- Cổng thi trắc nghiệm trực tuyến chống gian lận qua WebRTC Proctored Camera.
- Nghiên cứu và triển khai giải pháp Bảng vẽ kỹ thuật số thuần Vue 3 (Vue-native Whiteboard) thay thế cho Excalidraw/React IFrame.

### Changed
- **Digital Whiteboard:** Tạm hoãn giải pháp Excalidraw qua IFrame (chuyển trạng thái ADR-0003 sang `Deferred`) để ưu tiên tìm kiếm giải pháp tương thích sâu với Nuxt UI / Vue 3.

---

## [2.8.0] - 2026-09-23

### Added
- **Class Coordinator Operations & Student Care CRM:**
  - Bảng điều hành ca học hôm nay `/coordinator/today` kèm bộ lọc theo cơ sở chi nhánh và trạng thái ca học.
  - Check-in Watchdog cảnh báo cờ đỏ nhấp nháy khi GV/TA chưa check-in sau 10 phút vào lớp.
  - 1-Click Class Broadcast phát thông báo khẩn qua Zalo OA, Mobile Push và SMS khi có sự cố.
  - Hồ sơ chăm sóc học viên và can thiệp nguy cơ bỏ học (`student_care_logs`) tích hợp Churn Radar.
  - Quy trình chuyển lớp và bảo lưu khóa học 2 cấp (`class_transfers`), tự động tính học phí bảo lưu (`reservedCredit`) và giải phóng ghế ngồi.
  - Quản lý biên bản sự cố phòng học & thiết bị (`session_incidents`), tự động cấp link Meet chuyển sang học Online trong 60s.
  - Sổ liên lạc điện tử (`coordinatorNote`) và quản lý bàn giao học liệu đầu khóa (`student_materials`).
- **CSDL & Migrations:**
  - `database/migrations/20260923000018_create_coordinator_care_transfers.sql`
  - `database/migrations/20260923000019_create_coordinator_incidents_materials.sql`
  - Nâng tổng số bảng lên **59 bảng** (100% 6 audit fields & soft delete policy).
- **APIs:**
  - Bổ sung 10 Go REST Handlers mới thuộc domain `internal/domain/coordinator` (Nâng tổng số lên **105+ endpoints**).
- **QA & Test Cases:**
  - Bổ sung Nhóm 11 Test Cases (`TC-COORD-SHIFT-01` đến `TC-COORD-MATERIAL-01`).

---

## [2.7.0] - 2026-09-23

### Added
- **Teacher Portal Advanced & Live Class Cockpit:**
  - Khoang lái lớp học trực tuyến 1-Click khởi động lớp, tự động check-in giảng viên và mở Meet/Zoom.
  - Tích hợp bảng vẽ kỹ thuật số Excalidraw thời gian thực qua IFrame container độc lập, autosave 3s, multiplayer Go WebSocket Hub và xuất bản PDF tự động.
  - Quick Mini Poll 2 phút đo lường mức độ tiếp thu bài giảng kèm biểu đồ kết quả thời gian thực.
  - Trợ lý AI Code Review phân tích Clean Code, phát hiện data race / memory leak và gợi ý bản nháp nhận xét sư phạm.
  - Trình ghi âm nhận xét giọng nói (Voice Note 30s-2m) qua Web MediaRecorder API lưu trữ MinIO/S3.
  - Hộp thư Q&A tập trung thu gom thắc mắc từ video và bài tập, kho câu trả lời mẫu (Snippets) và phân công Trợ giảng kèm SLA 2 giờ.
  - Chợ dạy thay ngang hàng (`substitute_requests`) và quản lý khung giờ rảnh hàng tuần (`teacher_availabilities`).
  - Sổ tay sư phạm bảo mật (`student_pedagogical_notes`) và ngân hàng đề bài mẫu cá nhân (`assignment_banks`).
- **CSDL & Migrations:**
  - Bổ sung 8 bảng CSDL mới (Bảng 48 - 55) qua Migrations 15, 16, 17.

---

## [2.6.0] - 2026-09-23

### Added
- **Super Admin System Management & Website Customization:**
  - Quản trị cấu hình nhận diện website: thay đổi Favicon, Logo, Màu sắc Header/Footer, SEO Meta tags, Hotline trung tâm.
  - Quản lý người dùng tập trung (User Management): phân quyền RBAC động, khóa/mở tài khoản, reset mật khẩu, lịch sử đăng nhập.
  - Phân hệ quản lý đa cơ sở chi nhánh (`campuses`) và phòng học (`rooms`), tích hợp thuật toán chống trùng lịch phòng học (`OVERLAPS`).
  - Lên lịch học bù & dạy bù ghép lớp song song hoặc kèm 1-1 với trợ giảng (`makeup_sessions`).
  - Khảo thí & Ngân hàng câu hỏi trắc nghiệm tự động (`question_banks`, `quizzes`, `quiz_attempts`).
  - Radar phát hiện học viên có nguy cơ bỏ học (Churn Radar) và Hợp đồng đào tạo điện tử xuất PDF (`contracts`).

---

## [2.5.0] - 2026-09-23

### Added
- **Hạ Tầng OpenTelemetry Observability (Theo chuẩn repo go8):**
  - Cấu hình OTel Collector, Jaeger (Distributed Tracing), Prometheus (Application Metrics) và Grafana Dashboard.
  - Middleware Go-chi tracing bọc 100% incoming HTTP requests kèm W3C Trace Context propagation.
  - Log correlations tự động tiêm `trace_id` và `span_id` vào Zap structured logger gửi Loki.

---

## [2.0.0] - 2026-09-22

### Added
- Khởi tạo kiến trúc Clean Layered Architecture (`gmhafiz/go8`) cho Backend Go-chi.
- Thiết kế Dynamic RBAC 8 roles với các bảng `roles`, `permissions`, `role_permissions`, `user_roles`.
- Module Trình phát video chống tua (Anti-Seeking Video Player) kèm heartbeat 5s và xác thực tiến độ.
- Module Cổng Mua Khóa Học Trực Tuyến & Cổng Thanh Toán VietQR động (Napas247) tự động kích hoạt qua Webhook.
- Module Hội đồng chấm Đồ án tốt nghiệp (Capstone Project Defense) chấm điểm theo Rubric 4 tiêu chí.
- Thiết kế hệ thống thông báo tự động (Notification Engine) qua SMS, Push và Zalo OA.

---

## [1.0.0] - 2026-09-20

### Added
- Khởi tạo dự án Monorepo LMS Center Platform.
- Tài liệu đặc tả yêu cầu người dùng (SRS) phiên bản ban đầu.
- Bản đặc tả thiết kế giao diện `DESIGN.md` và mã màu Design Tokens.

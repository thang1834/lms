# Kế Hoạch Soạn Thảo Tài Liệu Kỹ Thuật & Thiết Kế (Documentation Suite Plan)

> **File:** `lms-documentation-plan.md`  
> **Dự án:** LMS Center Platform (Hệ thống Quản lý Học tập & Vận hành Đào tạo Đa hình thức)  
> **Chế độ:** PLANNING ONLY (No code writing in this phase)  
> **Phiên bản:** 1.0.0

---

## 1. Tổng Quan & Mục Tiêu (Overview)

Nhằm đảm bảo quá trình lập trình diễn ra chuẩn xác, chặt chẽ, không bị sai lệch nghiệp vụ và giúp đội ngũ phát triển (hoặc các AI Agents) có một **"Nguồn Chân Lý Duy Nhất" (Single Source of Truth)** trước khi viết bất kỳ dòng mã nào, hệ thống cần được đặc tả chi tiết thông qua bộ 6 tài liệu kỹ thuật & thiết kế chuyên nghiệp.

Bộ tài liệu này bao quát toàn diện các tính năng cốt lõi đã thống nhất:
- **Quản lý Giáo viên & Lịch dạy** (Hồ sơ, phân công, thời khóa biểu).
- **Điểm danh hai chiều** (Giáo viên check-in/out tính công & Học sinh Offline/Online).
- **Quản lý tài liệu** (Tài liệu bài học, slide, code mẫu, tài liệu BTVN).
- **Bài tập CNTT/Đa ngành & Chấm điểm** (Monaco Code Editor trên web, nộp link GitHub, rubric chấm điểm, Gradebook).
- **Cổng Học viên & Phụ huynh** (Theo dõi tiến độ, chuyên cần, điểm số, nhận xét).
- **Thanh toán học phí đa kênh** (VietQR Napas, Thẻ Visa/Mastercard, Tiền mặt).

---

## 2. Danh Mục Các Tài Liệu Cần Soạn Thảo (Documentation Suite)

```plaintext
LMS/
├── DESIGN.md                     # [1] Bộ quy chuẩn Design Tokens & UI/UX (Bắt buộc trước khi code UI)
├── docs/
│   ├── requirements.md           # [2] Đặc tả Yêu cầu Phần mềm (SRS / User Stories / Acceptance Criteria)
│   ├── architecture.md           # [3] Thiết kế Kiến trúc Hệ thống & CSDL (ERD, Data Dictionary, RBAC)
│   ├── api-specification.md      # [4] Đặc tả API Contracts & Server Actions (Payloads, DTOs, Zod schemas)
│   ├── user-flows.md             # [5] Sơ đồ Luồng Người dùng & Bố cục Giao diện (User Journeys & Wireframes)
│   └── test-plan.md              # [6] Kế hoạch Kiểm thử & Bộ Test Cases Nghiệp vụ (QA Strategy)
```

---

## 3. Nội Dung Chi Tiết Của Từng Tài Liệu

### 📄 Tài Liệu 1: `DESIGN.md` (Design System & Tokens)
- **Vị trí:** Ngay tại thư mục gốc dự án (`./DESIGN.md`) theo chuẩn `design-spec`.
- **Mục đích:** Là khung quy chuẩn giao diện trước khi xây dựng bất kỳ trang web hay component nào.
- **Nội dung:**
  - **YAML Design Tokens (Frontmatter):** Bảng màu (Primary Blue, Slate, Emerald, Amber, Rose - tuân thủ nghiêm ngặt quy tắc Không dùng màu tím tùy tiện), Typography (Inter / JetBrains Mono cho code), Spacing, Border Radius, Elevation (Shadows), Component tokens (Buttons, Cards, Inputs, Tables).
  - **Bố cục Giao diện 4 Portals:** Sidebar cố định, Topbar trạng thái, Content area cuộn mượt mà cho Admin, Teacher, Student và Parent.
  - **Code Editor Theme:** Bố cục và tông màu cho Monaco Editor (Dark/Light mode hỗ trợ học viên lập trình).
  - **Do's & Don'ts:** Các nguyên tắc UX/UI giáo dục (khoảng cách chạm trên di động, độ tương phản chuẩn WCAG AA).

---

### 📄 Tài Liệu 2: `docs/requirements.md` (Software Requirements Specification - SRS)
- **Mục đích:** Định nghĩa chính xác tất cả các chức năng, vai trò và tiêu chí chấp nhận (Acceptance Criteria).
- **Nội dung:**
  - **Phân nhóm Đối tượng & Ma trận Trách nhiệm:**
    - `ADMIN`: Quản trị viên trung tâm, giáo vụ, thu ngân.
    - `TEACHER`: Giảng viên chính, trợ giảng (TA).
    - `STUDENT`: Học viên.
    - `PARENT`: Phụ huynh theo dõi con.
  - **User Stories & Acceptance Criteria chi tiết cho 6 phân hệ:**
    1. *Quản lý Giáo viên:* Tạo hồ sơ, gán chuyên môn, cấu hình thù lao giờ dạy, phân công lớp.
    2. *Thời khóa biểu & Lịch dạy:* Hiển thị lịch tuần/tháng theo ca, phòng học hoặc link Google Meet.
    3. *Điểm danh 2 chiều:* Nút Check-in/out của giáo viên (tính giờ dạy thực tế, đối soát timesheet); Bảng điểm danh học sinh lớp Hybrid (phân loại `PRESENT_OFFLINE`, `PRESENT_ONLINE`, `LATE`, `ABSENT`).
    4. *Quản lý Tài liệu:* Tải lên và phân loại slide PDF, starter code mẫu, audio nghe; đính kèm file đề bài & dataset vào BTVN.
    5. *Bài tập & Chấm điểm:* Học viên làm bài trên web bằng Monaco Code Editor hoặc nộp GitHub link; Giáo viên chấm theo thang điểm và nhận xét chi tiết; Sổ điểm lớp học (Gradebook).
    6. *Thanh toán Đa kênh:* Sinh mã VietQR động chuẩn Napas; thanh toán thẻ Visa; ghi nhận thu tiền mặt tại quầy và xuất hóa đơn PDF.
  - **Yêu cầu Phi chức năng (Non-Functional Requirements):** Bảo mật JWT, tốc độ phản hồi < 200ms, mã hóa dữ liệu nhạy cảm.

---

### 📄 Tài Liệu 3: `docs/architecture.md` (System Architecture & Database Design)
- **Mục đích:** Kiến trúc kỹ thuật và thiết kế dữ liệu sâu rộng.
- **Nội dung:**
  - **Sơ đồ Kiến trúc Tổng thể:** Client (Next.js 16 React 19) ↔ Server Actions / API Routes ↔ Prisma ORM ↔ PostgreSQL Database.
  - **Thiết kế CSDL Toàn diện (ERD Mermaid Diagram):**
    - Toàn bộ bảng: `User`, `TeacherProfile`, `StudentProfile`, `ParentProfile`, `Course`, `Module`, `Lesson`, `Class`, `ClassSession`, `TeacherAttendance`, `StudentAttendance`, `Material`, `Assignment`, `Submission`, `GradeFeedback`, `TuitionInvoice`, `PaymentTransaction`.
  - **Từ điển Dữ liệu (Data Dictionary):** Tên trường, kiểu dữ liệu, ràng buộc (Primary Key, Foreign Key, Unique, Enum, Index).
  - **Mô hình Phân quyền RBAC & Bảo mật:**
    - Cấu trúc Session Token (chứa `userId`, `role`, `profileId`).
    - Quy tắc bảo vệ routes tại `src/middleware.ts`.
  - **Chiến lược Tích hợp Dịch vụ Thứ ba:**
    - Thuật toán sinh mã VietQR Napas (ngân hàng, STK, số tiền, nội dung mã hóa).
    - Tích hợp YouTube Player IFrame API (xử lý state tracking tiến độ xem).
    - Lưu trữ file Cloud Storage & xuất PDF bằng `@react-pdf/renderer`.

---

### 📄 Tài Liệu 4: `docs/api-specification.md` (API & Interface Specification)
- **Mục đích:** Chuẩn hóa giao tiếp giữa Client và Server (API contracts).
- **Nội dung:**
  - **Quy ước chung:** Định dạng Request/Response JSON, mã lỗi HTTP (200, 201, 400, 401, 403, 404, 500).
  - **Chi tiết từng Endpoint / Server Action:**
    - `POST /api/auth/...`: Đăng nhập, đăng xuất, lấy session.
    - `GET/POST /api/teachers`: CRUD thông tin giảng viên, gán lớp.
    - `GET /api/schedule`: Lấy lịch dạy của giáo viên hoặc lịch học của học viên theo tuần/tháng.
    - `POST /api/attendance/teacher-checkin`: Giáo viên check-in/out ca dạy.
    - `POST /api/attendance/students`: Lưu điểm danh học sinh buổi học.
    - `GET/POST /api/materials`: Upload và tải tài liệu học tập/BTVN.
    - `POST /api/assignments`: Tạo bài tập, giao cho lớp.
    - `POST /api/submissions`: Học viên nộp bài làm (code/link/file).
    - `POST /api/grading`: Giáo viên chấm điểm và trả nhận xét.
    - `GET/POST /api/payments/vietqr`: Sinh mã VietQR cho hóa đơn học phí; xác nhận tiền mặt.
  - **Zod Validation Schemas:** Các schema kiểm tra tính hợp lệ dữ liệu đầu vào.

---

### 📄 Tài Liệu 5: `docs/user-flows.md` (User Journey & Flow Diagrams)
- **Mục đích:** Mô hình hóa trải nghiệm và luồng thao tác của người dùng qua sơ đồ trực quan.
- **Nội dung:**
  - **Flow 1: Giáo viên Bắt đầu Ca dạy & Điểm danh:**
    - Xem lịch dạy → Bấm Check-in ca dạy → Mở lớp học → Điểm danh học viên (chọn Offline hoặc Online cho lớp Hybrid) → Ghi chú buổi dạy → Kết thúc ca (Check-out).
  - **Flow 2: Học viên Học Bài & Nộp BTVN CNTT:**
    - Xem lịch học → Xem video bài giảng YouTube & tải tài liệu slide/code mẫu → Mở bài tập về nhà → Soạn code trên Monaco Editor trực tiếp → Kiểm tra và bấm Nộp bài.
  - **Flow 3: Giáo viên Chấm bài & Phụ huynh Nhận kết quả:**
    - Giáo viên vào danh sách bài nộp → Xem code học viên đã viết → Nhập điểm và nhận xét → Học viên và Phụ huynh nhận thông báo kết quả.
  - **Flow 4: Phụ huynh / Học viên Đóng học phí:**
    - Xem hóa đơn học phí → Bấm "Thanh toán VietQR" → App hiển thị mã QR động chuẩn Napas → Mở app ngân hàng quét mã → Giao dịch hoàn tất.

---

### 📄 Tài Liệu 6: `docs/test-plan.md` (QA Strategy & Test Matrix)
- **Mục đích:** Kế hoạch kiểm thử chất lượng toàn diện trước khi bàn giao hệ thống.
- **Nội dung:**
  - **Phạm vi kiểm thử:** Unit tests, Integration tests, Security tests, E2E tests.
  - **Danh mục Test Cases quan trọng:**
    - *TC-AUTH-01:* Chặn truy cập chéo vai trò (Học sinh cố tình vào `/admin` hoặc `/teacher`).
    - *TC-ATT-01:* Giáo viên check-in và check-out ca dạy, kiểm tra tính toán tổng số phút dạy thực tế.
    - *TC-ATT-02:* Điểm danh lớp Hybrid: học viên A học tại lớp (`PRESENT_OFFLINE`), học viên B học online (`PRESENT_ONLINE`), kiểm tra dữ liệu hiển thị đúng trên portal phụ huynh.
    - *TC-SUB-01:* Học viên nộp code trên Monaco Editor, kiểm tra mã nguồn lưu trữ nguyên vẹn, không bị mất định dạng.
    - *TC-PAY-01:* Sinh mã VietQR và kiểm tra độ chính xác của payload Napas (đúng STK, số tiền, cú pháp chuyển khoản).
    - *TC-MAT-01:* Kiểm tra tải lên và tải về tài liệu bài học (PDF, zip code).

---

## 4. Kế Hoạch Thực Hiện Soạn Thảo (Task Breakdown)

- [ ] **Task DOC-1: Soạn thảo `DESIGN.md` (Design Tokens & UI Guidelines)**
  - **Agent:** `frontend-specialist`
  - **Skill:** `design-spec`, `frontend-design`
  - **Output:** File `DESIGN.md` tại thư mục gốc với đầy đủ YAML tokens và mô tả giao diện 4 portals.

- [ ] **Task DOC-2: Soạn thảo `docs/requirements.md` (SRS & Business Requirements)**
  - **Agent:** `product-manager` / `project-planner`
  - **Skill:** `plan-writing`, `documentation-templates`
  - **Output:** File `docs/requirements.md` đặc tả chi tiết 6 phân hệ nghiệp vụ.

- [ ] **Task DOC-3: Soạn thảo `docs/architecture.md` (Architecture & Database ERD)**
  - **Agent:** `database-architect` / `backend-specialist`
  - **Skill:** `database-design`, `architecture`
  - **Output:** File `docs/architecture.md` chứa sơ đồ ERD, từ điển dữ liệu và cơ chế RBAC.

- [ ] **Task DOC-4: Soạn thảo `docs/api-specification.md` (API Contracts & Payloads)**
  - **Agent:** `backend-specialist`
  - **Skill:** `api-patterns`, `documentation-templates`
  - **Output:** File `docs/api-specification.md` liệt kê toàn bộ Route Handlers và Server Actions kèm Zod schemas.

- [ ] **Task DOC-5: Soạn thảo `docs/user-flows.md` (User Journeys & UI Wireframes)**
  - **Agent:** `frontend-specialist`
  - **Skill:** `frontend-design`, `documentation-templates`
  - **Output:** File `docs/user-flows.md` với các sơ đồ Mermaid trực quan hóa luồng thao tác.

- [ ] **Task DOC-6: Soạn thảo `docs/test-plan.md` (QA Strategy & Test Cases)**
  - **Agent:** `test-engineer`
  - **Skill:** `testing-patterns`, `webapp-testing`
  - **Output:** File `docs/test-plan.md` chứa ma trận kiểm thử và danh sách test cases chi tiết.

---

## 5. Tiêu Chuẩn Hoàn Thành (Definition of Done)

Trước khi chuyển sang giai đoạn viết mã (Implementation):
- [ ] Tất cả 6 tài liệu trên phải được tạo đầy đủ, mạch lạc, thống nhất thuật ngữ và cấu trúc dữ liệu với nhau.
- [ ] Đảm bảo không có xung đột giữa thiết kế CSDL trong `architecture.md` và các API trong `api-specification.md`.
- [ ] `DESIGN.md` tuân thủ nghiêm ngặt quy tắc Design Tokens và quy chuẩn màu sắc (No purple ban).
- [ ] Người dùng review và phê duyệt bộ tài liệu.

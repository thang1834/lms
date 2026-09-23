# Kế Hoạch Chi Tiết: Hệ Thống LMS Cho Trung Tâm Đào Tạo (IT & Đa Ngành)

> **File:** `lms-center-platform.md`  
> **Dự án:** LMS Center Platform (Hệ thống Quản lý Học tập, Giảng viên & Vận hành Đào tạo Đa hình thức)  
> **Kiến trúc:** Monorepo (Frontend: Nuxt UI + Vue 3 | Backend: `gmhafiz/go8` + Go-chi + PostgreSQL)  
> **Chế độ:** PLANNING ONLY (No code writing in this phase)  
> **Phiên bản:** 2.1.0 (Chuẩn hóa Backend theo blueprint `gmhafiz/go8` & Frontend Nuxt UI)  
> **Ngày cập nhật:** 23/09/2026  

---

## 1. Tổng Quan & Mục Tiêu Dự Án (Overview)

Hệ thống LMS được thiết kế riêng cho trung tâm đào tạo hiện đại với trọng tâm là **Công nghệ thông tin (CNTT)**, đồng thời linh hoạt mở rộng hoàn hảo cho **Ngoại ngữ** và các bộ môn khác. Backend được xây dựng theo blueprint **`gmhafiz/go8`** (Go-chi, Layered Architecture, Goose migrations, Taskfile) kết hợp Frontend **Nuxt UI** cho 4 cổng người dùng.


Hệ thống tích hợp toàn diện 10 trụ cột nghiệp vụ:
1. **Chuẩn Hóa Phân Quyền Hạt Nhân RBAC (Dynamic RBAC Architecture)**:
   - Các bảng `roles`, `permissions`, `role_permissions`, `user_roles`.
   - Phân cấp 8 vai trò: Super Admin, Academic Manager (Giáo vụ trưởng), Class Coordinator (Vận hành lớp & Chăm sóc học viên), Giảng viên chính, Trợ giảng (Optional), Giám khảo / Hội đồng phản biện (Examiner), Học viên và Phụ huynh.
2. **Cơ Chế Xếp Lịch Học Linh Hoạt (Flexible Scheduling Engine)**:
   - Thiết lập lịch định kỳ: 1 tuần 1 buổi hoặc 1 tuần nhiều buổi. Trợ giảng là tùy chọn (không bắt buộc).
   - Tùy biến từng buổi: dời ngày học, đổi giờ ca dạy, đổi phòng học hoặc đổi link Meet/Zoom, phân công giáo viên dạy thay.
3. **Mô Hình Vận Hành Lớp & Chăm Sóc Học Viên (Class Operations & Student Care)**:
   - Quản lý lớp đồng hành, Chuyên viên Vận hành nhận cảnh báo GV trễ, theo dõi chuyên cần, nhập `coordinatorNote` gửi phụ huynh.
4. **Điểm Danh Hai Chiều & Đánh Giá Chất Lượng Giáo Viên**:
   - Điểm danh học sinh lớp Hybrid (Offline/Online/Late/Absent).
   - Giáo viên Check-in/out ca dạy chấm công.
   - Học sinh đánh giá nhanh 1-5 sao sau mỗi buổi học (Per-Session Feedback).
5. **Hệ Thống Tin Nhắn & Cảnh Báo Tự Động (Automated Notification Engine)**:
   - *Điểm danh sau 15 phút:* Tự động quét sĩ số vắng/đủ, gửi tin báo vắng cho phụ huynh và báo cáo cho Vận hành & GV.
   - *Cảnh báo giáo viên đi muộn sau 10 phút:* Nếu sau 10p GV chưa check-in $\to$ gửi SMS khẩn cho GV và báo động đỏ cho Quản lý & Vận hành lớp.
   - *Nhận xét sau buổi học:* GV lưu nhận xét $\to$ gửi tin tóm tắt tới Phụ huynh & Học sinh.
   - *Nhắc lịch học trước 24h & 2h:* Gửi tin nhắc lịch, link Meet và dặn dò đồ dùng học tập.
6. **Mua Trực Tiếp Khóa Học Trực Tuyến (Course Store & Direct Enrollment)**:
   - Khóa học Miễn Phí (Free): Đăng ký học ngay 1-Click không cần thanh toán.
   - Khóa học Trả Phí (Paid): Mua trực tiếp, sinh mã VietQR Napas247 động, tự động kích hoạt trong 3s qua Webhook.
7. **Hội Đồng Giám Khảo & Chấm Đồ Án Tốt Nghiệp (Capstone Project Defense)**:
   - Giám khảo (Examiner) đăng nhập cổng chấm thi, xem GitHub repo, live demo, slide thuyết trình.
   - Chấm điểm theo Rubric 4 tiêu chí (Hoàn thiện 30%, Kiến trúc/Clean Code 25%, Thuyết trình 25%, Sáng tạo 20%).
8. **Trình Phát Video Chống Tua Cho Khóa Học Tự Học (Anti-Seeking Video Player)**:
   - Cấm tua nhanh vượt quá mốc đã xem, hỗ trợ tua lùi ôn tập; client heartbeat 5s chống gian lận DevTools; mở khóa tua tự do khi hoàn thành 100%.
9. **Kế Thừa & Tinh Gọn Tính Năng Moodle & Frappe LMS**:
   - Drip content tuần tự, Sổ điểm đa trọng số (Weighted Gradebook), Timestamped Q&A trong video, Chứng chỉ số xác minh công khai `/verify/[code]`.
10. **Học Liệu & Bài Tập CNTT (Monaco Editor)**:
    - Trình soạn thảo Monaco Editor làm bài tập trên web, nộp link GitHub / file ghi âm.

---

## 2. Mô Hình Phân Quyền Hạt Nhân (Roles & Permissions)

| Vai trò (Role) | Mã Role | Quyền hạn chính |
| :--- | :--- | :--- |
| **Super Admin** | `SUPER_ADMIN` | Toàn quyền cấu hình hệ thống, tạo vai trò mới, quản lý tài chính & báo cáo |
| **Quản lý Đào tạo / Giáo vụ** | `ACADEMIC_MANAGER` | Quản lý khóa học, xếp lớp, xếp lịch học linh hoạt, phân công giáo viên & mời hội đồng chấm thi |
| **Chuyên viên Vận hành lớp & CSKH** | `CLASS_COORDINATOR` | Đồng hành cùng lớp, theo dõi sĩ số, nhận cảnh báo GV trễ, liên hệ học sinh vắng/trễ, hỗ trợ phụ huynh |
| **Giảng viên chính** | `TEACHER` | Xem thời khóa biểu, Check-in ca dạy, điểm danh học sinh, upload tài liệu, giao bài & chấm điểm |
| **Trợ giảng (Tùy chọn)** | `TA` | Vai trò tùy chọn: Hỗ trợ lớp học, điểm danh, hỗ trợ giải đáp thắc mắc bài tập code cho học viên |
| **Giám khảo / Phản biện** | `EXAMINER` | Xem sản phẩm đồ án tốt nghiệp, xem GitHub & Live Demo, chấm điểm theo Rubric và nhập nhận xét |
| **Học viên** | `STUDENT` | Mua khóa học trực tiếp (Free/VietQR), xem video chống tua, làm code Monaco Editor, nộp bài |
| **Phụ huynh** | `PARENT` | Theo dõi chuyên cần của con, nhận cảnh báo vắng sau 15p, xem điểm số & nhận xét, thanh toán VietQR |

---

## 3. Kiến Trúc Cơ Sở Dữ Liệu Nâng Cấp (Prisma Schema Overview)

```prisma
// 1. Phân quyền RBAC
model Role {
  id          String           @id @default(uuid())
  code        String           @unique // SUPER_ADMIN, ACADEMIC_MANAGER, CLASS_COORDINATOR, TEACHER, etc.
  name        String
  description String?
  isSystem    Boolean          @default(false)
  permissions RolePermission[]
  users       UserRole[]
}

model Permission {
  id          String           @id @default(uuid())
  code        String           @unique // e.g. classes.schedule, teachers.evaluate
  name        String
  module      String           // CLASSES, TEACHERS, ATTENDANCE, FINANCE
  description String?
  roles       RolePermission[]
}

model RolePermission {
  id           String     @id @default(uuid())
  roleId       String
  permissionId String
  role         Role       @relation(fields: [roleId], references: [id], onDelete: Cascade)
  permission   Permission @relation(fields: [permissionId], references: [id], onDelete: Cascade)
  @@unique([roleId, permissionId])
}

model UserRole {
  id         String   @id @default(uuid())
  userId     String
  roleId     String
  assignedAt DateTime @default(now())
  user       User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  role       Role     @relation(fields: [roleId], references: [id], onDelete: Cascade)
  @@unique([userId, roleId])
}

// 2. Lớp học & Vận hành lớp
model Class {
  id            String         @id @default(uuid())
  courseId      String
  name          String         // Mã lớp: FE-K32
  classType     ClassType      // OFFLINE, ONLINE, HYBRID
  mainTeacherId String
  taTeacherId   String?
  coordinatorId String?        // Chuyên viên Vận hành / CSKH lớp
  roomName      String?
  meetUrl       String?
  scheduleRule  Json           // Quy tắc lặp: [{"dayOfWeek": 2, "time": "19:30-21:30"}]
  sessions      ClassSession[]
  // ... relations
}

// 3. Buổi học linh hoạt (Flexible Sessions)
model ClassSession {
  id                  String             @id @default(uuid())
  classId             String
  sessionNumber       Int
  sessionDate         DateTime           @db.Date
  startTime           String             // 19:30
  endTime             String             // 21:30
  assignedTeacherId   String?            // Hỗ trợ giáo viên dạy thay
  roomName            String?            // Hỗ trợ đổi phòng riêng buổi này
  meetUrl             String?            // Hỗ trợ đổi link meet riêng buổi này
  topic               String?
  status              SessionStatus      // SCHEDULED, RESCHEDULED, IN_PROGRESS, COMPLETED, CANCELLED
  rescheduleReason    String?
  originalDate        DateTime?          @db.Date
  teacherAttendance   TeacherAttendance?
  studentAttendances  StudentAttendance[]
}

// 4. Đánh giá chất lượng giáo viên
model TeacherEvaluation {
  id             String         @id @default(uuid())
  teacherId      String
  classId        String?
  evaluatorId    String
  evaluationType EvaluationType // MANAGER_AUDIT, STUDENT_SURVEY
  criteriaScores Json           // Điểm chi tiết tiêu chí sư phạm, chuyên môn
  overallScore   Decimal        @db.Decimal(3, 2)
  comment        String
  evaluatedAt    DateTime       @default(now())
}
```

---

## 4. Kế Hoạch Triển Khai & Kiểm Thử (Nuxt UI + Go-chi Stack)

Toàn bộ các tác vụ sẽ được phân rã theo cấu trúc Monorepo (`frontend/` và `backend/`), tích hợp giao diện xếp lịch trực quan, cổng vận hành lớp và hệ thống đánh giá giáo viên.

```bash
# 1. Kiểm tra mã nguồn Backend Go-chi
cd backend
go vet ./...
go test -v ./...

# 2. Kiểm tra mã nguồn Frontend Nuxt UI
cd ../frontend
npm run typecheck
npm run lint

# 3. Kiểm tra tính toàn vẹn AG Kit
cd ..
python .agents/scripts/validate_kit.py
```


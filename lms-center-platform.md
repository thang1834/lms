# Kế Hoạch Chi Tiết: Hệ Thống LMS Cho Trung Tâm Đào Tạo (IT & Đa Ngành)

> **File:** `lms-center-platform.md`  
> **Dự án:** LMS Center Platform (Hệ thống Quản lý Học tập, Giảng viên & Vận hành Đào tạo Đa hình thức)  
> **Chế độ:** PLANNING ONLY (No code writing in this phase)  
> **Phiên bản:** 1.2.0 (Cập nhật Chuẩn RBAC, Vận Hành Lớp, Lịch Học Linh Hoạt & Đánh Giá Moodle)  

---

## 1. Tổng Quan & Mục Tiêu Dự Án (Overview)

Hệ thống LMS được thiết kế riêng cho trung tâm đào tạo hiện đại với trọng tâm là **Công nghệ thông tin (CNTT)**, đồng thời linh hoạt mở rộng hoàn hảo cho **Ngoại ngữ** và các bộ môn khác.

Hệ thống tích hợp toàn diện 7 trụ cột nghiệp vụ:
1. **Chuẩn Hóa Phân Quyền Hạt Nhân RBAC (Dynamic RBAC Architecture)**:
   - Các bảng `roles`, `permissions`, `role_permissions`, `user_roles`.
   - Phân cấp đa tầng: Super Admin, Academic Manager (Giáo vụ trưởng: xếp lớp, lịch dạy, đánh giá giáo viên), Class Coordinator (Vận hành lớp & Chăm sóc học viên), Giảng viên, Trợ giảng, Học viên và Phụ huynh.
2. **Cơ Chế Xếp Lịch Học Linh Hoạt (Flexible Scheduling Engine)**:
   - Thiết lập lịch định kỳ: 1 tuần 1 buổi (ví dụ: Chủ nhật) hoặc 1 tuần nhiều buổi (T2-T4-T6, T3-T5).
   - Tùy biến từng buổi: dời ngày học, đổi giờ ca dạy, đổi phòng học cơ sở hoặc đổi link Google Meet/Zoom, bù buổi học phát sinh, phân công giáo viên dạy thay theo buổi.
3. **Mô Hình Vận Hành Lớp & Chăm Sóc Học Viên (Class Operations & Student Care)**:
   - Mỗi lớp học do bộ ba phụ trách: Giảng viên chính + Trợ giảng + Chuyên viên Vận hành lớp (Coordinator).
   - Chuyên viên Vận hành theo dõi chuyên cần, liên hệ học sinh vắng, nhập `coordinatorNote` gửi phụ huynh, đôn đốc nộp BTVN trước hạn.
4. **Điểm Danh Hai Chiều & Đánh Giá Chất Lượng Giáo Viên**:
   - **Điểm danh học sinh:** Phân loại rõ Có mặt Offline (tại lớp), Có mặt Online (qua Meet/Zoom), Đi trễ, Vắng có phép/không phép.
   - **Điểm danh giáo viên (Teacher Check-in / Timesheet):** Chấm công giờ dạy thực tế đối soát thù lao.
   - **Đánh giá giáo viên:** Quản lý Đào tạo dự giờ đánh giá sư phạm định kỳ + Học viên khảo sát ẩn danh cuối khóa.
5. **Kế Thừa & Tinh Gọn Tính Năng Moodle**:
   - *Activity Completion & Drip Content:* Mở khóa bài học tuần tự theo tiến độ.
   - *Weighted Gradebook:* Sổ điểm đa trọng số (Chuyên cần %, BTVN %, Giữa kỳ %, Cuối khóa %).
   - *Question Bank & Random Quiz:* Ngân hàng câu hỏi & sinh đề trắc nghiệm ngẫu nhiên.
   - *Course Template Cloning:* Nhân bản khóa học/lớp học chỉ với 1 click.
6. **Học Liệu & Bài Tập CNTT (Monaco Editor)**:
   - Bài giảng nhúng YouTube, đính kèm slide PDF, source code mẫu.
   - Trình soạn thảo **Monaco Code Editor** làm bài tập trên web, nộp link GitHub / file ghi âm ngoại ngữ.
7. **Thanh Toán Học Phí Đa Kênh**:
   - Sinh mã **VietQR** động (chuẩn Napas247), thanh toán thẻ **Visa/Mastercard**, và ghi nhận thu **tiền mặt/quẹt thẻ tại quầy** có xuất biên lai PDF.

---

## 2. Mô Hình Phân Quyền Hạt Nhân (Roles & Permissions)

| Vai trò (Role) | Mã Role | Quyền hạn chính |
| :--- | :--- | :--- |
| **Super Admin** | `SUPER_ADMIN` | Toàn quyền cấu hình hệ thống, tạo vai trò mới, quản lý tài chính & báo cáo |
| **Quản lý Đào tạo / Giáo vụ** | `ACADEMIC_MANAGER` | Quản lý khóa học, xếp lớp, xếp lịch học linh hoạt, phân công giáo viên, đánh giá KPI giáo viên |
| **Chuyên viên Vận hành lớp & CSKH** | `CLASS_COORDINATOR` | Đồng hành cùng lớp, theo dõi sĩ số, liên hệ học sinh vắng/trễ, đôn đốc BTVN, hỗ trợ phụ huynh |
| **Giảng viên chính** | `TEACHER` | Xem thời khóa biểu, Check-in ca dạy, điểm danh học sinh, upload tài liệu, giao bài & chấm điểm |
| **Trợ giảng** | `TA` | Hỗ trợ lớp học, điểm danh, hỗ trợ giải đáp thắc mắc bài tập code cho học viên |
| **Học viên** | `STUDENT` | Xem lịch học, học bài YouTube, làm code trên Monaco Editor, nộp bài, thanh toán học phí VietQR |
| **Phụ huynh** | `PARENT` | Theo dõi chuyên cần của con, xem điểm số & nhận xét, thanh toán học phí qua mã VietQR |

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

## 4. Kế Hoạch Triển Khai & Kiểm Thử

Toàn bộ các tác vụ sẽ được phân rã theo đúng chuẩn RBAC, tích hợp giao diện xếp lịch trực quan, cổng vận hành lớp và hệ thống đánh giá giáo viên.

```bash
# Kiểm tra tính toàn vẹn CSDL
npx prisma validate

# Kiểm tra Linter & Type Safety
npm run lint
npx tsc --noEmit

# Kiểm tra tính toàn vẹn AG Kit
python .agents/scripts/validate_kit.py
```

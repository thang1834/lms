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

## 3. Kiến Trúc Cơ Sở Dữ Liệu & Mô Hình Thực Thể Go (Go Domain Models & Goose Migrations)

Toàn bộ CSDL được phiên bản hóa qua **Goose SQL Migrations** (`database/migrations/*.sql`) và biểu diễn dưới dạng các struct Go trong kiến trúc `gmhafiz/go8`:

```go
package model

import (
	"time"
	"github.com/google/uuid"
)

// 1. Phân quyền RBAC động
type Role struct {
	ID          uuid.UUID `json:"id" db:"id"`
	Code        string    `json:"code" db:"code"` // SUPER_ADMIN, ACADEMIC_MANAGER, CLASS_COORDINATOR, TEACHER, TA, EXAMINER, STUDENT, PARENT
	Name        string    `json:"name" db:"name"`
	Description *string   `json:"description" db:"description"`
	IsSystem    bool      `json:"isSystem" db:"is_system"`
}

type Permission struct {
	ID          uuid.UUID `json:"id" db:"id"`
	Code        string    `json:"code" db:"code"` // classes.schedule, capstone.evaluate, etc.
	Name        string    `json:"name" db:"name"`
	Module      string    `json:"module" db:"module"`
	Description *string   `json:"description" db:"description"`
}

// 2. Lớp học & Vận hành lớp (Trợ giảng là tùy chọn)
type Class struct {
	ID                     uuid.UUID  `json:"id" db:"id"`
	CourseID               uuid.UUID  `json:"courseId" db:"course_id"`
	Name                   string     `json:"name" db:"name"` // Ví dụ: FE-K32
	ClassType              string     `json:"classType" db:"class_type"` // OFFLINE, ONLINE_VIRTUAL, HYBRID
	MainTeacherID          uuid.UUID  `json:"mainTeacherId" db:"main_teacher_id"`
	TaTeacherID            *uuid.UUID `json:"taTeacherId,omitempty" db:"ta_teacher_id"` // Trợ giảng Tùy chọn (Optional)
	CoordinatorID          *uuid.UUID `json:"coordinatorId,omitempty" db:"coordinator_id"` // Chuyên viên Vận hành lớp
	RoomName               *string    `json:"roomName,omitempty" db:"room_name"`
	MeetURL                *string    `json:"meetUrl,omitempty" db:"meet_url"`
	ScheduleRule           string     `json:"scheduleRule" db:"schedule_rule"` // JSON: [{"dayOfWeek": 2, "time": "19:30-21:30"}]
	AttendanceAlertMinutes int        `json:"attendanceAlertMinutes" db:"attendance_alert_minutes"` // Mặc định 15p
	Status                 string     `json:"status" db:"status"` // UPCOMING, ACTIVE, COMPLETED
}

// 3. Buổi học linh hoạt (Flexible Session Scheduling)
type ClassSession struct {
	ID                   uuid.UUID  `json:"id" db:"id"`
	ClassID              uuid.UUID  `json:"classId" db:"class_id"`
	SessionNumber        int        `json:"sessionNumber" db:"session_number"`
	SessionDate          time.Time  `json:"sessionDate" db:"session_date"`
	StartTime            string     `json:"startTime" db:"start_time"` // "19:30"
	EndTime              string     `json:"endTime" db:"end_time"`     // "21:30"
	AssignedTeacherID    *uuid.UUID `json:"assignedTeacherId,omitempty" db:"assigned_teacher_id"` // Giáo viên dạy thay
	RoomName             *string    `json:"roomName,omitempty" db:"room_name"`
	MeetURL              *string    `json:"meetUrl,omitempty" db:"meet_url"`
	Topic                *string    `json:"topic,omitempty" db:"topic"`
	Status               string     `json:"status" db:"status"` // SCHEDULED, RESCHEDULED, IN_PROGRESS, COMPLETED
	RescheduleReason     *string    `json:"rescheduleReason,omitempty" db:"reschedule_reason"`
	OriginalDate         *time.Time `json:"originalDate,omitempty" db:"original_date"`
	AttendanceAlertSentAt *time.Time `json:"attendanceAlertSentAt,omitempty" db:"attendance_alert_sent_at"`
}

// 4. Hội đồng Giám khảo & Chấm đồ án tốt nghiệp
type CapstoneEvaluation struct {
	ID                uuid.UUID `json:"id" db:"id"`
	ProjectID         uuid.UUID `json:"projectId" db:"project_id"`
	ExaminerID        uuid.UUID `json:"examinerId" db:"examiner_id"`
	ScoreCompletion   float64   `json:"scoreCompletion" db:"score_completion"`     // 30%
	ScoreArchitecture float64   `json:"scoreArchitecture" db:"score_architecture"` // 25%
	ScorePresentation float64   `json:"scorePresentation" db:"score_presentation"` // 25%
	ScoreCreativity   float64   `json:"scoreCreativity" db:"score_creativity"`     // 20%
	FinalScore        float64   `json:"finalScore" db:"final_score"`
	EvaluationNotes   string    `json:"evaluationNotes" db:"evaluation_notes"`
	EvaluatedAt       time.Time `json:"evaluatedAt" db:"evaluated_at"`
}
```

---

## 4. Kế Hoạch Triển Khai & Kiểm Thử Tự Động Với Taskfile (Go8 Workflow)

Toàn bộ các tác vụ backend được tự động hóa qua `Taskfile.yml` theo chuẩn blueprint `gmhafiz/go8`:

```bash
# 1. Chạy Backend API (Go-chi + Hot reload Air)
cd backend
task dev

# 2. Quản lý CSDL (Goose Migrations)
task migrate

# 3. Tự động sinh tài liệu Swagger/OpenAPI từ Go Handlers
task swagger

# 4. Kiểm tra mã nguồn, linting & quét lỗ hổng bảo mật
task check
task test

# 5. Kiểm tra mã nguồn Frontend Nuxt UI (Vue 3 / TypeScript)
cd ../frontend
npm run typecheck
npm run lint

# 6. Kiểm tra tính toàn vẹn AG Kit
cd ..
python .agents/scripts/validate_kit.py
```


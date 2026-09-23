# Kế Hoạch Chi Tiết: Hệ Thống LMS Cho Trung Tâm Đào Tạo (IT & Đa Ngành)

> **File:** `lms-center-platform.md`  
> **Dự án:** LMS Center Platform (Hệ thống Quản lý Học tập, Giảng viên & Vận hành Đào tạo Đa hình thức)  
> **Kiến trúc:** Monorepo (Frontend: Nuxt UI + Vue 3 | Backend: `gmhafiz/go8` + Go-chi + PostgreSQL)  
> **Chế độ:** PLANNING ONLY (No code writing in this phase)  
> **Phiên bản:** 2.2.0 (Bổ sung Live Class Cockpit, AI Code Review, Voice Note, Hộp Thư Q&A, Chợ Dạy Thay & Sổ Tay Sư Phạm)  
> **Ngày cập nhật:** 23/09/2026  

---

## 1. Tổng Quan & Mục Tiêu Dự Án (Overview)

Hệ thống LMS được thiết kế riêng cho trung tâm đào tạo hiện đại với trọng tâm là **Công nghệ thông tin (CNTT)**, đồng thời linh hoạt mở rộng hoàn hảo cho **Ngoại ngữ** và các bộ môn khác. Backend được xây dựng theo blueprint **`gmhafiz/go8`** (Go-chi, Layered Architecture, Goose migrations, Taskfile) kết hợp Frontend **Nuxt UI** cho 4 cổng người dùng.


Hệ thống tích hợp toàn diện 18 trụ cột nghiệp vụ:
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
11. **Quản Lý Đa Cơ Sở & Chống Trùng Lịch Phòng Học (Multi-Campus & Room Conflict Guard)**:
    - Quản lý mạng lưới chi nhánh cơ sở, phân loại phòng học (Lab PC, Lý thuyết, Hội trường).
    - Bộ lọc kiểm tra xung đột thời gian `OVERLAPS` tự động ngăn ngừa việc xếp 2 lớp trùng phòng cùng khung giờ.
12. **Lên Lịch Học Bù & Dạy Bù Cho Học Sinh Vắng (Make-up Class Scheduling Engine)**:
    - Tự động phát hiện học sinh vắng sau ca học, gợi ý 2 phương án: Ghép lớp song song cùng chủ đề bài giảng hoặc Kèm 1-1 với Trợ giảng/Giáo viên.
    - Điểm danh buổi học bù và tự động đồng bộ chuyên cần để học viên không bị hổng kiến thức và đủ điều kiện thi.
13. **Khảo Thí & Ngân Hàng Câu Hỏi Tự Động (Question Bank & Auto-Quiz)**:
    - Ngân hàng câu hỏi trắc nghiệm theo môn học và cấp độ khó (Dễ, TB, Khó).
    - Thuật toán sinh đề ngẫu nhiên, xáo trộn câu hỏi/đáp án chống gian lận, chấm điểm tức thì và đồng bộ Sổ điểm lớp.
14. **Trung Tâm Điều Hành & Radar Cảnh Báo Nguy Cơ Bỏ Học (Executive Analytics & Churn Radar)**:
    - Dashboard KPIs tài chính và vận hành thời gian thực.
    - Radar tự động gắn cờ đỏ học sinh có nguy cơ bỏ học (vắng 2 buổi liên tiếp hoặc thiếu 3 BTVN) để CSKH can thiệp kịp thời.
    - Ma trận đánh giá hiệu quả giảng viên (Teacher Performance Matrix) theo feedback, đúng giờ và tốc độ trả bài.
15. **Khoang Lái Lớp Học Trực Tuyến & Bảng Vẽ Kỹ Thuật Số (Live Class Cockpit & Digital Whiteboard)**:
    - 1-click khởi động lớp: tự động mở Meet/Zoom, tự động check-in giảng viên và hiển thị phòng học.
    - Điểm danh nhanh 1 chạm cho cả lớp.
    - Bảng vẽ tương tác nhiều người (Excalidraw/Tldraw) với autosave JSON định kỳ 10s và xuất bản PDF đính kèm buổi học.
    - Tạo nhanh câu hỏi bình chọn trực tiếp (Mini Poll 2 phút) kèm biểu đồ kết quả thời gian thực.
16. **Trợ Lý AI Chấm Bài Code & Nhận Xét Bằng Giọng Nói (AI Code Review & Voice Note Feedback)**:
    - Trợ lý AI quét phân tích mã nguồn Clean Code, phát hiện lỗi tiềm ẩn và tạo bản nháp nhận xét sư phạm kèm điểm đề xuất.
    - Ghi âm nhận xét trực tiếp trên trình duyệt (Voice Note 30s-2m) giúp phản hồi sinh động, tiết kiệm 70% thời gian gõ phím.
17. **Hộp Thư Q&A Tập Trung & Ủy Quyền Trợ Giảng (Unified Q&A Inbox & TA Delegation)**:
    - Hộp thư thống nhất thu gom toàn bộ thắc mắc của học sinh từ video bài giảng (kèm timestamp) và bài tập về nhà.
    - Kho câu trả lời mẫu (Snippets) dùng lại câu giải thích thường gặp.
    - Cơ chế ủy quyền câu hỏi cho Trợ giảng phụ trách kèm cam kết thời gian phản hồi (SLA 2h).
18. **Chợ Dạy Thay, Lịch Rảnh & Sổ Tay Sư Phạm Cá Nhân (Substitute Marketplace, Availability & Pedagogy)**:
    - Thiết lập lịch rảnh cố định trong tuần (`teacher_availabilities`).
    - Chợ dạy thay: Đăng yêu cầu dạy thay, hệ thống tự khớp nối với GV cùng chuyên môn có lịch rảnh, tự động điều chuyển thù lao ca dạy.
    - Sổ tay ghi chú sư phạm cá nhân (`student_pedagogical_notes`): Ghi chú bảo mật về tính cách, điểm mạnh/yếu học sinh (chỉ GV và TA đọc được).
    - Ngân hàng đề bài mẫu cá nhân (`assignment_banks`) kèm nút nhân bản 1-click sang các lớp học mới.

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

// 5. Chuẩn 6 trường Audit & Xóa mềm bắt buộc (Universal Audit & Soft Delete)
type AuditFields struct {
	CreatedAt time.Time  `json:"createdAt" db:"created_at"`
	CreatedBy *uuid.UUID `json:"createdBy,omitempty" db:"created_by"`
	UpdatedAt time.Time  `json:"updatedAt" db:"updated_at"`
	UpdatedBy *uuid.UUID `json:"updatedBy,omitempty" db:"updated_by"`
	DeletedAt *time.Time `json:"deletedAt,omitempty" db:"deleted_at"` // Xóa mềm (Soft Delete)
	DeletedBy *uuid.UUID `json:"deletedBy,omitempty" db:"deleted_by"`
}

// 6. Môn học động (Dynamic Subjects CRUD)
type Subject struct {
	ID          uuid.UUID  `json:"id" db:"id"`
	Code        string     `json:"code" db:"code"`
	Name        string     `json:"name" db:"name"`
	Description *string    `json:"description,omitempty" db:"description"`
	IconURL     *string    `json:"iconUrl,omitempty" db:"icon_url"`
	IsActive    bool       `json:"isActive" db:"is_active"`
	AuditFields
}

// 7. Bậc lương & Bảng lương giáo viên (Salary Grades & Payroll)
type SalaryGrade struct {
	ID                 uuid.UUID `json:"id" db:"id"`
	GradeName          string    `json:"gradeName" db:"grade_name"` // INTERN_TA, STANDARD_TEACHER, SENIOR_TEACHER, MASTER_TEACHER
	DisplayName        string    `json:"displayName" db:"display_name"`
	BaseHourlyRate     float64   `json:"baseHourlyRate" db:"base_hourly_rate"`
	OvertimeMultiplier float64   `json:"overtimeMultiplier" db:"overtime_multiplier"`
	KpiBonusRate       float64   `json:"kpiBonusRate" db:"kpi_bonus_rate"`
	AuditFields
}

type TeacherPayroll struct {
	ID                 uuid.UUID  `json:"id" db:"id"`
	TeacherID          uuid.UUID  `json:"teacherId" db:"teacher_id"`
	SalaryGradeID      uuid.UUID  `json:"salaryGradeId" db:"salary_grade_id"`
	Period             string     `json:"period" db:"period"` // "2026-10"
	TotalTeachingHours float64    `json:"totalTeachingHours" db:"total_teaching_hours"`
	BaseSalaryAmount   float64    `json:"baseSalaryAmount" db:"base_salary_amount"`
	KpiBonusAmount     float64    `json:"kpiBonusAmount" db:"kpi_bonus_amount"`
	LatePenaltyAmount  float64    `json:"latePenaltyAmount" db:"late_penalty_amount"`
	NetSalaryAmount    float64    `json:"netSalaryAmount" db:"net_salary_amount"`
	Status             string     `json:"status" db:"status"` // DRAFT, APPROVED, PAID
	ApprovedBy         *uuid.UUID `json:"approvedBy,omitempty" db:"approved_by"`
	ApprovedAt         *time.Time `json:"approvedAt,omitempty" db:"approved_at"`
	AuditFields
}

// 8. Khuyến mãi & Voucher giảm giá (Discounts & Coupons)
type Discount struct {
	ID                uuid.UUID  `json:"id" db:"id"`
	Code              string     `json:"code" db:"code"`
	Description       *string    `json:"description,omitempty" db:"description"`
	DiscountType      string     `json:"discountType" db:"discount_type"` // PERCENT, FIXED
	Value             float64    `json:"value" db:"value"`
	MaxDiscountAmount *float64   `json:"maxDiscountAmount,omitempty" db:"max_discount_amount"`
	MinOrderAmount    float64    `json:"minOrderAmount" db:"min_order_amount"`
	UsageLimit        *int       `json:"usageLimit,omitempty" db:"usage_limit"`
	UsedCount         int        `json:"usedCount" db:"used_count"`
	ValidFrom         time.Time  `json:"validFrom" db:"valid_from"`
	ValidUntil        time.Time  `json:"validUntil" db:"valid_until"`
	IsActive          bool       `json:"isActive" db:"is_active"`
	AuditFields
}

// 9. Bài tập về nhà theo lớp & Chấm điểm (Class Assignments & Grading)
type Assignment struct {
	ID          uuid.UUID  `json:"id" db:"id"`
	ClassID     uuid.UUID  `json:"classId" db:"class_id"`
	Title       string     `json:"title" db:"title"`
	Description string     `json:"description" db:"description"`
	MaxScore    float64    `json:"maxScore" db:"max_score"`
	DueAt       time.Time  `json:"dueAt" db:"due_at"`
	AuditFields
}

type Submission struct {
	ID           uuid.UUID  `json:"id" db:"id"`
	AssignmentID uuid.UUID  `json:"assignmentId" db:"assignment_id"`
	StudentID    uuid.UUID  `json:"studentId" db:"student_id"`
	CodeContent  *string    `json:"codeContent,omitempty" db:"code_content"`
	GithubURL    *string    `json:"githubUrl,omitempty" db:"github_url"`
	Score        *float64   `json:"score,omitempty" db:"score"`
	TeacherNotes *string    `json:"teacherNotes,omitempty" db:"teacher_notes"`
	GradedBy     *uuid.UUID `json:"gradedBy,omitempty" db:"graded_by"`
	GradedAt     *time.Time `json:"gradedAt,omitempty" db:"graded_at"`
	Status       string     `json:"status" db:"status"` // SUBMITTED, GRADED, LATE
	AuditFields
}

// 10. Cấu hình Website thương hiệu & Nhật ký kiểm toán (Website Settings & Audit Logs)
type WebsiteSettings struct {
	ID              uuid.UUID  `json:"id" db:"id"`
	SiteTitle       string     `json:"siteTitle" db:"site_title"`
	SiteTagline     *string    `json:"siteTagline,omitempty" db:"site_tagline"`
	LogoURL         *string    `json:"logoUrl,omitempty" db:"logo_url"`
	LogoDarkURL     *string    `json:"logoDarkUrl,omitempty" db:"logo_dark_url"`
	FaviconURL      *string    `json:"faviconUrl,omitempty" db:"favicon_url"`
	HeaderBgColor   string     `json:"headerBgColor" db:"header_bg_color"`
	HeaderTextColor string     `json:"headerTextColor" db:"header_text_color"`
	FooterBgColor   string     `json:"footerBgColor" db:"footer_bg_color"`
	FooterTextColor string     `json:"footerTextColor" db:"footer_text_color"`
	FooterCopyright *string    `json:"footerCopyright,omitempty" db:"footer_copyright"`
	PrimaryColor    string     `json:"primaryColor" db:"primary_color"`
	BannerURL       *string    `json:"bannerUrl,omitempty" db:"banner_url"`
	ContactEmail    *string    `json:"contactEmail,omitempty" db:"contact_email"`
	ContactPhone    *string    `json:"contactPhone,omitempty" db:"contact_phone"`
	Hotline         *string    `json:"hotline,omitempty" db:"hotline"`
	Address         *string    `json:"address,omitempty" db:"address"`
	SocialLinks     *string    `json:"socialLinks,omitempty" db:"social_links"` // JSON string
	SeoMeta         *string    `json:"seoMeta,omitempty" db:"seo_meta"`         // JSON string
	AuditFields
}

type SystemAuditLog struct {
	ID           uuid.UUID  `json:"id" db:"id"`
	ActorID      uuid.UUID  `json:"actorId" db:"actor_id"`
	ActorEmail   string     `json:"actorEmail" db:"actor_email"`
	Action       string     `json:"action" db:"action"`
	ResourceType string     `json:"resourceType" db:"resource_type"`
	ResourceID   string     `json:"resourceId" db:"resource_id"`
	DiffJSON     *string    `json:"diffJson,omitempty" db:"diff_json"`
	IPAddress    *string    `json:"ipAddress,omitempty" db:"ip_address"`
	UserAgent    *string    `json:"userAgent,omitempty" db:"user_agent"`
	CreatedAt    time.Time  `json:"createdAt" db:"created_at"`
}

// 12. Cơ sở, phòng học & Lịch học bù
type Campus struct {
	ID        uuid.UUID `json:"id" db:"id"`
	Code      string    `json:"code" db:"code"`
	Name      string    `json:"name" db:"name"`
	Address   string    `json:"address" db:"address"`
	Phone     *string   `json:"phone,omitempty" db:"phone"`
	Email     *string   `json:"email,omitempty" db:"email"`
	IsActive  bool      `json:"isActive" db:"is_active"`
	AuditFields
}

type Room struct {
	ID         uuid.UUID `json:"id" db:"id"`
	CampusID   uuid.UUID `json:"campusId" db:"campus_id"`
	Code       string    `json:"code" db:"code"`
	Name       string    `json:"name" db:"name"`
	Capacity   int       `json:"capacity" db:"capacity"`
	RoomType   string    `json:"roomType" db:"room_type"` // LAB_PC, THEORY_ROOM, HALL, STUDIO
	Facilities *string   `json:"facilities,omitempty" db:"facilities"` // JSON string
	IsActive   bool      `json:"isActive" db:"is_active"`
	AuditFields
}

type MakeupSession struct {
	ID                uuid.UUID  `json:"id" db:"id"`
	OriginalSessionID uuid.UUID  `json:"originalSessionId" db:"original_session_id"`
	StudentID         uuid.UUID  `json:"studentId" db:"student_id"`
	MakeupType        string     `json:"makeupType" db:"makeup_type"` // PARALLEL_CLASS, TUTOR_1ON1
	TargetClassID     *uuid.UUID `json:"targetClassId,omitempty" db:"target_class_id"`
	TargetSessionID   *uuid.UUID `json:"targetSessionId,omitempty" db:"target_session_id"`
	InstructorID      *uuid.UUID `json:"instructorId,omitempty" db:"instructor_id"`
	ScheduledDate     time.Time  `json:"scheduledDate" db:"scheduled_date"`
	StartTime         string     `json:"startTime" db:"start_time"`
	EndTime           string     `json:"endTime" db:"end_time"`
	RoomID            *uuid.UUID `json:"roomId,omitempty" db:"room_id"`
	MeetURL           *string    `json:"meetUrl,omitempty" db:"meet_url"`
	Status            string     `json:"status" db:"status"` // SCHEDULED, ATTENDED, ABSENT, CANCELLED
	CoordinatorNotes  *string    `json:"coordinatorNotes,omitempty" db:"coordinator_notes"`
	AuditFields
}

// 13. Khảo thí & Ngân hàng câu hỏi
type QuestionBank struct {
	ID          uuid.UUID `json:"id" db:"id"`
	SubjectID   uuid.UUID `json:"subjectId" db:"subject_id"`
	Code        string    `json:"code" db:"code"`
	Name        string    `json:"name" db:"name"`
	Description *string   `json:"description,omitempty" db:"description"`
	AuditFields
}

type Question struct {
	ID           uuid.UUID `json:"id" db:"id"`
	BankID       uuid.UUID `json:"bankId" db:"bank_id"`
	QuestionType string    `json:"questionType" db:"question_type"` // SINGLE_CHOICE, MULTIPLE_CHOICE, TRUE_FALSE, SHORT_ANSWER
	Content      string    `json:"content" db:"content"`
	MediaURL     *string   `json:"mediaUrl,omitempty" db:"media_url"`
	Options      string    `json:"options" db:"options"` // JSON string
	Difficulty   string    `json:"difficulty" db:"difficulty"` // EASY, MEDIUM, HARD
	DefaultPoints float64  `json:"defaultPoints" db:"default_points"`
	AuditFields
}

type Quiz struct {
	ID                 uuid.UUID  `json:"id" db:"id"`
	CourseID           uuid.UUID  `json:"courseId" db:"course_id"`
	ClassID            *uuid.UUID `json:"classId,omitempty" db:"class_id"`
	Title              string     `json:"title" db:"title"`
	DurationMinutes    int        `json:"durationMinutes" db:"duration_minutes"`
	PassingScore       float64    `json:"passingScore" db:"passing_score"`
	MaxAttempts        int        `json:"maxAttempts" db:"max_attempts"`
	IsShuffleQuestions bool       `json:"isShuffleQuestions" db:"is_shuffle_questions"`
	IsShuffleOptions   bool       `json:"isShuffleOptions" db:"is_shuffle_options"`
	Status             string     `json:"status" db:"status"` // DRAFT, PUBLISHED, CLOSED
	AuditFields
}

type QuizAttempt struct {
	ID            uuid.UUID  `json:"id" db:"id"`
	QuizID        uuid.UUID  `json:"quizId" db:"quiz_id"`
	StudentID     uuid.UUID  `json:"studentId" db:"student_id"`
	AttemptNumber int        `json:"attemptNumber" db:"attempt_number"`
	StartedAt     time.Time  `json:"startedAt" db:"started_at"`
	SubmittedAt   *time.Time `json:"submittedAt,omitempty" db:"submitted_at"`
	TotalScore    float64    `json:"totalScore" db:"total_score"`
	IsPassed      bool       `json:"isPassed" db:"is_passed"`
	AuditFields
}

// 14. Hợp đồng đào tạo điện tử
type Contract struct {
	ID            uuid.UUID  `json:"id" db:"id"`
	ContractCode  string     `json:"contractCode" db:"contract_code"`
	StudentID     uuid.UUID  `json:"studentId" db:"student_id"`
	CourseID      *uuid.UUID `json:"courseId,omitempty" db:"course_id"`
	ClassID       *uuid.UUID `json:"classId,omitempty" db:"class_id"`
	ContractType  string     `json:"contractType" db:"contract_type"` // TRAINING_COMMITMENT, TUITION_INSTALLMENT, JOB_PLACEMENT
	Title         string     `json:"title" db:"title"`
	TermsContent  string     `json:"termsContent" db:"terms_content"`
	FilePdfURL    *string    `json:"filePdfUrl,omitempty" db:"file_pdf_url"`
	SignedAt      *time.Time `json:"signedAt,omitempty" db:"signed_at"`
	SignatureData *string    `json:"signatureData,omitempty" db:"signature_data"` // JSON string
	Status        string     `json:"status" db:"status"` // DRAFT, SENT, SIGNED, EXPIRED, TERMINATED
	AuditFields
}

// 15. Khoang lái lớp học trực tuyến & Bảng vẽ kỹ thuật số
type ClassWhiteboard struct {
	ID           uuid.UUID `json:"id" db:"id"`
	SessionID    uuid.UUID `json:"sessionId" db:"session_id"`
	TeacherID    uuid.UUID `json:"teacherId" db:"teacher_id"`
	Title        string    `json:"title" db:"title"`
	BoardData    string    `json:"boardData" db:"board_data"` // JSON (Excalidraw/Tldraw scene data)
	ExportPdfURL *string   `json:"exportPdfUrl,omitempty" db:"export_pdf_url"`
	AuditFields
}

type QuickPoll struct {
	ID              uuid.UUID `json:"id" db:"id"`
	SessionID       uuid.UUID `json:"sessionId" db:"session_id"`
	QuestionText    string    `json:"questionText" db:"question_text"`
	Options         string    `json:"options" db:"options"` // JSON array: [{"id": "opt1", "text": "Đã hiểu"}]
	CorrectOptionID *string   `json:"correctOptionId,omitempty" db:"correct_option_id"`
	IsActive        bool      `json:"isActive" db:"is_active"`
	DurationSeconds int       `json:"durationSeconds" db:"duration_seconds"`
	AuditFields
}

type PollVote struct {
	ID               uuid.UUID `json:"id" db:"id"`
	PollID           uuid.UUID `json:"pollId" db:"poll_id"`
	StudentID        uuid.UUID `json:"studentId" db:"student_id"`
	SelectedOptionID string    `json:"selectedOptionId" db:"selected_option_id"`
	VotedAt          time.Time `json:"votedAt" db:"voted_at"`
}

// 16. Trợ lý AI Code Review & Voice Note Feedback
type AICodeReview struct {
	ID             uuid.UUID `json:"id" db:"id"`
	SubmissionID   uuid.UUID `json:"submissionId" db:"submission_id"`
	LintIssues     string    `json:"lintIssues" db:"lint_issues"` // JSON array
	SuggestedScore float64   `json:"suggestedScore" db:"suggested_score"`
	FeedbackDraft  string    `json:"feedbackDraft" db:"feedback_draft"`
	Status         string    `json:"status" db:"status"` // PENDING, GENERATED, APPLIED
	AuditFields
}

// 17. Sổ tay sư phạm & Ngân hàng đề bài riêng
type StudentPedagogicalNote struct {
	ID             uuid.UUID `json:"id" db:"id"`
	StudentID      uuid.UUID `json:"studentId" db:"student_id"`
	TeacherID      uuid.UUID `json:"teacherId" db:"teacher_id"`
	ClassID        uuid.UUID `json:"classId" db:"class_id"`
	NoteContent    string    `json:"noteContent" db:"note_content"`
	IsSharedWithTA bool      `json:"isSharedWithTa" db:"is_shared_with_ta"` // Confidential from Student & Parent
	AuditFields
}

type AssignmentBank struct {
	ID                 uuid.UUID `json:"id" db:"id"`
	TeacherID          uuid.UUID `json:"teacherId" db:"teacher_id"`
	SubjectID          uuid.UUID `json:"subjectId" db:"subject_id"`
	Title              string    `json:"title" db:"title"`
	Description        string    `json:"description" db:"description"`
	Format             string    `json:"format" db:"format"` // MONACO_CODE, ESSAY, GITHUB_URL, QUIZ
	StarterCode        *string   `json:"starterCode,omitempty" db:"starter_code"`
	SolutionCode       *string   `json:"solutionCode,omitempty" db:"solution_code"`
	RubricCriteria     string    `json:"rubricCriteria" db:"rubric_criteria"` // JSON
	IsSharedWithCenter bool      `json:"isSharedWithCenter" db:"is_shared_with_center"`
	AuditFields
}

// 18. Chợ dạy thay & Lịch rảnh tuần
type TeacherAvailability struct {
	ID        uuid.UUID `json:"id" db:"id"`
	TeacherID uuid.UUID `json:"teacherId" db:"teacher_id"`
	DayOfWeek int       `json:"dayOfWeek" db:"day_of_week"` // 1 (Mon) - 7 (Sun)
	StartTime string    `json:"startTime" db:"start_time"` // "18:00"
	EndTime   string    `json:"endTime" db:"end_time"`     // "21:00"
	IsActive  bool      `json:"isActive" db:"is_active"`
	AuditFields
}

type SubstituteRequest struct {
	ID                  uuid.UUID  `json:"id" db:"id"`
	SessionID           uuid.UUID  `json:"sessionId" db:"session_id"`
	OriginalTeacherID   uuid.UUID  `json:"originalTeacherId" db:"original_teacher_id"`
	SubstituteTeacherID *uuid.UUID `json:"substituteTeacherId,omitempty" db:"substitute_teacher_id"`
	Reason              string     `json:"reason" db:"reason"`
	CompensationRate    float64    `json:"compensationRate" db:"compensation_rate"`
	Status              string     `json:"status" db:"status"` // OPEN, ACCEPTED, REJECTED, CANCELLED
	ResolvedAt          *time.Time `json:"resolvedAt,omitempty" db:"resolved_at"`
	AuditFields
}
```

---

## 4. Kế Hoạch Triển Khai & Kiểm Thử Tự Động Với Taskfile (Go8 Workflow)

Toàn bộ các tác vụ backend được tự động hóa qua `Taskfile.yml` theo chuẩn blueprint `gmhafiz/go8`:

```bash
# 1. Khởi chạy cụm hạ tầng OpenTelemetry Observability (Prometheus, Jaeger, Loki, Grafana)
cd backend
task infra:up
# Truy cập Grafana Dashboard: http://localhost:3300 (admin/admin)
# Truy cập Jaeger Traces: http://localhost:16686

# 2. Chạy Backend API (Go-chi + Hot reload Air)
task dev

# 3. Quản lý CSDL (Goose Migrations)
task migrate

# 4. Tự động sinh tài liệu Swagger/OpenAPI từ Go Handlers
task swagger

# 5. Kiểm tra mã nguồn, linting & quét lỗ hổng bảo mật
task check
task test

# 6. Kiểm tra mã nguồn Frontend Nuxt UI (Vue 3 / TypeScript)
cd ../frontend
npm run typecheck
npm run lint

# 7. Kiểm tra tính toàn vẹn AG Kit
cd ..
python .agents/scripts/validate_kit.py
```


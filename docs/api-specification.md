# Đặc Tả Giao Diện Lập Trình Ứng Dụng (Go-chi REST API & Swagger/OpenAPI Specification)

> **Dự án:** LMS Center Platform (Hệ thống Quản lý Học tập, Giảng viên & Vận hành Đào tạo Đa hình thức)  
> **Tài liệu:** `docs/api-specification.md`  
> **Phiên bản:** 2.5.0 (Bổ sung Phân hệ 21: Cổng Vận Hành Lớp, Chăm Sóc Học Viên & Chuyển Lớp CLASS_COORDINATOR)  
> **Ngày cập nhật:** 23/09/2026  

---

## 1. Quy Chuẩn Thiết Kế API & Swagger/OpenAPI Chuẩn Blueprint `gmhafiz/go8`

Toàn bộ các giao tiếp dữ liệu giữa Client (Nuxt UI) và Server (Go Backend) tuân thủ tiêu chuẩn:
- **Router:** `go-chi/chi/v5` với kiến trúc 3 tầng: `Handler` (HTTP layer) $\to$ `UseCase` (Business layer) $\to$ `Repository` (Data layer).
- **Swagger Engine:** Sử dụng `swaggo/swag` v1.16+ quét mã nguồn và sinh tài liệu Swagger 2.0 / OpenAPI 3.0 tự động khi chạy lệnh `task swagger`.
- **Validation:** Thư viện `go-playground/validator/v10` kiểm tra ràng buộc struct tags (`validate:"required,min=5,email"`).

### 1.1 Chuẩn Hóa Phản Hồi (JSON Response Envelope - `pkg/response`)

```go
package response

// Envelope định dạng chuẩn cho mọi phản hồi JSON từ Go Backend
type Envelope struct {
	Success bool         `json:"success" example:"true"`
	Data    any          `json:"data,omitempty"`
	Message string       `json:"message,omitempty" example:"Thao tác thực hiện thành công"`
	Error   *ErrorDetail `json:"error,omitempty"`
}

// ErrorDetail cấu trúc chi tiết lỗi trả về cho client
type ErrorDetail struct {
	Code    string   `json:"code" example:"INVALID_CREDENTIALS"`
	Message string   `json:"message" example:"Email hoặc mật khẩu không chính xác"`
	Details []string `json:"details,omitempty"`
}
```

---

## 2. Chi Tiết Handlers, Swagger Annotations & Sample DTOs Theo Domain

---

### Phân Hệ 1: Xác Thực & Người Dùng (`internal/domain/auth`)

#### 1.1 `POST /api/v1/auth/login` - Đăng nhập hệ thống

```go
package handler

import (
	"net/http"
	"backend/internal/domain/auth/dto"
	"backend/pkg/response"
)

// Login godoc
// @Summary Đăng nhập hệ thống
// @Description Xác thực tài khoản bằng email & mật khẩu, trả về JWT Access Token (hạn 24h), Refresh Token và thông tin vai trò
// @Tags Auth
// @Accept json
// @Produce json
// @Param request body dto.LoginRequest true "Thông tin tài khoản đăng nhập"
// @Success 200 {object} response.Envelope{data=dto.LoginResponse} "Đăng nhập thành công"
// @Failure 400 {object} response.Envelope{error=response.ErrorDetail} "Dữ liệu yêu cầu không hợp lệ"
// @Failure 401 {object} response.Envelope{error=response.ErrorDetail} "Email hoặc mật khẩu không chính xác"
// @Router /api/v1/auth/login [post]
func (h *AuthHandler) Login(w http.ResponseWriter, r *http.Request) {
	var req dto.LoginRequest
	if err := h.validator.BindAndValidate(r, &req); err != nil {
		response.BadRequest(w, "VALIDATION_FAILED", err.Error())
		return
	}
	res, err := h.useCase.Login(r.Context(), req)
	if err != nil {
		response.Unauthorized(w, "INVALID_CREDENTIALS", "Email hoặc mật khẩu không chính xác")
		return
	}
	response.OK(w, res, "Đăng nhập thành công")
}
```

- **DTO Request & Response Structs:**
```go
package dto

import "github.com/google/uuid"

type LoginRequest struct {
	Email    string `json:"email" validate:"required,email" example:"tuan.nguyen@lms.edu.vn"`
	Password string `json:"password" validate:"required,min=6" example:"Secret@123456"`
}

type LoginResponse struct {
	AccessToken  string    `json:"accessToken" example:"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."`
	RefreshToken string    `json:"refreshToken" example:"d7a8f9c2-4e6a-4f9e-87a1-2b3c4d5e6f7a"`
	ExpiresIn    int64     `json:"expiresIn" example:"86400"`
	User         UserInfo  `json:"user"`
}

type UserInfo struct {
	ID          uuid.UUID `json:"id" example:"a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11"`
	Email       string    `json:"email" example:"tuan.nguyen@lms.edu.vn"`
	FullName    string    `json:"fullName" example:"Nguyễn Văn Tuấn"`
	Roles       []string  `json:"roles" example:"[\"TEACHER\", \"ACADEMIC_MANAGER\"]"`
	Permissions []string  `json:"permissions" example:"[\"classes.schedule\", \"attendance.record\"]"`
}
```

---

### Phân Hệ 2: Khóa Học, Xếp Lớp & Lịch Học Linh Hoạt (`internal/domain/class`)

#### 2.1 `POST /api/v1/classes` - Mở lớp học với quy tắc lịch lặp lại (Recurrence) & Trợ giảng tùy chọn

```go
package handler

import (
	"net/http"
	"backend/internal/domain/class/dto"
	"backend/pkg/response"
)

// CreateClass godoc
// @Summary Tạo lớp học mới và sinh lịch học tự động
// @Description Khởi tạo lớp học với cấu hình lịch tuần lặp lại (1 buổi hoặc nhiều buổi/tuần). Trợ giảng là tùy chọn (Optional). Hệ thống tự động sinh toàn bộ danh sách các buổi học (class_sessions)
// @Tags Classes
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param request body dto.CreateClassRequest true "Cấu hình lớp học và quy tắc thời khóa biểu"
// @Success 201 {object} response.Envelope{data=dto.ClassDetailResponse} "Tạo lớp học và sinh lịch thành công"
// @Failure 400 {object} response.Envelope{error=response.ErrorDetail} "Dữ liệu đầu vào sai định dạng"
// @Failure 403 {object} response.Envelope{error=response.ErrorDetail} "Không đủ quyền hạn (Yêu cầu ACADEMIC_MANAGER hoặc SUPER_ADMIN)"
// @Router /api/v1/classes [post]
func (h *ClassHandler) CreateClass(w http.ResponseWriter, r *http.Request) {
	var req dto.CreateClassRequest
	if err := h.validator.BindAndValidate(r, &req); err != nil {
		response.BadRequest(w, "VALIDATION_FAILED", err.Error())
		return
	}
	res, err := h.useCase.CreateClassWithSchedule(r.Context(), req)
	if err != nil {
		response.InternalServerError(w, "CLASS_CREATION_FAILED", err.Error())
		return
	}
	response.Created(w, res, "Khởi tạo lớp học và sinh thời khóa biểu thành công")
}
```

- **DTO Request & Response Structs:**
```go
package dto

import (
	"time"
	"github.com/google/uuid"
)

type ScheduleRuleItem struct {
	DayOfWeek int    `json:"dayOfWeek" validate:"required,min=1,max=7" example:"2"` // 2: Thứ 2, 7: Thứ 7, 1: Chủ Nhật
	StartTime string `json:"startTime" validate:"required" example:"19:30"`
	EndTime   string `json:"endTime" validate:"required" example:"21:30"`
}

type CreateClassRequest struct {
	CourseID               uuid.UUID          `json:"courseId" validate:"required" example:"b1eebc99-9c0b-4ef8-bb6d-6bb9bd380b22"`
	Name                   string             `json:"name" validate:"required,min=3" example:"LMS-FE-K32"`
	ClassType              string             `json:"classType" validate:"required,oneof=OFFLINE ONLINE_VIRTUAL HYBRID" example:"HYBRID"`
	MainTeacherID          uuid.UUID          `json:"mainTeacherId" validate:"required" example:"c2eebc99-9c0b-4ef8-bb6d-6bb9bd380c33"`
	TaTeacherID            *uuid.UUID         `json:"taTeacherId,omitempty" example:"d3eebc99-9c0b-4ef8-bb6d-6bb9bd380d44"` // Tùy chọn (Optional)
	CoordinatorID          *uuid.UUID         `json:"coordinatorId,omitempty" example:"e4eebc99-9c0b-4ef8-bb6d-6bb9bd380e55"`
	RoomName               string             `json:"roomName,omitempty" example:"Phòng Lab 302 - Cơ sở 1"`
	MeetURL                string             `json:"meetUrl,omitempty" example:"https://meet.google.com/abc-xyz-lms"`
	StartDate              string             `json:"startDate" validate:"required" example:"2026-10-01"`
	TotalSessions          int                `json:"totalSessions" validate:"required,min=1" example:"24"`
	ScheduleRules          []ScheduleRuleItem `json:"scheduleRules" validate:"required,min=1"`
	AttendanceAlertMinutes int                `json:"attendanceAlertMinutes" validate:"min=5" example:"15"`
}

type ClassDetailResponse struct {
	ID                     uuid.UUID `json:"id" example:"f5eebc99-9c0b-4ef8-bb6d-6bb9bd380f66"`
	Name                   string    `json:"name" example:"LMS-FE-K32"`
	ClassType              string    `json:"classType" example:"HYBRID"`
	TotalGeneratedSessions int       `json:"totalGeneratedSessions" example:"24"`
	StartDate              string    `json:"startDate" example:"2026-10-01"`
	EndDate                string    `json:"endDate" example:"2026-12-20"`
	Status                 string    `json:"status" example:"UPCOMING"`
}
```

#### 2.2 `PUT /api/v1/sessions/{id}/reschedule` - Dời lịch học & phân công giáo viên dạy thay

```go
// RescheduleSession godoc
// @Summary Dời ngày học, đổi phòng học hoặc phân công giáo viên dạy thay
// @Description Điều chỉnh thông tin của một buổi học cụ thể (class_sessions). Hệ thống tự động gửi thông báo cập nhật lịch học tới Giảng viên, Học viên và Phụ huynh
// @Tags Classes
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param id path string true "ID buổi học (UUID)" format(uuid)
// @Param request body dto.RescheduleSessionRequest true "Thông tin điều chỉnh lịch"
// @Success 200 {object} response.Envelope{data=dto.SessionItemResponse} "Điều chỉnh lịch học thành công"
// @Failure 400 {object} response.Envelope{error=response.ErrorDetail} "Dữ liệu không hợp lệ"
// @Failure 404 {object} response.Envelope{error=response.ErrorDetail} "Không tìm thấy buổi học"
// @Router /api/v1/sessions/{id}/reschedule [put]
func (h *ClassHandler) RescheduleSession(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

- **DTO Request Struct:**
```go
type RescheduleSessionRequest struct {
	NewSessionDate     string     `json:"newSessionDate" validate:"required" example:"2026-10-15"`
	NewStartTime       string     `json:"newStartTime" validate:"required" example:"20:00"`
	NewEndTime         string     `json:"newEndTime" validate:"required" example:"22:00"`
	AssignedTeacherID  *uuid.UUID `json:"assignedTeacherId,omitempty" example:"c2eebc99-9c0b-4ef8-bb6d-6bb9bd380c33"` // Giáo viên dạy thay
	RoomName           string     `json:"roomName,omitempty" example:"Phòng 405"`
	MeetURL            string     `json:"meetUrl,omitempty" example:"https://meet.google.com/new-link-lms"`
	RescheduleReason   string     `json:"rescheduleReason" validate:"required,min=5" example:"Giảng viên chính bận công tác đột xuất, đổi sang thầy Nam dạy thay"`
}
```

---

### Phân Hệ 3: Điểm Danh, Chấm Công & Cảnh Báo Trễ (`internal/domain/attendance`)

#### 3.1 `POST /api/v1/attendance/teacher-checkin` - Giáo viên Check-in ca dạy

```go
// TeacherCheckIn godoc
// @Summary Giáo viên check-in hoặc check-out ca dạy
// @Description Ghi nhận giờ vào lớp/ra lớp thực tế của giáo viên để tính công thù lao và chấm dứt cảnh báo đi muộn
// @Tags Attendance
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param request body dto.TeacherCheckInRequest true "Thông tin ca dạy và hành động"
// @Success 200 {object} response.Envelope{data=dto.TeacherAttendanceResponse} "Ghi nhận chấm công thành công"
// @Failure 400 {object} response.Envelope{error=response.ErrorDetail} "Hành động hoặc mã buổi học không hợp lệ"
// @Router /api/v1/attendance/teacher-checkin [post]
func (h *AttendanceHandler) TeacherCheckIn(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

- **DTO Request & Response Structs:**
```go
type TeacherCheckInRequest struct {
	SessionID uuid.UUID `json:"sessionId" validate:"required" example:"a1eebc99-9c0b-4ef8-bb6d-6bb9bd380a11"`
	Action    string    `json:"action" validate:"required,oneof=CHECK_IN CHECK_OUT" example:"CHECK_IN"`
	Note      string    `json:"note,omitempty" example:"Bắt đầu buổi học đúng giờ"`
}

type TeacherAttendanceResponse struct {
	ID          uuid.UUID  `json:"id" example:"b2eebc99-9c0b-4ef8-bb6d-6bb9bd380b22"`
	SessionID   uuid.UUID  `json:"sessionId" example:"a1eebc99-9c0b-4ef8-bb6d-6bb9bd380a11"`
	CheckInTime *time.Time `json:"checkInTime" example:"2026-10-01T19:28:00Z"`
	Status      string     `json:"status" example:"ON_TIME"`
	Message     string     `json:"message" example:"Check-in ca dạy thành công lúc 19:28"`
}
```

#### 3.2 `POST /api/v1/attendance/student-batch` - Điểm danh học sinh đa mô hình (Hybrid) & Ghi chú chăm sóc

```go
// BatchStudentAttendance godoc
// @Summary Lưu bảng điểm danh học sinh của buổi học
// @Description Ghi nhận chuyên cần từng học sinh (Offline/Online/Late/Absent) kèm ghi chú chăm sóc của Chuyên viên Vận hành lớp
// @Tags Attendance
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param request body dto.BatchStudentAttendanceRequest true "Danh sách điểm danh học sinh"
// @Success 200 {object} response.Envelope{data=dto.BatchAttendanceResult} "Lưu điểm danh thành công"
// @Failure 400 {object} response.Envelope{error=response.ErrorDetail} "Dữ liệu học sinh không hợp lệ"
// @Router /api/v1/attendance/student-batch [post]
func (h *AttendanceHandler) BatchStudentAttendance(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

- **DTO Request Struct:**
```go
type StudentAttendanceRecord struct {
	StudentID       uuid.UUID `json:"studentId" validate:"required"`
	Status          string    `json:"status" validate:"required,oneof=PRESENT_OFFLINE PRESENT_ONLINE LATE ABSENT" example:"PRESENT_OFFLINE"`
	LateMinutes     int       `json:"lateMinutes" example:"0"`
	IsExcused       bool      `json:"isExcused" example:"false"`
	TeacherNote     string    `json:"teacherNote,omitempty" example:"Nắm bài tốt, giải quyết bài toán nhanh"`
	CoordinatorNote string    `json:"coordinatorNote,omitempty" example:"Đã liên hệ phụ huynh, gia đình xác nhận có mặt"`
}

type BatchStudentAttendanceRequest struct {
	SessionID uuid.UUID                 `json:"sessionId" validate:"required"`
	Records   []StudentAttendanceRecord `json:"records" validate:"required,min=1"`
}
```

---

### Phân Hệ 4: Đánh Giá Chất Lượng Giáo Viên & Buổi Học (`internal/domain/evaluation`)

#### 4.1 `POST /api/v1/sessions/{sessionId}/feedback` - Học sinh đánh giá giáo viên sau mỗi buổi học

```go
// SubmitSessionFeedback godoc
// @Summary Học sinh gửi đánh giá nhanh chất lượng buổi học
// @Description Học sinh đánh giá chất lượng Giảng viên chính, Trợ giảng (nếu có), mức độ hiểu bài và tốc độ giảng dạy trong vòng 24h sau buổi học
// @Tags Evaluations
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param sessionId path string true "ID buổi học (UUID)" format(uuid)
// @Param request body dto.SessionFeedbackRequest true "Phiếu đánh giá buổi học"
// @Success 201 {object} response.Envelope{data=dto.SessionFeedbackResponse} "Gửi nhận xét thành công"
// @Failure 400 {object} response.Envelope{error=response.ErrorDetail} "Dữ liệu đánh giá không hợp lệ hoặc đã đánh giá trước đó"
// @Router /api/v1/sessions/{sessionId}/feedback [post]
func (h *EvaluationHandler) SubmitSessionFeedback(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

- **DTO Request & Response Structs:**
```go
type SessionFeedbackRequest struct {
	TeacherRating      int    `json:"teacherRating" validate:"required,min=1,max=5" example:"5"`
	TaRating           *int   `json:"taRating,omitempty" validate:"omitempty,min=1,max=5" example:"5"` // Tùy chọn nếu lớp có TA
	UnderstandingLevel string `json:"understandingLevel" validate:"required,oneof=POOR AVERAGE GOOD EXCELLENT" example:"EXCELLENT"`
	PacingFeedback     string `json:"pacingFeedback" validate:"required,oneof=TOO_SLOW JUST_RIGHT TOO_FAST" example:"JUST_RIGHT"`
	Comment            string `json:"comment,omitempty" example:"Thầy giải thích phần State Management rất dễ hiểu và thực tế"`
	IsAnonymous        bool   `json:"isAnonymous" example:"true"`
}

type SessionFeedbackResponse struct {
	ID          uuid.UUID `json:"id" example:"e1eebc99-9c0b-4ef8-bb6d-6bb9bd380e11"`
	SessionID   uuid.UUID `json:"sessionId" example:"a1eebc99-9c0b-4ef8-bb6d-6bb9bd380a11"`
	SubmittedAt time.Time `json:"submittedAt" example:"2026-10-01T21:45:00Z"`
}
```

---

### Phân Hệ 5: Hội Đồng Giám Khảo & Chấm Đồ Án Tốt Nghiệp (`internal/domain/capstone`)

#### 5.1 `POST /api/v1/capstone/evaluations` - Giám khảo chấm điểm đồ án tốt nghiệp

```go
// EvaluateCapstoneProject godoc
// @Summary Giám khảo chấm điểm đồ án tốt nghiệp và phản biện thuyết trình
// @Description Thành viên Hội đồng Giám khảo (Examiner / Reviewer) nhập điểm theo Rubric 4 tiêu chí và đưa ra nhận xét chuyên môn độc lập cho đồ án của học viên
// @Tags Capstone Jury
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param request body dto.CapstoneEvaluationRequest true "Phiếu chấm điểm đồ án tốt nghiệp"
// @Success 201 {object} response.Envelope{data=dto.CapstoneEvaluationResult} "Nộp phiếu chấm đồ án thành công"
// @Failure 400 {object} response.Envelope{error=response.ErrorDetail} "Dữ liệu chấm điểm không hợp lệ"
// @Failure 403 {object} response.Envelope{error=response.ErrorDetail} "Chỉ tài khoản có vai trò EXAMINER mới có quyền chấm điểm"
// @Router /api/v1/capstone/evaluations [post]
func (h *CapstoneHandler) EvaluateCapstoneProject(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

- **DTO Request & Response Structs:**
```go
type CapstoneEvaluationRequest struct {
	ProjectID         uuid.UUID `json:"projectId" validate:"required" example:"f1eebc99-9c0b-4ef8-bb6d-6bb9bd380f11"`
	ScoreCompletion   float64   `json:"scoreCompletion" validate:"required,min=0,max=100" example:"90.0"`   // Trọng số 30%
	ScoreArchitecture float64   `json:"scoreArchitecture" validate:"required,min=0,max=100" example:"85.0"` // Trọng số 25%
	ScorePresentation float64   `json:"scorePresentation" validate:"required,min=0,max=100" example:"95.0"` // Trọng số 25%
	ScoreCreativity   float64   `json:"scoreCreativity" validate:"required,min=0,max=100" example:"80.0"`   // Trọng số 20%
	EvaluationNotes   string    `json:"evaluationNotes" validate:"required,min=10" example:"Dự án có kiến trúc microservices tốt, xử lý concurrency mượt mà. Cần bổ sung thêm unit test coverage."`
}

type CapstoneEvaluationResult struct {
	EvaluationID    uuid.UUID `json:"evaluationId" example:"71eebc99-9c0b-4ef8-bb6d-6bb9bd380711"`
	ProjectID       uuid.UUID `json:"projectId" example:"f1eebc99-9c0b-4ef8-bb6d-6bb9bd380f11"`
	FinalScore      float64   `json:"finalScore" example:"88.0"`
	EvaluatedAt     time.Time `json:"evaluatedAt" example:"2026-10-15T15:30:00Z"`
	Passed          bool      `json:"passed" example:"true"`
}
```

---

### Phân Hệ 6: Trình Phát Video Chống Tua & Heartbeat Anti-Cheat (`internal/domain/course_video`)

#### 6.1 `POST /api/v1/courses/{courseId}/lessons/{lessonId}/heartbeat` - Client Heartbeat kiểm soát tua video

```go
// VideoHeartbeat godoc
// @Summary Heartbeat kiểm soát tiến độ xem video và chống tua nhanh bất thường
// @Description Client gửi heartbeat định kỳ mỗi 5 giây khi xem bài giảng YouTube. Server kiểm tra chênh lệch thời gian để chặn gian lận DevTools. Mở khóa tua tự do khi hoàn thành >= 95%
// @Tags Course Video
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param courseId path string true "ID khóa học (UUID)" format(uuid)
// @Param lessonId path string true "ID bài học (UUID)" format(uuid)
// @Param request body dto.VideoHeartbeatRequest true "Thông số phát video từ client"
// @Success 200 {object} response.Envelope{data=dto.VideoHeartbeatResponse} "Cập nhật tiến độ thành công"
// @Failure 400 {object} response.Envelope{error=response.ErrorDetail} "Phát hiện tua lách luật hoặc dữ liệu sai lệch"
// @Router /api/v1/courses/{courseId}/lessons/{lessonId}/heartbeat [post]
func (h *VideoHandler) VideoHeartbeat(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

- **DTO Request & Response Structs:**
```go
type VideoHeartbeatRequest struct {
	CurrentSeconds  float64 `json:"currentSeconds" validate:"required,min=0" example:"145.5"`
	PlaybackRate    float64 `json:"playbackRate" validate:"required,min=0.25,max=2.0" example:"1.0"`
	TotalDuration   float64 `json:"totalDuration" validate:"required,min=1" example:"600.0"`
	ClientTimestamp int64   `json:"clientTimestamp" validate:"required" example:"1727072400"`
}

type VideoHeartbeatResponse struct {
	MaxWatchedSeconds float64 `json:"maxWatchedSeconds" example:"145.5"`
	IsCompleted       bool    `json:"isCompleted" example:"false"`
	AllowFreeSeeking  bool    `json:"allowFreeSeeking" example:"false"`
	ProgressPercent   float64 `json:"progressPercent" example:"24.25"`
}
```

---

### Phân Hệ 7: Cổng Mua Khóa Học & Thanh Toán VietQR (`internal/domain/billing`)

#### 7.1 `POST /api/v1/courses/{id}/enroll` - Đăng ký học ngay với Khóa học Miễn Phí (1-Click)

```go
// EnrollFreeCourse godoc
// @Summary Đăng ký học ngay khóa học trực tuyến Miễn Phí
// @Description Kích hoạt ngay lập tức quyền học bài giảng cho học viên đối với các khóa học miễn phí (isFree = true) mà không cần qua cổng thanh toán
// @Tags Billing & Store
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param id path string true "ID khóa học (UUID)" format(uuid)
// @Success 200 {object} response.Envelope{data=dto.EnrollmentSuccessResponse} "Kích hoạt khóa học thành công"
// @Failure 400 {object} response.Envelope{error=response.ErrorDetail} "Khóa học không phải miễn phí"
// @Router /api/v1/courses/{id}/enroll [post]
func (h *BillingHandler) EnrollFreeCourse(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

#### 7.2 `POST /api/v1/courses/{id}/checkout` - Mua khóa học Trả Phí, Áp Mã Giảm Giá & Sinh VietQR Động

```go
// CheckoutCourse godoc
// @Summary Khởi tạo đơn mua khóa học, áp mã giảm giá và tạo mã thanh toán VietQR động
// @Description Tạo hóa đơn điện tử cho khóa học trả phí. Hỗ trợ áp mã giảm giá (couponCode) kiểm tra tính hợp lệ và tự động tính lại số tiền thanh toán thực tế (finalAmount). Sinh payload mã QR Napas247 chính xác số tiền cùng cú pháp định danh
// @Tags Billing & Store
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param id path string true "ID khóa học (UUID)" format(uuid)
// @Param request body dto.CourseCheckoutRequest false "Thông tin mua hàng kèm mã giảm giá (nếu có)"
// @Success 201 {object} response.Envelope{data=dto.CourseCheckoutResponse} "Tạo đơn mua và mã VietQR thành công"
// @Failure 400 {object} response.Envelope{error=response.ErrorDetail} "Khóa học không hợp lệ hoặc mã giảm giá hết hạn/không đủ điều kiện"
// @Router /api/v1/courses/{id}/checkout [post]
func (h *BillingHandler) CheckoutCourse(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

- **DTO Request & Response Structs:**
```go
type CourseCheckoutRequest struct {
	CouponCode *string `json:"couponCode,omitempty" example:"CHAOHOCVIEN2026"`
}

type CourseCheckoutResponse struct {
	InvoiceID       uuid.UUID `json:"invoiceId" example:"81eebc99-9c0b-4ef8-bb6d-6bb9bd380811"`
	InvoiceCode     string    `json:"invoiceCode" example:"INV-2026-CRS-108"`
	CourseTitle     string    `json:"courseTitle" example:"Lập Trình Web Fullstack Go & Nuxt UI"`
	OriginalAmount  float64   `json:"originalAmount" example:"1500000"`
	DiscountAmount  float64   `json:"discountAmount" example:"300000"`
	FinalAmount     float64   `json:"finalAmount" example:"1200000"`
	CouponCode      *string   `json:"couponCode,omitempty" example:"CHAOHOCVIEN2026"`
	FormattedAmount string    `json:"formattedAmount" example:"1.200.000 đ"`
	BankID          string    `json:"bankId" example:"MB"`
	AccountNo       string    `json:"accountNo" example:"0988888888"`
	AccountName     string    `json:"accountName" example:"TRUNG TAM LMS CENTER"`
	TransferContent string    `json:"transferContent" example:"LMS CRS INV108"`
	VietQRImageURL  string    `json:"vietqrImageUrl" example:"https://img.vietqr.io/image/MB-0988888888-compact2.png?amount=1200000&addInfo=LMS%20CRS%20INV108"`
}
```

#### 7.3 `POST /api/v1/webhooks/payment` - Webhook ngân hàng xác nhận giao dịch & Tự động kích hoạt

```go
// PaymentWebhook godoc
// @Summary Nhận thông báo giao dịch chuyển khoản thành công từ ngân hàng (Webhook)
// @Description Cổng tiếp nhận Webhook ngân hàng (SePay / Casso). Tự động khớp nội dung chuyển khoản và kích hoạt khóa học trong vòng 3 giây
// @Tags Billing & Store
// @Accept json
// @Produce json
// @Param request body dto.PaymentWebhookPayload true "Payload dữ liệu giao dịch ngân hàng"
// @Success 200 {object} response.Envelope{data=dto.WebhookProcessResult} "Xử lý kích hoạt đơn hàng thành công"
// @Failure 400 {object} response.Envelope{error=response.ErrorDetail} "Dữ liệu webhook không hợp lệ"
// @Router /api/v1/webhooks/payment [post]
func (h *BillingHandler) PaymentWebhook(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

---

### Phân Hệ 8: Hệ Thống Tin Nhắn & Cảnh Báo Tự Động (`internal/domain/notification`)

#### 8.1 `POST /api/v1/cron/teacher-late-alerts` - Cảnh báo Giảng viên đi muộn sau 10 phút

```go
// ScanTeacherLateAlerts godoc
// @Summary Quét và gửi cảnh báo khẩn khi Giảng viên đi muộn / chưa vào lớp
// @Description Cron định kỳ kiểm tra các ca học đã bắt đầu quá 10 phút mà Giảng viên chính chưa Check-in. Tự động gửi SMS/Push cho Giáo viên và bắn cảnh báo đỏ cho Quản lý & Vận hành lớp
// @Tags Notification Engine
// @Accept json
// @Produce json
// @Security ApiKeyAuth
// @Success 200 {object} response.Envelope{data=dto.TeacherLateAlertSummary} "Quét và phát cảnh báo thành công"
// @Failure 401 {object} response.Envelope{error=response.ErrorDetail} "Mã bí mật Cron không hợp lệ"
// @Router /api/v1/cron/teacher-late-alerts [post]
func (h *NotificationHandler) ScanTeacherLateAlerts(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

- **DTO Response Struct:**
```go
type TeacherLateAlertSummary struct {
	ScannedSessions    int      `json:"scannedSessions" example:"12"`
	LateTeachersFound  int      `json:"lateTeachersFound" example:"1"`
	AlertedTeacherIDs  []string `json:"alertedTeacherIds" example:"[\"tuan.nguyen@lms.edu.vn\"]"`
	ManagersNotified   int      `json:"managersNotified" example:"2"`
	ExecutedAt         time.Time `json:"executedAt" example:"2026-10-01T19:40:00Z"`
}
```

#### 8.2 `POST /api/v1/sessions/{sessionId}/attendance-alert` - Bắn tin sĩ số vắng/đủ sau 15 phút

```go
// DispatchAttendanceAlert godoc
// @Summary Kích hoạt kiểm tra và phát thông báo sĩ số sau 15 phút vào lớp
// @Description Quét bảng điểm danh buổi học. Nếu lớp vắng, tự động gửi tin nhắn Zalo/SMS cho Phụ huynh học sinh vắng và báo cáo tổng hợp cho Vận hành & GV. Nếu đủ 100%, gửi thông báo chúc mừng
// @Tags Notification Engine
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param sessionId path string true "ID buổi học (UUID)" format(uuid)
// @Success 200 {object} response.Envelope{data=dto.AttendanceAlertDispatchResult} "Phát tin thông báo sĩ số thành công"
// @Failure 404 {object} response.Envelope{error=response.ErrorDetail} "Không tìm thấy buổi học"
// @Router /api/v1/sessions/{sessionId}/attendance-alert [post]
func (h *NotificationHandler) DispatchAttendanceAlert(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

---

### Phân Hệ 9: Bài Tập Về Nhà, Monaco Code Editor & Chấm Điểm (`internal/domain/assignment`)

#### 9.1 `POST /api/v1/submissions` - Học viên nộp mã nguồn Monaco Editor

```go
// SubmitAssignment godoc
// @Summary Học viên nộp bài làm trực tiếp trên web (Monaco Code Editor)
// @Description Nộp mã nguồn lập trình, đường dẫn GitHub hoặc tài liệu đính kèm cho bài tập về nhà được giao
// @Tags Assignments
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param request body dto.SubmitAssignmentRequest true "Dữ liệu bài làm học viên"
// @Success 201 {object} response.Envelope{data=dto.SubmissionResultResponse} "Nộp bài thành công"
// @Failure 400 {object} response.Envelope{error=response.ErrorDetail} "Dữ liệu bài nộp không hợp lệ hoặc quá deadline"
// @Router /api/v1/submissions [post]
func (h *AssignmentHandler) SubmitAssignment(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

- **DTO Request Struct:**
```go
type SubmitAssignmentRequest struct {
	AssignmentID uuid.UUID `json:"assignmentId" validate:"required" example:"91eebc99-9c0b-4ef8-bb6d-6bb9bd380911"`
	CodeContent  string    `json:"codeContent,omitempty" example:"function binarySearch(arr, target) { ... }"`
	GithubURL    string    `json:"githubUrl,omitempty" example:"https://github.com/student/lms-homework-1"`
	FileURL      string    `json:"fileUrl,omitempty" example:"https://storage.lms.edu.vn/submissions/report.pdf"`
}
```

#### 9.2 `GET /api/v1/classes/{classId}/assignments` - Danh sách bài tập của lớp học

```go
// ListClassAssignments godoc
// @Summary Lấy danh sách bài tập được giao cho một lớp học cụ thể
// @Description Giảng viên, Trợ giảng hoặc Học viên của lớp xem toàn bộ các bài tập (homework), hạn nộp và thống kê số lượng bài đã nộp/chưa nộp
// @Tags Assignments
// @Produce json
// @Security BearerAuth
// @Param classId path string true "ID lớp học (UUID)" format(uuid)
// @Success 200 {object} response.Envelope{data=[]dto.ClassAssignmentItem} "Lấy danh sách bài tập thành công"
// @Failure 404 {object} response.Envelope{error=response.ErrorDetail} "Không tìm thấy lớp học"
// @Router /api/v1/classes/{classId}/assignments [get]
func (h *AssignmentHandler) ListClassAssignments(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

#### 9.3 `GET /api/v1/classes/{classId}/assignments/{assignmentId}/submissions` - Danh sách bài nộp của học viên trong lớp

```go
// ListAssignmentSubmissions godoc
// @Summary Giáo viên xem toàn bộ danh sách bài nộp của học viên trong lớp
// @Description Cung cấp cho Giảng viên/Trợ giảng danh sách học viên trong lớp, trạng thái nộp bài, thời điểm nộp, link GitHub, mã nguồn Monaco và điểm số hiện tại
// @Tags Assignments
// @Produce json
// @Security BearerAuth
// @Param classId path string true "ID lớp học (UUID)" format(uuid)
// @Param assignmentId path string true "ID bài tập (UUID)" format(uuid)
// @Success 200 {object} response.Envelope{data=[]dto.StudentSubmissionSummary} "Lấy danh sách bài nộp thành công"
// @Failure 403 {object} response.Envelope{error=response.ErrorDetail} "Không có quyền quản lý lớp này"
// @Router /api/v1/classes/{classId}/assignments/{assignmentId}/submissions [get]
func (h *AssignmentHandler) ListAssignmentSubmissions(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

#### 9.4 `POST /api/v1/classes/{classId}/assignments/{assignmentId}/submissions/{submissionId}/grade` - Giáo viên chấm bài tập học viên

```go
// GradeSubmission godoc
// @Summary Giáo viên chấm điểm và nhận xét chi tiết bài làm của học viên
// @Description Giáo viên chấm bài theo Rubric, nhập điểm số, nhận xét Markdown, link audio dặn dò. Hệ thống tự động đồng bộ kết quả vào Sổ điểm lớp học (Gradebook) và gửi thông báo cho Học viên & Phụ huynh
// @Tags Assignments
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param classId path string true "ID lớp học (UUID)" format(uuid)
// @Param assignmentId path string true "ID bài tập (UUID)" format(uuid)
// @Param submissionId path string true "ID bài nộp của học viên (UUID)" format(uuid)
// @Param request body dto.GradeSubmissionRequest true "Thông tin điểm số và phản hồi chấm bài"
// @Success 200 {object} response.Envelope{data=dto.GradeSubmissionResult} "Chấm bài và lưu điểm thành công"
// @Failure 400 {object} response.Envelope{error=response.ErrorDetail} "Điểm số không hợp lệ (ngoài thang điểm 0-100)"
// @Failure 403 {object} response.Envelope{error=response.ErrorDetail} "Chỉ Giảng viên chính hoặc Trợ giảng của lớp mới được chấm điểm"
// @Router /api/v1/classes/{classId}/assignments/{assignmentId}/submissions/{submissionId}/grade [post]
func (h *AssignmentHandler) GradeSubmission(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

- **DTO Structs Bổ Sung (Assignments & Grading):**
```go
type ClassAssignmentItem struct {
	ID             uuid.UUID `json:"id" example:"91eebc99-9c0b-4ef8-bb6d-6bb9bd380911"`
	ClassID        uuid.UUID `json:"classId" example:"f5eebc99-9c0b-4ef8-bb6d-6bb9bd380f66"`
	Title          string    `json:"title" example:"Bài tập 03: Xây dựng REST API với Go-chi và PostgreSQL"`
	DueAt          time.Time `json:"dueAt" example:"2026-10-20T23:59:59Z"`
	TotalStudents  int       `json:"totalStudents" example:"25"`
	SubmittedCount int       `json:"submittedCount" example:"22"`
	GradedCount    int       `json:"gradedCount" example:"18"`
}

type StudentSubmissionSummary struct {
	SubmissionID uuid.UUID  `json:"submissionId" example:"a1eebc99-9c0b-4ef8-bb6d-6bb9bd380111"`
	StudentID    uuid.UUID  `json:"studentId" example:"b2eebc99-9c0b-4ef8-bb6d-6bb9bd380222"`
	StudentName  string     `json:"studentName" example:"Trần Minh Hoàng"`
	Status       string     `json:"status" example:"GRADED"` // SUBMITTED, GRADED, LATE, MISSING
	CodeContent  *string    `json:"codeContent,omitempty" example:"package main\n\nfunc main() {...}"`
	GithubURL    *string    `json:"githubUrl,omitempty" example:"https://github.com/hoang/hw3"`
	SubmittedAt  time.Time  `json:"submittedAt" example:"2026-10-19T20:15:00Z"`
	Score        *float64   `json:"score,omitempty" example:"95.0"`
	TeacherNotes *string    `json:"teacherNotes,omitempty" example:"Code rất sạch, xử lý transaction cẩn thận"`
}

type GradeSubmissionRequest struct {
	Score        float64 `json:"score" validate:"required,min=0,max=100" example:"95.0"`
	TeacherNotes string  `json:"teacherNotes" validate:"required,min=5" example:"Thuật toán tối ưu, đã xử lý hết edge cases. Tiếp tục phát huy!"`
	AudioURL     *string `json:"audioUrl,omitempty" example:"https://storage.lms.edu.vn/feedback/audio-grade-95.mp3"`
}

type GradeSubmissionResult struct {
	SubmissionID uuid.UUID `json:"submissionId" example:"a1eebc99-9c0b-4ef8-bb6d-6bb9bd380111"`
	Score        float64   `json:"score" example:"95.0"`
	GradedAt     time.Time `json:"gradedAt" example:"2026-10-21T09:30:00Z"`
	GradedBy     uuid.UUID `json:"gradedBy" example:"c2eebc99-9c0b-4ef8-bb6d-6bb9bd380c33"`
}
```

---

### Phân Hệ 10: Chứng Chỉ Số Có URL Xác Thực Công Khai (`internal/domain/certificate`)

#### 10.1 `GET /api/v1/verify/{certificateCode}` - Xác thực chứng chỉ công khai

```go
// VerifyCertificate godoc
// @Summary Xác thực chứng chỉ tốt nghiệp công khai
// @Description API công khai (không cần đăng nhập) cho phép đối tác, nhà tuyển dụng kiểm tra tính chính danh của chứng chỉ số được cấp bởi trung tâm
// @Tags Certificate Verification
// @Produce json
// @Param certificateCode path string true "Mã chứng chỉ duy nhất" example("CERT-2026-REACT-8F29")
// @Success 200 {object} response.Envelope{data=dto.PublicCertificateResponse} "Chứng chỉ hợp lệ"
// @Failure 404 {object} response.Envelope{error=response.ErrorDetail} "Mã chứng chỉ không tồn tại hoặc đã bị thu hồi"
// @Router /api/v1/verify/{certificateCode} [get]
func (h *CertificateHandler) VerifyCertificate(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

---

### Phân Hệ 11: Quản Lý Môn Học Động (`internal/domain/subject`)

#### 11.1 `GET /api/v1/subjects` - Lấy danh sách môn học động

```go
// ListSubjects godoc
// @Summary Lấy danh sách môn học động trong toàn trung tâm
// @Description Hỗ trợ tìm kiếm theo từ khóa tên môn học, mã môn học và lọc theo trạng thái hoạt động (isActive)
// @Tags Subjects
// @Produce json
// @Param search query string false "Từ khóa tìm kiếm (tên hoặc mã môn)" example("Frontend")
// @Param isActive query bool false "Lọc trạng thái hoạt động" example(true)
// @Success 200 {object} response.Envelope{data=[]dto.SubjectResponse} "Lấy danh sách môn học thành công"
// @Router /api/v1/subjects [get]
func (h *SubjectHandler) ListSubjects(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

#### 11.2 `POST /api/v1/subjects` - Tạo môn học mới

```go
// CreateSubject godoc
// @Summary Tạo mới môn học trong chương trình đào tạo
// @Description Admin hoặc Academic Manager tạo môn học mới (ví dụ: Lập trình Web Frontend, Khoa học Dữ liệu, v.v.)
// @Tags Subjects
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param request body dto.CreateSubjectRequest true "Dữ liệu môn học mới"
// @Success 201 {object} response.Envelope{data=dto.SubjectResponse} "Tạo môn học thành công"
// @Failure 400 {object} response.Envelope{error=response.ErrorDetail} "Dữ liệu không hợp lệ hoặc mã môn học đã tồn tại"
// @Failure 403 {object} response.Envelope{error=response.ErrorDetail} "Không đủ quyền hạn"
// @Router /api/v1/subjects [post]
func (h *SubjectHandler) CreateSubject(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

#### 11.3 `PUT /api/v1/subjects/{id}` - Cập nhật thông tin môn học

```go
// UpdateSubject godoc
// @Summary Cập nhật thông tin môn học
// @Description Chỉnh sửa tên, mã môn, biểu tượng (icon) hoặc trạng thái kích hoạt của môn học
// @Tags Subjects
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param id path string true "ID môn học (UUID)" format(uuid)
// @Param request body dto.UpdateSubjectRequest true "Thông tin cập nhật môn học"
// @Success 200 {object} response.Envelope{data=dto.SubjectResponse} "Cập nhật môn học thành công"
// @Failure 400 {object} response.Envelope{error=response.ErrorDetail} "Dữ liệu không hợp lệ"
// @Failure 404 {object} response.Envelope{error=response.ErrorDetail} "Không tìm thấy môn học"
// @Router /api/v1/subjects/{id} [put]
func (h *SubjectHandler) UpdateSubject(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

#### 11.4 `DELETE /api/v1/subjects/{id}` - Xóa mềm môn học (Soft Delete)

```go
// DeleteSubject godoc
// @Summary Xóa mềm môn học (Soft Delete)
// @Description Thực hiện cập nhật deleted_at = NOW() và deleted_by = current_user_id. Nghiêm cấm xóa vật lý (Hard Delete) để bảo toàn tính toàn vẹn khóa ngoại của các khóa học đã mở
// @Tags Subjects
// @Produce json
// @Security BearerAuth
// @Param id path string true "ID môn học (UUID)" format(uuid)
// @Success 200 {object} response.Envelope{message=string} "Xóa mềm môn học thành công"
// @Failure 400 {object} response.Envelope{error=response.ErrorDetail} "Môn học đang có các khóa học hoạt động, không thể xóa"
// @Failure 404 {object} response.Envelope{error=response.ErrorDetail} "Không tìm thấy môn học"
// @Router /api/v1/subjects/{id} [delete]
func (h *SubjectHandler) DeleteSubject(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

- **DTO Request & Response Structs (Subjects):**
```go
type CreateSubjectRequest struct {
	Code        string  `json:"code" validate:"required,min=2,max=30" example:"WEB_FE"`
	Name        string  `json:"name" validate:"required,min=3,max=100" example:"Lập Trình Web Frontend"`
	Description *string `json:"description,omitempty" example:"Chuyên ngành đào tạo phát triển giao diện Web với Vue/Nuxt & React"`
	IconURL     *string `json:"iconUrl,omitempty" example:"https://storage.lms.edu.vn/icons/frontend.svg"`
	IsActive    bool    `json:"isActive" example:"true"`
}

type UpdateSubjectRequest struct {
	Name        string  `json:"name" validate:"required,min=3,max=100" example:"Lập Trình Web Frontend Hiện Đại"`
	Description *string `json:"description,omitempty" example:"Cập nhật giáo trình 2026"`
	IconURL     *string `json:"iconUrl,omitempty" example:"https://storage.lms.edu.vn/icons/frontend-v2.svg"`
	IsActive    bool    `json:"isActive" example:"true"`
}

type SubjectResponse struct {
	ID          uuid.UUID `json:"id" example:"e1eebc99-9c0b-4ef8-bb6d-6bb9bd380123"`
	Code        string    `json:"code" example:"WEB_FE"`
	Name        string    `json:"name" example:"Lập Trình Web Frontend"`
	Description *string   `json:"description,omitempty" example:"Chuyên ngành đào tạo phát triển giao diện"`
	IconURL     *string   `json:"iconUrl,omitempty" example:"https://storage.lms.edu.vn/icons/frontend.svg"`
	IsActive    bool      `json:"isActive" example:"true"`
	CreatedAt   time.Time `json:"createdAt" example:"2026-09-01T08:00:00Z"`
}
```

---

### Phân Hệ 12: Quản Lý Khuyến Mãi & Voucher Giảm Giá (`internal/domain/discount`)

#### 12.1 `GET /api/v1/discounts` - Danh sách mã giảm giá

```go
// ListDiscounts godoc
// @Summary Danh sách các mã khuyến mãi trong hệ thống
// @Description Quản trị viên và Marketing xem danh sách các mã giảm giá, số lượt đã sử dụng, hạn dùng và trạng thái kích hoạt
// @Tags Discounts
// @Produce json
// @Security BearerAuth
// @Param isActive query bool false "Lọc mã đang hoạt động" example(true)
// @Success 200 {object} response.Envelope{data=[]dto.DiscountItemResponse} "Lấy danh sách mã giảm giá thành công"
// @Router /api/v1/discounts [get]
func (h *DiscountHandler) ListDiscounts(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

#### 12.2 `POST /api/v1/discounts` - Tạo mã giảm giá mới

```go
// CreateDiscount godoc
// @Summary Khởi tạo mã khuyến mãi mới
// @Description Hỗ trợ giảm theo phần trăm (PERCENT) hoặc số tiền cố định (FIXED), thiết lập đơn hàng tối thiểu, mức giảm tối đa, giới hạn số lượt và thời gian áp dụng
// @Tags Discounts
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param request body dto.CreateDiscountRequest true "Thông tin mã khuyến mãi"
// @Success 201 {object} response.Envelope{data=dto.DiscountItemResponse} "Tạo mã giảm giá thành công"
// @Failure 400 {object} response.Envelope{error=response.ErrorDetail} "Mã khuyến mãi bị trùng lặp hoặc cấu hình không hợp lệ"
// @Router /api/v1/discounts [post]
func (h *DiscountHandler) CreateDiscount(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

#### 12.3 `POST /api/v1/discounts/validate` - Xác thực mã giảm giá trước thanh toán

```go
// ValidateDiscount godoc
// @Summary Kiểm tra tính hợp lệ của mã khuyến mãi trước khi tạo hóa đơn
// @Description Kiểm tra mã có tồn tại, còn hạn, còn lượt sử dụng và đáp ứng giá trị đơn hàng tối thiểu không. Trả về số tiền được khấu trừ dự kiến
// @Tags Discounts
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param request body dto.ValidateDiscountRequest true "Mã coupon và giá trị đơn hàng"
// @Success 200 {object} response.Envelope{data=dto.ValidateDiscountResponse} "Mã giảm giá hợp lệ"
// @Failure 400 {object} response.Envelope{error=response.ErrorDetail} "Mã không hợp lệ, hết lượt hoặc đơn hàng không đủ điều kiện"
// @Router /api/v1/discounts/validate [post]
func (h *DiscountHandler) ValidateDiscount(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

- **DTO Request & Response Structs (Discounts):**
```go
type CreateDiscountRequest struct {
	Code              string     `json:"code" validate:"required,min=3,max=30" example:"KHAIGIANG2026"`
	Description       *string    `json:"description,omitempty" example:"Giảm giá 20% nhân dịp khai giảng quý 4"`
	DiscountType      string     `json:"discountType" validate:"required,oneof=PERCENT FIXED" example:"PERCENT"`
	Value             float64    `json:"value" validate:"required,min=1" example:"20.0"`
	MaxDiscountAmount *float64   `json:"maxDiscountAmount,omitempty" example:"500000"`
	MinOrderAmount    float64    `json:"minOrderAmount" example:"1000000"`
	UsageLimit        *int       `json:"usageLimit,omitempty" example:"100"`
	ValidFrom         time.Time  `json:"validFrom" validate:"required" example:"2026-10-01T00:00:00Z"`
	ValidUntil        time.Time  `json:"validUntil" validate:"required" example:"2026-10-31T23:59:59Z"`
	IsActive          bool       `json:"isActive" example:"true"`
	CourseIDs         []uuid.UUID `json:"courseIds,omitempty"` // Rỗng = Áp dụng toàn bộ khóa học
}

type ValidateDiscountRequest struct {
	CouponCode  string     `json:"couponCode" validate:"required" example:"KHAIGIANG2026"`
	CourseID    uuid.UUID  `json:"courseId" validate:"required"`
	OrderAmount float64    `json:"orderAmount" validate:"required,min=0" example:"2000000"`
}

type ValidateDiscountResponse struct {
	Valid          bool    `json:"valid" example:"true"`
	DiscountAmount float64 `json:"discountAmount" example:"400000"`
	FinalAmount    float64 `json:"finalAmount" example:"1600000"`
	Message        string  `json:"message" example:"Mã giảm giá 20% áp dụng thành công"`
}

type DiscountItemResponse struct {
	ID                uuid.UUID  `json:"id" example:"d1eebc99-9c0b-4ef8-bb6d-6bb9bd380456"`
	Code              string     `json:"code" example:"KHAIGIANG2026"`
	DiscountType      string     `json:"discountType" example:"PERCENT"`
	Value             float64    `json:"value" example:"20.0"`
	MaxDiscountAmount *float64   `json:"maxDiscountAmount,omitempty" example:"500000"`
	MinOrderAmount    float64    `json:"minOrderAmount" example:"1000000"`
	UsageLimit        *int       `json:"usageLimit,omitempty" example:"100"`
	UsedCount         int        `json:"usedCount" example:"34"`
	ValidUntil        time.Time  `json:"validUntil" example:"2026-10-31T23:59:59Z"`
	IsActive          bool       `json:"isActive" example:"true"`
}
```

---

### Phân Hệ 13: Bảng Lương Giáo Viên & Bậc Lương Giảng Dạy (`internal/domain/payroll`)

#### 13.1 `GET /api/v1/salary-grades` - Danh sách bậc lương giảng viên

```go
// ListSalaryGrades godoc
// @Summary Lấy danh sách định mức bậc lương giảng viên
// @Description Quản trị viên xem cấu hình mức thù lao theo giờ, hệ số làm ngoài giờ và tỷ lệ thưởng KPI của từng cấp bậc (Intern/TA, Standard, Senior, Master)
// @Tags Payroll
// @Produce json
// @Security BearerAuth
// @Success 200 {object} response.Envelope{data=[]dto.SalaryGradeResponse} "Lấy danh sách bậc lương thành công"
// @Router /api/v1/salary-grades [get]
func (h *PayrollHandler) ListSalaryGrades(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

#### 13.2 `POST /api/v1/salary-grades` - Cấu hình định mức bậc lương

```go
// UpsertSalaryGrade godoc
// @Summary Tạo hoặc cập nhật mức thù lao của bậc lương
// @Description Cấu hình mức thù lao cơ bản theo giờ (baseHourlyRate), hệ số overtime và tỷ lệ thưởng hoàn thành KPI
// @Tags Payroll
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param request body dto.UpsertSalaryGradeRequest true "Cấu hình định mức bậc lương"
// @Success 200 {object} response.Envelope{data=dto.SalaryGradeResponse} "Cấu hình bậc lương thành công"
// @Failure 400 {object} response.Envelope{error=response.ErrorDetail} "Dữ liệu định mức không hợp lệ"
// @Router /api/v1/salary-grades [post]
func (h *PayrollHandler) UpsertSalaryGrade(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

#### 13.3 `POST /api/v1/payrolls/generate` - Tự động tổng hợp bảng lương tháng

```go
// GenerateMonthlyPayroll godoc
// @Summary Tự động tổng hợp bảng lương giáo viên theo kỳ tháng
// @Description Tổng hợp toàn bộ số giờ dạy thực tế từ các buổi học hoàn thành, tính lương cơ sở theo bậc, cộng thưởng KPI nếu điểm đánh giá của học sinh >= 4.5, và trừ tiền phạt đi muộn (dựa trên late_minutes)
// @Tags Payroll
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param request body dto.GeneratePayrollRequest true "Thông số kỳ tổng hợp lương"
// @Success 201 {object} response.Envelope{data=dto.GeneratePayrollResult} "Tổng hợp bảng lương thành công"
// @Failure 400 {object} response.Envelope{error=response.ErrorDetail} "Định dạng kỳ lương sai (yêu cầu YYYY-MM)"
// @Router /api/v1/payrolls/generate [post]
func (h *PayrollHandler) GenerateMonthlyPayroll(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

#### 13.4 `GET /api/v1/payrolls` - Danh sách bảng lương giáo viên

```go
// ListPayrolls godoc
// @Summary Danh sách phiếu lương giáo viên theo kỳ
// @Description Admin hoặc Kế toán lọc bảng lương theo tháng (period: "2026-10"), trạng thái (DRAFT, APPROVED, PAID) và theo giáo viên
// @Tags Payroll
// @Produce json
// @Security BearerAuth
// @Param period query string false "Kỳ tính lương (YYYY-MM)" example("2026-10")
// @Param status query string false "Trạng thái bảng lương" example("DRAFT")
// @Param teacherId query string false "ID giáo viên (UUID)" format(uuid)
// @Success 200 {object} response.Envelope{data=[]dto.TeacherPayrollSummary} "Lấy danh sách bảng lương thành công"
// @Router /api/v1/payrolls [get]
func (h *PayrollHandler) ListPayrolls(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

#### 13.5 `GET /api/v1/payrolls/{id}` - Xem chi tiết phiếu lương giáo viên

```go
// GetPayrollDetail godoc
// @Summary Xem chi tiết phiếu lương cá nhân kèm danh sách buổi dạy
// @Description Giáo viên xem chi tiết thu nhập của mình trong tháng hoặc Admin kiểm tra chi tiết các mục giờ dạy, thưởng KPI, phạt đi muộn
// @Tags Payroll
// @Produce json
// @Security BearerAuth
// @Param id path string true "ID phiếu lương (UUID)" format(uuid)
// @Success 200 {object} response.Envelope{data=dto.TeacherPayrollDetailResponse} "Lấy chi tiết phiếu lương thành công"
// @Failure 404 {object} response.Envelope{error=response.ErrorDetail} "Không tìm thấy phiếu lương"
// @Router /api/v1/payrolls/{id} [get]
func (h *PayrollHandler) GetPayrollDetail(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

#### 13.6 `POST /api/v1/payrolls/{id}/approve` - Phê duyệt bảng lương

```go
// ApprovePayroll godoc
// @Summary Quản trị viên duyệt quyết toán phiếu lương
// @Description Chuyển trạng thái phiếu lương từ DRAFT sang APPROVED, khóa dữ liệu chấm công kỳ đó và sẵn sàng chi trả ngân hàng
// @Tags Payroll
// @Produce json
// @Security BearerAuth
// @Param id path string true "ID phiếu lương (UUID)" format(uuid)
// @Success 200 {object} response.Envelope{message=string} "Phê duyệt bảng lương thành công"
// @Failure 400 {object} response.Envelope{error=response.ErrorDetail} "Phiếu lương đã được duyệt hoặc không ở trạng thái DRAFT"
// @Router /api/v1/payrolls/{id}/approve [post]
func (h *PayrollHandler) ApprovePayroll(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

- **DTO Request & Response Structs (Payroll):**
```go
type SalaryGradeResponse struct {
	ID                 uuid.UUID `json:"id" example:"a1eebc99-9c0b-4ef8-bb6d-6bb9bd380789"`
	GradeName          string    `json:"gradeName" example:"SENIOR_TEACHER"`
	DisplayName        string    `json:"displayName" example:"Giảng viên Cao cấp (Senior)"`
	BaseHourlyRate     float64   `json:"baseHourlyRate" example:"350000"`
	OvertimeMultiplier float64   `json:"overtimeMultiplier" example:"1.5"`
	KpiBonusRate       float64   `json:"kpiBonusRate" example:"0.15"`
}

type UpsertSalaryGradeRequest struct {
	GradeName          string  `json:"gradeName" validate:"required" example:"SENIOR_TEACHER"`
	DisplayName        string  `json:"displayName" validate:"required" example:"Giảng viên Cao cấp (Senior)"`
	BaseHourlyRate     float64 `json:"baseHourlyRate" validate:"required,min=50000" example:"350000"`
	OvertimeMultiplier float64 `json:"overtimeMultiplier" validate:"required,min=1.0" example:"1.5"`
	KpiBonusRate       float64 `json:"kpiBonusRate" validate:"min=0,max=1.0" example:"0.15"`
}

type GeneratePayrollRequest struct {
	Period string `json:"period" validate:"required" example:"2026-10"`
}

type GeneratePayrollResult struct {
	Period           string  `json:"period" example:"2026-10"`
	GeneratedRecords int     `json:"generatedRecords" example:"18"`
	TotalGrossAmount float64 `json:"totalGrossAmount" example:"158400000"`
}

type TeacherPayrollSummary struct {
	ID                 uuid.UUID `json:"id" example:"f1eebc99-9c0b-4ef8-bb6d-6bb9bd380999"`
	TeacherID          uuid.UUID `json:"teacherId" example:"c2eebc99-9c0b-4ef8-bb6d-6bb9bd380c33"`
	TeacherName        string    `json:"teacherName" example:"ThS. Nguyễn Văn Tuấn"`
	SalaryGradeName    string    `json:"salaryGradeName" example:"Senior"`
	Period             string    `json:"period" example:"2026-10"`
	TotalTeachingHours float64   `json:"totalTeachingHours" example:"42.5"`
	BaseSalaryAmount   float64   `json:"baseSalaryAmount" example:"14875000"`
	KpiBonusAmount     float64   `json:"kpiBonusAmount" example:"2231250"`
	LatePenaltyAmount  float64   `json:"latePenaltyAmount" example:"100000"`
	NetSalaryAmount    float64   `json:"netSalaryAmount" example:"17006250"`
	Status             string    `json:"status" example:"DRAFT"` // DRAFT, APPROVED, PAID
}

type TeacherPayrollDetailResponse struct {
	Summary TeacherPayrollSummary      `json:"summary"`
	Items   []TeacherPayrollItemDetail `json:"items"`
}

type TeacherPayrollItemDetail struct {
	SessionID   uuid.UUID `json:"sessionId" example:"a1eebc99-9c0b-4ef8-bb6d-6bb9bd380a11"`
	SessionDate string    `json:"sessionDate" example:"2026-10-05"`
	ClassName   string    `json:"className" example:"LMS-FE-K32"`
	HoursWorked float64   `json:"hoursWorked" example:"2.0"`
	HourlyRate  float64   `json:"hourlyRate" example:"350000"`
	Amount      float64   `json:"amount" example:"700000"`
	LateMinutes int       `json:"lateMinutes" example:"0"`
	RatingScore *float64  `json:"ratingScore,omitempty" example:"4.8"`
}
```

---

### Phân Hệ 14: Quản Trị Hệ Thống, Nhận Diện Thương Hiệu Website & Kiểm Toán (`internal/domain/system` & `admin`)

#### 14.1 `GET /api/v1/settings/website` - Lấy cấu hình nhận diện thương hiệu & giao diện website (Public)

```go
// GetWebsiteSettings godoc
// @Summary Lấy cấu hình nhận diện thương hiệu công khai của website
// @Description API công khai (không cần login) cho Client Nuxt UI gọi lúc khởi tạo để hiển thị Logo, Favicon, Tiêu đề trang, Màu Header/Footer, Hotline và thông tin liên hệ
// @Tags System Settings
// @Produce json
// @Success 200 {object} response.Envelope{data=dto.WebsiteSettingsResponse} "Lấy cấu hình website thành công"
// @Router /api/v1/settings/website [get]
func (h *SystemHandler) GetWebsiteSettings(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

#### 14.2 `PUT /api/v1/admin/settings/website` - Super Admin tùy biến giao diện & thương hiệu website

```go
// UpdateWebsiteSettings godoc
// @Summary Super Admin tùy biến thương hiệu website (Logo, Favicon, Màu sắc Header/Footer)
// @Description Cho phép Super Admin thay đổi nhận diện thương hiệu toàn diện: Tên website, Slogan, Favicon, Logo sáng/tối, Banner, Màu nền & chữ Header/Footer, Màu nhấn (primary color), Hotline và SEO
// @Tags System Settings
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param request body dto.UpdateWebsiteSettingsRequest true "Cấu hình thương hiệu mới"
// @Success 200 {object} response.Envelope{data=dto.WebsiteSettingsResponse} "Cập nhật cấu hình website thành công"
// @Failure 400 {object} response.Envelope{error=response.ErrorDetail} "Dữ liệu cấu hình không hợp lệ"
// @Failure 403 {object} response.Envelope{error=response.ErrorDetail} "Chỉ SUPER_ADMIN mới có quyền cập nhật"
// @Router /api/v1/admin/settings/website [put]
func (h *SystemHandler) UpdateWebsiteSettings(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

#### 14.3 `GET /api/v1/admin/users` - Super Admin quản lý danh sách người dùng toàn trung tâm

```go
// ListAdminUsers godoc
// @Summary Danh sách người dùng toàn trung tâm (kèm bộ lọc đa chiều)
// @Description Quản trị viên tra cứu người dùng theo từ khóa (Tên, Email, SĐT, Mã HV), lọc theo Vai trò (Role) và Trạng thái kích hoạt (isActive)
// @Tags User Management
// @Produce json
// @Security BearerAuth
// @Param search query string false "Từ khóa tìm kiếm" example("Nguyễn Văn")
// @Param role query string false "Mã vai trò" example("TEACHER")
// @Param isActive query bool false "Lọc trạng thái hoạt động" example(true)
// @Param page query int false "Trang hiện tại" example(1)
// @Param pageSize query int false "Kích thước trang" example(20)
// @Success 200 {object} response.Envelope{data=[]dto.AdminUserItemResponse} "Lấy danh sách người dùng thành công"
// @Failure 403 {object} response.Envelope{error=response.ErrorDetail} "Không đủ quyền hạn"
// @Router /api/v1/admin/users [get]
func (h *UserHandler) ListAdminUsers(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

#### 14.4 `POST /api/v1/admin/users` - Super Admin tạo tài khoản nhân sự mới

```go
// CreateAdminUser godoc
// @Summary Tạo mới tài khoản nhân sự (Giảng viên, Trợ giảng, Giám khảo, Vận hành)
// @Description Khởi tạo tài khoản người dùng và gán vai trò ban đầu cùng thông tin hồ sơ chuyên ngành
// @Tags User Management
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param request body dto.CreateAdminUserRequest true "Thông tin tài khoản mới"
// @Success 201 {object} response.Envelope{data=dto.AdminUserItemResponse} "Tạo người dùng thành công"
// @Failure 400 {object} response.Envelope{error=response.ErrorDetail} "Email đã tồn tại hoặc mật khẩu không đủ mạnh"
// @Failure 403 {object} response.Envelope{error=response.ErrorDetail} "Không đủ quyền hạn"
// @Router /api/v1/admin/users [post]
func (h *UserHandler) CreateAdminUser(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

#### 14.5 `PUT /api/v1/admin/users/{id}/roles` - Super Admin gán/thu hồi vai trò

```go
// UpdateUserRoles godoc
// @Summary Gán hoặc thu hồi vai trò của người dùng (Multi-Role)
// @Description Cập nhật danh sách các vai trò (roles) được cấp cho một người dùng cụ thể
// @Tags User Management
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param id path string true "ID người dùng (UUID)" format(uuid)
// @Param request body dto.UpdateUserRolesRequest true "Danh sách mã vai trò mới"
// @Success 200 {object} response.Envelope{message=string} "Cập nhật vai trò thành công"
// @Failure 404 {object} response.Envelope{error=response.ErrorDetail} "Không tìm thấy người dùng"
// @Router /api/v1/admin/users/{id}/roles [put]
func (h *UserHandler) UpdateUserRoles(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

#### 14.6 `PUT /api/v1/admin/users/{id}/status` - Super Admin khóa hoặc mở khóa tài khoản

```go
// ToggleUserStatus godoc
// @Summary Kích hoạt hoặc Vô hiệu hóa tài khoản người dùng
// @Description Cho phép Super Admin vô hiệu hóa ngay quyền đăng nhập của nhân sự nghỉ việc hoặc học sinh vi phạm quy chế
// @Tags User Management
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param id path string true "ID người dùng (UUID)" format(uuid)
// @Param request body dto.ToggleUserStatusRequest true "Trạng thái kích hoạt mới"
// @Success 200 {object} response.Envelope{message=string} "Cập nhật trạng thái người dùng thành công"
// @Router /api/v1/admin/users/{id}/status [put]
func (h *UserHandler) ToggleUserStatus(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

#### 14.7 `GET /api/v1/admin/audit-logs` - Super Admin xem nhật ký kiểm toán toàn hệ thống

```go
// ListAuditLogs godoc
// @Summary Tra cứu nhật ký kiểm toán hệ thống (Audit Logs)
// @Description Cho phép Super Admin kiểm tra lịch sử các hành động nhạy cảm: Thay đổi điểm số, Duyệt lương, Dời lịch học, Cấp quyền, Đổi cấu hình
// @Tags System Audit
// @Produce json
// @Security BearerAuth
// @Param action query string false "Lọc theo hành động" example("PAYROLL_APPROVE")
// @Param resourceType query string false "Loại tài nguyên" example("PAYROLL")
// @Param page query int false "Trang hiện tại" example(1)
// @Param pageSize query int false "Kích thước trang" example(50)
// @Success 200 {object} response.Envelope{data=[]dto.AuditLogItemResponse} "Lấy nhật ký kiểm toán thành công"
// @Failure 403 {object} response.Envelope{error=response.ErrorDetail} "Chỉ SUPER_ADMIN mới có quyền tra cứu"
// @Router /api/v1/admin/audit-logs [get]
func (h *SystemHandler) ListAuditLogs(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

- **DTO Request & Response Structs (System, Website & User Management):**
```go
type WebsiteSettingsResponse struct {
	SiteTitle       string            `json:"siteTitle" example:"LMS Center - Nền Tảng Đào Tạo Công Nghệ"`
	SiteTagline     string            `json:"siteTagline" example:"Học thực chiến, việc làm ngay"`
	LogoURL         string            `json:"logoUrl" example:"https://storage.lms.edu.vn/brand/logo.svg"`
	LogoDarkURL     string            `json:"logoDarkUrl" example:"https://storage.lms.edu.vn/brand/logo-dark.svg"`
	FaviconURL      string            `json:"faviconUrl" example:"https://storage.lms.edu.vn/brand/favicon.ico"`
	HeaderBgColor   string            `json:"headerBgColor" example:"#0f172a"`
	HeaderTextColor string            `json:"headerTextColor" example:"#f8fafc"`
	FooterBgColor   string            `json:"footerBgColor" example:"#0f172a"`
	FooterTextColor string            `json:"footerTextColor" example:"#94a3b8"`
	FooterCopyright string            `json:"footerCopyright" example:"© 2026 LMS Center Platform. All rights reserved."`
	PrimaryColor    string            `json:"primaryColor" example:"#0284c7"`
	BannerURL       string            `json:"bannerUrl" example:"https://storage.lms.edu.vn/brand/hero-banner.webp"`
	Hotline         string            `json:"hotline" example:"1900 6868"`
	ContactEmail    string            `json:"contactEmail" example:"contact@lms.edu.vn"`
	Address         string            `json:"address" example:"Tòa nhà Công nghệ, Cầu Giấy, Hà Nội"`
	SocialLinks     map[string]string `json:"socialLinks"`
}

type UpdateWebsiteSettingsRequest struct {
	SiteTitle       string            `json:"siteTitle" validate:"required,min=2" example:"LMS Center Platform"`
	SiteTagline     *string           `json:"siteTagline,omitempty" example:"Nền tảng đào tạo lập trình thực chiến"`
	LogoURL         *string           `json:"logoUrl,omitempty" example:"https://storage.lms.edu.vn/brand/logo.svg"`
	LogoDarkURL     *string           `json:"logoDarkUrl,omitempty" example:"https://storage.lms.edu.vn/brand/logo-dark.svg"`
	FaviconURL      *string           `json:"faviconUrl,omitempty" example:"https://storage.lms.edu.vn/brand/favicon.ico"`
	HeaderBgColor   string            `json:"headerBgColor" validate:"required" example:"#0f172a"`
	HeaderTextColor string            `json:"headerTextColor" validate:"required" example:"#f8fafc"`
	FooterBgColor   string            `json:"footerBgColor" validate:"required" example:"#0f172a"`
	FooterTextColor string            `json:"footerTextColor" validate:"required" example:"#94a3b8"`
	FooterCopyright *string           `json:"footerCopyright,omitempty" example:"© 2026 LMS Center. Bản quyền thuộc về Trung tâm Đào tạo."`
	PrimaryColor    string            `json:"primaryColor" validate:"required" example:"#0284c7"`
	BannerURL       *string           `json:"bannerUrl,omitempty"`
	Hotline         *string           `json:"hotline,omitempty" example:"1900 6868"`
	ContactEmail    *string           `json:"contactEmail,omitempty" example:"support@lms.edu.vn"`
	Address         *string           `json:"address,omitempty"`
	SocialLinks     map[string]string `json:"socialLinks,omitempty"`
}

type AdminUserItemResponse struct {
	ID        uuid.UUID `json:"id" example:"a1eebc99-9c0b-4ef8-bb6d-6bb9bd380123"`
	Email     string    `json:"email" example:"nam.le@lms.edu.vn"`
	FullName  string    `json:"fullName" example:"Lê Hoàng Nam"`
	Phone     *string   `json:"phone,omitempty" example:"0988888888"`
	AvatarURL *string   `json:"avatarUrl,omitempty"`
	IsActive  bool      `json:"isActive" example:"true"`
	Roles     []string  `json:"roles" example:"[\"TEACHER\"]"`
	CreatedAt time.Time `json:"createdAt" example:"2026-09-01T08:00:00Z"`
}

type CreateAdminUserRequest struct {
	Email    string   `json:"email" validate:"required,email" example:"giangvien.moi@lms.edu.vn"`
	Password string   `json:"password" validate:"required,min=8" example:"SecurePassword@2026"`
	FullName string   `json:"fullName" validate:"required,min=2" example:"Trần Văn B"`
	Phone    *string  `json:"phone,omitempty" example:"0977777777"`
	RoleCodes []string `json:"roleCodes" validate:"required,min=1" example:"[\"TEACHER\"]"`
}

type UpdateUserRolesRequest struct {
	RoleCodes []string `json:"roleCodes" validate:"required,min=1" example:"[\"TEACHER\", \"ACADEMIC_MANAGER\"]"`
}

type ToggleUserStatusRequest struct {
	IsActive bool   `json:"isActive" example:"false"`
	Reason   string `json:"reason" validate:"required,min=5" example:"Nhân sự đã thanh lý hợp đồng lao động"`
}

type AuditLogItemResponse struct {
	ID           uuid.UUID `json:"id" example:"e1eebc99-9c0b-4ef8-bb6d-6bb9bd380456"`
	ActorID      uuid.UUID `json:"actorId" example:"c2eebc99-9c0b-4ef8-bb6d-6bb9bd380c33"`
	ActorEmail   string    `json:"actorEmail" example:"admin@lms.edu.vn"`
	Action       string    `json:"action" example:"PAYROLL_APPROVE"`
	ResourceType string    `json:"resourceType" example:"PAYROLL"`
	ResourceID   string    `json:"resourceId" example:"f1eebc99-9c0b-4ef8-bb6d-6bb9bd380999"`
	DiffJSON     *string   `json:"diffJson,omitempty"`
	IPAddress    *string   `json:"ipAddress,omitempty" example:"14.226.24.12"`
	CreatedAt    time.Time `json:"createdAt" example:"2026-10-25T14:30:00Z"`
}
```

---

### Phân Hệ 13: Quản Lý Cơ Sở, Phòng Học & Chống Trùng Lịch (`internal/domain/campus`)

#### 13.1 `GET /api/v1/campuses` - Danh sách chi nhánh cơ sở đào tạo

```go
package handler

import (
	"net/http"
	"backend/internal/domain/campus/dto"
	"backend/pkg/response"
)

// ListCampuses godoc
// @Summary Danh sách chi nhánh cơ sở đào tạo
// @Description Lấy toàn bộ danh sách các cơ sở / chi nhánh đang hoạt động của trung tâm
// @Tags Campus & Room
// @Accept json
// @Produce json
// @Success 200 {object} response.Envelope{data=[]dto.CampusResponse} "Lấy danh sách thành công"
// @Failure 500 {object} response.Envelope "Lỗi nội bộ máy chủ"
// @Router /api/v1/campuses [get]
func (h *CampusHandler) ListCampuses(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.ListCampuses
}
```

#### 13.2 `POST /api/v1/campuses` - Tạo mới cơ sở chi nhánh

```go
// CreateCampus godoc
// @Summary Tạo mới cơ sở đào tạo
// @Description Thêm một chi nhánh trung tâm mới với thông tin địa chỉ và hotline
// @Tags Campus & Room
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param request body dto.CreateCampusRequest true "Thông tin cơ sở đào tạo"
// @Success 201 {object} response.Envelope{data=dto.CampusResponse} "Tạo cơ sở thành công"
// @Failure 400 {object} response.Envelope "Dữ liệu không hợp lệ hoặc mã cơ sở đã tồn tại"
// @Failure 401 {object} response.Envelope "Chưa xác thực danh tính"
// @Failure 403 {object} response.Envelope "Không có quyền quản trị cơ sở"
// @Router /api/v1/campuses [post]
func (h *CampusHandler) CreateCampus(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.CreateCampus
}
```

#### 13.3 `GET /api/v1/rooms/conflicts` - Kiểm tra xung đột phòng học (Room Conflict Guard)

```go
// CheckRoomConflicts godoc
// @Summary Kiểm tra xung đột lịch phòng học
// @Description Kiểm tra xem phòng học có bị trùng lịch với ca học nào khác trong khoảng thời gian chỉ định hay không
// @Tags Campus & Room
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param roomId query string true "Mã định danh phòng học (UUID)"
// @Param date query string true "Ngày cần kiểm tra (YYYY-MM-DD)" example("2026-10-15")
// @Param startTime query string true "Giờ bắt đầu (HH:mm)" example("19:30")
// @Param endTime query string true "Giờ kết thúc (HH:mm)" example("21:30")
// @Param excludeSessionId query string false "ID buổi học loại trừ khi đang sửa lịch (UUID)"
// @Success 200 {object} response.Envelope{data=dto.RoomConflictCheckResponse} "Kiểm tra hoàn tất (hasConflict: true/false)"
// @Failure 400 {object} response.Envelope "Tham số truy vấn không hợp lệ"
// @Router /api/v1/rooms/conflicts [get]
func (h *CampusHandler) CheckRoomConflicts(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.CheckRoomConflicts
}
```

---

### Phân Hệ 14: Lên Lịch Học Bù & Dạy Bù Cho Học Sinh Vắng (`internal/domain/makeup`)

#### 14.1 `GET /api/v1/makeup-sessions` - Danh sách ca học bù / dạy bù

```go
package handler

import (
	"net/http"
	"backend/internal/domain/makeup/dto"
	"backend/pkg/response"
)

// ListMakeupSessions godoc
// @Summary Danh sách ca học bù / dạy bù
// @Description Lấy danh sách các lịch học bù cho học sinh vắng, hỗ trợ lọc theo học sinh, lớp, trạng thái và ngày học
// @Tags Make-up Class
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param studentId query string false "Lọc theo ID học sinh (UUID)"
// @Param classId query string false "Lọc theo ID lớp học (UUID)"
// @Param status query string false "Trạng thái (SCHEDULED, ATTENDED, ABSENT, CANCELLED)"
// @Param page query int false "Số trang" default(1)
// @Param limit query int false "Số bản ghi mỗi trang" default(20)
// @Success 200 {object} response.Envelope{data=[]dto.MakeupSessionItemResponse} "Lấy danh sách thành công"
// @Failure 401 {object} response.Envelope "Chưa xác thực"
// @Router /api/v1/makeup-sessions [get]
func (h *MakeupHandler) ListMakeupSessions(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.ListMakeupSessions
}
```

#### 14.2 `POST /api/v1/makeup-sessions` - Lên lịch học bù ghép lớp hoặc kèm 1-1

```go
// CreateMakeupSession godoc
// @Summary Lên lịch học bù cho học sinh vắng
// @Description Điều phối viên tạo lịch học bù cho học sinh đã vắng buổi trước (chọn Ghép Lớp Song Song hoặc Kèm 1-1 với GV/TA)
// @Tags Make-up Class
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param request body dto.CreateMakeupSessionRequest true "Thông tin sắp xếp ca học bù"
// @Success 201 {object} response.Envelope{data=dto.MakeupSessionItemResponse} "Lên lịch học bù thành công"
// @Failure 400 {object} response.Envelope "Dữ liệu không hợp lệ hoặc trùng lịch phòng/giáo viên"
// @Failure 403 {object} response.Envelope "Không có quyền xếp lịch bù"
// @Router /api/v1/makeup-sessions [post]
func (h *MakeupHandler) CreateMakeupSession(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.CreateMakeupSession
}
```

#### 14.3 `PUT /api/v1/makeup-sessions/{id}/attend` - Xác nhận tham gia & Đồng bộ chuyên cần

```go
// MarkMakeupAttended godoc
// @Summary Điểm danh buổi học bù & Đồng bộ chuyên cần
// @Description Giáo viên hoặc Trợ giảng xác nhận học sinh đã tham gia buổi học bù, hệ thống tự động cập nhật cờ bù buổi học gốc
// @Tags Make-up Class
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param id path string true "Mã định danh ca học bù (UUID)"
// @Param request body dto.MarkMakeupAttendedRequest true "Ghi chú nhận xét buổi học bù"
// @Success 200 {object} response.Envelope{data=dto.MakeupSessionItemResponse} "Ghi nhận tham gia thành công"
// @Failure 404 {object} response.Envelope "Không tìm thấy ca học bù"
// @Router /api/v1/makeup-sessions/{id}/attend [put]
func (h *MakeupHandler) MarkMakeupAttended(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.MarkMakeupAttended
}
```

---

### Phân Hệ 15: Khảo Thí & Ngân Hàng Đề Thi Trắc Nghiệm Tự Động (`internal/domain/quiz`)

#### 15.1 `POST /api/v1/quizzes/generate` - Sinh đề thi ngẫu nhiên từ ngân hàng câu hỏi

```go
package handler

import (
	"net/http"
	"backend/internal/domain/quiz/dto"
	"backend/pkg/response"
)

// GenerateQuiz godoc
// @Summary Sinh đề thi ngẫu nhiên từ ngân hàng câu hỏi
// @Description Tự động rút ngẫu nhiên các câu hỏi theo phân bố cấp độ (Dễ, Trung bình, Khó) từ ngân hàng đề của môn học
// @Tags Quizzes & Question Bank
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param request body dto.GenerateQuizRequest true "Cấu hình sinh đề thi"
// @Success 201 {object} response.Envelope{data=dto.QuizDetailResponse} "Sinh đề thi thành công"
// @Failure 400 {object} response.Envelope "Số lượng câu hỏi trong ngân hàng không đủ theo yêu cầu"
// @Router /api/v1/quizzes/generate [post]
func (h *QuizHandler) GenerateQuiz(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.GenerateQuiz
}
```

#### 15.2 `POST /api/v1/quizzes/{id}/submit` - Nộp bài thi trắc nghiệm & Chấm điểm tự động

```go
// SubmitQuizAttempt godoc
// @Summary Nộp bài thi trắc nghiệm & Chấm điểm tức thì
// @Description Học viên nộp bài làm, hệ thống tự động tính điểm theo đáp án chuẩn, trả kết quả và đồng bộ Sổ điểm lớp học
// @Tags Quizzes & Question Bank
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param id path string true "Mã bài thi quiz (UUID)"
// @Param request body dto.SubmitQuizAttemptRequest true "Danh sách câu trả lời của học viên"
// @Success 200 {object} response.Envelope{data=dto.QuizAttemptResultResponse} "Chấm điểm thành công"
// @Failure 400 {object} response.Envelope "Bài thi đã quá thời gian làm bài hoặc hết lượt thi"
// @Router /api/v1/quizzes/{id}/submit [post]
func (h *QuizHandler) SubmitQuizAttempt(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.SubmitQuizAttempt
}
```

---

### Phân Hệ 16: Trung Tâm Điều Hành & Radar Cảnh Báo Nguy Cơ Bỏ Học (`internal/domain/analytics`)

#### 16.1 `GET /api/v1/admin/analytics/kpis` - Báo cáo chỉ số điều hành Real-time

```go
package handler

import (
	"net/http"
	"backend/internal/domain/analytics/dto"
	"backend/pkg/response"
)

// GetExecutiveKPIs godoc
// @Summary Chỉ số điều hành tổng quan trung tâm (Real-time KPIs)
// @Description Báo cáo doanh thu, sĩ số học viên đang hoạt động, tỷ lệ chuyên cần và số lớp đang mở
// @Tags Executive Analytics
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param period query string false "Kỳ phân tích (month, quarter, year)" default("month")
// @Success 200 {object} response.Envelope{data=dto.ExecutiveKPIsResponse} "Lấy dữ liệu KPI thành công"
// @Failure 403 {object} response.Envelope "Chỉ dành cho Ban Giám đốc và Super Admin"
// @Router /api/v1/admin/analytics/kpis [get]
func (h *AnalyticsHandler) GetExecutiveKPIs(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.GetExecutiveKPIs
}
```

#### 16.2 `GET /api/v1/admin/analytics/churn-risk` - Radar cảnh báo học sinh nguy cơ thôi học

```go
// GetChurnRiskRadar godoc
// @Summary Radar cảnh báo nguy cơ học sinh bỏ học (Churn Radar)
// @Description Quét danh sách học viên vắng 2 buổi liên tiếp, chuyên cần dưới 70% hoặc thiếu 3 bài tập về nhà
// @Tags Executive Analytics
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param classId query string false "Lọc theo lớp học cụ thể (UUID)"
// @Param campusId query string false "Lọc theo cơ sở chi nhánh (UUID)"
// @Success 200 {object} response.Envelope{data=[]dto.ChurnRiskStudentResponse} "Lấy danh sách cảnh báo thành công"
// @Router /api/v1/admin/analytics/churn-risk [get]
func (h *AnalyticsHandler) GetChurnRiskRadar(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.GetChurnRiskRadar
}
```

#### 16.3 `GET /api/v1/admin/analytics/teacher-matrix` - Ma trận xếp hạng hiệu quả giảng viên

```go
// GetTeacherMatrix godoc
// @Summary Ma trận đánh giá hiệu quả giảng viên (Teacher Performance Matrix)
// @Description Xếp hạng giảng viên theo điểm đánh giá học viên, tỷ lệ đúng giờ và tốc độ trả bài chấm điểm
// @Tags Executive Analytics
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param period query string false "Tháng tính toán (YYYY-MM)" example("2026-10")
// @Success 200 {object} response.Envelope{data=[]dto.TeacherMatrixItemResponse} "Lấy ma trận thành công"
// @Router /api/v1/admin/analytics/teacher-matrix [get]
func (h *AnalyticsHandler) GetTeacherMatrix(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.GetTeacherMatrix
}
```

#### DTO Structs Bổ Sung (Campus, Make-up, Quiz & Analytics)

```go
package dto

import (
	"time"
	"github.com/google/uuid"
)

type CampusResponse struct {
	ID        uuid.UUID `json:"id" example:"c1eebc99-9c0b-4ef8-bb6d-6bb9bd380001"`
	Code      string    `json:"code" example:"CS_CAUGIAY"`
	Name      string    `json:"name" example:"Cơ sở Cầu Giấy - Hà Nội"`
	Address   string    `json:"address" example:"Tòa nhà Công nghệ, Dịch Vọng Hậu, Cầu Giấy"`
	Phone     *string   `json:"phone,omitempty" example:"024 7300 8888"`
	Email     *string   `json:"email,omitempty" example:"caugiay@lms.edu.vn"`
	IsActive  bool      `json:"isActive" example:"true"`
	RoomCount int       `json:"roomCount" example:"6"`
}

type CreateCampusRequest struct {
	Code    string  `json:"code" validate:"required,min=3" example:"CS_HADONG"`
	Name    string  `json:"name" validate:"required,min=3" example:"Cơ sở Hà Đông"`
	Address string  `json:"address" validate:"required" example:"Số 10 Trần Phú, Hà Đông, Hà Nội"`
	Phone   *string `json:"phone,omitempty" example:"024 7300 9999"`
	Email   *string `json:"email,omitempty" example:"hadong@lms.edu.vn"`
}

type RoomConflictCheckResponse struct {
	HasConflict  bool      `json:"hasConflict" example:"true"`
	ConflictMsg  *string   `json:"conflictMsg,omitempty" example:"Phòng LAB-201 đã được đặt bởi lớp FE-K31 (19:30 - 21:30)"`
	ConflictingClass *string `json:"conflictingClass,omitempty" example:"FE-K31"`
}

type CreateMakeupSessionRequest struct {
	OriginalSessionID uuid.UUID  `json:"originalSessionId" validate:"required" example:"s1eebc99-9c0b-4ef8-bb6d-6bb9bd380111"`
	StudentID         uuid.UUID  `json:"studentId" validate:"required" example:"u1eebc99-9c0b-4ef8-bb6d-6bb9bd380222"`
	MakeupType        string     `json:"makeupType" validate:"required,oneof=PARALLEL_CLASS TUTOR_1ON1" example:"PARALLEL_CLASS"`
	TargetClassID     *uuid.UUID `json:"targetClassId,omitempty" example:"c2eebc99-9c0b-4ef8-bb6d-6bb9bd380333"`
	TargetSessionID   *uuid.UUID `json:"targetSessionId,omitempty" example:"s2eebc99-9c0b-4ef8-bb6d-6bb9bd380444"`
	InstructorID      *uuid.UUID `json:"instructorId,omitempty"`
	ScheduledDate     string     `json:"scheduledDate" validate:"required" example:"2026-10-18"`
	StartTime         string     `json:"startTime" validate:"required" example:"19:30"`
	EndTime           string     `json:"endTime" validate:"required" example:"21:30"`
	RoomID            *uuid.UUID `json:"roomId,omitempty"`
	MeetURL           *string    `json:"meetUrl,omitempty"`
	CoordinatorNotes  *string    `json:"coordinatorNotes,omitempty" example:"Học sinh bận thi giữa kỳ trường đại học, xếp ghép lớp FE-K33"`
}

type MakeupSessionItemResponse struct {
	ID                uuid.UUID  `json:"id" example:"m1eebc99-9c0b-4ef8-bb6d-6bb9bd380555"`
	OriginalSessionID uuid.UUID  `json:"originalSessionId"`
	OriginalTopic     string     `json:"originalTopic" example:"Buổi 4: Goroutines & Channels"`
	StudentID         uuid.UUID  `json:"studentId"`
	StudentName       string     `json:"studentName" example:"Nguyễn Hoàng Nam"`
	MakeupType        string     `json:"makeupType" example:"PARALLEL_CLASS"`
	ScheduledDate     string     `json:"scheduledDate" example:"2026-10-18"`
	TimeRange         string     `json:"timeRange" example:"19:30 - 21:30"`
	Status            string     `json:"status" example:"SCHEDULED"`
	InstructorName    *string    `json:"instructorName,omitempty" example:"ThS. Vũ Hải Đăng"`
}

type MarkMakeupAttendedRequest struct {
	TeacherNotes string `json:"teacherNotes" validate:"required" example:"Học sinh nắm vững kiến thức channel và hoàn thành bài lab tại lớp"`
}

type GenerateQuizRequest struct {
	CourseID        uuid.UUID  `json:"courseId" validate:"required"`
	ClassID         *uuid.UUID `json:"classId,omitempty"`
	Title           string     `json:"title" validate:"required" example:"Bài kiểm tra trắc nghiệm số 1: Go Fundamentals"`
	BankID          uuid.UUID  `json:"bankId" validate:"required"`
	EasyCount       int        `json:"easyCount" validate:"min=0" example:"5"`
	MediumCount     int        `json:"mediumCount" validate:"min=0" example:"10"`
	HardCount       int        `json:"hardCount" validate:"min=0" example:"5"`
	DurationMinutes int        `json:"durationMinutes" validate:"min=5" example:"20"`
	PassingScore    float64    `json:"passingScore" validate:"min=0,max=100" example:"60.0"`
}

type QuizDetailResponse struct {
	ID              uuid.UUID              `json:"id" example:"q1eebc99-9c0b-4ef8-bb6d-6bb9bd380777"`
	Title           string                 `json:"title" example:"Bài kiểm tra trắc nghiệm số 1: Go Fundamentals"`
	DurationMinutes int                    `json:"durationMinutes" example:"20"`
	TotalQuestions  int                    `json:"totalQuestions" example:"20"`
	Questions       []QuizQuestionItemView `json:"questions"`
}

type QuizQuestionItemView struct {
	ID           uuid.UUID `json:"id"`
	QuestionType string    `json:"questionType" example:"SINGLE_CHOICE"`
	Content      string    `json:"content" example:"Từ khóa nào trong Go dùng để khởi chạy một Goroutine?"`
	Options      []string  `json:"options" example:"[\"go\", \"async\", \"thread\", \"routine\"]"`
	Points       float64   `json:"points" example:"1.0"`
}

type SubmitQuizAttemptRequest struct {
	AttemptID uuid.UUID                     `json:"attemptId" validate:"required"`
	Answers   []QuizStudentAnswerSubmission `json:"answers" validate:"required,min=1"`
}

type QuizStudentAnswerSubmission struct {
	QuestionID    uuid.UUID `json:"questionId" validate:"required"`
	StudentAnswer []string  `json:"studentAnswer" validate:"required" example:"[\"go\"]"`
}

type QuizAttemptResultResponse struct {
	AttemptID   uuid.UUID `json:"attemptId"`
	TotalScore  float64   `json:"totalScore" example:"85.0"`
	IsPassed    bool      `json:"isPassed" example:"true"`
	CorrectCount int      `json:"correctCount" example:"17"`
	TotalCount   int      `json:"totalCount" example:"20"`
	SubmittedAt time.Time `json:"submittedAt"`
}

type ExecutiveKPIsResponse struct {
	MonthlyRevenue     float64 `json:"monthlyRevenue" example:"452000000"`
	RevenueGrowthRate  float64 `json:"revenueGrowthRate" example:"14.8"`
	ActiveStudents     int     `json:"activeStudents" example:"284"`
	ActiveClasses      int     `json:"activeClasses" example:"16"`
	AvgAttendanceRate  float64 `json:"avgAttendanceRate" example:"91.5"`
	CompletionRate     float64 `json:"completionRate" example:"88.2"`
}

type ChurnRiskStudentResponse struct {
	StudentID          uuid.UUID `json:"studentId"`
	StudentCode        string    `json:"studentCode" example:"HV-2026-042"`
	StudentName        string    `json:"studentName" example:"Lê Hoàng Long"`
	ClassName          string    `json:"className" example:"FE-K32"`
	RiskLevel          string    `json:"riskLevel" example:"HIGH"` // HIGH, MEDIUM
	ConsecutiveAbsence int       `json:"consecutiveAbsence" example:"2"`
	AttendanceRate     float64   `json:"attendanceRate" example:"62.5"`
	MissingHomeworks   int       `json:"missingHomeworks" example:"3"`
	CoordinatorNotes   *string   `json:"coordinatorNotes,omitempty"`
}

type TeacherMatrixItemResponse struct {
	TeacherID        uuid.UUID `json:"teacherId"`
	TeacherName      string    `json:"teacherName" example:"ThS. Nguyễn Văn Tuấn"`
	RatingScore      float64   `json:"ratingScore" example:"4.85"`
	TotalFeedbacks   int       `json:"totalFeedbacks" example:"142"`
	OnTimeRate       float64   `json:"onTimeRate" example:"98.2"`
	LateSessionsCount int      `json:"lateSessionsCount" example:"1"`
	AvgGradingHours  float64   `json:"avgGradingHours" example:"18.5"` // Thời gian trả bài trung bình (giờ)
}
```

---

### Phân Hệ 17: Bảng Điều Khiển Ca Dạy Trực Tiếp & Bảng Trắng (`internal/domain/cockpit`)

#### 17.1 `GET /api/v1/teacher/cockpit/{sessionId}` - Không gian ca dạy tập trung (Live Cockpit)

```go
package handler

import (
	"net/http"
	"backend/internal/domain/cockpit/dto"
	"backend/pkg/response"
)

// GetLiveCockpit godoc
// @Summary Bảng điều khiển ca dạy tập trung (Live Cockpit)
// @Description Tải toàn bộ thông tin ca dạy: link Meet/Zoom, trạng thái check-in, danh sách học viên, bài giảng và quick poll
// @Tags Teacher Live Cockpit
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param sessionId path string true "Mã định danh buổi học (UUID)"
// @Success 200 {object} response.Envelope{data=dto.LiveCockpitResponse} "Tải không gian ca dạy thành công"
// @Failure 403 {object} response.Envelope "Chỉ giảng viên phụ trách hoặc trợ giảng mới có quyền truy cập"
// @Router /api/v1/teacher/cockpit/{sessionId} [get]
func (h *CockpitHandler) GetLiveCockpit(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.GetLiveCockpit
}
```

#### 17.2 `POST /api/v1/teacher/cockpit/{sessionId}/quick-poll` - Phát câu hỏi bình chọn tức thời

```go
// CreateQuickPoll godoc
// @Summary Tạo câu hỏi bình chọn nhanh 2 phút trong ca dạy
// @Description Giảng viên phát câu hỏi kiểm tra mức độ hiểu bài ngay tại lớp để học viên bình chọn real-time
// @Tags Teacher Live Cockpit
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param sessionId path string true "Mã định danh buổi học (UUID)"
// @Param request body dto.CreateQuickPollRequest true "Nội dung câu hỏi và các lựa chọn"
// @Success 201 {object} response.Envelope{data=dto.QuickPollResponse} "Phát bình chọn thành công"
// @Router /api/v1/teacher/cockpit/{sessionId}/quick-poll [post]
func (h *CockpitHandler) CreateQuickPoll(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.CreateQuickPoll
}
```

#### 17.3 `POST /api/v1/teacher/cockpit/{sessionId}/whiteboard` - Lưu bảng trắng kỹ thuật số (Vue-native Whiteboard)

```go
// SaveWhiteboard godoc
// @Summary Lưu nét vẽ bảng trắng kỹ thuật số & Autosave (Vue-native Whiteboard)
// @Description Lưu dữ liệu nét vẽ vector scene JSON của buổi học (Tạm hoãn Excalidraw, sẵn sàng cho Vue-native canvas)
// @Tags Teacher Live Cockpit
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param sessionId path string true "Mã định danh buổi học (UUID)"
// @Param request body dto.SaveWhiteboardRequest true "Dữ liệu nét vẽ vector JSON và tiêu đề sơ đồ"
// @Success 200 {object} response.Envelope{data=dto.WhiteboardResponse} "Lưu bảng trắng thành công"
// @Router /api/v1/teacher/cockpit/{sessionId}/whiteboard [post]
func (h *CockpitHandler) SaveWhiteboard(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.SaveWhiteboard
}
```

#### 17.4 `POST /api/v1/teacher/whiteboards/{id}/export-pdf` - Xuất file PDF bảng vẽ đính kèm buổi học

```go
// ExportWhiteboardPDF godoc
// @Summary Xuất bảng vẽ ra file PDF chất lượng cao (Vue-native Whiteboard)
// @Description Chuyển đổi sơ đồ nét vẽ vector thành file PDF, lưu trữ lên MinIO/S3 và đính kèm vào mục tài liệu buổi học
// @Tags Teacher Live Cockpit
// @Accept multipart/form-data
// @Produce json
// @Security BearerAuth
// @Param id path string true "Mã định danh bảng vẽ (UUID)"
// @Param file formData file true "File hình ảnh/PDF render từ canvas (PNG hoặc PDF)"
// @Param title formData string false "Tiêu đề tài liệu sơ đồ"
// @Success 200 {object} response.Envelope{data=dto.WhiteboardExportPDFResponse} "Xuất PDF và đính kèm buổi học thành công"
// @Router /api/v1/teacher/whiteboards/{id}/export-pdf [post]
func (h *CockpitHandler) ExportWhiteboardPDF(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.ExportWhiteboardPDF
}
```

---

### Phân Hệ 18: Trợ Lý Chấm Điểm AI & Voice Note (`internal/domain/assignment`)

#### 18.1 `POST /api/v1/submissions/{id}/ai-review` - Trợ lý AI quét code review

```go
package handler

import (
	"net/http"
	"backend/internal/domain/assignment/dto"
	"backend/pkg/response"
)

// GenerateAICodeReview godoc
// @Summary Trợ lý AI quét code review & Gợi ý nhận xét sư phạm
// @Description Phân tích mã nguồn học viên nộp, phát hiện lỗi cú pháp, gợi ý điểm số và soạn bản nháp nhận xét Clean Code
// @Tags Teacher Grading
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param id path string true "Mã bài nộp submission (UUID)"
// @Success 200 {object} response.Envelope{data=dto.AICodeReviewDraftResponse} "Sinh nhận xét AI thành công"
// @Failure 404 {object} response.Envelope "Không tìm thấy bài nộp"
// @Router /api/v1/submissions/{id}/ai-review [post]
func (h *AssignmentHandler) GenerateAICodeReview(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.GenerateAICodeReview
}
```

#### 18.2 `POST /api/v1/submissions/{id}/voice-feedback` - Tải lên audio nhận xét dặn dò

```go
// UploadVoiceFeedback godoc
// @Summary Tải lên audio nhận xét dặn dò (Voice Note)
// @Description Giảng viên tải lên file ghi âm dặn dò trực tiếp trên trình duyệt (thời lượng 30s - 2 phút)
// @Tags Teacher Grading
// @Accept multipart/form-data
// @Produce json
// @Security BearerAuth
// @Param id path string true "Mã bài nộp submission (UUID)"
// @Param audio formData file true "File ghi âm giọng nói (audio/webm, audio/mp3)"
// @Success 200 {object} response.Envelope{data=dto.VoiceFeedbackResponse} "Tải audio thành công"
// @Router /api/v1/submissions/{id}/voice-feedback [post]
func (h *AssignmentHandler) UploadVoiceFeedback(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.UploadVoiceFeedback
}
```

---

### Phân Hệ 19: Hộp Thư Hỏi Đáp Tập Trung & Ủy Quyền Trợ Giảng (`internal/domain/inbox`)

#### 19.1 `GET /api/v1/teacher/inbox/questions` - Danh sách câu hỏi cần giải đáp

```go
package handler

import (
	"net/http"
	"backend/internal/domain/inbox/dto"
	"backend/pkg/response"
)

// ListTeacherQuestions godoc
// @Summary Hộp thư hỏi đáp tập trung của giảng viên
// @Description Gom toàn bộ thắc mắc từ video bài giảng (Timestamped Q&A) và BTVN của các lớp phụ trách
// @Tags Teacher Inbox
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param status query string false "Trạng thái (UNANSWERED, DELEGATED_TA, RESOLVED)"
// @Param classId query string false "Lọc theo lớp học (UUID)"
// @Success 200 {object} response.Envelope{data=[]dto.TeacherQuestionItemResponse} "Lấy danh sách thành công"
// @Router /api/v1/teacher/inbox/questions [get]
func (h *InboxHandler) ListTeacherQuestions(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.ListTeacherQuestions
}
```

#### 19.2 `PUT /api/v1/teacher/inbox/questions/{id}/delegate-ta` - Phân công câu hỏi cho Trợ giảng

```go
// DelegateQuestionToTA godoc
// @Summary Ủy quyền câu hỏi cho Trợ giảng kèm SLA
// @Description Giảng viên giao câu hỏi cho Trợ giảng phụ trách lớp trả lời kèm cam kết thời hạn SLA 2 giờ
// @Tags Teacher Inbox
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param id path string true "Mã câu hỏi thảo luận (UUID)"
// @Param request body dto.DelegateTARequest true "Thông tin trợ giảng và ghi chú dặn dò"
// @Success 200 {object} response.Envelope{data=dto.TeacherQuestionItemResponse} "Giao việc cho trợ giảng thành công"
// @Router /api/v1/teacher/inbox/questions/{id}/delegate-ta [put]
func (h *InboxHandler) DelegateQuestionToTA(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.DelegateQuestionToTA
}
```

---

### Phân Hệ 20: Sàn Dạy Thay & Lịch Rảnh Giảng Viên (`internal/domain/substitute`)

#### 20.1 `POST /api/v1/teacher/sessions/{sessionId}/substitute-request` - Đề xuất nhờ dạy thay

```go
package handler

import (
	"net/http"
	"backend/internal/domain/substitute/dto"
	"backend/pkg/response"
)

// CreateSubstituteRequest godoc
// @Summary Đề xuất nhờ đồng nghiệp dạy thay ca học
// @Description Giảng viên bận đột xuất tạo yêu cầu nhờ dạy thay; hệ thống tự động tìm đồng nghiệp cùng chuyên môn đang rảnh ca đó
// @Tags Teacher Substitute
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param sessionId path string true "Mã định danh buổi học cần nhờ dạy (UUID)"
// @Param request body dto.CreateSubstituteRequest true "Lý do xin nghỉ và tài liệu bàn giao"
// @Success 201 {object} response.Envelope{data=dto.SubstituteRequestResponse} "Tạo yêu cầu nhờ dạy thay thành công"
// @Router /api/v1/teacher/sessions/{sessionId}/substitute-request [post]
func (h *SubstituteHandler) CreateSubstituteRequest(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.CreateSubstituteRequest
}
```

#### 20.2 `PUT /api/v1/teacher/substitute-requests/{id}/accept` - Đồng nghiệp nhận ca dạy thay

```go
// AcceptSubstituteRequest godoc
// @Summary Đồng nghiệp nhận ca dạy thay
// @Description Giảng viên rảnh ca nhận dạy thay cho đồng nghiệp; hệ thống tự động chuyển giao ca học và thù lao buổi dạy
// @Tags Teacher Substitute
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param id path string true "Mã yêu cầu dạy thay (UUID)"
// @Success 200 {object} response.Envelope{data=dto.SubstituteRequestResponse} "Nhận ca dạy thay thành công"
// @Router /api/v1/teacher/substitute-requests/{id}/accept [put]
func (h *SubstituteHandler) AcceptSubstituteRequest(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.AcceptSubstituteRequest
}
```

---

### Phân Hệ 21: Cổng Vận Hành Lớp, Chăm Sóc Học Viên & Chuyển Lớp (`internal/domain/coordinator`)

#### 21.1 `GET /api/v1/coordinator/shifts/today` - Bảng điều hành ca học hôm nay & Giám sát Check-in

```go
package handler

import (
	"net/http"
	"backend/internal/domain/coordinator/dto"
	"backend/pkg/response"
)

// GetTodayShifts godoc
// @Summary Lấy danh sách toàn bộ ca học diễn ra trong ngày kèm trạng thái Check-in GV
// @Description Chuyên viên Vận hành lớp (Class Coordinator) giám sát các phòng học, đường link Meet, trạng thái giáo viên đã check-in hay có nguy cơ trễ
// @Tags Coordinator Operations
// @Produce json
// @Security BearerAuth
// @Param campusId query string false "Lọc theo chi nhánh cơ sở (UUID)" format(uuid)
// @Success 200 {object} response.Envelope{data=dto.TodayShiftResponse} "Lấy danh sách ca học hôm nay thành công"
// @Router /api/v1/coordinator/shifts/today [get]
func (h *CoordinatorHandler) GetTodayShifts(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.GetTodayShifts
}
```

#### 21.2 `POST /api/v1/coordinator/sessions/{id}/broadcast` - Phát loa thông báo khẩn cấp cả lớp

```go
// BroadcastToClass godoc
// @Summary Phát thông báo khẩn cấp tới toàn bộ học sinh và giáo viên trong lớp
// @Description Gửi thông báo tức thì qua Zalo ZNS / Push Notification / SMS khi đổi phòng học, link Google Meet hoặc dời giờ khẩn cấp
// @Tags Coordinator Operations
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param id path string true "Mã định danh buổi học (UUID)" format(uuid)
// @Param request body dto.BroadcastClassRequest true "Nội dung thông báo khẩn và kênh gửi"
// @Success 200 {object} response.Envelope{data=dto.BroadcastClassResponse} "Phát thông báo khẩn thành công"
// @Router /api/v1/coordinator/sessions/{id}/broadcast [post]
func (h *CoordinatorHandler) BroadcastToClass(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.BroadcastToClass
}
```

#### 21.3 `POST /api/v1/coordinator/care-logs` - Ghi nhận nhật ký chăm sóc học sinh (Care CRM)

```go
// CreateCareLog godoc
// @Summary Ghi lại nhật ký gọi điện / trao đổi chăm sóc học sinh
// @Description Coordinator ghi nhận nội dung cuộc gọi với Học sinh/Phụ huynh, phân loại lý do nghỉ/đuối kiến thức, cam kết giải pháp và hẹn lịch follow-up
// @Tags Coordinator Operations
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param request body dto.CreateCareLogRequest true "Thông tin cuộc gọi và giải pháp cam kết"
// @Success 201 {object} response.Envelope{data=dto.CareLogResponse} "Lưu nhật ký chăm sóc thành công"
// @Router /api/v1/coordinator/care-logs [post]
func (h *CoordinatorHandler) CreateCareLog(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.CreateCareLog
}
```

#### 21.4 `GET /api/v1/coordinator/students/{id}/care-history` - Xem toàn bộ lịch sử chăm sóc học viên

```go
// GetStudentCareHistory godoc
// @Summary Xem toàn bộ lịch sử chăm sóc và tương tác của một học sinh
// @Description Lấy danh sách toàn bộ các cuộc gọi, lý do vắng, ca học bù đã xếp và ghi chú của Coordinator đối với học sinh
// @Tags Coordinator Operations
// @Produce json
// @Security BearerAuth
// @Param id path string true "Mã định danh học sinh (UUID)" format(uuid)
// @Success 200 {object} response.Envelope{data=[]dto.CareLogResponse} "Lấy lịch sử chăm sóc thành công"
// @Router /api/v1/coordinator/students/{id}/care-history [get]
func (h *CoordinatorHandler) GetStudentCareHistory(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.GetStudentCareHistory
}
```

#### 21.5 `POST /api/v1/coordinator/transfers` - Tiếp nhận đơn xin chuyển lớp hoặc bảo lưu

```go
// CreateClassTransferRequest godoc
// @Summary Tạo đơn xin chuyển lớp học hoặc xin bảo lưu khóa học
// @Description Coordinator tạo đề xuất chuyển lớp (sang ca khác) hoặc bảo lưu khóa học (tối đa 6 tháng) kèm số tiền học phí bảo lưu
// @Tags Coordinator Operations
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param request body dto.CreateClassTransferRequest true "Thông tin lớp chuyển đến hoặc thời hạn bảo lưu"
// @Success 201 {object} response.Envelope{data=dto.ClassTransferResponse} "Tạo đơn chuyển lớp/bảo lưu thành công"
// @Router /api/v1/coordinator/transfers [post]
func (h *CoordinatorHandler) CreateClassTransfer(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.CreateClassTransfer
}
```

#### 21.6 `PUT /api/v1/coordinator/transfers/{id}/approve` - Phê duyệt chuyển lớp hoặc bảo lưu

```go
// ApproveClassTransfer godoc
// @Summary Giáo vụ trưởng hoặc Admin phê duyệt đơn chuyển lớp / bảo lưu
// @Description Xác nhận chấp thuận chuyển lớp, tự động cập nhật sĩ số 2 lớp hoặc chuyển trạng thái học viên sang DEFERRED
// @Tags Coordinator Operations
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param id path string true "Mã đơn chuyển lớp/bảo lưu (UUID)" format(uuid)
// @Param request body dto.ApproveTransferRequest true "Quyết định phê duyệt (APPROVED/REJECTED)"
// @Success 200 {object} response.Envelope{data=dto.ClassTransferResponse} "Xử lý đơn chuyển lớp thành công"
// @Router /api/v1/coordinator/transfers/{id}/approve [put]
func (h *CoordinatorHandler) ApproveClassTransfer(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.ApproveClassTransfer
}
```

#### 21.7 `POST /api/v1/coordinator/sessions/{id}/incidents` - Báo cáo sự cố buổi học

```go
// ReportSessionIncident godoc
// @Summary Báo cáo sự cố phòng học hoặc ca dạy
// @Description Ghi nhận sự cố kỹ thuật (mất điện, rớt mạng, điều hòa hỏng, GV ốm) và giải pháp khắc phục tạm thời
// @Tags Coordinator Operations
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param id path string true "Mã định danh buổi học (UUID)" format(uuid)
// @Param request body dto.ReportSessionIncidentRequest true "Chi tiết sự cố và phương án xử lý"
// @Success 201 {object} response.Envelope{data=dto.SessionIncidentResponse} "Ghi nhận sự cố thành công"
// @Router /api/v1/coordinator/sessions/{id}/incidents [post]
func (h *CoordinatorHandler) ReportSessionIncident(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.ReportSessionIncident
}
```

#### 21.8 `GET /api/v1/coordinator/classes/{id}/materials` - Danh sách cấp phát học liệu & đồng phục

```go
// ListClassMaterials godoc
// @Summary Danh sách theo dõi cấp phát giáo trình, áo đồng phục của lớp học
// @Description Hiển thị danh sách học viên trong lớp kèm trạng thái đã nhận giáo trình in, áo đồng phục (size) hay chưa
// @Tags Coordinator Operations
// @Produce json
// @Security BearerAuth
// @Param id path string true "Mã định danh lớp học (UUID)" format(uuid)
// @Success 200 {object} response.Envelope{data=[]dto.StudentMaterialItemDTO} "Lấy danh sách cấp phát thành công"
// @Router /api/v1/coordinator/classes/{id}/materials [get]
func (h *CoordinatorHandler) ListClassMaterials(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.ListClassMaterials
}
```

#### 21.9 `PUT /api/v1/coordinator/materials/{id}/deliver` - Xác nhận bàn giao học liệu

```go
// DeliverStudentMaterial godoc
// @Summary Xác nhận đã bàn giao học liệu / áo đồng phục cho học viên
// @Description Coordinator tích chọn xác nhận đã bàn giao giáo trình hoặc đồng phục khi học viên nhận trực tiếp tại quầy
// @Tags Coordinator Operations
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param id path string true "Mã bản ghi cấp phát học liệu (UUID)" format(uuid)
// @Param request body dto.DeliverMaterialRequest true "Thông tin người ký nhận"
// @Success 200 {object} response.Envelope{data=dto.StudentMaterialItemDTO} "Xác nhận bàn giao thành công"
// @Router /api/v1/coordinator/materials/{id}/deliver [put]
func (h *CoordinatorHandler) DeliverStudentMaterial(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.DeliverStudentMaterial
}
```

#### 21.10 `POST /api/v1/coordinator/sessions/{id}/digest` - Gửi báo cáo tóm tắt buổi học tới Phụ huynh

```go
// SendSessionDigest godoc
// @Summary Soạn và gửi báo cáo tóm tắt buổi học tới Phụ huynh và Học sinh
// @Description Gửi tin nhắn tổng hợp nội dung buổi học, học viên tích cực và bài tập cần làm qua Zalo ZNS / App Phụ huynh
// @Tags Coordinator Operations
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param id path string true "Mã định danh buổi học (UUID)" format(uuid)
// @Param request body dto.SendSessionDigestRequest true "Nội dung tóm tắt buổi học và danh sách nhận tin"
// @Success 200 {object} response.Envelope{data=dto.SessionDigestResponse} "Gửi báo cáo buổi học thành công"
// @Router /api/v1/coordinator/sessions/{id}/digest [post]
func (h *CoordinatorHandler) SendSessionDigest(w http.ResponseWriter, r *http.Request) {
	// Implementation calls usecase.SendSessionDigest
}
```

---

#### DTO Structs Bổ Sung (Teacher Live Cockpit, AI Review, Inbox, Substitute & Coordinator)

```go
package dto

import (
	"time"
	"github.com/google/uuid"
)

type LiveCockpitResponse struct {
	SessionID       uuid.UUID                 `json:"sessionId"`
	ClassName       string                    `json:"className" example:"FE-K32"`
	Topic           string                    `json:"topic" example:"Buổi 4: Goroutines & Channels"`
	MeetingPlatform string                    `json:"meetingPlatform" example:"GOOGLE_MEET"`
	MeetingURL      string                    `json:"meetingUrl" example:"https://meet.google.com/abc-defg-hij"`
	IsCheckedIn     bool                      `json:"isCheckedIn" example:"true"`
	CheckedInAt     *time.Time                `json:"checkedInAt,omitempty"`
	Students        []CockpitStudentStatusDTO `json:"students"`
	ActivePoll      *QuickPollResponse        `json:"activePoll,omitempty"`
	WhiteboardURL   *string                   `json:"whiteboardUrl,omitempty"`
}

type CockpitStudentStatusDTO struct {
	StudentID      uuid.UUID `json:"studentId"`
	StudentName    string    `json:"studentName" example:"Nguyễn Hoàng Nam"`
	AttendanceType string    `json:"attendanceType" example:"PRESENT_ONLINE"` // PRESENT_OFFLINE, PRESENT_ONLINE, ABSENT
	IsRaisedHand   bool      `json:"isRaisedHand" example:"false"`
}

type CreateQuickPollRequest struct {
	QuestionText    string   `json:"questionText" validate:"required,min=5" example:"Channel không đệm có chặn goroutine gửi không?"`
	Options         []string `json:"options" validate:"required,min=2" example:"[\"Có chặn\", \"Không chặn\"]"`
	DurationSeconds int      `json:"durationSeconds" validate:"min=30,max=300" example:"120"`
}

type QuickPollResponse struct {
	PollID          uuid.UUID          `json:"pollId"`
	QuestionText    string             `json:"questionText"`
	Options         []PollOptionVoteDTO `json:"options"`
	TotalVotes      int                `json:"totalVotes" example:"18"`
	RemainingSeconds int               `json:"remainingSeconds" example:"85"`
}

type PollOptionVoteDTO struct {
	ID         string  `json:"id" example:"A"`
	Text       string  `json:"text" example:"Có chặn"`
	VoteCount  int     `json:"voteCount" example:"15"`
	Percentage float64 `json:"percentage" example:"83.3"`
}

type SaveWhiteboardRequest struct {
	Title     string `json:"title" validate:"required" example:"Sơ đồ kiến trúc Go Worker Pool"`
	// BoardData chứa dữ liệu vector canvas Scene JSON:
	// bao gồm các phần tử vector (elements), trạng thái hiển thị (state) và nét vẽ
	BoardData string `json:"boardData" validate:"required" example:"{\"version\":1,\"elements\":[...],\"state\":{...}}"`
}

type WhiteboardResponse struct {
	WhiteboardID uuid.UUID `json:"whiteboardId"`
	SessionID    uuid.UUID `json:"sessionId"`
	Title        string    `json:"title"`
	ExportPdfURL *string   `json:"exportPdfUrl,omitempty" example:"https://storage.lms.edu.vn/whiteboards/session-01.pdf"`
	UpdatedAt    time.Time `json:"updatedAt"`
}

type WhiteboardExportPDFResponse struct {
	WhiteboardID uuid.UUID `json:"whiteboardId"`
	SessionID    uuid.UUID `json:"sessionId"`
	Title        string    `json:"title"`
	ExportPdfURL string    `json:"exportPdfUrl" example:"https://storage.lms.edu.vn/whiteboards/session-01.pdf"`
	FileSizeKB   int       `json:"fileSizeKb" example:"320"`
	GeneratedAt  time.Time `json:"generatedAt"`
}

type AICodeReviewDraftResponse struct {
	SubmissionID   uuid.UUID `json:"submissionId"`
	CleanCodeScore float64   `json:"cleanCodeScore" example:"88.0"`
	LintIssues     []string  `json:"lintIssues" example:"[\"Line 24: Unhandled error in file read\", \"Line 38: Channel not closed\"]"`
	FeedbackDraft  string    `json:"feedbackDraft" example:"Em hoàn thành tốt logic concurrency. Cần lưu ý đóng channel tại dòng 38 để tránh rò rỉ bộ nhớ."`
}

type VoiceFeedbackResponse struct {
	SubmissionID uuid.UUID `json:"submissionId"`
	AudioURL     string    `json:"audioUrl" example:"https://storage.lms.edu.vn/feedback/audio-submission-01.webm"`
	DurationSec  int       `json:"durationSec" example:"45"`
}

type TeacherQuestionItemResponse struct {
	QuestionID      uuid.UUID  `json:"questionId"`
	StudentName     string     `json:"studentName" example:"Lê Hoàng Long"`
	ClassName       string     `json:"className" example:"FE-K32"`
	LessonTopic     string     `json:"lessonTopic" example:"Buổi 4: Goroutines"`
	VideoTimestamp  *int       `json:"videoTimestamp,omitempty" example:"845"`
	Content         string     `json:"content" example:"Thầy ơi tại sao chỗ này dùng unbuffered channel lại bị deadlock?"`
	Status          string     `json:"status" example:"DELEGATED_TA"`
	AssignedTAName  *string    `json:"assignedTaName,omitempty" example:"Trần Văn TA"`
	CreatedAt       time.Time  `json:"createdAt"`
}

type DelegateTARequest struct {
	TAUserID uuid.UUID `json:"taUserId" validate:"required"`
	Notes    *string   `json:"notes,omitempty" example:"Học sinh bị deadlock do gửi trước khi có receiver, em chỉ dẫn lại slide 12 nhé"`
}

type CreateSubstituteRequest struct {
	Reason      string  `json:"reason" validate:"required,min=5" example:"Bị sốt xuất huyết nhập viện, nhờ đồng nghiệp dạy thay tối T5"`
	HandoffNotes *string `json:"handoffNotes,omitempty" example:"Đã up slide Buổi 4 và starter code trong bài giảng"`
}

type SubstituteRequestResponse struct {
	RequestID           uuid.UUID  `json:"requestId"`
	SessionID           uuid.UUID  `json:"sessionId"`
	SessionDate         string     `json:"sessionDate" example:"2026-10-18"`
	TimeRange           string     `json:"timeRange" example:"19:30 - 21:30"`
	OriginalTeacherName string     `json:"originalTeacherName" example:"ThS. Nguyễn Văn Tuấn"`
	SubstituteTeacherName *string  `json:"substituteTeacherName,omitempty"`
	Status              string     `json:"status" example:"PENDING_OFFER"`
}

// Coordinator Operations DTOs
type TodayShiftResponse struct {
	Date          string                `json:"date" example:"2026-10-18"`
	TotalSessions int                   `json:"totalSessions" example:"12"`
	LateAlerts    int                   `json:"lateAlerts" example:"1"`
	Sessions      []ShiftSessionItemDTO `json:"sessions"`
}

type ShiftSessionItemDTO struct {
	SessionID       uuid.UUID  `json:"sessionId"`
	ClassName       string     `json:"className" example:"FE-K32"`
	SubjectName     string     `json:"subjectName" example:"Golang Backend Master"`
	CampusName      string     `json:"campusName" example:"Cơ sở Cầu Giấy"`
	RoomName        string     `json:"roomName" example:"LAB-201"`
	TimeRange       string     `json:"timeRange" example:"19:30 - 21:30"`
	TeacherName     string     `json:"teacherName" example:"Nguyễn Văn Tuấn"`
	IsTeacherLate   bool       `json:"isTeacherLate" example:"false"`
	IsCheckedIn     bool       `json:"isCheckedIn" example:"true"`
	CheckedInAt     *time.Time `json:"checkedInAt,omitempty"`
	PresentStudents int        `json:"presentStudents" example:"22"`
	TotalStudents   int        `json:"totalStudents" example:"24"`
	Status          string     `json:"status" example:"IN_PROGRESS"`
}

type BroadcastClassRequest struct {
	Title    string   `json:"title" validate:"required" example:"Thông báo đổi phòng học tối nay"`
	Message  string   `json:"message" validate:"required,min=10" example:"Lớp FE-K32 tối nay chuyển sang phòng LAB-302 do phòng LAB-201 bảo trì điều hòa."`
	Channels []string `json:"channels" validate:"required" example:"[\"ZALO\", \"PUSH\"]"`
}

type BroadcastClassResponse struct {
	BroadcastID uuid.UUID `json:"broadcastId"`
	Recipients  int       `json:"recipients" example:"25"`
	DeliveredAt time.Time `json:"deliveredAt"`
}

type CreateCareLogRequest struct {
	StudentID      uuid.UUID  `json:"studentId" validate:"required"`
	ClassID        uuid.UUID  `json:"classId" validate:"required"`
	ContactType    string     `json:"contactType" validate:"required" example:"PHONE_CALL"`
	ContactTarget  string     `json:"contactTarget" validate:"required" example:"PARENT"`
	ReasonCategory string     `json:"reasonCategory" validate:"required" example:"DEMOTIVATED"`
	NoteContent    string     `json:"noteContent" validate:"required,min=10" example:"Học sinh cảm thấy bài tập Concurrency khó, phụ huynh nhờ trung tâm kèm thêm"`
	ActionTaken    string     `json:"actionTaken" validate:"required" example:"TA_TUTORING"`
	NextFollowUpAt *time.Time `json:"nextFollowUpAt,omitempty"`
}

type CareLogResponse struct {
	LogID           uuid.UUID  `json:"logId"`
	StudentName     string     `json:"studentName" example:"Lê Hoàng Long"`
	CoordinatorName string     `json:"coordinatorName" example:"Phạm Thị Quản Nhiệm"`
	ContactType     string     `json:"contactType" example:"PHONE_CALL"`
	ContactTarget   string     `json:"contactTarget" example:"PARENT"`
	ReasonCategory  string     `json:"reasonCategory" example:"DEMOTIVATED"`
	NoteContent     string     `json:"noteContent"`
	ActionTaken     string     `json:"actionTaken" example:"TA_TUTORING"`
	NextFollowUpAt  *time.Time `json:"nextFollowUpAt,omitempty"`
	CreatedAt       time.Time  `json:"createdAt"`
}

type CreateClassTransferRequest struct {
	StudentID    uuid.UUID  `json:"studentId" validate:"required"`
	FromClassID  uuid.UUID  `json:"fromClassId" validate:"required"`
	ToClassID    *uuid.UUID `json:"toClassId,omitempty"`
	TransferType string     `json:"transferType" validate:"required" example:"DEFERRAL"`
	Reason       string     `json:"reason" validate:"required,min=5" example:"Học sinh bận công tác đột xuất 2 tháng, xin bảo lưu sang khóa sau"`
	DeferMonths  int        `json:"deferMonths" validate:"min=1,max=6" example:"3"`
}

type ClassTransferResponse struct {
	TransferID     uuid.UUID  `json:"transferId"`
	StudentID      uuid.UUID  `json:"studentId"`
	StudentName    string     `json:"studentName" example:"Lê Hoàng Long"`
	FromClassName  string     `json:"fromClassName" example:"FE-K32"`
	ToClassName    *string    `json:"toClassName,omitempty"`
	TransferType   string     `json:"transferType" example:"DEFERRAL"`
	ReservedCredit float64    `json:"reservedCredit" example:"3500000.0"`
	DeferUntilDate *time.Time `json:"deferUntilDate,omitempty"`
	Status         string     `json:"status" example:"PENDING"`
	CreatedAt      time.Time  `json:"createdAt"`
}

type ApproveTransferRequest struct {
	IsApproved bool    `json:"isApproved"`
	Note       *string `json:"note,omitempty" example:"Đồng ý bảo lưu 3 tháng, bảo lưu số tiền 3.500.000 VNĐ"`
}

type ReportSessionIncidentRequest struct {
	IncidentType string `json:"incidentType" validate:"required" example:"PROJECTOR_BROKEN"`
	Severity     string `json:"severity" validate:"required" example:"MEDIUM"`
	Description  string `json:"description" validate:"required,min=10" example:"Máy chiếu phòng 201 chập bóng đèn không lên hình"`
	Resolution   string `json:"resolution" validate:"required" example:"Đã mượn máy chiếu di động từ phòng hành chính thay thế"`
}

type SessionIncidentResponse struct {
	IncidentID      uuid.UUID `json:"incidentId"`
	SessionID       uuid.UUID `json:"sessionId"`
	IncidentType    string    `json:"incidentType" example:"PROJECTOR_BROKEN"`
	Severity        string    `json:"severity" example:"MEDIUM"`
	Description     string    `json:"description"`
	Resolution      *string   `json:"resolution,omitempty"`
	CoordinatorName string    `json:"coordinatorName" example:"Phạm Thị Quản Nhiệm"`
	CreatedAt       time.Time `json:"createdAt"`
}

type StudentMaterialItemDTO struct {
	MaterialID   uuid.UUID  `json:"materialId"`
	StudentID    uuid.UUID  `json:"studentId"`
	StudentName  string     `json:"studentName" example:"Nguyễn Hoàng Nam"`
	ItemName     string     `json:"itemName" example:"Áo đồng phục LMS"`
	ItemType     string     `json:"itemType" example:"UNIFORM"`
	SizeOption   *string    `json:"sizeOption,omitempty" example:"L"`
	IsDelivered  bool       `json:"isDelivered" example:"true"`
	DeliveredAt  *time.Time `json:"deliveredAt,omitempty"`
	ReceiverName *string    `json:"receiverName,omitempty" example:"Nguyễn Hoàng Nam"`
}

type DeliverMaterialRequest struct {
	ReceiverName string `json:"receiverName" validate:"required" example:"Nguyễn Hoàng Nam"`
}

type SendSessionDigestRequest struct {
	Highlights   string   `json:"highlights" validate:"required" example:"Buổi 4 lớp học tốt, 100% học sinh hoàn thành bài thực hành goroutine cơ bản."`
	HomeworkDue  string   `json:"homeworkDue" example:"Hạn nộp BTVN bài 4: 23:59 Chủ Nhật"`
	SendChannels []string `json:"sendChannels" validate:"required" example:"[\"APP_NOTIFICATION\", \"ZALO\"]"`
}

type SessionDigestResponse struct {
	SessionID uuid.UUID `json:"sessionId"`
	SentCount int       `json:"sentCount" example:"24"`
	SentAt    time.Time `json:"sentAt"`
}
```

---

## 3. Quy Trình Khởi Tạo & Xem Swagger UI Trên Môi Trường Local

1. **Khởi tạo và cập nhật tài liệu Swagger:**
   ```bash
   # Chạy task swaggo để quét toàn bộ annotations và sinh docs/swagger.json, docs/swagger.yaml
   task swagger
   ```
2. **Khởi chạy máy chủ API:**
   ```bash
   task dev
   ```
3. **Truy cập Swagger UI:**
   - Mở trình duyệt tại đường dẫn: `http://localhost:8080/swagger/index.html`
   - Nhấn nút **Authorize** và điền Bearer token: `Bearer <jwt_token_here>` để thực hiện kiểm thử trực tiếp các API cần phân quyền.

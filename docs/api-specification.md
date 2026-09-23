# Đặc Tả Giao Diện Lập Trình Ứng Dụng (Go-chi REST API & Swagger/OpenAPI Specification)

> **Dự án:** LMS Center Platform (Hệ thống Quản lý Học tập, Giảng viên & Vận hành Đào tạo Đa hình thức)  
> **Tài liệu:** `docs/api-specification.md`  
> **Phiên bản:** 2.2.0 (Chuẩn hóa Go 1.23+ / `gmhafiz/go8` Layered Architecture, Chi Router, Go Struct DTOs & Toàn bộ Swagger/OpenAPI Comments)  
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

#### 7.2 `POST /api/v1/courses/{id}/checkout` - Mua khóa học Trả Phí & Sinh VietQR Động

```go
// CheckoutCourse godoc
// @Summary Khởi tạo đơn mua khóa học và tạo mã thanh toán VietQR động
// @Description Tạo hóa đơn điện tử cho khóa học trả phí và sinh payload mã QR Napas247 chính xác số tiền cùng cú pháp định danh
// @Tags Billing & Store
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param id path string true "ID khóa học (UUID)" format(uuid)
// @Success 201 {object} response.Envelope{data=dto.CourseCheckoutResponse} "Tạo đơn mua và mã VietQR thành công"
// @Failure 400 {object} response.Envelope{error=response.ErrorDetail} "Khóa học không hợp lệ"
// @Router /api/v1/courses/{id}/checkout [post]
func (h *BillingHandler) CheckoutCourse(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

- **DTO Request & Response Structs:**
```go
type CourseCheckoutResponse struct {
	InvoiceID       uuid.UUID `json:"invoiceId" example:"81eebc99-9c0b-4ef8-bb6d-6bb9bd380811"`
	InvoiceCode     string    `json:"invoiceCode" example:"INV-2026-CRS-108"`
	CourseTitle     string    `json:"courseTitle" example:"Lập Trình Web Fullstack Go & Nuxt UI"`
	Amount          float64   `json:"amount" example:"1500000"`
	FormattedAmount string    `json:"formattedAmount" example:"1.500.000 đ"`
	BankID          string    `json:"bankId" example:"MB"`
	AccountNo       string    `json:"accountNo" example:"0988888888"`
	AccountName     string    `json:"accountName" example:"TRUNG TAM LMS CENTER"`
	TransferContent string    `json:"transferContent" example:"LMS CRS INV108"`
	VietQRImageURL  string    `json:"vietqrImageUrl" example:"https://img.vietqr.io/image/MB-0988888888-compact2.png?amount=1500000&addInfo=LMS%20CRS%20INV108"`
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

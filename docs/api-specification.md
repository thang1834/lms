# Đặc Tả Giao Diện Lập Trình Ứng Dụng (Go-chi REST API Specification)

> **Dự án:** LMS Center Platform (Hệ thống Quản lý Học tập, Giảng viên & Vận hành Đào tạo Đa hình thức)  
> **Tài liệu:** `docs/api-specification.md`  
> **Phiên bản:** 2.0.0 (Chuẩn hóa Go-chi RESTful API & Nuxt UI Client)  
> **Ngày cập nhật:** 23/09/2026  

---

## 1. Quy Chuẩn Thiết Kế API (Go-chi RESTful Conventions)

Toàn bộ các giao tiếp dữ liệu giữa Client (Nuxt UI) và Server (Go-chi) trong hệ thống tuân thủ nghiêm ngặt các nguyên tắc:
- **Chuẩn hóa phản hồi (JSON Response Envelope):**
  - **Thành công (200, 201):**
    ```json
    {
      "success": true,
      "data": { ... },
      "message": "Thông điệp thành công (nếu có)"
    }
    ```
  - **Thất bại (400, 401, 403, 404, 500):**
    ```json
    {
      "success": false,
      "error": {
        "code": "MA_LOI_CU_THE",
        "message": "Mô tả lỗi chi tiết thân thiện với người dùng",
        "details": [ ... ]
      }
    }
    ```
- **Xác thực dữ liệu (Input Validation):** 
  - Phía **Go Backend**: Xác thực dữ liệu đầu vào bằng Go Struct Tags (`validate:"required,email"`, thư viện `go-playground/validator/v10`) trước khi xử lý tầng Service.
  - Phía **Nuxt UI Client**: Xác thực form tức thời bằng Zod Schema tích hợp cùng component `<UForm>` của Nuxt UI.
- **Bảo mật & Session:** Mọi request được gắn context người dùng từ Bearer JWT token qua `JWTMiddleware` và phân quyền qua `RBACMiddleware` của Go-chi.

---

## 2. Danh Mục Các API Chi Tiết (Chi Router Groups)

```go
// Chi Router Definition (internal/api/router.go)
r.Route("/api", func(r chi.Router) {
    r.Use(middleware.Logger, middleware.Recoverer, middleware.CORS)
    // Public routes (Auth, VietQR verification)
    // Protected routes (JWT & RBAC Middleware)
})
```

### 2.1 Nhóm Quản Lý Giáo Viên & Lịch Dạy (Teachers & Schedule)

#### `GET /api/teachers`
- **Mô tả:** Lấy danh sách giảng viên và trợ giảng (hỗ trợ tìm kiếm, lọc theo chuyên môn).
- **Quyền hạn:** `ADMIN`.
- **Query Params:** `search`, `specialization`, `page`, `limit`.
- **Response 200:**
  ```json
  {
    "success": true,
    "data": {
      "teachers": [
        {
          "id": "t-uuid-1",
          "userId": "u-uuid-1",
          "fullName": "Nguyễn Văn Tuấn",
          "email": "tuan.nguyen@lms.edu.vn",
          "specialization": "Fullstack Web & React",
          "hourlyRate": 350000,
          "contractType": "FULLTIME",
          "activeClassesCount": 3
        }
      ],
      "total": 15
    }
  }
  ```

#### `GET /api/schedule/teacher`
- **Mô tả:** Lấy thời khóa biểu lịch dạy của giáo viên hiện tại theo khoảng thời gian.
- **Quyền hạn:** `TEACHER`, `ADMIN`.
- **Query Params:** `startDate=2026-09-21&endDate=2026-09-28`.
- **Response 200:**
  ```json
  {
    "success": true,
    "data": [
      {
        "sessionId": "sess-uuid-101",
        "className": "LMS-FE-K32",
        "courseTitle": "Lập trình React & Next.js",
        "sessionNumber": 5,
        "sessionDate": "2026-09-23",
        "startTime": "19:30",
        "endTime": "21:30",
        "classType": "HYBRID",
        "roomName": "Phòng Lab 302 - Cơ sở 1",
        "meetUrl": "https://meet.google.com/abc-xyz-lms",
        "isCheckIn": false,
        "topic": "Xây dựng Custom Hooks và Quản lý State"
      }
    ]
  }
  ```

#### `POST /api/attendance/teacher-checkin`
- **Mô tả:** Giảng viên check-in hoặc check-out ca dạy để chấm công giảng dạy thực tế.
- **Quyền hạn:** `TEACHER`.
- **Zod Schema:**
  ```typescript
  export const TeacherCheckInSchema = z.object({
    sessionId: z.string().uuid(),
    action: z.enum(["CHECK_IN", "CHECK_OUT"]),
    note: z.string().optional()
  });
  ```
- **Response 200:**
  ```json
  {
    "success": true,
    "data": {
      "attendanceId": "ta-uuid-1",
      "sessionId": "sess-uuid-101",
      "checkInTime": "2026-09-23T19:25:00.000Z",
      "checkOutTime": null,
      "status": "ON_TIME",
      "message": "Check-in ca dạy thành công lúc 19:25."
    }
  }
  ```

---

### 2.2 Nhóm Điểm Danh Học Sinh Đa Mô Hình (Student Attendance)

#### `GET /api/attendance/session/:sessionId`
- **Mô tả:** Lấy danh sách điểm danh học sinh của một buổi học cụ thể.
- **Quyền hạn:** `TEACHER`, `ADMIN`.
- **Response 200:**
  ```json
  {
    "success": true,
    "data": {
      "classInfo": {
        "name": "LMS-FE-K32",
        "classType": "HYBRID",
        "sessionNumber": 5
      },
      "students": [
        {
          "studentId": "st-uuid-1",
          "studentCode": "HV2026-001",
          "fullName": "Trần Thị Mai",
          "avatarUrl": "https://...",
          "status": "PRESENT_OFFLINE",
          "lateMinutes": 0,
          "isExcused": false,
          "teacherNote": "Nắm bài tốt, tích cực phát biểu"
        }
      ]
    }
  }
  ```

#### `POST /api/attendance/students`
- **Mô tả:** Giảng viên lưu bảng điểm danh học sinh cho buổi học (hỗ trợ lớp Hybrid: phân loại rõ Offline/Online).
- **Quyền hạn:** `TEACHER`, `ADMIN`.
- **Zod Schema:**
  ```typescript
  export const SaveStudentAttendanceSchema = z.object({
    sessionId: z.string().uuid(),
    records: z.array(z.object({
      studentId: z.string().uuid(),
      status: z.enum(["PRESENT_OFFLINE", "PRESENT_ONLINE", "LATE", "ABSENT"]),
      lateMinutes: z.number().int().min(0).default(0),
      isExcused: z.boolean().default(false),
      teacherNote: z.string().optional()
    }))
  });
  ```
- **Response 200:**
  ```json
  {
    "success": true,
    "message": "Đã lưu kết quả điểm danh cho 18 học sinh."
  }
  ```

---

### 2.3 Nhóm Quản Lý Tài Liệu & Bài Giảng YouTube (Materials & Lessons)

#### `POST /api/materials/upload`
- **Mô tả:** Upload và gắn tài liệu vào Bài giảng (PDF, Starter Code, Audio) hoặc Bài tập về nhà (Đề bài, Dataset).
- **Quyền hạn:** `TEACHER`, `ADMIN`.
- **Zod Schema:**
  ```typescript
  export const UploadMaterialSchema = z.object({
    title: z.string().min(3),
    fileUrl: z.string().url(),
    fileType: z.enum(["PDF_SLIDE", "CODE_STARTER", "AUDIO_MP3", "DATASET", "DOCUMENT"]),
    lessonId: z.string().uuid().optional(),
    assignmentId: z.string().uuid().optional(),
    isPublic: z.boolean().default(true)
  });
  ```

#### `POST /api/lessons/:lessonId/progress`
- **Mô tả:** Cập nhật thời lượng học viên đã xem video YouTube và lưu vết hoàn thành bài học.
- **Quyền hạn:** `STUDENT`.
- **Request Body:**
  ```json
  {
    "watchedSeconds": 750,
    "totalSeconds": 900,
    "isCompleted": true
  }
  ```
- **Response 200:**
  ```json
  {
    "success": true,
    "data": {
      "lessonId": "les-uuid-1",
      "isCompleted": true,
      "updatedCourseProgressPercent": 42
    }
  }
  ```

---

### 2.4 Nhóm Bài Tập Về Nhà, Monaco Code Editor & Chấm Điểm (Assignments & Grading)

#### `POST /api/assignments`
- **Mô tả:** Giảng viên tạo và giao bài tập về nhà cho lớp học.
- **Quyền hạn:** `TEACHER`, `ADMIN`.
- **Zod Schema:**
  ```typescript
  export const CreateAssignmentSchema = z.object({
    classId: z.string().uuid(),
    title: z.string().min(5),
    description: z.string().min(10), // Markdown content
    format: z.enum(["CODE_MONACO", "GITHUB_REPO", "FILE_UPLOAD", "AUDIO_RECORDING", "ESSAY"]),
    allowedLanguage: z.string().default("javascript"),
    deadline: z.string().datetime(),
    maxScore: z.number().int().positive().default(100)
  });
  ```

#### `POST /api/submissions`
- **Mô tả:** Học viên nộp bài làm trực tiếp trên web (mã nguồn Monaco Editor, link GitHub, file hoặc audio).
- **Quyền hạn:** `STUDENT`.
- **Zod Schema:**
  ```typescript
  export const SubmitAssignmentSchema = z.object({
    assignmentId: z.string().uuid(),
    codeContent: z.string().optional(), // Text code gõ trên Monaco Editor
    githubUrl: z.string().url().optional(),
    fileUrl: z.string().url().optional(),
    audioUrl: z.string().url().optional()
  });
  ```
- **Response 201:**
  ```json
  {
    "success": true,
    "data": {
      "submissionId": "sub-uuid-1",
      "submittedAt": "2026-09-23T20:15:00.000Z",
      "isLate": false,
      "status": "SUBMITTED",
      "message": "Nộp bài tập thành công!"
    }
  }
  ```

#### `POST /api/grading`
- **Mô tả:** Giảng viên hoặc Trợ giảng chấm điểm bài nộp và gửi nhận xét chi tiết.
- **Quyền hạn:** `TEACHER`, `ADMIN`.
- **Zod Schema:**
  ```typescript
  export const GradeSubmissionSchema = z.object({
    submissionId: z.string().uuid(),
    score: z.number().min(0).max(100),
    rubricCriteria: z.record(z.string(), z.number()).optional(),
    comment: z.string().min(5)
  });
  ```
- **Response 200:**
  ```json
  {
    "success": true,
    "data": {
      "gradeId": "gr-uuid-1",
      "score": 95,
      "gradedAt": "2026-09-24T08:30:00.000Z",
      "message": "Đã lưu điểm và nhận xét cho học viên."
    }
  }
  ```

---

### 2.5 Nhóm Thanh Toán Đa Kênh & VietQR (Billing & Payments)

#### `GET /api/invoices/:invoiceId/vietqr`
- **Mô tả:** Sinh thông tin mã VietQR động chuẩn Napas247 cho hóa đơn học phí.
- **Quyền hạn:** `STUDENT`, `PARENT`, `ADMIN`.
- **Response 200:**
  ```json
  {
    "success": true,
    "data": {
      "invoiceId": "inv-uuid-1",
      "invoiceCode": "INV-2026-0042",
      "studentName": "Trần Thị Mai",
      "amount": 3500000,
      "formattedAmount": "3.500.000 đ",
      "bankInfo": {
        "bankId": "MB",
        "bankName": "Ngân hàng Quân Đội (MBBank)",
        "accountNo": "0988888888",
        "accountName": "TRUNG TAM DAO TAO LMS CENTER"
      },
      "transferContent": "HP HV2026001 INV20260042",
      "qrImageUrl": "https://img.vietqr.io/image/MB-0988888888-compact2.png?amount=3500000&addInfo=HP%20HV2026001%20INV20260042&accountName=TRUNG%20TAM%20DAO%20TAO%20LMS%20CENTER"
    }
  }
  ```

#### `POST /api/payments/cash-confirm`
- **Mô tả:** Giáo vụ/thu ngân xác nhận đã thu tiền mặt tại quầy và tạo biên lai điện tử.
- **Quyền hạn:** `ADMIN`.
- **Zod Schema:**
  ```typescript
  export const ConfirmCashPaymentSchema = z.object({
    invoiceId: z.string().uuid(),
    amountReceived: z.number().positive(),
    note: z.string().optional()
  });
  ```
- **Response 200:**
  ```json
  {
    "success": true,
    "data": {
      "transactionId": "tx-uuid-1",
      "receiptPdfUrl": "/api/receipts/tx-uuid-1.pdf",
      "status": "PAID",
      "message": "Đã ghi nhận thu tiền mặt thành công. Biên lai PDF đã sẵn sàng."
    }
  }
  ```

---

### 2.6 Nhóm Thông Báo & Tin Nhắn Tự Động (Automated Notifications)

#### `POST /api/sessions/:sessionId/attendance-alert`
- **Mô tả:** Trigger quét điểm danh sau $N$ phút (mặc định 15p) kể từ giờ bắt đầu buổi học. Nếu lớp vắng, tự động bắn tin nhắn Zalo/SMS cho Phụ huynh học sinh vắng và báo cáo tổng hợp cho GV & Vận hành lớp. Nếu lớp đủ, gửi tin xác nhận sĩ số 100%.
- **Quyền hạn:** `SYSTEM_CRON`, `COORDINATOR`, `ADMIN`.
- **Response 200:**
  ```json
  {
    "success": true,
    "data": {
      "sessionId": "sess-uuid-101",
      "className": "LMS-FE-K32",
      "totalEnrolled": 20,
      "presentCount": 18,
      "absentCount": 2,
      "absentStudents": ["Nguyễn Văn A", "Trần Thị B"],
      "alertsSent": {
        "parentsNotified": 2,
        "coordinatorsNotified": 1,
        "teachersNotified": 1
      },
      "status": "ALERTED_ABSENT",
      "sentAt": "2026-09-23T19:45:00.000Z"
    }
  }
  ```

#### `POST /api/sessions/:sessionId/feedbacks/notify`
- **Mô tả:** Tự động gửi nhận xét buổi học của giáo viên tới từng Phụ huynh và Học sinh sau ca dạy.
- **Quyền hạn:** `TEACHER`, `COORDINATOR`, `ADMIN`.
- **Zod Schema:**
  ```typescript
  export const BroadcastFeedbackSchema = z.object({
    sessionId: z.string().uuid(),
    notifyChannels: z.array(z.enum(["ZALO_ZNS", "SMS", "IN_APP", "PUSH", "EMAIL"])).default(["IN_APP", "ZALO_ZNS"])
  });
  ```
- **Response 200:**
  ```json
  {
    "success": true,
    "data": {
      "sessionId": "sess-uuid-101",
      "recipientCount": 18,
      "dispatchedAt": "2026-09-23T21:40:00.000Z",
      "message": "Đã gửi nhận xét của giáo viên tới 18 phụ huynh và học viên."
    }
  }
  ```

#### `POST /api/cron/class-reminders`
- **Mô tả:** Cron job tự động quét các buổi học sắp diễn ra (trước 24h và trước 2h) để gửi tin nhắn nhắc nhở, đính kèm link phòng học trực tuyến (Meet/Zoom) và dặn dò đồ dùng học tập (`preparationNotes`).
- **Quyền hạn:** `SYSTEM_CRON` (Bảo vệ bằng `CRON_SECRET` Bearer Token).
- **Response 200:**
  ```json
  {
    "success": true,
    "data": {
      "scannedSessions": 8,
      "reminders24hSent": 45,
      "reminders2hSent": 38,
      "executedAt": "2026-09-23T10:00:00.000Z"
    }
  }
  ```

---

### 2.7 Nhóm Trình Phát Video & Chống Tua Cho Khóa Học Tự Học (Anti-Seeking Video Player)

#### `POST /api/courses/:courseId/lessons/:lessonId/heartbeat`
- **Mô tả:** Heartbeat định kỳ 5 giây gửi từ client trong lúc phát video bài giảng. Backend kiểm tra mốc thời gian xem hợp lệ (chống can thiệp `currentTime` bất thường) và mở khóa tua tự do khi hoàn thành 100%.
- **Quyền hạn:** `STUDENT`.
- **Zod Schema:**
  ```typescript
  export const VideoHeartbeatSchema = z.object({
    currentSeconds: z.number().nonnegative(),
    playbackRate: z.number().min(0.25).max(2.0).default(1.0),
    totalDuration: z.number().positive(),
    clientTimestamp: z.number()
  });
  ```
- **Response 200:**
  ```json
  {
    "success": true,
    "data": {
      "lessonId": "les-uuid-1",
      "maxWatchedSeconds": 150,
      "isCompleted": false,
      "allowFreeSeeking": false,
      "percentComplete": 45.2,
      "isCheatDetected": false
    }
  }
  ```

#### `GET /api/courses/:courseId/lessons/:lessonId/progress`
- **Mô tả:** Lấy thông tin tiến độ xem video và quyền tua của học viên khi mở bài học.
- **Quyền hạn:** `STUDENT`.
- **Response 200:**
  ```json
  {
    "success": true,
    "data": {
      "lessonId": "les-uuid-1",
      "watchedSeconds": 150,
      "maxWatchedSeconds": 150,
      "totalDuration": 320,
      "isCompleted": false,
      "allowFreeSeeking": false,
      "resumeAtSeconds": 150
    }
  }
  ```


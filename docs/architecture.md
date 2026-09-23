# Thiết Kế Kiến Trúc Hệ Thống & Cơ Sở Dữ Liệu (System Architecture & Database Design)

> **Dự án:** LMS Center Platform (Hệ thống Quản lý Học tập, Giảng viên & Vận hành Đào tạo Đa hình thức)  
> **Tài liệu:** `docs/architecture.md`  
> **Phiên bản:** 2.1.0 (Kiến trúc Go Backend chuẩn hóa theo blueprint `gmhafiz/go8` + Frontend Nuxt UI)  
> **Ngày cập nhật:** 23/09/2026  

---

## 1. Tổng Quan Kiến Trúc Hệ Thống (System Architecture Overview)

Hệ thống **LMS Center Platform** được thiết kế theo mô hình kiến trúc **Tách biệt Frontend - Backend (Decoupled Clean Architecture)** tối ưu hiệu năng cao:
- **Frontend Layer:** Xây dựng trên nền tảng **Nuxt 3/4 + Nuxt UI (Vue 3, TypeScript, Tailwind CSS v4, Pinia)** mang lại trải nghiệm tương tác mượt mà, hỗ trợ cả SSR (Server-Side Rendering cho SEO trang công khai) và SPA tốc độ cao cho các cổng Dashboard quản trị.
- **Backend API Layer:** Xây dựng bằng ngôn ngữ **Go (Golang 1.23+)** kế thừa trọn vẹn kiến trúc phân tầng chuyên nghiệp từ blueprint **[`gmhafiz/go8`](https://github.com/gmhafiz/go8)**:
  - **Router:** `go-chi/chi` v5 (tiêu chuẩn cộng đồng Go, 100% tương thích `net/http`).
  - **Mô hình Phân tầng (Layered Architecture):** `Handler` (Controller, DTO validation) $\to$ `UseCase` (Business logic) $\to$ `Repository` (Data access).
  - **Quản lý CSDL & Migrations:** Quản lý phiên bản bảng bằng **Goose migrations** (`database/migrations/*.sql`), kết nối truy vấn tốc độ cao qua `sqlx` / `GORM`.
  - **Quản lý Cấu hình:** Struct-based Configuration đọc từ `.env` qua `kelseyhightower/envconfig`.
  - **Dependency Injection:** Khởi tạo tường minh tại `internal/server/initDomains.go`.
  - **Task Runner:** Điều phối tự động hóa qua `Taskfile.yml` (`task dev`, `task migrate`, `task routes`, `task swagger`).
  - **Concurrency & Workers:** Tận dụng tối đa Go Goroutines & Channels cho Cron quét điểm danh sau 15p, nhắc lịch trước 24h/2h và Heartbeat chống tua video mà không cần Redis.
- **Database Layer:** **PostgreSQL 16** đảm bảo toàn vẹn dữ liệu, giao dịch ACID tin cậy và tốc độ truy vấn tối ưu.

```mermaid
graph TD
    subgraph FrontendLayer ["1. Tầng Giao Diện Người Dùng (Nuxt UI + Vue 3)"]
        AdminUI["Admin & Academic Manager Portal (/admin)"]
        CoordUI["Class Coordinator / Care Portal (/operations)"]
        TeacherUI["Teacher Portal (/teacher)"]
        StudentUI["Student Portal (/student)"]
        ParentUI["Parent Portal (/parent)"]
        Monaco["Monaco Editor (Vue Component)"]
        RestrictedPlayer["Restricted Video Player (Anti-Seeking Component)"]
        QRCard["VietQR Display Component"]
    end

    subgraph Go8Backend ["2. Tầng Backend Hiệu Năng Cao (gmhafiz/go8 Architecture)"]
        subgraph Middlewares ["Chi Middleware Stack (internal/middleware)"]
            Logger["chi/middleware.Logger & RequestID"]
            Recoverer["chi/middleware.Recoverer"]
            CORS["cors.Handler (Allowed Origins)"]
            JWTAuth["JWT Authentication Middleware"]
            RBACAuth["Dynamic RBAC Permission Guard"]
        end

        subgraph Domains ["Domain Packages (internal/domain/{feature})"]
            subgraph DomainStructure ["Chuẩn 3 tầng mỗi Domain"]
                Handler["Handler (Parse DTO, Validator v10)"]
                UseCase["Use Case (Business Logic)"]
                Repo["Repository (Postgres / sqlx / GORM)"]
                Handler --> UseCase
                UseCase --> Repo
            end
            
            AuthD["auth: Login, Register, RBAC"]
            ClassD["class: Schedule, Recurrence, Reschedule"]
            AttendD["attendance: Student Attendance, Timesheet"]
            VideoD["course_video: Anti-Seeking, Heartbeat"]
            AssignD["assignment: Monaco Submissions, Grading"]
            NotifD["notification: Automated Message Engine"]
            BillD["billing: VietQR, Invoices, Cash"]
            CertD["certificate: Public Verifiable Certs"]
        end

        subgraph GoWorkers ["Goroutine Workers & Cron (internal/worker)"]
            AttendanceAlertWorker["Attendance Alert Worker (sau 15p)"]
            ReminderWorker["Class Reminder Worker (24h & 2h)"]
            MessageDispatcher["Async Message Dispatcher Pool (Zalo/SMS/Email)"]
        end

        subgraph ServerCore ["Server Core & DI (internal/server)"]
            ServerStruct["Server Struct & Configs (envconfig)"]
            InitDomains["initDomains.go (Dependency Injection)"]
        end
    end

    subgraph DataLayer ["3. Tầng Dữ Liệu (Data Layer)"]
        Postgres[(PostgreSQL 16 Database)]
        GooseMigrations["Goose SQL Migrations (database/migrations)"]
    end

    subgraph ExternalServices ["4. Dịch Vụ Bên Ngoài (External Gateways)"]
        YouTube["YouTube IFrame API"]
        ZaloZNS["Zalo ZNS / SMS Gateway"]
        VietQR["Napas247 VietQR Generator"]
        Storage["Cloud Storage (S3 / Cloudinary)"]
        PDFEngine["Go PDF Invoice & Certificate Generator"]
    end

    FrontendLayer --> Middlewares
    Middlewares --> Handler
    Repo --> Postgres
    GooseMigrations --> Postgres
    GoWorkers --> Domains
    Domains --> ExternalServices
    FrontendLayer -.-> YouTube
    RestrictedPlayer -.-> VideoD
```



---

## 2. Thiết Kế Cơ Sở Dữ Liệu Toàn Diện (Database Schema & ERD)

### 2.1 Sơ Đồ Thực Thể Quan Hệ (Entity Relationship Diagram - ERD)

```mermaid
erDiagram
    User ||--o{ UserRole : has
    Role ||--o{ UserRole : assigned_to
    Role ||--o{ RolePermission : contains
    Permission ||--o{ RolePermission : defines

    User ||--o{ TeacherProfile : has
    User ||--o{ StudentProfile : has
    User ||--o{ ParentProfile : has
    ParentProfile ||--o{ ParentStudent : connects
    StudentProfile ||--o{ ParentStudent : belongs_to
    
    Course ||--o{ Module : contains
    Module ||--o{ Lesson : contains
    Lesson ||--o{ Material : includes
    Course ||--o{ GradebookConfig : defines
    
    Course ||--o{ Class : instances
    TeacherProfile ||--o{ Class : teaches_main
    TeacherProfile ||--o{ Class : teaches_ta
    User ||--o{ Class : coordinates
    
    Class ||--o{ ClassSession : schedules
    Class ||--o{ Enrollment : registers
    StudentProfile ||--o{ Enrollment : joins
    
    ClassSession ||--o{ TeacherAttendance : logs
    ClassSession ||--o{ StudentAttendance : logs
    StudentProfile ||--o{ StudentAttendance : records
    ClassSession ||--o{ SessionFeedback : receives_student_review
    StudentProfile ||--o{ SessionFeedback : submits_review
    
    TeacherProfile ||--o{ TeacherEvaluation : evaluated
    User ||--o{ TeacherEvaluation : audits

    Lesson ||--o{ LessonDiscussion : has_qa
    User ||--o{ LessonDiscussion : posts_qa
    Lesson ||--o{ LessonNote : has_notes
    StudentProfile ||--o{ LessonNote : writes_notes
    Lesson ||--o{ LessonProgress : tracks
    StudentProfile ||--o{ LessonProgress : achieves
    
    ClassSession ||--o{ NotificationLog : triggers
    NotificationTemplate ||--o{ NotificationLog : formats
    User ||--o{ NotificationLog : receives
    
    Class ||--o{ Assignment : assigns
    Assignment ||--o{ Material : attaches
    Assignment ||--o{ Submission : receives
    StudentProfile ||--o{ Submission : submits
    Submission ||--o{ GradeFeedback : grades
    TeacherProfile ||--o{ GradeFeedback : evaluates
    
    StudentProfile ||--o{ Certificate : awarded
    Course ||--o{ Certificate : certifies
    
    StudentProfile ||--o{ TuitionInvoice : billed_to
    TuitionInvoice ||--o{ PaymentTransaction : settles
```


---

### 2.2 Từ Điển Dữ Liệu Chi Tiết (Data Dictionary)

#### Nhóm 1: Hệ Thống Phân Quyền Động Chuẩn RBAC (Core Dynamic RBAC)

1. **`users`**:
   - `id` (UUID, PK): Mã người dùng duy nhất.
   - `email` (String, Unique): Email đăng nhập.
   - `passwordHash` (String): Mật khẩu mã hóa bcrypt.
   - `fullName` (String): Họ và tên.
   - `phone` (String, Nullable): Số điện thoại liên hệ.
   - `avatarUrl` (String, Nullable): Ảnh đại diện.
   - `isActive` (Boolean, Default `true`): Trạng thái kích hoạt.
   - `createdAt`, `updatedAt` (Timestamp).

2. **`roles`** (Bảng quản lý vai trò):
   - `id` (UUID, PK).
   - `code` (String, Unique): Mã định danh (ví dụ: `SUPER_ADMIN`, `ACADEMIC_MANAGER`, `CLASS_COORDINATOR`, `TEACHER`, `TA`, `STUDENT`, `PARENT`).
   - `name` (String): Tên hiển thị (ví dụ: "Quản lý Đào tạo / Giáo vụ trưởng", "Chuyên viên Vận hành & CSKH lớp").
   - `description` (String, Nullable): Mô tả quyền hạn.
   - `isSystem` (Boolean, Default `false`): Role hệ thống mặc định (không được xóa).

3. **`permissions`** (Bảng quản lý quyền hạn chi tiết - Granular Permissions):
   - `id` (UUID, PK).
   - `code` (String, Unique): Mã quyền (ví dụ: `classes.create`, `classes.schedule`, `classes.reschedule`, `teachers.evaluate`, `attendance.record_student`, `attendance.record_teacher`, `students.care_notes`, `finance.manage_invoices`).
   - `name` (String): Tên quyền (ví dụ: "Xếp thời khóa biểu lớp học", "Đánh giá chất lượng giáo viên").
   - `module` (String): Phân hệ (ví dụ: `CLASSES`, `TEACHERS`, `ATTENDANCE`, `FINANCE`).
   - `description` (String, Nullable).

4. **`role_permissions`** (Bảng trung gian N-N gán quyền cho vai trò):
   - `id` (UUID, PK).
   - `roleId` (UUID, FK -> `roles.id`).
   - `permissionId` (UUID, FK -> `permissions.id`).
   - Unique Constraint: `(roleId, permissionId)`.

5. **`user_roles`** (Bảng trung gian N-N gán vai trò cho người dùng):
   - `id` (UUID, PK).
   - `userId` (UUID, FK -> `users.id`).
   - `roleId` (UUID, FK -> `roles.id`).
   - `assignedAt` (Timestamp, Default `now()`).
   - `assignedBy` (UUID, Nullable, FK -> `users.id`): Người cấp quyền.
   - Unique Constraint: `(userId, roleId)`.

---

#### Nhóm 2: Hồ Sơ Người Dùng & Quan Hệ Gia Đình (Profiles)

6. **`teacher_profiles`**:
   - `id` (UUID, PK).
   - `userId` (UUID, FK -> `users.id`, Unique).
   - `specialization` (String): Lĩnh vực chuyên môn (Frontend, Backend, Python AI, IELTS...).
   - `bio` (Text, Nullable): Kinh nghiệm & chứng chỉ.
   - `hourlyRate` (Decimal): Đơn giá thù lao giờ dạy (VNĐ).
   - `contractType` (Enum: `FULLTIME`, `PARTTIME`, `VISITING`).
   - `kpiRating` (Decimal, Default `5.0`): Điểm đánh giá trung bình từ học viên và giáo vụ.

7. **`student_profiles`**:
   - `id` (UUID, PK).
   - `userId` (UUID, FK -> `users.id`, Unique).
   - `studentCode` (String, Unique): Mã học viên (ví dụ: `HV-2026-001`).
   - `dateOfBirth` (Date, Nullable).
   - `currentLevel` (String, Nullable).

8. **`parent_profiles`**:
   - `id` (UUID, PK).
   - `userId` (UUID, FK -> `users.id`, Unique).
   - `address` (String, Nullable).

9. **`parent_students`**:
   - `id` (UUID, PK).
   - `parentId` (UUID, FK -> `parent_profiles.id`).
   - `studentId` (UUID, FK -> `student_profiles.id`).
   - `relationship` (String): Bố, Mẹ, Người giám hộ.
   - Unique Constraint: `(parentId, studentId)`.

---

#### Nhóm 3: Đào Tạo, Khóa Học & Lớp Học Đa Hình Thức (Courses & Classes)

10. **`courses`**:
    - `id` (UUID, PK).
    - `title` (String): Tên khóa học.
    - `slug` (String, Unique).
    - `description` (Text).
    - `courseType` (Enum: `SELF_PACED_ONLINE`, `INSTRUCTOR_LED`): Phân loại khóa học tự học (cấm tua video) hay lớp có giáo viên.
    - `subjectType` (Enum: `IT`, `LANGUAGE`, `GENERAL`).
    - `thumbnailUrl` (String, Nullable).
    - `price` (Decimal): Học phí niêm yết.
    - `isPublished` (Boolean, Default `false`).

11. **`gradebook_configs`** (Cấu hình trọng số điểm theo chuẩn Moodle):
    - `id` (UUID, PK).
    - `courseId` (UUID, FK -> `courses.id`, Unique).
    - `attendanceWeight` (Int, Default `10`): % điểm chuyên cần.
    - `homeworkWeight` (Int, Default `30`): % điểm bài tập về nhà.
    - `midtermWeight` (Int, Default `20`): % điểm thi giữa kỳ.
    - `finalProjectWeight` (Int, Default `40`): % điểm đồ án / thi cuối khóa.
    - *Ràng buộc:* Tổng trọng số phải bằng 100%.

12. **`modules`**:
    - `id` (UUID, PK).
    - `courseId` (UUID, FK -> `courses.id`).
    - `title` (String): Tên chương.
    - `orderIndex` (Int): Thứ tự.

13. **`lessons`**:
    - `id` (UUID, PK).
    - `moduleId` (UUID, FK -> `modules.id`).
    - `title` (String): Tên bài học.
    - `orderIndex` (Int).
    - `youtubeUrl` (String, Nullable): Link video YouTube (Unlisted/Public/Playlist).
    - `content` (Text, Nullable): Nội dung tóm tắt Markdown.
    - `durationMinutes` (Int, Default `0`).
    - `isDripLocked` (Boolean, Default `false`): Mở khóa tuần tự theo tiến độ học (tính năng kế thừa Moodle).

14. **`classes`** (Lớp học với sự tham gia của 3 vai trò: Giáo viên, Trợ giảng và Vận hành/CSKH):
    - `id` (UUID, PK).
    - `courseId` (UUID, FK -> `courses.id`).
    - `name` (String): Mã lớp (ví dụ: `FE-K32`).
    - `classType` (Enum: `OFFLINE`, `ONLINE_VIRTUAL`, `HYBRID`).
    - `mainTeacherId` (UUID, FK -> `teacher_profiles.id`): Giảng viên chính.
    - `taTeacherId` (UUID, FK -> `teacher_profiles.id`, Nullable): Trợ giảng.
    - `coordinatorId` (UUID, FK -> `users.id`, Nullable): **Chuyên viên Vận hành lớp & Chăm sóc học viên (Class Coordinator / Care)**.
    - `roomName` (String, Nullable): Tên phòng học mặc định.
    - `meetUrl` (String, Nullable): Link Google Meet / Zoom mặc định.
    - `startDate` (Date), `endDate` (Date).
    - `scheduleRule` (JSON): Quy tắc lịch học lặp lại (ví dụ: `[{"dayOfWeek": 2, "startTime": "19:30", "endTime": "21:30"}, {"dayOfWeek": 4, "startTime": "19:30", "endTime": "21:30"}]`).
    - `attendanceAlertMinutes` (Int, Default `15`): Thời gian phát cảnh báo sĩ số vắng/đủ sau khi buổi học bắt đầu.
    - `reminderHours` (JSON, Default `[24, 2]`): Các mốc thời gian tự động bắn tin nhắn nhắc lịch học trước buổi.
    - `allowVideoSeeking` (Boolean, Default `false`): Mặc định cấm tua đối với video tự học.
    - `maxStudents` (Int, Default `20`).
    - `status` (Enum: `UPCOMING`, `ACTIVE`, `COMPLETED`, `CANCELLED`).

15. **`class_sessions`** (Buổi học linh hoạt - Flexible Session Scheduling):
    - `id` (UUID, PK): Buổi học cụ thể.
    - `classId` (UUID, FK -> `classes.id`).
    - `sessionNumber` (Int): Buổi số (1, 2, 3...).
    - `sessionDate` (Date): Ngày diễn ra buổi học.
    - `startTime` (Time): Giờ bắt đầu (có thể điều chỉnh linh hoạt từng buổi).
    - `endTime` (Time): Giờ kết thúc.
    - `assignedTeacherId` (UUID, FK -> `teacher_profiles.id`): Giáo viên phụ trách buổi này (hỗ trợ phân công giáo viên dạy thay).
    - `roomName` (String, Nullable): Phòng học riêng của buổi này (nếu đổi phòng).
    - `meetUrl` (String, Nullable): Link meet riêng của buổi này.
    - `meetingPlatform` (Enum: `GOOGLE_MEET`, `ZOOM`, `MS_TEAMS`, `JITSI`, Nullable).
    - `preparationNotes` (Text, Nullable): Dặn dò chuẩn bị trước giờ học (laptop, tài liệu, bài tập nợ) để gửi tin nhắn nhắc nhở.
    - `topic` (String, Nullable): Chủ đề giảng dạy.
    - `status` (Enum: `SCHEDULED`, `RESCHEDULED`, `IN_PROGRESS`, `COMPLETED`, `CANCELLED`).
    - `attendanceAlertSentAt` (Timestamp, Nullable): Thời điểm hệ thống đã tự động gửi tin nhắn báo sĩ số.
    - `rescheduleReason` (Text, Nullable): Lý do dời lịch hoặc bù buổi.
    - `originalDate` (Date, Nullable): Ngày học ban đầu nếu là buổi dời/bù.

16. **`enrollments`**:
    - `id` (UUID, PK).
    - `classId` (UUID, FK -> `classes.id`).
    - `studentId` (UUID, FK -> `student_profiles.id`).
    - `enrolledAt` (Timestamp).
    - `status` (Enum: `STUDYING`, `DROPPED`, `COMPLETED`).

---

#### Nhóm 4: Điểm Danh, Chấm Công & Đánh Giá Chất Lượng Giáo Viên

17. **`teacher_attendance`** (Chấm công ca dạy giáo viên):
    - `id` (UUID, PK).
    - `sessionId` (UUID, FK -> `class_sessions.id`, Unique).
    - `teacherId` (UUID, FK -> `teacher_profiles.id`).
    - `checkInTime` (Timestamp, Nullable).
    - `checkOutTime` (Timestamp, Nullable).
    - `actualDurationMinutes` (Int, Default `0`).
    - `note` (Text, Nullable).
    - `status` (Enum: `ON_TIME`, `LATE`, `ABSENT`, `SUBSTITUTED`).

18. **`student_attendance`** (Điểm danh học sinh đa mô hình):
    - `id` (UUID, PK).
    - `sessionId` (UUID, FK -> `class_sessions.id`).
    - `studentId` (UUID, FK -> `student_profiles.id`).
    - `status` (Enum: `PRESENT_OFFLINE`, `PRESENT_ONLINE`, `LATE`, `ABSENT`).
    - `lateMinutes` (Int, Default `0`).
    - `isExcused` (Boolean, Default `false`).
    - `teacherNote` (String, Nullable).
    - `coordinatorNote` (String, Nullable): **Ghi chú chăm sóc của Chuyên viên Vận hành lớp** (ví dụ: "Đã gọi phụ huynh, học sinh bị ốm xin nghỉ").
    - Unique Constraint: `(sessionId, studentId)`.

19. **`teacher_evaluations`** (Đánh giá giáo viên & QA Đào tạo định kỳ):
    - `id` (UUID, PK).
    - `teacherId` (UUID, FK -> `teacher_profiles.id`).
    - `classId` (UUID, FK -> `classes.id`, Nullable).
    - `evaluatorId` (UUID, FK -> `users.id`): Người đánh giá (Academic Manager hoặc Học sinh đánh giá ẩn danh).
    - `evaluationType` (Enum: `MANAGER_AUDIT`, `STUDENT_SURVEY`).
    - `criteriaScores` (JSON): Điểm chi tiết (Kiến thức chuyên môn, Kỹ năng sư phạm, Tác phong đúng giờ, Độ nhiệt tình hỗ trợ...).
    - `overallScore` (Decimal): Điểm trung bình (Thang điểm 5 hoặc 10).
    - `comment` (Text): Nhận xét góp ý.
    - `evaluatedAt` (Timestamp).

20. **`session_feedbacks`** (Đánh giá giáo viên theo từng buổi học từ học sinh - Per-Session Student Feedback):
    - `id` (UUID, PK).
    - `sessionId` (UUID, FK -> `class_sessions.id`): Buổi học được đánh giá.
    - `studentId` (UUID, FK -> `student_profiles.id`): Học sinh đánh giá.
    - `teacherRating` (Int, 1-5 sao): Đánh giá chất lượng giảng viên trong buổi.
    - `taRating` (Int, Nullable, 1-5 sao): Đánh giá trợ giảng trong buổi.
    - `understandingLevel` (Enum: `POOR`, `AVERAGE`, `GOOD`, `EXCELLENT`): Mức độ hiểu bài trong buổi.
    - `pacingFeedback` (Enum: `TOO_SLOW`, `JUST_RIGHT`, `TOO_FAST`): Tốc độ giảng bài của giáo viên.
    - `comment` (Text, Nullable): Góp ý cụ thể (ẩn danh).
    - `isAnonymous` (Boolean, Default `true`): Ẩn danh tính với giáo viên (chỉ Quản lý Đào tạo & Vận hành lớp xem được thống kê).
    - `createdAt` (Timestamp, Default `now()`).
    - Unique Constraint: `(sessionId, studentId)`: Mỗi học sinh chỉ đánh giá 1 lần mỗi buổi học.

---

#### Nhóm 5: Tài Liệu, BTVN, Monaco Editor & Chấm Điểm

20. **`materials`**:
    - `id` (UUID, PK).
    - `title` (String): Tên tài liệu.
    - `fileUrl` (String): Link Cloud Storage.
    - `fileType` (Enum: `PDF_SLIDE`, `CODE_STARTER`, `AUDIO_MP3`, `DATASET`, `DOCUMENT`).
    - `lessonId` (UUID, FK -> `lessons.id`, Nullable).
    - `assignmentId` (UUID, FK -> `assignments.id`, Nullable).
    - `isPublic` (Boolean, Default `true`).

21. **`assignments`**:
    - `id` (UUID, PK).
    - `classId` (UUID, FK -> `classes.id`).
    - `title` (String).
    - `description` (Text): Hướng dẫn đề bài Markdown.
    - `format` (Enum: `CODE_MONACO`, `GITHUB_REPO`, `FILE_UPLOAD`, `AUDIO_RECORDING`, `ESSAY`).
    - `allowedLanguage` (String, Default `"javascript"`).
    - `deadline` (Timestamp).
    - `maxScore` (Int, Default `100`).

22. **`submissions`**:
    - `id` (UUID, PK).
    - `assignmentId` (UUID, FK -> `assignments.id`).
    - `studentId` (UUID, FK -> `student_profiles.id`).
    - `codeContent` (Text, Nullable): Code học viên viết trên Monaco Editor.
    - `githubUrl` (String, Nullable).
    - `fileUrl` (String, Nullable).
    - `submittedAt` (Timestamp).
    - `isLate` (Boolean, Default `false`).
    - `status` (Enum: `SUBMITTED`, `GRADED`, `RESUBMIT_REQUIRED`).
    - Unique Constraint: `(assignmentId, studentId)`.

23. **`grade_feedbacks`**:
    - `id` (UUID, PK).
    - `submissionId` (UUID, FK -> `submissions.id`, Unique).
    - `teacherId` (UUID, FK -> `teacher_profiles.id`).
    - `score` (Decimal).
    - `rubricCriteria` (JSON, Nullable).
    - `comment` (Text).
    - `gradedAt` (Timestamp).

---

#### Nhóm 6: Tài Chính & Học Phí Đa Kênh

24. **`tuition_invoices`**:
    - `id` (UUID, PK).
    - `invoiceCode` (String, Unique).
    - `studentId` (UUID, FK -> `student_profiles.id`).
    - `classId` (UUID, FK -> `classes.id`, Nullable).
    - `title` (String).
    - `amount` (Decimal), `paidAmount` (Decimal, Default `0`).
    - `dueDate` (Date).
    - `status` (Enum: `PENDING`, `PARTIALLY_PAID`, `PAID`, `OVERDUE`).
    - `vietqrPayload` (Text, Nullable).

25. **`payment_transactions`**:
    - `id` (UUID, PK).
    - `invoiceId` (UUID, FK -> `tuition_invoices.id`).
    - `amount` (Decimal).
    - `method` (Enum: `VIETQR`, `CREDIT_CARD`, `CASH`, `BANK_TRANSFER`).
    - `transactionRef` (String, Nullable).
    - `receivedByUserId` (UUID, FK -> `users.id`, Nullable).
    - `receiptPdfUrl` (String, Nullable).
    - `paidAt` (Timestamp).

---

#### Nhóm 7: Tính Năng Khóa Học Online Chuyên Sâu (Kế Thừa Từ Frappe LMS)

26. **`lesson_discussions`** (Hỏi đáp & Thảo luận ngay trong bài học có Timestamp):
    - `id` (UUID, PK).
    - `lessonId` (UUID, FK -> `lessons.id`).
    - `userId` (UUID, FK -> `users.id`).
    - `videoTimestampSeconds` (Int, Nullable): Giây thứ mấy trong video bài giảng YouTube mà học viên đang thắc mắc (click vào nhảy ngay đến đoạn video đó).
    - `content` (Text): Nội dung câu hỏi hoặc phản hồi.
    - `parentId` (UUID, Nullable, FK -> `lesson_discussions.id`): Phản hồi cho câu hỏi nào (Threaded Q&A).
    - `isVerifiedAnswer` (Boolean, Default `false`): Dấu tích xanh xác nhận câu trả lời chính xác từ Giảng viên / Trợ giảng.
    - `createdAt` (Timestamp, Default `now()`).

27. **`lesson_notes`** (Ghi chú cá nhân của học sinh trong khi xem bài giảng):
    - `id` (UUID, PK).
    - `lessonId` (UUID, FK -> `lessons.id`).
    - `studentId` (UUID, FK -> `student_profiles.id`).
    - `videoTimestampSeconds` (Int, Nullable): Mốc thời gian video lúc ghi chú.
    - `noteText` (Text): Nội dung ghi chú cá nhân (Markdown).
    - `createdAt`, `updatedAt` (Timestamp).

28. **`certificates`** (Chứng chỉ hoàn thành khóa học có URL xác thực công khai - Public Verifiable Certificate):
    - `id` (UUID, PK).
    - `certificateCode` (String, Unique): Mã chứng chỉ duy nhất (ví dụ: `CERT-2026-REACT-8F29`).
    - `studentId` (UUID, FK -> `student_profiles.id`).
    - `courseId` (UUID, FK -> `courses.id`).
    - `classId` (UUID, FK -> `classes.id`).
    - `issueDate` (Date).
    - `gradeClassification` (String): Xếp loại (Xuất sắc, Giỏi, Khá).
    - `verificationSlug` (String, Unique): Đường dẫn kiểm tra công khai `/verify/[certificateCode]`.
    - `pdfUrl` (String): Link file PDF chứng chỉ có mã QR xác thực.

---

#### Nhóm 8: Tiến Độ Học Tập & Kiểm Soát Video Chống Tua (Anti-Seeking Video Progress)

29. **`lesson_progress`** (Quản lý tiến độ xem video & cấm tua đối phó):
    - `id` (UUID, PK).
    - `lessonId` (UUID, FK -> `lessons.id`).
    - `studentId` (UUID, FK -> `student_profiles.id`).
    - `watchedSeconds` (Int, Default `0`): Tổng số giây đã thực sự xem.
    - `maxWatchedSeconds` (Int, Default `0`): Mốc giây xa nhất đã xem (dùng để chặn thao tác tua vượt mốc).
    - `totalDuration` (Int, Default `0`): Tổng thời lượng bài giảng (lấy từ YouTube API).
    - `isCompleted` (Boolean, Default `false`): Đánh dấu hoàn thành khi `maxWatchedSeconds >= totalDuration * 0.95`.
    - `allowFreeSeeking` (Boolean, Default `false`): Chỉ bật `true` khi học sinh đã hoàn thành bài học lần đầu tiên, cho phép tự do tua để ôn tập.
    - `lastHeartbeatAt` (Timestamp): Mốc thời gian lần heartbeat gần nhất để chống nhảy cóc (cheat detection).
    - Unique Constraint: `(lessonId, studentId)`.

---

#### Nhóm 9: Hệ Thống Thông Báo & Tin Nhắn Tự Động (Automated Notification Engine)

30. **`notification_templates`** (Mẫu nội dung thông báo đa kênh):
    - `id` (UUID, PK).
    - `code` (String, Unique): Mã mẫu thông báo (`ATTENDANCE_ALERT_ABSENT`, `ATTENDANCE_ALERT_FULL`, `TEACHER_SESSION_FEEDBACK`, `CLASS_REMINDER_24H`, `CLASS_REMINDER_2H`).
    - `title` (String): Tiêu đề thông báo.
    - `contentTemplate` (Text): Mẫu nội dung hỗ trợ placeholder (`{{studentName}}`, `{{className}}`, `{{sessionTime}}`, `{{meetingUrl}}`, `{{feedbackSummary}}`).
    - `channels` (JSON): Mảng kênh áp dụng (ví dụ: `["ZALO_ZNS", "SMS", "IN_APP", "PUSH", "EMAIL"]`).
    - `isActive` (Boolean, Default `true`).

31. **`notification_logs`** (Nhật ký phát tin nhắn & trạng thái chuyển phát):
    - `id` (UUID, PK).
    - `templateCode` (String): Mã mẫu áp dụng.
    - `recipientUserId` (UUID, FK -> `users.id`): Người nhận (Phụ huynh, Học sinh, Giáo viên hoặc Vận hành).
    - `recipientPhone` (String, Nullable): Số điện thoại nhận Zalo/SMS.
    - `sessionId` (UUID, FK -> `class_sessions.id`, Nullable): Gắn với buổi học nào.
    - `channel` (Enum: `ZALO_ZNS`, `SMS`, `IN_APP`, `PUSH`, `EMAIL`).
    - `payload` (JSON): Dữ liệu truyền vào template.
    - `sentStatus` (Enum: `PENDING`, `SENT`, `DELIVERED`, `FAILED`).
    - `errorDetails` (Text, Nullable).
    - `createdAt` (Timestamp, Default `now()`).

---

## 3. Ma Trận Phân Quyền Hạt Nhân RBAC (Granular RBAC Matrix)

| Chức năng / Permission Code | Super Admin | Academic Manager (Giáo vụ) | Class Coordinator (Vận hành/CSKH) | Teacher (Giảng viên) | Student (Học viên) | Parent (Phụ huynh) |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| Quản trị hệ thống, Cấu hình RBAC | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Tạo khóa học, phân công giáo viên | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Xếp lớp, cấu hình lịch học linh hoạt | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Dời lịch học / đổi phòng / đổi Meet link | ✅ | ✅ | ✅ (lớp phụ trách) | ⚠️ (đề xuất) | ❌ | ❌ |
| Cấu hình thời gian gửi tin tự động | ✅ | ✅ | ✅ (lớp phụ trách) | ❌ | ❌ | ❌ |
| Đánh giá chất lượng giáo viên (Audit) | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Đánh giá giáo viên theo từng buổi | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ |
| Ghi chú chăm sóc học sinh vắng | ✅ | ✅ | ✅ (lớp phụ trách) | ❌ | ❌ | ❌ |
| Check-in/out ca dạy (chấm công) | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ |
| Điểm danh học sinh lớp Hybrid | ✅ | ✅ | ✅ (hỗ trợ) | ✅ | ❌ | ❌ |
| Upload tài liệu học tập, code mẫu | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ |
| Giao bài tập, chấm điểm theo rubric | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ |
| Làm BTVN trên Monaco Code Editor | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ |
| Xem chuyên cần, điểm số của con | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Thanh toán học phí VietQR, thẻ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| Xác nhận thu tiền mặt tại quầy | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |

---

## 4. Cơ Chế Xếp Lịch Học Linh Hoạt (Flexible Scheduling Engine)

Thuật toán sinh và điều chỉnh buổi học:
1. **Lịch định kỳ chuẩn (Base Recurrence):**
   - Khi mở lớp, Quản lý đào tạo chọn tần suất:
     - *1 buổi/tuần:* Chọn thứ (ví dụ: Chủ nhật 08:30 - 11:30).
     - *Nhiều buổi/tuần:* Chọn các thứ (ví dụ: T2-T4-T6 hoặc T3-T5 lúc 19:30 - 21:30).
   - Hệ thống tự động tạo trước danh sách $N$ buổi học tương ứng trong bảng `class_sessions`.
2. **Tùy biến từng buổi học (Per-Session Customization):**
   - Mỗi buổi học trong `class_sessions` là một thực thể độc lập có `id` riêng.
   - Quản lý đào tạo hoặc Điều phối viên lớp có thể:
     - **Dời ngày học (Reschedule):** Chuyển từ ngày $D$ sang ngày $D'$; hệ thống tự động lưu `originalDate` và lý do `rescheduleReason`.
     - **Đổi phòng học / Link Meet:** Cập nhật `roomName` hoặc `meetUrl` riêng cho buổi đó nếu phòng cũ bảo trì.
     - **Phân công giáo viên dạy thay:** Cập nhật `assignedTeacherId` riêng cho buổi học khi giáo viên chính bận đột xuất.
     - **Bù buổi:** Thêm một buổi học mới ngoài lịch cố định.
   - Mọi thay đổi lịch học được bắn thông báo tức thời (Email/Push) đến Giảng viên, Học viên và Phụ huynh.

---

## 5. Kiến Trúc Động Cơ Tin Nhắn & Thông Báo Tự Động (Automated Notification Engine)

```mermaid
sequenceDiagram
    autonumber
    participant Cron as Cron Task / Scheduler
    participant Engine as NotificationEngineService
    participant DB as PostgreSQL (GORM)
    participant Gateway as Zalo ZNS / SMS Gateway
    participant Recipient as Parent / Teacher / Coordinator

    Note over Cron, Engine: Kịch bản 1: Cảnh báo điểm danh sau N phút (15 phút)
    Cron->>Engine: Trigger checkAttendanceAlert()
    Engine->>DB: Truy vấn ca học đang diễn ra (now - startTime >= alertMinutes)
    DB-->>Engine: Danh sách buổi học & trạng thái điểm danh
    alt Lớp có học sinh vắng mặt
        Engine->>Gateway: Gửi Zalo/SMS tới Phụ huynh của từng học sinh vắng
        Engine->>Gateway: Gửi tin nhắn tổng hợp vắng X/Y tới Giảng viên & Vận hành lớp
    else Lớp đã đi đủ 100%
        Engine->>Gateway: Gửi tin nhắn chúc mừng sĩ số đủ 100% tới Giảng viên & Vận hành
    end
    Engine->>DB: Đánh dấu class_sessions.attendanceAlertSentAt = now()

    Note over Cron, Engine: Kịch bản 2: Nhắc nhở lịch học trước 24h & 2h
    Cron->>Engine: Trigger scanUpcomingSessions()
    Engine->>DB: Lấy ca học diễn ra trong 24h tới và 2h tới
    DB-->>Engine: Danh sách ca học kèm preparationNotes & meetUrl
    Engine->>Gateway: Bắn thông báo nhắc lịch học, dặn dò đồ dùng & Link phòng học trực tuyến

    Note over Cron, Engine: Kịch bản 3: Tự động gửi nhận xét buổi học
    participant Teacher as Giảng viên
    Teacher->>Engine: Lưu sổ nhận xét buổi học (teacherNotes)
    Engine->>Gateway: Bắn tin nhắn tóm tắt kết quả ca học tới Phụ huynh & Học sinh
    Engine->>DB: Lưu nhật ký notification_logs (sentStatus = SENT)
```

---

## 6. Cơ Chế Video Player Chống Tua & Heartbeat Anti-Cheat (Restricted Video Player)

Nhằm bảo đảm học viên của các khóa học tự học (Self-Paced) xem trọn vẹn bài giảng video nhúng YouTube:

1. **Client-Side Player Shield (Nuxt / Vue 3 Component):**
   - Trình phát sử dụng YouTube IFrame API bọc trong component bảo vệ Vue 3 (`components/course/RestrictedVideoPlayer.vue`).
   - Ẩn điều khiển tua mặc định của YouTube hoặc chặn sự kiện `seekTo`:
     - Nếu vị trí người dùng tua tới $> \text{maxWatchedSeconds} + 2\text{s}$, player ngay lập tức ép `seekTo(maxWatchedSeconds)`.
     - Cho phép tua lùi thoải mái ($\le \text{maxWatchedSeconds}$) để nghe lại bài giảng.
2. **Anti-Cheat Heartbeat Protocol:**
   - Mỗi $5$ giây khi video đang ở trạng thái `PLAYING`, client gửi heartbeat:
     ```json
     {
       "lessonId": "uuid",
       "currentSeconds": 145,
       "playbackRate": 1.0,
       "clientTimestamp": 1727072400
     }
     ```
   - **Xác thực phía Server (Go Backend Verification):**
     - $\Delta T_{\text{client}} = \text{currentSeconds} - \text{lastCurrentSeconds}$.
     - $\Delta T_{\text{server}} = \text{now}() - \text{lastHeartbeatAt}$.
     - Nếu $\Delta T_{\text{client}} > \Delta T_{\text{server}} \times \text{playbackRate} + 3\text{s}$ (phát hiện tua lách luật qua DevTools/Script) $\to$ Server từ chối cập nhật `maxWatchedSeconds` và trả mã cảnh báo `400 Bad Request`.
3. **Mở khóa sau khi hoàn thành (Post-Completion Unlock):**
   - Khi `maxWatchedSeconds >= totalDuration * 0.95`, hệ thống cập nhật `isCompleted = true` và `allowFreeSeeking = true`.
   - Các lần xem tiếp theo, học sinh được tự do tua nhanh/chậm phục vụ việc tra cứu và ôn tập.

---

## 7. Cấu Trúc Mã Nguồn Dự Án (Chuẩn hóa theo blueprint gmhafiz/go8 + Nuxt UI)

Hệ thống được tổ chức theo chuẩn **Monorepo** với Backend tuân thủ nguyên tắc Clean/Layered Architecture của **`gmhafiz/go8`**:

```text
LMS/
├── backend/                                   # Golang REST API Server (gmhafiz/go8 Blueprint)
│   ├── cmd/
│   │   ├── go8/
│   │   │   └── main.go                        # Entrypoint chính khởi chạy API Server
│   │   ├── migrate/
│   │   │   └── main.go                        # Trình thực thi database migrations (Goose)
│   │   ├── route/
│   │   │   └── main.go                        # CLI in danh sách toàn bộ routes đã đăng ký
│   │   └── seed/
│   │       └── main.go                        # Seeder dữ liệu mẫu (Roles, Admin, Permissions)
│   ├── configs/                               # Cấu hình Struct-based (envconfig & .env)
│   │   ├── configs.go                         # Root Config struct
│   │   ├── api.go                             # Cấu hình Host, Port, Read/Write Timeout
│   │   ├── database.go                        # Cấu hình PostgreSQL connection pool
│   │   ├── cors.go                            # Cấu hình CORS allowed origins
│   │   └── jwt.go                             # Cấu hình JWT secret key & expiration
│   ├── database/
│   │   └── migrations/                        # Goose SQL Migrations (*.sql)
│   │       ├── 20260923000001_create_roles_permissions.sql
│   │       ├── 20260923000002_create_users_profiles.sql
│   │       ├── 20260923000003_create_courses_classes.sql
│   │       ├── 20260923000004_create_sessions_attendance.sql
│   │       ├── 20260923000005_create_assignments_submissions.sql
│   │       ├── 20260923000006_create_invoices_payments.sql
│   │       ├── 20260923000007_create_video_progress_discussions.sql
│   │       └── 20260923000008_create_notification_logs_templates.sql
│   ├── internal/
│   │   ├── server/                            # Khởi tạo Server & Dependency Injection
│   │   │   ├── server.go                      # Server struct & lifecycle
│   │   │   ├── init.go                        # Khởi tạo Router, DB, Validator, Middlewares
│   │   │   └── initDomains.go                 # Wire-up dependencies (Repo -> Usecase -> Handler)
│   │   ├── middleware/                        # Chaining Middlewares cho Chi Router
│   │   │   ├── auth.go                        # JWT Authentication Middleware
│   │   │   ├── rbac.go                        # Dynamic RBAC Permission Guard
│   │   │   ├── cors.go                        # Cross-Origin Resource Sharing
│   │   │   └── request_id.go                  # Gắn Request ID phục vụ truy vết log
│   │   ├── domain/                            # Các phân hệ nghiệp vụ độc lập (Clean Layered)
│   │   │   ├── auth/                          # Đăng nhập, đăng ký, cấp phát Token, phân quyền
│   │   │   ├── class/                         # Khóa học, Module, Lớp học & Buổi học linh hoạt
│   │   │   ├── attendance/                    # Điểm danh học sinh Hybrid & Chấm công giáo viên
│   │   │   ├── evaluation/                    # Đánh giá giáo viên theo buổi (Student Feedback) & Audit
│   │   │   ├── course_video/                  # Trình phát video chống tua & Heartbeat anti-cheat
│   │   │   ├── assignment/                    # BTVN, nộp code Monaco & Chấm điểm Rubric
│   │   │   ├── notification/                  # Engine gửi tin nhắn Zalo/SMS (sau 15p, nhắc 24h/2h)
│   │   │   ├── billing/                       # Học phí, sinh mã VietQR Napas247, xác nhận tiền mặt
│   │   │   └── certificate/                   # Cấp chứng chỉ & URL xác minh công khai (/verify)
│   │   │       # Mỗi domain tuân thủ cấu trúc 3 tầng chuẩn của go8:
│   │   │       # ├── handler/ (HTTP Handlers, register.go, DTO validator)
│   │   │       # ├── usecase/ (Business Logic implementation)
│   │   │       # ├── repository/ (Postgres data access)
│   │   │       # ├── model.go (Entity struct)
│   │   │       # ├── dto.go (Request/Response DTOs)
│   │   │       # ├── repository.go (Interface)
│   │   │       # └── usecase.go (Interface)
│   │   └── worker/                            # Tiến trình nền Goroutines & Scheduled Cron
│   │       ├── cron.go                        # robfig/cron setup
│   │       ├── attendance_alert_job.go        # Quét và gửi tin điểm danh sau 15p
│   │       ├── class_reminder_job.go          # Quét và gửi tin nhắc lịch học trước 24h & 2h
│   │       └── message_dispatcher.go          # Worker pool bắn tin Zalo ZNS / SMS / Email
│   ├── pkg/                                   # Thư viện tiện ích dùng chung
│   │   ├── response/                          # Chuẩn hóa JSON Response Envelope
│   │   ├── validator/                         # Custom validation rules (go-playground/validator)
│   │   ├── vietqr/                            # Package tạo payload & mã VietQR Napas247
│   │   └── pdf/                               # Package sinh file PDF biên lai & chứng chỉ
│   ├── Taskfile.yml                           # Task runner tự động hóa (build, dev, test, migrate)
│   ├── .air.toml                              # Cấu hình hot reload (Air)
│   ├── env.example                            # Biến môi trường mẫu
│   ├── go.mod
│   └── go.sum
│
├── frontend/                                  # Nuxt UI Application (Vue 3 + Tailwind CSS v4)
│   ├── assets/css/main.css                    # Tailwind CSS v4 & Design Tokens (DESIGN.md)
│   ├── components/                            # Reusable Nuxt UI components
│   │   ├── common/                            # Navigation, Modal, Toast, VietQR Card
│   │   ├── course/                            # RestrictedVideoPlayer.vue, TimestampedQA.vue
│   │   ├── editor/                            # MonacoCodeEditor.vue
│   │   └── portals/                           # Components riêng của 4 Portal
│   ├── composables/                           # Vue 3 Composables (useAuth, useApi, useAttendance)
│   ├── layouts/                               # default.vue, admin.vue, portal.vue
│   ├── pages/                                 # File-based routing (Admin, Teacher, Student, Parent)
│   ├── stores/                                # Pinia Stores (auth.ts, class.ts)
│   ├── nuxt.config.ts                         # Cấu hình Nuxt UI, Tailwind, API proxy
│   ├── package.json
│   └── tsconfig.json
│
├── docs/                                      # Bộ tài liệu kỹ thuật toàn diện
│   ├── requirements.md                        # SRS v1.3.0
│   ├── architecture.md                        # Kiến trúc v2.1.0 (go8 + Nuxt UI)
│   ├── api-specification.md                   # Đặc tả REST API chuẩn Chi Router
│   ├── user-flows.md                          # Sơ đồ tương tác người dùng & Wireframes
│   └── test-plan.md                           # Kế hoạch QA & Lệnh kiểm thử
├── DESIGN.md                                  # Quy chuẩn Design Tokens & Typography
└── .gitignore
```

---

## 8. Quy Trình Phát Triển & Vận Hành Với `go8` & Taskfile (Go8 Developer Workflow)

Nhờ áp dụng blueprint `gmhafiz/go8`, toàn bộ các thao tác phát triển và kiểm thử ở backend được gói gọn trong công cụ **Task runner (`Taskfile.yml`)**:

| Lệnh `task` | Thao Tác Thực Hiện Phía Backend |
| :--- | :--- |
| `task dev` | Khởi chạy máy chủ API với cơ chế **Hot Reload (Air)**, tự động biên dịch lại khi sửa mã nguồn Go. |
| `task migrate` | Chạy toàn bộ các file migration trong `database/migrations/*.sql` bằng **Goose**. |
| `task migrate:create -- name` | Tạo nhanh một file migration mới (Up & Down SQL). |
| `task migrate:rollback` | Rollback migration gần nhất nếu cần quay lại phiên bản CSDL cũ. |
| `task routes` | In ra toàn bộ bảng danh sách các API Route đã đăng ký trên terminal để kiểm tra nhanh. |
| `task swagger` | Tự động quét annotations trong handlers và sinh tài liệu **Swagger/OpenAPI** trực quan tại `/swagger/`. |
| `task test` | Chạy toàn bộ Unit Test và Integration Test của Repository, UseCase và Handler. |
| `task check` | Chạy kiểm tra tổng hợp: `go fmt` (định dạng), `go vet` (biên dịch), `golangci-lint` và quét lỗ hổng bảo mật `govulncheck`. |
| `task build` | Đóng gói nhị phân (statically-linked binary) tối ưu cho môi trường Production (Linux/Docker). |




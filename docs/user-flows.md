# Sơ Đồ Luồng Thao Tác Người Dùng (User Journeys & Interaction Flows)

> **Dự án:** LMS Center Platform (Hệ thống Quản lý Học tập, Giảng viên & Vận hành Đào tạo Đa hình thức)  
> **Tài liệu:** `docs/user-flows.md`  
> **Phiên bản:** 1.0.0  
> **Ngày cập nhật:** 23/09/2026  

---

## 1. Luồng 1: Giảng Viên Check-In Ca Dạy & Điểm Danh Lớp Hybrid

Luồng thao tác từ lúc giáo viên mở hệ thống chuẩn bị cho buổi dạy, chấm công ca dạy, điểm danh học viên đến khi kết thúc ca.

```mermaid
sequenceDiagram
    autonumber
    actor Teacher as Giảng viên
    participant UI as Teacher Portal (/teacher)
    participant API as Backend Server
    participant DB as PostgreSQL Database
    actor Parent as Phụ huynh & Học viên

    Teacher ->> UI: Mở Thời khóa biểu (/teacher/schedule)
    UI ->> Teacher: Hiển thị ca dạy hôm nay (19:30 - 21:30, Lớp Hybrid)
    Teacher ->> UI: Nhấn nút [ Check-in Ca Dạy ]
    UI ->> API: POST /api/attendance/teacher-checkin (sessionId, action: CHECK_IN)
    API ->> DB: Ghi nhận checkInTime thực tế của giáo viên
    API -->> UI: Trả về trạng thái "Đang diễn ra (In Progress)"

    Note over Teacher, UI: Trong giờ học / Đầu giờ học
    Teacher ->> UI: Mở tab [ Điểm danh học sinh ]
    UI ->> Teacher: Hiển thị danh sách học viên trong lớp
    loop Điểm danh từng học viên
        Teacher ->> UI: Chọn [ Có mặt tại lớp (Offline) ] hoặc [ Tham gia Online (Meet) ] hoặc [ Đi trễ / Vắng ]
        opt Nhập ghi chú
            Teacher ->> UI: Ghi chú tình hình học sinh
        end
    end
    Teacher ->> UI: Nhấn [ Lưu Bảng Điểm Danh ]
    UI ->> API: POST /api/attendance/students
    API ->> DB: Lưu kết quả điểm danh học sinh
    API -->> Parent: Thông báo chuyên cần tức thời đến Phụ huynh & Học viên

    Note over Teacher, UI: Kết thúc buổi học
    Teacher ->> UI: Nhấn [ Check-out Kết Thúc Ca Dạy ]
    UI ->> API: POST /api/attendance/teacher-checkin (action: CHECK_OUT)
    API ->> DB: Tính số phút dạy thực tế (Actual Timesheet Hours)
    API -->> UI: Thông báo hoàn thành buổi dạy (Ví dụ: 120 phút)
```

---

## 2. Luồng 2: Học Viên Học Bài Giảng YouTube & Làm BTVN Trực Tuyến (Monaco Editor)

Luồng thao tác của học viên CNTT khi theo dõi bài giảng, tải tài liệu và thực hành viết code nộp bài trực tiếp trên trình duyệt.

```mermaid
sequenceDiagram
    autonumber
    actor Student as Học viên
    participant UI as Student Portal (/student)
    participant Monaco as Monaco Code Editor
    participant YT as YouTube Player API
    participant API as Backend Server
    participant DB as PostgreSQL Database

    Student ->> UI: Đăng nhập vào Dashboard (/student)
    UI ->> Student: Hiển thị bài học tiếp theo & BTVN cần nộp
    Student ->> UI: Mở bài giảng số (/student/courses/:id/lessons/:id)
    UI ->> YT: Khởi chạy video YouTube bài giảng
    UI ->> Student: Hiển thị tab [ Tài liệu học tập (Slide PDF, Starter Code) ]
    Student ->> UI: Nhấp tải file `starter-code.zip` và slide bài học

    Note over Student, YT: Xem video bài giảng
    YT ->> UI: Theo dõi thời lượng xem (Progress >= 80%)
    UI ->> API: POST /api/lessons/:id/progress
    API ->> DB: Đánh dấu bài học "Đã hoàn thành"

    Note over Student, Monaco: Làm bài tập về nhà
    Student ->> UI: Mở trang Bài tập (/student/assignments/:id)
    UI ->> Student: Hiển thị Đề bài Markdown + Tài liệu BTVN (Dataset/File mẫu)
    UI ->> Monaco: Khởi động Monaco Code Editor (giao diện dark giống VS Code)
    Student ->> Monaco: Soạn thảo mã nguồn (JavaScript/TypeScript/Python)
    Monaco ->> Monaco: Auto-save lưu nháp vào LocalStorage mỗi 15s
    Student ->> UI: Bấm nút [ Nộp Bài Tập ]
    UI ->> UI: Hiển thị hộp thoại xác nhận nộp bài
    UI ->> API: POST /api/submissions (codeContent, submittedAt)
    API ->> DB: Lưu bài nộp, kiểm tra hạn nộp (Deadline)
    API -->> UI: Thông báo "Nộp bài thành công!"
```

---

## 3. Luồng 3: Giảng Viên Chấm Điểm & Phụ Huynh Nhận Kết Quả

Quy trình giáo viên chấm bài nộp và hệ thống tự động cập nhật sổ điểm, thông báo đến phụ huynh.

```mermaid
sequenceDiagram
    autonumber
    actor Teacher as Giảng viên
    participant UI as Teacher Portal
    participant API as Backend Server
    participant DB as PostgreSQL Database
    actor Student as Học viên
    actor Parent as Phụ huynh

    Teacher ->> UI: Mở mục [ Chấm bài tập ] (/teacher/assignments)
    UI ->> Teacher: Hiển thị danh sách học viên đã nộp bài
    Teacher ->> UI: Chọn học viên Trần Thị Mai (Bài nộp code Monaco)
    UI ->> Teacher: Màn hình chia đôi: Bên trái hiển thị Code học viên, Bên phải là Khung chấm điểm
    Teacher ->> UI: Nhập điểm số (Ví dụ: 95/100)
    Teacher ->> UI: Nhập nhận xét: "Thuật toán tối ưu, code sạch sẽ, chú ý thêm comment"
    Teacher ->> UI: Bấm [ Lưu & Trả Điểm ]
    UI ->> API: POST /api/grading
    API ->> DB: Lưu GradeFeedback và cập nhật trạng thái bài tập thành GRADED
    API ->> DB: Tự động tính lại điểm trung bình trong Sổ điểm lớp (Gradebook)
    API -->> Student: Thông báo điểm số và nhận xét trên Cổng học viên
    API -->> Parent: Thông báo kết quả bài tập của con trên Cổng phụ huynh
```

---

## 4. Luồng 4: Phụ Huynh & Học Viên Thanh Toán Học Phí Qua Mã VietQR Động

Luồng thanh toán học phí nhanh chóng, tự động điền thông tin và không thể chuyển khoản sai lệch.

```mermaid
sequenceDiagram
    autonumber
    actor User as Phụ huynh / Học viên
    participant UI as Student/Parent Portal
    participant BankApp as App Ngân Hàng (MB, VCB, MoMo...)
    participant API as Backend Server
    participant Napas as Napas247 / VietQR Gateway
    actor Admin as Giáo vụ / Thu ngân

    User ->> UI: Truy cập mục [ Học phí ] (/tuition)
    UI ->> User: Hiển thị hóa đơn học phí (Ví dụ: 3.500.000 đ - Khóa React K32)
    User ->> UI: Bấm chọn [ Thanh toán qua VietQR ]
    UI ->> API: GET /api/invoices/:id/vietqr
    API ->> Napas: Sinh URL mã VietQR QuickLink chuẩn Napas
    API -->> UI: Trả về mã QR động kèm số tài khoản và nội dung: "HP HV0123 INV0042"
    UI ->> User: Hiển thị ảnh VietQR lớn + Nút sao chép STK & nội dung
    User ->> BankApp: Mở App ngân hàng trên điện thoại cá nhân
    User ->> BankApp: Quét mã VietQR trên màn hình máy tính
    BankApp ->> User: Tự động điền đúng Tên trung tâm, Số tiền 3.500.000 đ, Nội dung chính xác
    User ->> BankApp: Xác nhận chuyển khoản (FaceID / OTP)
    BankApp -->> User: Giao dịch thành công!
    
    opt Thu tiền mặt tại quầy (Hình thức thay thế)
        User ->> Admin: Nộp tiền mặt trực tiếp tại quầy lễ tân
        Admin ->> UI: Bấm nút [ Xác nhận thu tiền mặt ]
        UI ->> API: POST /api/payments/cash-confirm
        API -->> Admin: Tự động xuất biên lai điện tử PDF (có thể in hoặc gửi email)
    end
```

---

## 5. Luồng 5: Động Cơ Thông Báo Tự Động (Điểm Danh Sau 15p, Nhận Xét Sau Buổi, Nhắc Lịch Học)

```mermaid
sequenceDiagram
    autonumber
    actor Parent as Phụ huynh
    actor Student as Học viên
    actor Teacher as Giảng viên
    actor Coord as Chuyên viên Vận hành
    participant Cron as Hệ thống Cron / Trigger
    participant API as Backend Notification Engine
    participant Gateway as Zalo ZNS / SMS Gateway

    Note over Cron, Gateway: Kịch bản A: Sau 15 phút vào lớp (Điểm danh vắng/đủ)
    Cron ->> API: POST /api/sessions/:id/attendance-alert
    alt Lớp có học sinh vắng
        API ->> Gateway: Bắn tin nhắn đến từng Phụ huynh của học sinh vắng
        Gateway -->> Parent: "Bé [Tên] chưa có mặt tại lớp [Mã lớp] bắt đầu lúc [Giờ]..."
        API ->> Gateway: Gửi báo cáo tổng hợp cho GV & Vận hành lớp
        Gateway -->> Teacher: "Lớp hiện có 18/20 bạn, vắng 2 bạn: [Tên]"
        Gateway -->> Coord: "Lớp hiện có 18/20 bạn, vắng 2 bạn: [Tên]"
    else Lớp đi đủ 100%
        API ->> Gateway: Gửi tin xác nhận lớp đủ quân số cho GV & Vận hành
    end

    Note over Teacher, Gateway: Kịch bản B: Sau buổi học (Giáo viên nhận xét)
    Teacher ->> API: Lưu nhận xét từng học sinh (thái độ, BTVN trên lớp)
    API ->> Gateway: Tự động gửi tin nhắn nhận xét đến Phụ huynh & Học sinh
    Gateway -->> Parent: "Nhận xét buổi học môn [Môn] của bé [Tên]: Thái độ tốt, thực hành đạt..."

    Note over Cron, Gateway: Kịch bản C: Nhắc nhở lịch học trước 24h và 2h
    Cron ->> API: Quét các buổi học trong 24h & 2h tới
    API ->> Gateway: Bắn tin nhắc lịch kèm dặn dò đồ dùng học tập & Link Meet/Zoom nếu học Online/Hybrid
    Gateway -->> Student: "Nhắc lịch học lớp [Mã lớp] vào ngày mai [Giờ]. Lưu ý: mang laptop..."
    Gateway -->> Parent: "Nhắc lịch học của bé [Tên]..."
```

---

## 6. Luồng 6: Trình Phát Video Bài Giảng Chống Tua (Anti-Seeking Video Player Flow)

```mermaid
sequenceDiagram
    autonumber
    actor Student as Học viên
    participant Player as Restricted Video Player (Client)
    participant API as Backend Server
    participant DB as PostgreSQL Database

    Student ->> Player: Bắt đầu phát video bài giảng YouTube
    Player ->> API: GET /api/courses/:id/lessons/:id/progress
    API ->> DB: Lấy maxWatchedSeconds và allowFreeSeeking
    DB -->> API: maxWatchedSeconds = 120s, allowFreeSeeking = false
    API -->> Player: Khởi chạy video từ giây 120

    alt Học viên cố tình click tua tới giây 300 (Chưa từng xem)
        Student ->> Player: Kéo tua thanh thời gian tới 05:00 (300s)
        Player ->> Player: Phát hiện 300s > maxWatchedSeconds (120s)
        Player ->> Student: Hiển thị cảnh báo: "Vui lòng xem tuần tự bài giảng, không tua nhanh"
        Player ->> Player: Ép tua trở lại vị trí 120s
    else Học viên tua lùi về giây 60 để nghe lại
        Student ->> Player: Kéo tua về 01:00 (60s)
        Player ->> Player: Chấp nhận thao tác (60s <= maxWatchedSeconds)
        Player ->> Student: Phát lại từ giây 60
    end

    loop Heartbeat định kỳ mỗi 5 giây xem hợp lệ
        Player ->> API: POST /api/courses/:id/lessons/:id/heartbeat (currentSeconds = 125, delta = 5s)
        API ->> API: Xác thực tốc độ xem hợp lệ (Không gian lận)
        API ->> DB: Cập nhật maxWatchedSeconds = 125s
    end

    opt Khi xem đạt >= 95% thời lượng bài học
        API ->> DB: Cập nhật isCompleted = true, allowFreeSeeking = true
        API -->> Player: Mở khóa thanh tua tự do (Free Seeking Unlocked)
        Player -->> Student: Thông báo: "Chúc mừng bạn đã hoàn thành bài học! Từ bây giờ bạn có thể tự do tua ôn tập."
    end
```

---

## 7. Bố Cục Giao Diện Màn Hình Trọng Yếu (Key Wireframe Schematics)

### 7.1 Giao diện Học viên làm bài tập CNTT (Monaco Code Workspace)
```
+-----------------------------------------------------------------------------+
| [← Quay lại lớp]  Bài tập 03: Xây dựng Todo List với React 19   [Deadline: 23:59] |
+---------------------------------------+-------------------------------------+
| ĐỀ BÀI & HƯỚNG DẪN (Markdown)        | MONACO CODE EDITOR                  |
| ------------------------------------  | Ngôn ngữ: [ TypeScript v ] [Prettier] |
| Yêu cầu:                              | 1  import React, { useState } from  |
| 1. Tạo component TodoList             | 2  'react';                         |
| 2. Cho phép thêm, xóa, sửa trạng thái | 3                                   |
| 3. Lưu trữ vào localStorage           | 4  export function TodoApp() {      |
|                                       | 5    const [todos, setTodos] = ...  |
| Tài liệu BTVN đính kèm:               | 6    // Viết code của bạn ở đây...   |
| [📄 starter-template.zip (Tải về)]    | 7  }                                |
| [📊 sample-todos.json (Tải về)]       |                                     |
|                                       | +---------------------------------+ |
|                                       | | [💾 Lưu nháp]   [🚀 NỘP BÀI TẬP] | |
|                                       | +---------------------------------+ |
+---------------------------------------+-------------------------------------+
```

### 7.2 Giao diện Giảng viên Điểm danh Lớp Hybrid (Attendance Sheet)
```
+-----------------------------------------------------------------------------+
| Lớp: LMS-FE-K32 (Hybrid) | Buổi 5 (23/09/2026) | [Check-out ca dạy: 21:30]   |
+-----------------------------------------------------------------------------+
| Tìm học viên: [ Tìm kiếm tên / mã... ]         Sĩ số: 18/20 học viên (Vắng: 2) |
| [⚠️ Đã gửi cảnh báo vắng sau 15p tới 2 phụ huynh lúc 19:45]                 |
|                                                                             |
| #  Mã HV      Họ và tên        Trạng thái tham gia          Ghi chú buổi học |
| 1  HV-001     Trần Thị Mai     (•) Offline  ( ) Online      [Thái độ tốt   ] |
|                                ( ) Đi trễ   ( ) Vắng                         |
| 2  HV-002     Lê Hoàng Nam     ( ) Offline  (•) Online      [Học qua Meet  ] |
|                                ( ) Đi trễ   ( ) Vắng                         |
| 3  HV-003     Phạm Minh Đức    ( ) Offline  ( ) Online      [Sốt xin nghỉ  ] |
|                                ( ) Đi trễ   (•) Vắng                         |
+-----------------------------------------------------------------------------+
| [ Tải lại ]                                         [ 💾 LƯU ĐIỂM DANH ]    |
+-----------------------------------------------------------------------------+
```

### 7.3 Giao diện Khóa Học Online Tự Học với Video Chống Tua & Timestamped Q&A
```
+-----------------------------------------------------------------------------+
| [← Danh sách bài học]  Bài 04: Quản lý Global State với Zustand             |
+-------------------------------------------------------+---------------------+
| TRÌNH PHÁT VIDEO BÀI GIẢNG YOUTUBE (CHỐNG TUA)        | HỎI ĐÁP & GHI CHÚ   |
| ----------------------------------------------------- | ------------------- |
| +---------------------------------------------------+ | [❓ Thảo luận] [📝] |
| |                                                   | |                     |
| |              [ Video YouTube Embed ]              | | [02:15] Nam:        |
| |                                                   | | Store này có lưu    |
| |                                                   | | localStorage k ạ?   |
| +---------------------------------------------------+ |  ↳ [GV Trả lời] ✅  |
| [▶ Play] 02:15 / 15:40 [🔒 Đã xem: 02:15 - Cấm tua]   | Có em, dùng persist |
| [======•--------------------------------------------] |                     |
| (• Đang khóa tua tiến | Chỉ được tua lùi hoặc nghe lại) | +-----------------+ |
|                                                       | | Nhập câu hỏi... | |
| [📄 Tải Slide PDF]   [💻 Tải Code Mẫu]   [✅ Hoàn thành] | +-----------------+ |
+-------------------------------------------------------+---------------------+
```

### 7.4 Giao diện Cổng Chấm Thi Giám Khảo Đồ Án Tốt Nghiệp (Capstone Jury Defense Portal)
```
+-----------------------------------------------------------------------------+
| HỘI ĐỒNG CHẤM ĐỒ ÁN TỐT NGHIỆP | Lớp: LMS-FE-K32 | Giám khảo: TS. Lê Văn Tuấn |
+-----------------------------------------------------------------------------+
| Đề tài: Nền tảng Đặt xe Công nghệ Real-time (Nhóm 02: 3 thành viên)          |
| Tài liệu: [🔗 GitHub Repo] [🌐 Live Demo] [📊 Slide PDF] [🎥 Video Demo]     |
| --------------------------------------------------------------------------- |
| PHIẾU ĐÁNH GIÁ RUBRIC CỦA GIÁM KHẢO:                                        |
| 1. Mức độ hoàn thiện sản phẩm & Chức năng (30%):    [ 90 / 100 ]            |
| 2. Kiến trúc mã nguồn, Code Quality & Clean Code (25%): [ 85 / 100 ]         |
| 3. Kỹ năng thuyết trình & Trả lời phản biện Q&A (25%):  [ 95 / 100 ]         |
| 4. Tính sáng tạo, Đột phá & Khả năng ứng dụng (20%):   [ 80 / 100 ]         |
| --------------------------------------------------------------------------- |
| => ĐIỂM TỔNG HỢP: 88.0 / 100  [✅ ĐẠT YÊU CẦU TỐT NGHIỆP]                   |
|                                                                             |
| Nhận xét chuyên môn & Góp ý định hướng:                                     |
| +-------------------------------------------------------------------------+ |
| | Kiến trúc WebSocket thời gian thực hoạt động tốt. Cần tối ưu lại index  | |
| | PostgreSQL để chịu tải cao hơn. Nhóm phản biện rất tự tin và sắc bén.   | |
| +-------------------------------------------------------------------------+ |
|                                                    [ 💾 NỘP PHIẾU CHẤM ]    |
+-----------------------------------------------------------------------------+
```

### 7.5 Giao diện Chấm Bài Tập Về Nhà Theo Lớp Học (Class Assignment & Monaco Code Grading Portal)
```
+-----------------------------------------------------------------------------+
| LỚP: LMS-FE-K32 > BÀI TẬP: BTVN-03: Xây dựng REST API với Go-chi            |
| Hạn nộp: 23:59 20/10/2026 | Đã nộp: 22/25 | Đã chấm: 18/22                 |
+-------------------------------------------------------+---------------------+
| DANH SÁCH BÀI LÀM HỌC VIÊN CỦA LỚP                    | TRÌNH XEM CODE &    |
| 1. Trần Minh Hoàng  [Đã chấm: 95đ] [Xem code]         | PHIẾU CHẤM ĐIỂM     |
| 2. Lê Hoàng Nam     [Chưa chấm]    [Đang chọn >]      | ------------------- |
| 3. Phạm Minh Đức    [Chưa nộp]     [Nhắc nộp]         | Học viên: Lê Hoàng Nam|
| ----------------------------------------------------- | GitHub: /nam/hw-03  |
| TRÌNH XEM CODE MONACO (Read-only + Diff View):        | ------------------- |
| package handler                                       | ĐIỂM SỐ (Thang 100):|
| func (h *UserHandler) GetProfile(w http.ResponseWriter)| [ 90       ] / 100  |
|     // Validate token                                 | ------------------- |
|     userID := auth.GetUserID(r.Context())             | Nhận xét giáo viên: |
|     user, err := h.useCase.FindByID(r.Context(), userID)| +-----------------+ |
|     if err != nil {                                   | | Xử lý context rất | |
|         response.NotFound(w, "NOT_FOUND", err.Error())| | chuẩn. Cần thêm   | |
|         return                                        | | unit test cho err.| |
|     }                                                 | +-----------------+ |
|     response.OK(w, user, "Success")                   | [🎙 Ghi âm dặn dò]  |
| }                                                     | [ 💾 LƯU ĐIỂM VÀO ] |
|                                                       | [   GRADEBOOK LỚP ] |
+-------------------------------------------------------+---------------------+
```

### 7.6 Giao diện Bảng Lương Giáo Viên & Bậc Lương Hàng Tháng (Admin Teacher Payroll Management)
```
+-----------------------------------------------------------------------------+
| QUẢN LÝ BẢNG LƯƠNG GIÁO VIÊN | Kỳ lương: [ Tháng 10/2026 ▼ ] | Trạng thái: [ Tất cả ▼ ]
| Tổng ngân sách: 158.400.000 đ | Số GV: 18 | Đã duyệt: 14 | Chờ duyệt: 4     |
| [ ⚡ TỰ ĐỘNG TỔNG HỢP LƯƠNG THÁNG ] [ ⚙️ Cấu hình Bậc Lương ] [ 📥 Xuất Excel ]
+-----------------------------------------------------------------------------+
| GV / Bậc Lương      | Giờ Dạy | Lương Giờ | Thưởng KPI | Phạt Muộn | Thực Lĩnh   | Trạng thái | Thao tác
| --------------------+---------+-----------+------------+-----------+-------------+------------+---------
| ThS. Nguyễn Văn Tuấn| 42.5 h  | 350.000 đ | +2.231.250 | -100.000  | 17.006.250 đ| [Chờ duyệt]| [Duyệt] [Chi tiết]
| (Bậc: Senior)       |         |           | (⭐ 4.85/5) | (20 phút) |             |            |
| KS. Đặng Hồng Sơn   | 36.0 h  | 250.000 đ | +900.000   |        0  |  9.900.000 đ| [Đã duyệt] | [Chi tiết]
| (Bậc: Standard)     |         |           | (⭐ 4.60/5) | (0 phút)  |             |            |
| Lê Thu Hà (TA)      | 28.0 h  | 120.000 đ |          0 |  -50.000  |  3.310.000 đ| [Chờ duyệt]| [Duyệt] [Chi tiết]
| (Bậc: Intern/TA)    |         |           | (⭐ 4.20/5) | (10 phút) |             |            |
+-----------------------------------------------------------------------------+
| [✓ Duyệt hàng loạt đã chọn]                       [ 💳 KẾT XUẤT LỆNH CHI TRẢ ]|
+-----------------------------------------------------------------------------+
```

---

## 8. Luồng 7: Giám Khảo Chấm Điểm Đồ Án Tốt Nghiệp & Phản Biện (Capstone Defense)

```mermaid
sequenceDiagram
    autonumber
    actor Examiner as Giám khảo / Hội đồng
    participant UI as Examiner Portal (/examiner)
    participant API as Backend Server
    participant DB as PostgreSQL Database
    actor Student as Học viên / Nhóm đồ án

    Examiner ->> UI: Đăng nhập với quyền EXAMINER
    UI ->> Examiner: Hiển thị danh sách đề tài đồ án cần chấm
    Examiner ->> UI: Chọn đề tài "Nền tảng Đặt xe Công nghệ"
    UI ->> API: GET /api/v1/capstone/projects/:id
    API ->> DB: Lấy chi tiết đề tài, link GitHub, demo URL, slide
    DB -->> API: Trả về thông tin đồ án
    API -->> UI: Hiển thị giao diện chấm thi & tài liệu

    Note over Examiner, UI: Phiên bảo vệ đồ án trực tiếp / trực tuyến
    Student ->> Examiner: Thuyết trình đề tài & demo sản phẩm
    Examiner ->> Student: Đặt câu hỏi phản biện kỹ thuật
    Student ->> Examiner: Trả lời phản biện Q&A

    Examiner ->> UI: Nhập điểm Rubric 4 tiêu chí & Lời nhận xét
    Examiner ->> UI: Bấm [ Nộp Phiếu Chấm ]
    UI ->> API: POST /api/v1/capstone/evaluations
    API ->> DB: Lưu phiếu chấm độc lập vào capstone_evaluations
    API ->> DB: Tự động tính điểm trung bình Hội đồng vào Sổ điểm lớp
    API -->> UI: Thông báo nộp điểm thành công
    API -->> Student: Nhận thông báo kết quả chấm thi & nhận xét của Giám khảo
```

---

## 9. Luồng 8: Mua Trực Tiếp Khóa Học Trực Tuyến & Thanh Toán VietQR Tự Động Kích Hoạt

```mermaid
sequenceDiagram
    autonumber
    actor Student as Học viên
    actor Bank as Ngân hàng / Cổng TT
    participant UI as Course Storefront
    participant API as Backend Server
    participant DB as PostgreSQL Database

    Student ->> UI: Duyệt danh mục khóa học trực tuyến (/courses)
    alt Khóa học Miễn Phí (isFree = true)
        Student ->> UI: Nhấn [ Đăng ký học ngay (Free) ]
        UI ->> API: POST /api/v1/courses/:id/enroll
        API ->> DB: Tạo enrollment trạng thái ACTIVE
        API -->> UI: Kích hoạt thành công, mở ngay bài học đầu tiên
    else Khóa học Trả Phí (isFree = false)
        Student ->> UI: Nhấn [ Mua khóa học ]
        UI ->> API: POST /api/v1/courses/:id/checkout
        API ->> DB: Tạo hóa đơn tuition_invoices kèm VietQR payload
        API -->> UI: Trả về mã VietQR Napas247 động
        UI ->> Student: Hiển thị mã VietQR kèm số tiền & cú pháp chuyển khoản
        Student ->> Bank: Mở App Banking quét mã VietQR và xác nhận chuyển khoản
        Bank ->> API: POST /api/v1/webhooks/payment (Số tiền, Mã hóa đơn)
        API ->> API: Khớp nội dung & số tiền thanh toán
        API ->> DB: Cập nhật invoice PAID & kích hoạt enrollment ACTIVE
        API -->> Student: Bắn Email biên lai & Thông báo chào mừng học viên vào học
    end
```

---

## 10. Luồng 9: Giáo Viên Quản Lý & Chấm Bài Tập Về Nhà Theo Lớp (Class Assignment Grading Portal)

```mermaid
sequenceDiagram
    autonumber
    actor Teacher as Giảng viên / Trợ giảng
    participant UI as Teacher Class Portal (/teacher/classes/:id)
    participant Monaco as Monaco Diff/Code Viewer
    participant API as Backend Server
    participant DB as PostgreSQL Database
    actor Student as Học viên
    actor Parent as Phụ huynh

    Teacher ->> UI: Mở chi tiết lớp học (/teacher/classes/:id)
    UI ->> Teacher: Hiển thị các tab [ Buổi học | Điểm danh | Bài tập | Sổ điểm ]
    Teacher ->> UI: Nhấp tab [ Bài tập ] > Chọn "BTVN-03: Xây dựng REST API"
    UI ->> API: GET /api/v1/classes/:id/assignments/:assignmentId/submissions
    API ->> DB: Lấy danh sách bài nộp của học viên trong lớp
    DB -->> API: Trả về trạng thái nộp, điểm hiện tại, codeContent, githubUrl
    API -->> UI: Hiển thị bảng danh sách học sinh (22/25 đã nộp)

    Teacher ->> UI: Chọn bài nộp của học viên "Lê Hoàng Nam"
    UI ->> Monaco: Tải mã nguồn của học viên lên Monaco Code Viewer (hỗ trợ Syntax highlight & Diff)
    Teacher ->> Monaco: Rà soát logic mã nguồn, kiểm tra xử lý lỗi & Clean Code
    Teacher ->> UI: Nhập điểm Rubric (90/100) & Nhận xét Markdown + Ghi âm dặn dò
    Teacher ->> UI: Nhấn [ Lưu Điểm Vào Sổ Điểm Lớp ]
    UI ->> API: POST /api/v1/classes/:id/assignments/:assignmentId/submissions/:subId/grade
    API ->> DB: Cập nhật submission (score, teacherNotes, gradedAt, gradedBy)
    API ->> DB: Tự động cập nhật cột điểm tương ứng trong Sổ điểm lớp học (Gradebook)
    API -->> UI: Thông báo chấm bài thành công
    API -->> Student: Gửi Push Notification "Bài tập BTVN-03 đã có điểm: 90/100"
    API -->> Parent: Thông báo kết quả học tập của con trên Parent Portal
```

---

## 11. Luồng 10: Áp Mã Khuyến Mãi (Discount / Coupon) Khi Mua Khóa Học & Sinh VietQR

```mermaid
sequenceDiagram
    autonumber
    actor Student as Học viên
    participant UI as Checkout Page (/checkout/:courseId)
    participant API as Backend Server
    participant DB as PostgreSQL Database
    actor Bank as Ngân hàng / Napas247

    Student ->> UI: Bấm mua khóa học (Giá gốc: 2.000.000 đ)
    UI ->> Student: Hiển thị form thanh toán kèm ô [ Nhập mã giảm giá ]
    Student ->> UI: Nhập coupon "KHAIGIANG2026" và bấm [ Áp Dụng ]
    UI ->> API: POST /api/v1/discounts/validate (couponCode, courseId, orderAmount: 2000000)
    API ->> DB: Kiểm tra discounts (còn hạn, isActive=true, minOrderAmount <= 2tr, usedCount < usageLimit)
    DB -->> API: Mã hợp lệ, loại PERCENT 20%, max 500.000 đ
    API -->> UI: Trả về discountAmount: 400.000 đ, finalAmount: 1.600.000 đ
    UI ->> Student: Cập nhật hóa đơn: Giảm -400.000 đ => Số tiền cần trả: 1.600.000 đ

    Student ->> UI: Bấm [ Xác nhận thanh toán VietQR ]
    UI ->> API: POST /api/v1/courses/:id/checkout (couponCode: "KHAIGIANG2026")
    API ->> DB: Tạo tuition_invoices (originalAmount: 2tr, discountAmount: 400k, finalAmount: 1.6tr, discountId)
    API ->> DB: Tăng usedCount của mã giảm giá (+1)
    API -->> UI: Trả về mã VietQR Napas247 đúng số tiền 1.600.000 đ kèm cú pháp chuyển khoản
    UI ->> Student: Hiển thị mã QR động để học viên quét App Banking
    Student ->> Bank: Quét mã VietQR chuyển đúng 1.600.000 đ
    Bank ->> API: POST /api/v1/webhooks/payment
    API ->> DB: Đánh dấu hóa đơn PAID và kích hoạt quyền học
    API -->> Student: Chúc mừng thanh toán thành công và mở khóa bài học
```

---

## 12. Luồng 11: Admin Tổng Hợp & Phê Duyệt Bảng Lương Giáo Viên Hàng Tháng (Teacher Monthly Payroll)

```mermaid
sequenceDiagram
    autonumber
    actor Admin as Quản trị viên / Giám đốc Đào tạo
    participant UI as Admin Payroll Portal (/admin/payrolls)
    participant API as Backend Server
    participant Engine as Payroll Calculation Engine
    participant DB as PostgreSQL Database
    actor Teacher as Giảng viên

    Admin ->> UI: Truy cập Quản lý Bảng Lương (/admin/payrolls)
    Admin ->> UI: Chọn kỳ tính lương "Tháng 10/2026" và nhấn [ Tự Động Tổng Hợp Lương ]
    UI ->> API: POST /api/v1/payrolls/generate (period: "2026-10")
    API ->> Engine: Kích hoạt thuật toán tổng hợp thù lao định mức

    Engine ->> DB: 1. Truy vấn các ca dạy hoàn thành trong tháng của từng GV
    Engine ->> DB: 2. Lấy định mức bậc lương (baseHourlyRate, kpiBonusRate) từ salary_grades
    Engine ->> DB: 3. Lấy điểm đánh giá trung bình từ session_feedbacks của học sinh
    Engine ->> DB: 4. Lấy số phút đi muộn từ teacher_attendances (lateMinutes)

    loop Xử lý từng Giảng viên
        Engine ->> Engine: baseSalary = totalTeachingHours * baseHourlyRate
        alt Điểm đánh giá TB >= 4.5
            Engine ->> Engine: kpiBonus = baseSalary * kpiBonusRate
        else Điểm < 4.5
            Engine ->> Engine: kpiBonus = 0
        end
        Engine ->> Engine: latePenalty = (lateMinutes / 10) * 50.000 đ
        Engine ->> Engine: netSalary = baseSalary + kpiBonus - latePenalty
    end

    Engine ->> DB: Lưu toàn bộ bản ghi teacher_payrolls & payroll_items (trạng thái DRAFT)
    Engine -->> API: Hoàn tất tổng hợp 18 giảng viên, tổng quỹ 158.400.000 đ
    API -->> UI: Hiển thị bảng lương chi tiết từng giảng viên

    Admin ->> UI: Soát xét phiếu lương ThS. Nguyễn Văn Tuấn (42.5h, Đánh giá 4.85/5 => Net 17.006.250 đ)
    Admin ->> UI: Nhấn [ Phê Duyệt Quyết Toán ]
    UI ->> API: POST /api/v1/payrolls/:id/approve
    API ->> DB: Cập nhật status = APPROVED, approved_by = adminId, approved_at = NOW()
    API -->> UI: Trạng thái đổi sang [ Đã duyệt ], cho phép xuất lệnh chuyển khoản ngân hàng
    API -->> Teacher: Gửi thông báo phiếu lương tháng 10/2026 tới ứng dụng Giảng viên
```

---

## 13. Luồng 12: Điều Phối Viên Lên Lịch Học Bù Cho Học Sinh Vắng (Make-up Class Scheduling)

```mermaid
sequenceDiagram
    autonumber
    actor Coord as Chuyên viên Vận hành / CSKH
    participant UI as Operations Portal (/operations/makeup)
    participant API as Backend Server
    participant DB as PostgreSQL Database
    actor Student as Học viên & Phụ huynh
    actor Teacher as Giáo viên / Trợ giảng kèm bù

    Note over Coord, DB: Sau buổi học, hệ thống phát hiện học sinh vắng
    Coord ->> UI: Mở tab [ Quản lý Học Bù ], thấy học viên "Nguyễn Hoàng Nam" vắng Buổi 4 lớp FE-K32
    Coord ->> UI: Bấm nút [ Lên lịch học bù ]
    UI ->> API: GET /api/v1/classes/parallel-topics?lessonId=...
    API ->> DB: Tìm các lớp song song cùng dạy bài "Goroutines & Channels" trong 7 ngày tới
    DB -->> API: Lớp FE-K33 học tối thứ Năm (19:30 - 21:30, phòng LAB-202, còn 3 chỗ)
    API -->> UI: Trả về danh sách gợi ý (Ghép lớp FE-K33 hoặc Kèm 1-1 với TA)

    Coord ->> UI: Chọn phương án [ Ghép lớp song song FE-K33 ], nhập ghi chú dặn dò
    Coord ->> UI: Bấm [ Lưu & Gửi thông báo học bù ]
    UI ->> API: POST /api/v1/makeup-sessions
    API ->> DB: Tạo bản ghi makeup_sessions (status: SCHEDULED, makeupType: PARALLEL_CLASS)
    API -->> Student: Gửi tin nhắn Zalo/Push: "Lịch học bù Buổi 4 vào 19:30 Thứ 5 tại Phòng LAB-202"
    API -->> UI: Thông báo đặt lịch học bù thành công

    Note over Teacher, DB: Đến buổi học bù tại lớp FE-K33
    Teacher ->> UI: Mở danh sách điểm danh lớp FE-K33 (có thêm Nam - diện Học bù)
    Teacher ->> UI: Tích chọn [ Nam: Đã tham gia học bù ] và nhập nhận xét
    UI ->> API: PUT /api/v1/makeup-sessions/:id/attend
    API ->> DB: Cập nhật status = ATTENDED; đồng bộ chuyên cần buổi 4 của Nam sang "ĐÃ HỌC BÙ"
    API -->> Student: Thông báo xác nhận hoàn thành buổi học bù thành công
```

---

## 14. Luồng 13: Xếp Lịch Phòng Học & Cảnh Báo Trùng Phòng Tự Động (Room Conflict Guard)

```mermaid
sequenceDiagram
    autonumber
    actor Staff as Giáo vụ / Điều phối viên
    participant UI as Class Scheduler (/admin/classes/schedule)
    participant API as Backend Server
    participant DB as PostgreSQL Database

    Staff ->> UI: Chọn xếp ca học cho lớp Python-K15 vào Phòng LAB-101 (19:30 - 21:30 ngày 15/10/2026)
    UI ->> API: GET /api/v1/rooms/conflicts?roomId=LAB-101&date=2026-10-15&startTime=19:30&endTime=21:30
    API ->> DB: SELECT COUNT(*) FROM class_sessions WHERE room_id = :id AND session_date = :date AND (start_time, end_time) OVERLAPS ('19:30', '21:30') AND status != 'CANCELLED'
    DB -->> API: Phát hiện lớp React-K08 đang dùng phòng từ 18:00 đến 20:00 (Giao thoa 30 phút!)
    API -->> UI: Trả về hasConflict: true, conflictingClass: "React-K08 (18:00 - 20:00)"

    UI ->> UI: Khóa nút [ Lưu Lịch ], viền đỏ ô phòng học kèm thông báo lỗi
    UI ->> Staff: Cảnh báo: "Phòng LAB-101 bị trùng với lớp React-K08 đến 20:00. Vui lòng chọn phòng khác!"
    Staff ->> UI: Đổi phòng sang LAB-102 (Sức chứa 30, còn trống)
    UI ->> API: GET /api/v1/rooms/conflicts?roomId=LAB-102&date=2026-10-15...
    API ->> DB: Kiểm tra phòng LAB-102
    DB -->> API: 0 xung đột
    API -->> UI: hasConflict: false
    UI ->> Staff: Hiển thị tích xanh [ Phòng khả dụng ]
    Staff ->> UI: Bấm [ Xác nhận lưu lịch ]
    UI ->> API: POST /api/v1/classes/:id/sessions (room_id: LAB-102)
    API ->> DB: Lưu thành công buổi học
```

---

## 15. Luồng 14: Sinh Đề Thi Trắc Nghiệm Tự Động & Chấm Điểm Khảo Thí (Question Bank & Quiz)

```mermaid
sequenceDiagram
    autonumber
    actor Teacher as Giáo viên
    actor Student as Học viên
    participant UI as Quiz Portal (/student/quizzes/:id)
    participant API as Backend Server
    participant Engine as Quiz Auto-Grading Engine
    participant DB as PostgreSQL Database

    Teacher ->> API: POST /api/v1/quizzes/generate (5 Dễ, 10 TB, 5 Khó từ ngân hàng QB_GOLANG)
    API ->> DB: Rút ngẫu nhiên 20 câu hỏi và lưu quiz_questions
    API -->> Teacher: Xuất bản đề thi thành công

    Student ->> UI: Truy cập bài thi trắc nghiệm [ Quiz 15 Phút: Go Concurrency ]
    UI ->> API: POST /api/v1/quizzes/:id/start
    API ->> DB: Tạo quiz_attempts (started_at = NOW())
    API -->> UI: Trả về 20 câu hỏi (đã xáo trộn ngẫu nhiên thứ tự câu hỏi và thứ tự A/B/C/D)
    
    UI ->> Student: Đếm ngược thời gian làm bài (15:00... 14:59...)
    Student ->> UI: Chọn đáp án cho 20 câu hỏi và nhấn [ Nộp Bài ]
    UI ->> API: POST /api/v1/quizzes/:id/submit
    API ->> Engine: So khớp đáp án học sinh với bảng questions
    Engine ->> Engine: Tính điểm (17/20 câu đúng => 85/100 điểm, Đạt)
    Engine ->> DB: Lưu quiz_answers, cập nhật quiz_attempts (total_score: 85, is_passed: true)
    Engine ->> DB: Tự động ghi điểm 85 vào Sổ điểm tổng kết lớp học (gradebook)
    API -->> UI: Trả về bảng điểm, số câu đúng và lời giải thích chi tiết
    UI ->> Student: Hiển thị kết quả "Chúc mừng bạn đã đạt 85/100 điểm!"
```

---

## 16. Luồng 15: Giám Sát Radar Nguy Cơ Bỏ Học & Chăm Sóc Học Viên (Student Churn Risk Radar)

```mermaid
sequenceDiagram
    autonumber
    actor Coord as Chuyên viên Vận hành / CSKH
    participant UI as Churn Radar Dashboard (/admin/analytics/churn-risk)
    participant API as Backend Server
    participant DB as PostgreSQL Database
    actor Parent as Phụ huynh học sinh

    Note over API, DB: Trigger hàng ngày sau các ca học
    API ->> DB: Quét học sinh có 2 buổi vắng liên tiếp HOẶC chuyên cần < 70% HOẶC nợ 3 BTVN
    DB -->> API: Danh sách 4 học sinh thuộc diện Cảnh Báo Nguy Cơ Cao

    Coord ->> UI: Truy cập [ Radar Nguy Cơ Bỏ Học ]
    UI ->> Coord: Cảnh báo đỏ: Học viên "Lê Hoàng Long" - Lớp FE-K32 (Vắng 2 buổi, nợ 3 BTVN)
    Coord ->> UI: Nhấp vào hồ sơ Long, xem lịch sử chuyên cần và số điện thoại phụ huynh
    Coord ->> Parent: Gọi điện thăm hỏi lý do vắng và hỗ trợ khó khăn bài tập
    Parent -->> Coord: Trao đổi học sinh bị sốt xuất huyết nằm viện tuần qua
    Coord ->> UI: Nhập ghi chú CSKH: "Nghỉ do ốm nằm viện, đề xuất xếp học bù 2 buổi vào tuần sau"
    Coord ->> UI: Bấm nút [ Tạo lịch học bù ưu tiên ]
    UI ->> API: Cập nhật coordinator_notes và mở popup xếp lịch bù
```

---

## 17. Thiết Kế Giao Diện Trực Quan (ASCII Wireframes)

### Wireframe 1: Giao diện Xếp Lịch Học Bù & Ghép Lớp Song Song (`/operations/makeup`)

```text
+---------------------------------------------------------------------------------------+
|  LMS CENTER - ĐIỀU PHỐI HỌC BÙ CHO HỌC SINH VẮNG                        [ Admin/Coord ]|
+---------------------------------------------------------------------------------------+
|  [!] CẢNH BÁO: Có 3 học viên vắng buổi học chưa được xếp học bù!                      |
|                                                                                       |
|  DANH SÁCH HỌC VIÊN CẦN HỌC BÙ:                                                       |
|  +--------------------+----------+-------------+----------------+------------------+  |
|  | Học Viên           | Lớp Gốc  | Buổi Vắng   | Chủ Đề Bài Học | Hành Động        |  |
|  +--------------------+----------+-------------+----------------+------------------+  |
|  | Nguyễn Hoàng Nam   | FE-K32   | Buổi 4 (T2) | Goroutines & Ch| [ Xếp Lịch Bù ]  |  |
|  | Trần Thảo Linh     | PY-K14   | Buổi 2 (T3) | Pandas & NumPy | [ Xếp Lịch Bù ]  |  |
|  +--------------------+----------+-------------+----------------+------------------+  |
|                                                                                       |
|  POPUP XẾP LỊCH HỌC BÙ: Nguyễn Hoàng Nam (FE-K32 - Buổi 4)                            |
|  +---------------------------------------------------------------------------------+  |
|  | Phương thức học bù:                                                             |  |
|  |  (o) Ghép Lớp Song Song (Được đề xuất)        ( ) Kèm 1-1 với Trợ Giảng/GV      |  |
|  |                                                                                 |  |
|  | Chọn lớp học song song có cùng chủ đề:                                          |  |
|  |  [ FE-K33 - Buổi 4: Goroutines (Thứ 5 19:30 - Phòng LAB-202 - Còn 3 chỗ)      v] |  |
|  |                                                                                 |  |
|  | Thời gian: 19:30 - 21:30, Ngày 18/10/2026                                       |  |
|  | Giảng viên lớp ghép: ThS. Vũ Hải Đăng                                           |  |
|  | Ghi chú CSKH: [Học viên thi giữa kỳ ở trường ĐH, xếp học bù tối T5           ]   |  |
|  |                                                                                 |  |
|  | [x] Tự động gửi tin nhắn Zalo/SMS xác nhận lịch cho Học viên & Phụ huynh        |  |
|  |                                                                                 |  |
|  |                       [ Hủy Bỏ ]    [ XÁC NHẬN LÊN LỊCH BÙ ]                    |  |
|  +---------------------------------------------------------------------------------+  |
+---------------------------------------------------------------------------------------+
```

### Wireframe 2: Bản Đồ Xếp Phòng Học & Cảnh Báo Trùng Phòng (`/admin/classes/schedule`)

```text
+---------------------------------------------------------------------------------------+
|  LMS CENTER - THỜI KHÓA BIỂU & XẾP PHÒNG HỌC (CAMPUS CẦU GIẤY)                        |
+---------------------------------------------------------------------------------------+
|  Ngày: [ 15/10/2026 ]  |  Cơ sở: [ Cơ sở Cầu Giấy v ]  |  Ca: [ Ca Tối: 19:30-21:30 v]|
|                                                                                       |
|  SƠ ĐỒ TRẠNG THÁI PHÒNG HỌC:                                                          |
|  +----------------------+----------------------+----------------------+               |
|  | LAB-101 (25 máy)     | LAB-102 (30 máy)     | TH-201 (Lý Thuyết)   |               |
|  | [X] ĐÃ CÓ LỚP        | [V] CÒN TRỐNG        | [X] ĐÃ CÓ LỚP        |               |
|  | Lớp: React-K08       | Sẵn sàng xếp lớp!    | Lớp: IELTS-K92       |               |
|  | GV: Hoàng Nam        |                      | GV: Ms. Linda        |               |
|  | 18:00 - 20:00 (Trùng!)|                      | 19:00 - 21:00        |               |
|  +----------------------+----------------------+----------------------+               |
|                                                                                       |
|  [!] CẢNH BÁO XUNG ĐỘT PHÒNG HỌC:                                                     |
|  "Không thể xếp lớp Python-K15 vào LAB-101 lúc 19:30 vì phòng đang bị lớp React-K08   |
|   chiếm dụng đến 20:00 (Xung đột 30 phút). Vui lòng chọn LAB-102."                   |
+---------------------------------------------------------------------------------------+
```

### Wireframe 3: Radar Cảnh Báo Nguy Cơ Học Sinh Bỏ Học (`/admin/analytics/churn-risk`)

```text
+---------------------------------------------------------------------------------------+
|  LMS CENTER - RADAR CẢNH BÁO NGUY CƠ BỎ HỌC (STUDENT CHURN RISK)        [ Ban Giám Đốc]|
+---------------------------------------------------------------------------------------+
|  BỘ LỌC: Cơ sở: [ Tất cả v ] | Lớp: [ FE-K32 v ] | Mức độ nguy cơ: [ [!] Nguy cơ cao v]|
|                                                                                       |
|  +--------+------------------+--------+-------------+------------+--------+--------+  |
|  | Mã HV  | Họ Và Tên        | Lớp    | Vắng Liên T.| Chuyên Cần | Nợ BTVN| Thao Tác|  |
|  +--------+------------------+--------+-------------+------------+--------+--------+  |
|  | HV-042 | Lê Hoàng Long    | FE-K32 | [!] 2 buổi  | 62.5%      | 3 bài  | [Chăm Sóc]|
|  | HV-108 | Phạm Thu Hà      | PY-K14 | [!] 3 buổi  | 55.0%      | 2 bài  | [Chăm Sóc]|
|  | HV-215 | Đỗ Minh Quân     | FE-K32 | 1 buổi      | 68.0%      | 4 bài  | [Chăm Sóc]|
|  +--------+------------------+--------+-------------+------------+--------+--------+  |
|                                                                                       |
|  HÀNH ĐỘNG NHANH CHO HỌC VIÊN: Lê Hoàng Long (0988.123.456 - Phụ huynh: 0912.789.012) |
|  [ Gọi Điện CSKH ]  [ Xếp Học Bù Buổi Vắng ]  [ Gia Hạn Deadline BTVN ]  [ Đổi Lớp ]  |
+---------------------------------------------------------------------------------------+
```

---

## 18. Luồng 16: Giảng Viên Sử Dụng Live Class Cockpit Trong Ca Dạy

```mermaid
sequenceDiagram
    autonumber
    actor Teacher as Giảng viên
    participant UI as Live Class Cockpit (/teacher/classes/:id/live)
    participant API as Backend Server
    participant DB as PostgreSQL Database
    actor Students as Học viên trong lớp

    Teacher ->> UI: Truy cập Live Cockpit trước giờ học 5 phút
    Teacher ->> UI: Bấm nút [ 1-Click Launch Meeting ]
    UI ->> API: POST /api/v1/attendance/teacher-checkin (action: CHECK_IN)
    API ->> DB: Ghi nhận check-in thực tế của giáo viên
    UI ->> Teacher: Tự động mở Google Meet / Zoom trong tab mới

    Note over Teacher, Students: Điểm danh nhanh 1 chạm (Quick Attendance)
    Teacher ->> UI: Nhìn danh sách thấy chỉ có 1 bạn Nam vắng
    Teacher ->> UI: Nhấn [ Tất cả có mặt ] -> Bỏ tích riêng Nam (Vắng) -> Bấm [ Lưu điểm danh ]
    UI ->> API: POST /api/v1/attendance/student-batch
    API ->> DB: Lưu điểm danh 19 có mặt, 1 vắng; kích hoạt cảnh báo vắng sau 15p

    Note over Teacher, Students: Giảng bài & Bảng trắng kỹ thuật số (Whiteboard)
    Teacher ->> UI: Mở tab [ Digital Whiteboard ] vẽ sơ đồ kiến trúc Concurrency
    Teacher ->> UI: Nhấn [ Lưu & Chia sẻ sơ đồ bài giảng ]
    UI ->> API: POST /api/v1/teacher/cockpit/:id/whiteboard
    API ->> DB: Lưu nét vẽ và xuất file PDF đính kèm buổi học

    Note over Teacher, Students: Kiểm tra độ hiểu bài ngay tại lớp (Quick Poll)
    Teacher ->> UI: Nhập câu hỏi: "Channel unbuffered có chặn sender không?" -> Bấm [ Phát Poll 2 Phút ]
    UI ->> API: POST /api/v1/teacher/cockpit/:id/quick-poll
    API -->> Students: Hiển thị popup bình chọn A / B trên màn hình học viên
    Students ->> UI: Bấm chọn đáp án
    UI -->> Teacher: Cập nhật biểu đồ phần trăm real-time (85% chọn "Có chặn")
    Teacher ->> Students: Khen ngợi cả lớp nắm vững bài và chuyển sang phần bài tập thực hành
```

---

## 19. Luồng 17: Giảng Viên Sử Dụng Trợ Lý AI Chấm Điểm & Nhận Xét Giọng Nói (Voice Note)

```mermaid
sequenceDiagram
    autonumber
    actor Teacher as Giảng viên
    participant UI as Teacher Grading Workspace
    participant AI as AI Code Review Engine
    participant API as Backend Server
    participant DB as PostgreSQL Database
    actor Student as Học viên nhận bài chấm

    Teacher ->> UI: Mở bài nộp lập trình của học sinh Tuấn Anh (Lớp FE-K32)
    Teacher ->> UI: Bấm nút [ 🤖 Trợ lý AI Quét Bài ]
    UI ->> API: POST /api/v1/submissions/:id/ai-review
    API ->> AI: Gửi mã nguồn Monaco / GitHub repo của học viên
    AI ->> AI: Quét linter, phát hiện 1 lỗi unclosed channel và vi phạm Clean Code
    AI -->> API: Trả về CleanCodeScore: 88, gợi ý bản nháp nhận xét sư phạm chi tiết
    API -->> UI: Điền sẵn nhận xét mẫu vào khung đánh giá

    Teacher ->> UI: Đọc lướt bản nháp của AI trong 10 giây, thấy rất chuẩn xác
    Teacher ->> UI: Bấm nút [ 🎙️ Ghi âm dặn dò (Voice Note) ]
    Teacher ->> UI: Thu âm 40 giây: "Tuấn Anh lưu ý nhớ dùng defer close(ch) tại dòng 38 nhé, code logic rất sáng tạo..."
    Teacher ->> UI: Bấm [ Lưu Điểm 90 & Gửi Bài ]
    UI ->> API: POST /api/v1/classes/:classId/assignments/:asgId/submissions/:id/grade
    API ->> DB: Lưu điểm 90, audio URL và nhận xét; đồng bộ Sổ điểm lớp học
    API -->> Student: Gửi thông báo kèm file âm thanh và nhận xét của thầy cô
```

---

## 20. Luồng 18: Giảng Viên Nhờ Đồng Nghiệp Dạy Thay & Tự Động Quyết Toán

```mermaid
sequenceDiagram
    autonumber
    actor TeacherA as Giảng viên A (Bận đột xuất)
    participant UI as Teacher Portal
    participant API as Backend Server
    participant DB as PostgreSQL Database
    actor TeacherB as Giảng viên B (Cùng chuyên môn, rảnh ca)
    actor Student as Học sinh trong lớp

    TeacherA ->> UI: Tại buổi học tối thứ Năm, bấm [ Nhờ Dạy Thay ]
    TeacherA ->> UI: Chọn lý do: "Bị ốm sốt xuất huyết" kèm ghi chú dặn dò giáo án
    UI ->> API: POST /api/v1/teacher/sessions/:id/substitute-request
    API ->> DB: 1. Truy vấn các GV dạy cùng bộ môn Golang
    API ->> DB: 2. Kiểm tra `teacher_availabilities` thấy GV B đang rảnh ca tối thứ Năm
    API ->> DB: Tạo bản ghi substitute_requests (status: PENDING_OFFER)
    API -->> TeacherB: Bắn thông báo push: "Đồng nghiệp Tuấn nhờ bạn dạy thay ca tối T5 (FE-K32)"

    TeacherB ->> UI: Mở ứng dụng, xem thông tin lớp và đề cương bài giảng buổi 4
    TeacherB ->> UI: Bấm nút [ Đồng Ý Nhận Ca Dạy ]
    UI ->> API: PUT /api/v1/teacher/substitute-requests/:id/accept
    API ->> DB: 1. Cập nhật `class_sessions.assigned_teacher_id = TeacherB`
    API ->> DB: 2. Ghi nhận thù lao ca dạy đó cộng vào bảng lương tháng của TeacherB
    API ->> DB: 3. Chuyển status = ACCEPTED
    API -->> TeacherA: Thông báo: "Thầy Đăng đã nhận ca dạy thay giúp bạn!"
    API -->> Student: Thông báo lớp: "Buổi 4 tối T5 ThS. Vũ Hải Đăng sẽ phụ trách lớp"
```

---

## 21. Bổ Sung Bản Thiết Kế Giao Diện Dành Cho Giáo Viên (Teacher Wireframes)

### Wireframe 4: Bảng Điều Khiển Ca Dạy Trực Tiếp (`/teacher/classes/{id}/live`)

```text
+---------------------------------------------------------------------------------------+
|  LMS CENTER - BẢNG ĐIỀU KHIỂN CA DẠY (LIVE COCKPIT)               GV: ThS. Hoàng Nam  |
+---------------------------------------------------------------------------------------+
|  LỚP: FE-K32 | CA: 19:30 - 21:30 | BÀI: Buổi 4 - Concurrency & Channels              |
|                                                                                       |
|  [ 🔴 GOOGLE MEET: meet.google.com/abc-xyz ]   [ ĐÃ CHECK-IN CA DẠY LÚC 19:28 V ]    |
|                                                                                       |
|  +---------------------------------------+  +--------------------------------------+  |
|  | DANH SÁCH ĐIỂM DANH NHANH (SĨ SỐ 20)  |  | BẢNG TƯƠNG TÁC & MINI POLL           |  |
|  +---------------------------------------+  +--------------------------------------+  |
|  | [ Tất Cả Có Mặt ] [ Lưu Điểm Danh ]   |  | [ Phát Quick Poll 2 Phút ]           |  |
|  | [x] 1. Nguyễn Hoàng Nam   (Online)    |  | Câu hỏi: Channel không đệm có chặn?  |  |
|  | [x] 2. Trần Thảo Linh     (Offline)   |  | (o) Có chặn [===========> ] 85%      |  |
|  | [ ] 3. Lê Hoàng Long      (Vắng mặt)  |  | ( ) Không chặn [==>       ] 15%      |  |
|  | [x] 4. Đỗ Minh Quân       (Giơ tay ✋) |  | Thời gian còn lại: 01:15             |  |
|  +---------------------------------------+  +--------------------------------------+  |
|                                                                                       |
|  [ 🎨 MỞ BẢNG TRẮNG DIGITAL WHITEBOARD ]    [ 📂 GIÁO ÁN SLIDE PDF ]   [ 💻 CODE MẪU ]|
+---------------------------------------------------------------------------------------+
```

### Wireframe 5: Không Gian Chấm Bài Code AI & Thu Âm Lời Dặn Dò

```text
+---------------------------------------------------------------------------------------+
|  CHẤM BÀI TẬP VỀ NHÀ: BTVN-03 Goroutines (Học viên: Trần Tuấn Anh)     [ Đóng / Lưu ]  |
+---------------------------------------------------------------------------------------+
|  MÃ NGUỒN HỌC VIÊN NỘP (MONACO VIEWER):  |  TRỢ LÝ CHẤM ĐIỂM & ĐÁNH GIÁ SƯ PHẠM:      |
|  1  package main                         |                                             |
|  2  func worker(ch chan int) {           |  [ 🤖 AI QUÉT BÀI TỰ ĐỘNG ]                 |
|  3     val := <-ch                       |  - Điểm Clean Code gợi ý: 88/100            |
|  4     // Logic xử lý                    |  - Cảnh báo: Dòng 14 channel chưa close.    |
|  5  }                                    |                                             |
|  ...                                     |  NHẬN XÉT SƯ PHẠM (ĐÃ ĐIỀN BẢN NHÁP AI):    |
|  14 ch := make(chan int)                 |  [Em làm rất tốt phần khởi tạo goroutine.  ]|
|  15 go worker(ch)                        |  [Lưu ý dòng 14 nhớ defer close(ch) nhé!   ]|
|                                          |                                             |
|                                          |  ĐIỂM SỐ: [ 90 ] / 100                      |
|                                          |                                             |
|                                          |  [ 🎙️ BẤM GHI ÂM DẶN DÒ (VOICE NOTE) ]     |
|                                          |  ▶️ || 00:38 / 00:40 (Đã ghi âm thành công) |
|                                          |                                             |
|                                          |  [ GHI CHÚ SƯ PHẠM KÍN (CHỈ TA XEM) ]       |
|                                          |  [Tuấn Anh tiếp thu nhanh, cần giao thêm lab]|
|                                          |                                             |
|                                          |        [ LƯU ĐIỂM & TRẢ BÀI HỌC VIÊN ]      |
+---------------------------------------------------------------------------------------+
```



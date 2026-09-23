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


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

## 5. Bố Cục Giao Diện Màn Hình Trọng Yếu (Key Wireframe Schematics)

### 5.1 Giao diện Học viên làm bài tập CNTT (Monaco Code Workspace)
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
+---------------------------------------+-------------------------------------+
```

### 5.2 Giao diện Giảng viên Điểm danh Lớp Hybrid (Attendance Sheet)
```
+-----------------------------------------------------------------------------+
| Lớp: LMS-FE-K32 (Hybrid) | Buổi 5 (23/09/2026) | [Check-out ca dạy: 21:30]   |
+-----------------------------------------------------------------------------+
| Tìm học viên: [ Tìm kiếm tên / mã... ]         Sĩ số: 18 học viên           |
|                                                                             |
| #  Mã HV      Họ và tên        Trạng thái tham gia          Ghi chú buổi học |
| 1  HV-001     Trần Thị Mai     (•) Offline  ( ) Online      [..............] |
|                                ( ) Đi trễ   ( ) Vắng                         |
| 2  HV-002     Lê Hoàng Nam     ( ) Offline  (•) Online      [Học qua Meet  ] |
|                                ( ) Đi trễ   ( ) Vắng                         |
| 3  HV-003     Phạm Minh Đức    ( ) Offline  ( ) Online      [Xin phép trễ  ] |
|                                (•) Đi trễ (15p) ( ) Vắng                     |
+-----------------------------------------------------------------------------+
| [ Tải lại ]                                         [ 💾 LƯU ĐIỂM DANH ]    |
+-----------------------------------------------------------------------------+
```

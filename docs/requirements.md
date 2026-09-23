# Đặc Tả Yêu Cầu Phần Mềm (Software Requirements Specification - SRS)

> **Dự án:** LMS Center Platform (Hệ thống Quản lý Học tập, Giảng viên & Vận hành Đào tạo Đa hình thức)  
> **Tài liệu:** `docs/requirements.md`  
> **Phiên bản:** 1.2.0 (Cập nhật Phân cấp Quản lý, Vận Hành Lớp, Lịch Học Linh Hoạt & Đánh Giá Moodle)  
> **Ngày cập nhật:** 23/09/2026  

---

## 1. Mục Đích & Phạm Vi Dự Án (Scope & Objectives)

Hệ thống **LMS Center Platform** là giải pháp phần mềm quản lý toàn diện dành cho các trung tâm đào tạo hiện đại với trọng tâm chuyên sâu là **Công nghệ Thông tin (CNTT)**, kết hợp mở rộng linh hoạt cho **Ngoại ngữ** và các bộ môn khác.

Hệ thống giải quyết triệt để các bài toán vận hành thực tế:
- **Chuẩn hóa quản trị RBAC phân cấp:** Phân định rõ quyền hạn giữa Super Admin, Quản lý đào tạo (xếp lớp, đánh giá giáo viên), Chuyên viên vận hành lớp (chăm sóc học viên, điều phối), Giảng viên, Học viên và Phụ huynh.
- **Xếp lịch học linh hoạt:** Hỗ trợ lịch 1 buổi/tuần, nhiều buổi/tuần, dời ngày, đổi phòng/meet link, bù buổi học dễ dàng.
- **Tự động hóa điểm danh hai chiều:** Điểm danh học viên lớp Offline, Online, Hybrid và Chấm công giáo viên (Timesheet).
- **Môi trường thực hành lập trình trực tuyến:** Tích hợp Monaco Code Editor cho môn CNTT, nộp link GitHub/zip, hỗ trợ ghi âm bài nói cho ngoại ngữ.
- **Kế thừa & Nâng cấp các tính năng tinh hoa từ Moodle:** Sổ điểm đa trọng số (Weighted Gradebook), Mở khóa bài học theo tiến độ (Drip Content), Ngân hàng câu hỏi & Đề thi trắc nghiệm (Question Bank & Quiz), Nhân bản khóa học (Course Cloning).

---

## 2. Các Tác Nhân Hệ Thống & Ma Trận Phân Cấp (Actors & Roles)

```mermaid
flowchart TD
    SuperAdmin["1. Super Admin (Chủ trung tâm / Giám đốc)"]
    AcademicMgr["2. Academic Manager (Quản lý Đào tạo / Giáo vụ trưởng)"]
    ClassCoord["3. Class Coordinator (Chuyên viên Vận hành lớp & CSKH)"]
    Teacher["4. Giảng viên & Trợ giảng (Teacher / TA)"]
    Student["5. Học viên (Student)"]
    Parent["6. Phụ huynh (Parent)"]

    SuperAdmin --> AcademicMgr
    SuperAdmin --> ClassCoord
    AcademicMgr --> |Xếp lớp, Lịch dạy, Đánh giá| Teacher
    ClassCoord --> |Đồng hành, Chăm sóc, Nhắc nhở| Student
    ClassCoord --> |Báo cáo chuyên cần, Hỗ trợ| Parent
    Teacher --> |Giảng dạy, Điểm danh, Chấm bài| Student
    Student -.-> Parent
```

### Chi tiết vai trò:
1. **Super Admin (Chủ trung tâm / Quản trị viên cấp cao):** Toàn quyền cấu hình hệ thống, quản lý tài chính, phân quyền các Role trong bảng `roles` và `permissions`.
2. **Academic Manager (Quản lý Đào tạo / Giáo vụ trưởng):**
   - Tạo khóa học, phân công giảng viên & trợ giảng.
   - Xếp lớp học, thiết lập thời khóa biểu định kỳ và xử lý dời lịch/bù buổi.
   - Đánh giá chất lượng giảng viên (Teacher Evaluation / Audit), duyệt giáo án và tài liệu khóa học.
3. **Class Coordinator / Operations & Care (Chuyên viên Vận hành lớp & Chăm sóc học viên):**
   - Mỗi lớp học có 1 Chuyên viên vận hành đồng hành cùng Giảng viên và Trợ giảng.
   - Theo dõi chuyên cần buổi học, trực tiếp liên hệ học sinh vắng hoặc đi trễ.
   - Ghi nhận `coordinatorNote` (lý do nghỉ ốm, hoàn cảnh học sinh) gửi cho Phụ huynh.
   - Đôn đốc học viên nộp BTVN trước deadline, tiếp nhận và giải quyết phản hồi về chất lượng buổi học.
4. **Giảng viên & Trợ giảng (Teacher & Teaching Assistant):**
   - Xem thời khóa biểu cá nhân, Check-in / Check-out ca dạy để chấm công.
   - Điểm danh học sinh lớp Hybrid (Offline/Online), upload tài liệu bài học, giao BTVN và chấm điểm theo rubric.
5. **Học viên (Student):**
   - Xem lịch học, học bài giảng YouTube, tải tài liệu bài học.
   - Làm BTVN CNTT trực tiếp trên Monaco Code Editor hoặc nộp link GitHub / file ghi âm.
   - Xem điểm số, nhận xét và thanh toán học phí qua mã VietQR.
6. **Phụ huynh (Parent):**
   - Theo dõi chuyên cần từng buổi, điểm số bài tập và nhận xét của thầy cô.
   - Nhận thông báo học phí và quét mã VietQR thanh toán.

---

## 3. Yêu Cầu Chức Năng Chi Tiết (Functional Requirements)

### Phân hệ 1: Quản Trị RBAC & Phân Quyền Hạt Nhân (Dynamic RBAC Engine)

- **FR-RBAC-01 (Quản lý Roles & Permissions):** CSDL sử dụng các bảng `roles`, `permissions`, `role_permissions`, `user_roles`. Super Admin có thể tạo thêm role mới (ví dụ: "Nhân viên Tư vấn Tuyển sinh", "Kế toán") và gán các permissions cụ thể.
- **FR-RBAC-02 (Cấp quyền linh hoạt):** Một người dùng có thể đảm nhiệm nhiều vai trò (ví dụ: vừa là Giảng viên vừa là Quản lý Đào tạo).

---

### Phân hệ 2: Xếp Lớp & Lịch Học Linh Hoạt (Flexible Scheduling Engine)

- **FR-SCH-01 (Quy tắc lịch lặp lại - Recurrence Rule):**
  - Khi mở lớp, Quản lý Đào tạo cấu hình lịch học:
    - *1 buổi/tuần:* Ví dụ sáng Chủ Nhật (08:30 - 11:30).
    - *Nhiều buổi/tuần:* Ví dụ T2-T4-T6 (19:30 - 21:30) hoặc T3-T5 (18:00 - 20:00).
  - Hệ thống tự động sinh toàn bộ danh sách các buổi học (`class_sessions`) cho cả khóa học.
- **FR-SCH-02 (Tùy biến & Dời lịch từng buổi - Rescheduling):**
  - Quản lý Đào tạo hoặc Chuyên viên Vận hành lớp có thể:
    - **Dời ngày học:** Đổi ngày buổi học sang ngày khác, hệ thống lưu `originalDate` và `rescheduleReason`.
    - **Đổi giờ học / Đổi phòng:** Chuyển ca học sang phòng khác nếu phòng cũ bảo trì.
    - **Đổi link Meet:** Cập nhật link Google Meet riêng cho buổi đó nếu cần.
    - **Bù buổi học:** Thêm buổi học phát sinh vào khóa học.
  - Tự động gửi thông báo cập nhật lịch học tới Giảng viên, Học viên và Phụ huynh.
- **FR-SCH-03 (Phân công Giảng viên dạy thay):** Khi giảng viên chính có việc bận, Quản lý Đào tạo có thể gán `assignedTeacherId` là một giáo viên khác chỉ riêng cho buổi học đó, hệ thống tự động ghi nhận công dạy buổi đó cho giáo viên dạy thay.

---

### Phân hệ 3: Vận Hành Lớp Học & Chăm Sóc Học Viên (Class Operations & Student Care)

- **FR-OPS-01 (Mô hình Tam Giác Lớp Học):** Mỗi lớp học được quản lý bởi bộ 3:
  1. *Giảng viên chính:* Chịu trách nhiệm chuyên môn & bài giảng.
  2. *Trợ giảng (TA):* Hỗ trợ kỹ thuật, giải đáp bài tập code cho học viên.
  3. *Chuyên viên Vận hành (Class Coordinator):* Quản lý kỷ luật, sĩ số, chăm sóc học viên và phụ huynh.
- **FR-OPS-02 (Ghi chú Chăm sóc Học viên - Student Care Log):**
  - Khi học sinh vắng mặt hoặc đi trễ, Chuyên viên Vận hành vào danh sách lớp xem thông tin liên hệ và gọi điện thoại.
  - Nhập `coordinatorNote` (ví dụ: "Đã liên hệ mẹ, bạn bị sốt xin nghỉ buổi 3, đã gửi link video xem lại").
  - Phụ huynh và Giáo vụ xem được lịch sử chăm sóc này.
- **FR-OPS-03 (Đôn đốc nộp bài tập):** Chuyên viên Vận hành có màn hình theo dõi tiến độ nộp BTVN của cả lớp, tự động lọc danh sách các học viên chưa nộp bài khi còn 24h trước deadline để gửi tin nhắn nhắc nhở.

---

### Phân hệ 4: Đánh Giá Chất Lượng Giáo Viên (Teacher Quality Evaluation)

- **FR-EVAL-01 (Đánh giá từ Quản lý Đào tạo - Academic Audit):**
  - Quản lý Đào tạo dự giờ hoặc kiểm tra định kỳ chất lượng lớp học.
  - Đánh giá giảng viên theo các tiêu chí: Kiến thức chuyên môn, Kỹ năng sư phạm, Mức độ tương tác với học viên, Tác phong đúng giờ (dựa trên dữ liệu check-in/out).
  - Lưu trữ kết quả và tính điểm KPI của giảng viên.
- **FR-EVAL-02 (Khảo sát học viên cuối khóa - Student Feedback Survey):**
  - Cuối khóa học hoặc định kỳ giữa kỳ, học viên làm form khảo sát ẩn danh đánh giá giáo viên và trợ giảng (chấm điểm sao từ 1-5 và nhận xét góp ý).
  - Hệ thống tự động tổng hợp báo cáo điểm hài lòng của giảng viên để Ban Giám đốc trung tâm xem xét thù lao và khen thưởng.

---

### Phân hệ 5: Các Tính Năng Kế Thừa & Nâng Cấp Từ Moodle (Moodle-Inspired Features)

Dưới đây là các tính năng tinh hoa từ hệ thống Moodle được chọn lọc và tinh gọn cho trải nghiệm trung tâm đào tạo:

| Tính Năng Moodle | Cơ Chế Ứng Dụng Trong Hệ Thống LMS Của Chúng Ta | Giá Trị Mang Lại Cho Trung Tâm |
| :--- | :--- | :--- |
| **1. Activity Completion & Drip Content** | Cấu hình bài học tuần tự: Học viên phải xem video YouTube đạt $\ge 80\%$ thời lượng hoặc nộp xong BTVN bài trước thì bài học tiếp theo mới tự động mở khóa. | Tránh học viên học nhảy cóc, đảm bảo lộ trình tiếp thu kiến thức bài bản. |
| **2. Weighted Gradebook (Sổ điểm đa trọng số)** | Cho phép Quản lý Đào tạo cấu hình trọng số điểm cho từng khóa học (ví dụ: Chuyên cần 10%, BTVN 30%, Thi giữa kỳ 20%, Đồ án cuối kỳ 40%). Hệ thống tự động tính điểm tổng kết. | Đánh giá năng lực học viên toàn diện, chuẩn hóa xếp loại tốt nghiệp (Xuất sắc, Giỏi, Khá, Trung bình). |
| **3. Question Bank & Random Quiz** | Ngân hàng câu hỏi trắc nghiệm chia theo chuyên đề và mức độ khó (Dễ, Trung bình, Khó). Khi học viên làm bài kiểm tra, hệ thống tự động bốc ngẫu nhiên câu hỏi theo cấu trúc đề để chống nhìn bài nhau. | Hỗ trợ thi chứng chỉ đầu ra, kiểm tra định kỳ tự động chấm điểm ngay tức thì. |
| **4. Course Template Cloning (Nhân bản khóa học)** | Quản lý Đào tạo khi mở lớp mới chỉ cần nhấn nút "Nhân bản khóa học" (Clone Course): toàn bộ Module, Bài giảng YouTube, Tài liệu slide và BTVN mẫu được sao chép sang lớp mới chỉ trong 2 giây. | Tiết kiệm 95% thời gian giáo vụ nhập liệu khi mở khóa mới hàng tháng. |
- **FR-EVAL-03 (Đánh giá giáo viên theo từng buổi học từ học sinh - Per-Session Feedback):**
  - Sau khi buổi học kết thúc hoặc trong vòng 24h, khi học sinh truy cập hệ thống sẽ xuất hiện form đánh giá nhanh buổi học (thao tác dưới 15 giây):
    - Đánh giá sao Giảng viên (1 đến 5 sao).
    - Đánh giá sao Trợ giảng (1 đến 5 sao, nếu có).
    - Mức độ tiếp thu bài học (`POOR`, `AVERAGE`, `GOOD`, `EXCELLENT`).
    - Tốc độ giảng dạy (`TOO_SLOW`, `JUST_RIGHT`, `TOO_FAST`).
    - Nhận xét ẩn danh (tùy chọn).
  - Dữ liệu thống kê được cập nhật ngay lập tức cho Quản lý Đào tạo và Chuyên viên Vận hành lớp. Nếu một buổi học có điểm trung bình $< 3.5$ sao hoặc nhiều học sinh báo "Không hiểu bài / Dạy quá nhanh", hệ thống lập tức gắn cờ cảnh báo đỏ để trung tâm can thiệp và hỗ trợ học sinh ngay trong tuần.

---

### Phân hệ 6: Tính Năng Khóa Học Online Chuyên Sâu (Kế Thừa Từ Frappe LMS)

Tham khảo từ kiến trúc hiện đại của **Frappe LMS** (`https://github.com/frappe/lms`), hệ thống tích hợp các tính năng tối ưu trải nghiệm học online:

| Tính Năng Frappe LMS | Ứng Dụng Trong Hệ Thống LMS Của Chúng Ta | Lợi Ích Trực Tiếp |
| :--- | :--- | :--- |
| **1. In-Lesson Timestamped Q&A** | **Hỏi đáp gắn mốc thời gian video bài giảng:** Học viên có thể đặt câu hỏi ngay tại phút/giây thứ $T$ của video YouTube. Giảng viên hoặc trợ giảng bấm vào câu hỏi là video tự động tua tới đúng đoạn đó để xem ngữ cảnh và trả lời. Giảng viên có nút tích xanh xác nhận *"Câu trả lời chuẩn xác"*. | Loại bỏ việc học sinh hỏi chung chung "thầy ơi đoạn đó em không hiểu", tiết kiệm 80% thời gian trao đổi. |
| **2. Personal Course Notes** | **Ghi chú cá nhân trong khi học:** Học viên có khung ghi chú riêng tư ngay cạnh video bài giảng, lưu tự động kèm mốc thời gian video. Học viên có thể xem lại toàn bộ ghi chú của khóa học hoặc xuất ra file Markdown/PDF. | Tăng tính chủ động ghi chép và ghi nhớ kiến thức của học viên khi học online. |
| **3. Public Verifiable Certificate** | **Chứng chỉ số có URL xác thực công khai:** Khi học sinh hoàn thành khóa học và đạt chuẩn đầu ra, hệ thống tự động sinh chứng chỉ PDF kèm một mã định danh duy nhất và đường dẫn công khai `/verify/[certificateCode]` để nhà tuyển dụng xác minh trực tiếp. | Tăng uy tín thương hiệu của trung tâm, học viên dễ dàng đính kèm vào CV hoặc chia sẻ lên LinkedIn. |
| **4. Cohort Batches & Self-Paced Blending** | **Mô hình học kết hợp Đợt (Batch) và Tự học:** Khóa học online được tổ chức theo từng Batch (khóa khai giảng). Học viên có thể xem trước các video lý thuyết theo nhịp độ cá nhân (Self-paced), sau đó tham gia các buổi thực hành / giải đáp trực tiếp theo lịch cố định của Batch. | Tối ưu hóa thời lượng giảng dạy của giáo viên, học viên chủ động học trước lý thuyết để dành trọn vẹn giờ học trực tuyến cho thực hành. |

---

## 4. Bổ Sung Ma Trận User Stories (Acceptance Criteria)

| Mã Story | Đối tượng | Hành động (User Story) | Tiêu chí chấp nhận (Acceptance Criteria) |
| :--- | :--- | :--- | :--- |
| **US-SCH-01** | Quản lý Đào tạo | Tạo lớp học với lịch 1 tuần 1 buổi hoặc nhiều buổi | - Chọn được lịch tuần linh hoạt (ví dụ: T2-T4-T6 lúc 19:30).<br>- Tự động sinh ra đúng $N$ buổi học trong bảng `class_sessions`. |
| **US-SCH-02** | Quản lý / Vận hành | Dời ngày học một buổi cụ thể khi có sự cố | - Đổi ngày từ ngày $D$ sang $D'$, nhập lý do dời lịch.<br>- Hệ thống cập nhật và gửi thông báo cho giáo viên, học sinh, phụ huynh. |
| **US-OPS-01** | Chuyên viên Vận hành | Ghi chú chăm sóc học viên vắng học | - Xem được danh sách học viên vắng sau khi giáo viên điểm danh.<br>- Nhập ghi chú chăm sóc `coordinatorNote`, phụ huynh đọc được ghi chú. |
| **US-EVAL-01** | Quản lý Đào tạo | Đánh giá chất lượng giáo viên định kỳ | - Form chấm điểm các tiêu chí sư phạm, chuyên môn, đúng giờ.<br>- Cập nhật điểm KPI vào hồ sơ giảng viên. |
| **US-EVAL-02** | Học viên | Đánh giá giáo viên ngay sau buổi học | - Form đánh giá nhanh 1-5 sao, mức độ hiểu bài, tốc độ dạy.<br>- Đánh giá ẩn danh, cảnh báo cờ đỏ cho quản lý nếu điểm dưới 3 sao. |
| **US-FRP-01** | Học viên | Đặt câu hỏi tại mốc thời gian video YouTube | - Nhập câu hỏi kèm timestamp video.<br>- Click câu hỏi video tự tua đến đúng đoạn; Giáo viên có nút tích xanh câu trả lời đúng. |
| **US-FRP-02** | Học viên | Nhận chứng chỉ có link xác thực công khai | - Hoàn thành khóa học tự động cấp chứng chỉ.<br>- Link `/verify/[code]` hiển thị thông tin khóa học và tên học viên công khai. |
| **US-MDL-01** | Quản lý Đào tạo | Cấu hình trọng số điểm cho khóa học | - Nhập % chuyên cần, % BTVN, % thi giữa kỳ, % đồ án cuối kỳ (tổng = 100%).<br>- Sổ điểm tự động tính điểm trung bình môn theo đúng công thức trọng số. |
| **US-MDL-02** | Quản lý Đào tạo | Nhân bản khóa học sang lớp mới | - Bấm "Nhân bản khóa học" sao chép đầy đủ nội dung bài học, video và BTVN.<br>- Không sao chép học viên hay điểm số của lớp cũ. |


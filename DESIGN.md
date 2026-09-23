---
tokens:
  colors:
    brand:
      primary: "#0284c7"         # Sky/Ocean Blue - tin cậy, tri thức
      primary_hover: "#0369a1"
      primary_active: "#075985"
      primary_light: "#e0f2fe"
    neutral:
      background: "#0f172a"      # Dark slate for dark theme / #ffffff for light
      surface: "#1e293b"         # Card & container background
      surface_hover: "#334155"
      border: "#334155"
      text_primary: "#f8fafc"
      text_secondary: "#94a3b8"
      text_muted: "#64748b"
    feedback:
      success: "#10b981"         # Emerald - Có mặt, Hoàn thành, Đã đóng học phí
      success_bg: "#064e3b"
      warning: "#f59e0b"         # Amber - Đi trễ, Bài tập sắp hết hạn, Chờ duyệt
      warning_bg: "#78350f"
      danger: "#ef4444"          # Rose/Red - Vắng mặt, Chưa đóng học phí, Quá hạn
      danger_bg: "#7f1d1d"
      info: "#38bdf8"            # Light blue - Thông báo, Buổi học online
      info_bg: "#0c4a6e"
    roles:
      admin: "#3b82f6"           # Blue badge
      teacher: "#10b981"         # Emerald badge
      student: "#0ea5e9"         # Sky badge
      parent: "#f59e0b"          # Amber badge
    modes:
      offline: "#10b981"         # Lớp offline / Học tại lớp
      online: "#0284c7"          # Lớp online / Google Meet
      hybrid: "#8b5cf6"          # Lớp hybrid / Linh hoạt (chú ý: dùng indigo/cyan làm chính)
  typography:
    fonts:
      sans: "Inter, -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif"
      mono: "'JetBrains Mono', 'Fira Code', Menlo, Monaco, Consolas, monospace"
    sizes:
      xs: "0.75rem"      # 12px
      sm: "0.875rem"     # 14px - Body secondary, table data
      base: "1rem"       # 16px - Standard body text
      lg: "1.125rem"     # 18px - Subheadings, card titles
      xl: "1.25rem"      # 20px - Section titles
      "2xl": "1.5rem"    # 24px - Page titles
      "3xl": "1.875rem"  # 30px - Dashboard main headers
    weights:
      normal: 400
      medium: 500
      semibold: 600
      bold: 700
  spacing:
    container_padding: "1.5rem"
    card_padding: "1.25rem"
    gap_sm: "0.5rem"
    gap_md: "1rem"
    gap_lg: "1.5rem"
  rounded:
    sm: "0.375rem"
    md: "0.5rem"
    lg: "0.75rem"
    xl: "1rem"
    full: "9999px"
  elevation:
    card: "0 1px 3px 0 rgba(0, 0, 0, 0.1), 0 1px 2px -1px rgba(0, 0, 0, 0.1)"
    card_hover: "0 10px 15px -3px rgba(0, 0, 0, 0.2), 0 4px 6px -4px rgba(0, 0, 0, 0.2)"
    modal: "0 20px 25px -5px rgba(0, 0, 0, 0.4), 0 8px 10px -6px rgba(0, 0, 0, 0.4)"
---

# LMS Center Platform - Design Specification & System Tokens

> **Nguồn chân lý duy nhất (Single Source of Truth) cho thiết kế giao diện hệ thống LMS Trung tâm Đào tạo.**  
> Tuân thủ định dạng Design Tokens chuẩn (`tokens.json` & Tailwind CSS v4) và các quy chuẩn UI/UX giáo dục hiện đại.

---

## 1. Tổng Quan Triết Lý Thiết Kế (Overview)

Hệ thống LMS được thiết kế theo phong cách **Modern Professional Educational Dashboard**:
- **Trọng tâm & Tối giản (Focus-driven):** Giảm thiểu yếu tố gây xao nhãng để học viên tập trung học bài và viết code; giảng viên thao tác chấm điểm và điểm danh nhanh chóng (< 3 cú nhấp chuột).
- **Tương phản cao & Công thái học (Ergonomics):** Hỗ trợ Dark Mode làm mặc định cho các màn hình lập trình (Monaco Editor) và Light Mode tươi sáng, tin cậy cho cổng Phụ huynh và Giáo vụ.
- **Rõ ràng theo vai trò (Role-based visual identity):** Mỗi portal (Admin, Teacher, Student, Parent) có màu huy hiệu nhận diện rõ ràng giúp người dùng luôn ý thức được ngữ cảnh làm việc hiện tại.

---

## 2. Màu Sắc & Nhận Diện (Colors)

### 2.1 Bảng màu chính (Brand & Feedback)
- **Primary Brand (`#0284c7` - Ocean Blue):** Đại diện cho tri thức, tư duy logic và công nghệ. Dùng cho các nút hành động chính (Primary CTA), liên kết điều hướng đang kích hoạt, thanh tiến độ học tập.
- **Success (`#10b981` - Emerald):** Dùng cho trạng thái `Có mặt tại lớp`, `Bài tập đã đạt`, `Hóa đơn đã thanh toán`.
- **Warning (`#f59e0b` - Amber):** Dùng cho trạng thái `Đi trễ`, `Bài tập sắp đến hạn`, `Lớp học sắp bắt đầu`, `Học phí còn nợ`.
- **Danger (`#ef4444` - Rose/Red):** Dùng cho trạng thái `Vắng mặt`, `Học phí quá hạn`, `Nộp trễ`.
- **Info (`#38bdf8` - Sky):** Dùng cho trạng thái `Lớp học Online`, `Link Google Meet`, `Thông báo chung`.

### 2.2 Quy chuẩn cấm (Color Bans)
- Tuyệt đối không sử dụng màu tím sặc sỡ / neon violet làm màu chủ đạo cho các nút CTA hoặc nền chính.
- Màu chữ luôn đảm bảo độ tương phản chuẩn WCAG 2.1 AA (tối thiểu 4.5:1 đối với văn bản thường).

---

## 3. Kiểu Chữ (Typography)

- **Font chữ giao diện chính (Sans-serif):** `Inter`  
  Đem lại sự sắc nét, dễ đọc trên mọi độ phân giải màn hình từ Desktop đến Mobile.
- **Font chữ mã nguồn & Dữ liệu kỹ thuật (Monospace):** `JetBrains Mono` hoặc `Fira Code`  
  Áp dụng cho toàn bộ khối code trong bài học, Monaco Code Editor, mã học viên (`HV-2026-001`), mã lớp học (`LMS-FE-01`), mã VietQR payload.

---

## 4. Bố Cục Giao Diện 4 Cổng Portal (Layouts)

Hệ thống sử dụng bố cục chuẩn **App Shell**:
- **Sidebar cố định bên trái (Desktop 260px / Thu gọn 72px trên Tablet):**
  - Logo trung tâm đào tạo & Tên cơ sở.
  - Menu điều hướng phân cấp rõ ràng theo vai trò người dùng.
  - Widget thông tin cá nhân thu gọn (Avatar, Tên, Vai trò, Nút Đăng xuất).
- **Topbar trên cùng (Chiều cao 64px):**
  - Thanh tìm kiếm nhanh (khóa học, lớp học, học viên).
  - Nút chuyển đổi Dark/Light mode.
  - Chuông thông báo (Thông báo nộp bài, điểm danh, thanh toán).
- **Content Canvas (Khu vực nội dung chính):**
  - Giới hạn độ rộng tối đa `1440px` để đảm bảo tầm mắt dễ chịu.
  - Hỗ trợ cuộn độc lập với Sidebar.

### Đặc điểm bố cục từng Portal:
1. **Admin Portal (`/admin`):** Bố cục dạng Dashboard với các thẻ thống kê tổng quan (Cards metric), bảng dữ liệu có bộ lọc nâng cao (Data Tables với phân trang, search, export).
2. **Teacher Portal (`/teacher`):** Tối ưu cho thao tác theo dòng sự kiện: Thời khóa biểu tuần, Nút Check-in ca dạy nổi bật, Bảng điểm danh học sinh nhanh và Giao diện chấm bài split-screen (bên trái là code học viên, bên phải là khung chấm điểm rubric).
3. **Student Portal (`/student`):** Tối ưu cho trải nghiệm học tập: Lịch học hôm nay trên đầu trang, Khóa học đang học với thanh tiến độ %, Danh sách bài tập cần nộp (To-do list), Giao diện làm bài tập Monaco Editor toàn màn hình.
4. **Parent Portal (`/parent`):** Bố cục thân thiện, dễ hiểu, ưu tiên hiển thị trên điện thoại: Tỷ lệ chuyên cần của con dạng biểu đồ tròn, Điểm số bài kiểm tra mới nhất, Thẻ thanh toán học phí với mã VietQR quét ngay tức thì.

---

## 5. Thiết Kế Các Thành Phần Đặc Thù (Specialized Components)

### 5.1 Trình Soạn Thảo Mã Nguồn (Monaco Code Editor Component)
- **Vị trí:** Màn hình làm BTVN CNTT của học viên & Màn hình chấm bài của giáo viên.
- **Quy chuẩn:**
  - Tích hợp thanh công cụ: Dropdown chọn ngôn ngữ (`JavaScript`, `TypeScript`, `Python`, `C++`, `Java`, `HTML/CSS`), nút Đặt lại code, nút Định dạng code (Prettier), nút Phóng to toàn màn hình.
  - Theme mặc định: `vs-dark` tạo cảm giác chuyên nghiệp giống Visual Studio Code.
  - Hỗ trợ tính năng Read-only khi giáo viên chấm bài hoặc khi đã quá hạn nộp bài.

### 5.2 Trình Phát Video YouTube & Học Liệu Đi Kèm
- **Tỉ lệ khung hình:** `16:9` chuẩn Responsive.
- **Tích hợp:** Nền tảng điều khiển nhúng YouTube IFrame API.
- **Thanh Tab học liệu bên dưới:**
  - Tab 1: *Nội dung bài học* (Tóm tắt kiến thức định dạng Markdown).
  - Tab 2: *Tài liệu đính kèm* (Danh sách file PDF slide, mã nguồn mẫu kèm nút tải về).
  - Tab 3: *Hỏi đáp (Q&A)* (Thảo luận giữa học viên và trợ giảng).

### 5.3 Thẻ Sinh Mã VietQR Thanh Toán Động (VietQR Card)
- **Khung chứa:** Thẻ Card màu trắng tương phản cao, bo góc `16px`, viền xám nhạt sang trọng.
- **Mã QR:** Kích thước `220x220px` hiển thị sắc nét, có logo Napas và ngân hàng ở giữa.
- **Thông tin giao dịch rõ ràng:**
  - Tên chủ tài khoản & Tên ngân hàng.
  - Số tiền (Ví dụ: `3.500.000 đ`).
  - Nội dung chuyển khoản (Ví dụ: `HP NGUYEN VAN A K32`).
  - Nút *Sao chép số tài khoản* và *Sao chép nội dung* nhanh.

### 5.4 Bảng Điểm Danh Học Sinh Lớp Hybrid (Attendance Sheet)
- **Dạng hiển thị:** Danh sách học viên có ảnh đại diện, họ tên và 4 nút chọn trạng thái nhanh (Segmented Control / Radio Buttons):
  - `[ Có mặt tại lớp (Offline) ]` (Xanh lá)
  - `[ Tham gia Online (Meet) ]` (Xanh dương)
  - `[ Đi trễ ]` (Cam)
  - `[ Vắng mặt ]` (Đỏ)
- Kèm theo ô nhập ghi chú cá nhân (ví dụ: "xin phép về sớm 15p").

---

## 6. Nguyên Tắc Trải Nghiệm (Do's & Don'ts)

| Nên Làm (Do's) | Tuyệt Đối Tránh (Don'ts) |
| :--- | :--- |
| Hiển thị trạng thái tải (Skeleton loaders) khi đang fetch bài giảng, lịch dạy | Không để màn hình trắng hoặc giật lag khi đang tải video YouTube |
| Cho phép học viên lưu nháp (Auto-save) bài tập code trên Monaco Editor | Không làm mất code của học viên khi vô tình refresh trang |
| Xác nhận rõ ràng (Confirmation Dialog) trước khi nộp bài tập hoặc xóa khóa học | Không tự ý submit bài tập khi chưa có sự xác nhận của học viên |
| Thiết kế nút bấm và mục chọn điểm danh kích thước tối thiểu `44x44px` trên điện thoại | Không để các nút bấm quá nhỏ gây ấn nhầm khi điểm danh trên mobile |
| Tự động sao chép mã chuyển khoản học phí chỉ bằng 1 cú nhấp chuột | Không bắt phụ huynh phải gõ tay cú pháp chuyển khoản dài dòng |

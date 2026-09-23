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

### 5.5 Khoang Lái Lớp Học Trực Tuyến & Bảng Trắng (Live Class Cockpit & Digital Whiteboard)
- **Vị trí:** Màn hình `/teacher/classes/{id}/live`.
- **Quy chuẩn:**
  - Nút Primary CTA: `[ Bắt đầu lớp học ]` màu `#0284c7` (kích thước lớn, kèm icon Video Camera) tự động mở Meet/Zoom trong tab mới và check-in GV.
  - Điểm danh nhanh 1 chạm (Quick Attendance 1-Click).
  - Khởi tạo nhanh Mini Poll 2 phút kèm biểu đồ kết quả thời gian thực.
  - **Bảng vẽ kỹ thuật số (Digital Whiteboard):** Tạm hoãn tích hợp thư viện Excalidraw (React/IFrame); khu vực bảng vẽ bố trí container sẵn sàng để tích hợp giải pháp bảng vẽ thuần **Vue 3 / Nuxt UI (Vue-native)** trong các giai đoạn phát triển tiếp theo.
  - Thanh trạng thái lớp học: Hiển thị thời gian ca dạy, số học viên có mặt và link phòng học ảo.

### 5.6 Bảng Điều Hành Ca Học Hôm Nay & Watchdog Cảnh Báo Đỏ (Live Shift Board & Watchdog Banner)
- **Vị trí:** Màn hình `/coordinator/today`.
- **Quy chuẩn:**
  - Bộ lọc cơ sở chi nhánh trên Topbar: Tabs chuyển nhanh giữa CS Cầu Giấy, CS Hà Đông, Online.
  - Thẻ ca học: Phân loại thẻ bằng viền màu: Xanh dương (`Đang diễn ra`), Xanh lá (`Đã kết thúc`), Xám (`Chưa bắt đầu`).
  - **Check-in Watchdog Banner:** Khi ca học đã bắt đầu 10 phút mà GV chưa check-in:
    - Banner nền đỏ đậm nhấp nháy (`bg-rose-950 border border-rose-500 text-rose-200`).
    - Hiển thị văn bản: `🚨 CẢNH BÁO: Ca học đã bắt đầu 12 phút nhưng GV Nguyễn Văn A chưa check-in! [Gọi ngay: 0912.xxx.xxx]`.
    - Nút `[ 📢 Phát Loa Thông Báo Khẩn ]` gửi tin khẩn đến toàn bộ học sinh trong 1 click.

### 5.7 Sổ Chăm Sóc Học Viên CRM (Student Retention & Care Log Drawer)
- **Vị trí:** Màn hình `/coordinator/students/{id}` hoặc Slideover Drawer mở từ Churn Radar.
- **Quy chuẩn:**
  - Huy hiệu mức độ nguy cơ: Đỏ rực (`HIGH RISK - Vắng 2 buổi liên tiếp`), Vàng (`MEDIUM RISK - Nợ 3 bài tập`).
  - Dòng thời gian chăm sóc (Timeline view): Hiển thị tuần tự các mốc liên hệ (Gọi điện, Nhắn tin Zalo, Trao đổi trực tiếp) kèm người liên hệ, nội dung trao đổi và ngày hẹn kế tiếp (`nextFollowUpAt`).
  - Nút hành động nhanh: `[ Xếp Lịch Học Bù ]`, `[ Đề Xuất Bảo Lưu / Chuyển Lớp ]`, `[ Giao TA Kèm 1-1 ]`.

### 5.8 Trình Ghi Âm Giọng Nói Nhận Xét (Voice Note Recorder Component)
- **Vị trí:** Giao diện chấm bài tập Monaco Editor của Giáo viên & Màn hình xem bài nộp của Học viên.
- **Quy chuẩn:**
  - Nút Micro màu đỏ khi đang ghi âm (`animate-pulse`), đồng hồ đếm giây giới hạn thời gian (tối đa 2 phút).
  - Trình phát sóng âm thanh (Audio Waveform Visualizer) hiển thị biên độ giọng nói, nút Play/Pause và thanh tiến độ mượt mà.
  - Hỗ trợ học viên nghe lại với các tốc độ `1.0x`, `1.25x`, `1.5x`.

### 5.9 Quy Chuẩn Các Trạng Thái Giao Diện (Error, Empty, Loading & Skeleton Standards)
- **Loading State (Trạng thái đang tải):**
  - *Thanh tiến trình toàn trang (Global Page Indicator):* Sử dụng `<NuxtLoadingIndicator color="#0284c7" :height="3" />` của Nuxt 4 gắn cố định tại mép trên cùng của trình duyệt, tự động kích hoạt khi chuyển route hoặc fetch dữ liệu ban đầu.
  - *Nút bấm đang xử lý (Button Loading):* Khi bấm submit form, nút chuyển sang trạng thái disabled (`opacity-60 cursor-not-allowed`), ẩn icon thường và thay bằng icon Spinner quay vòng (`animate-spin`), hiển thị text hành động dạng tiếp diễn (ví dụ: `[ Đang lưu dữ liệu... ]`, `[ Đang chấm bài... ]`).
- **Skeleton Loaders (Khung xương tải trước):**
  - Tuyệt đối không sử dụng icon xoay tròn toàn màn hình (full-page spinner) gây chớp giật mắt và mất cảm giác công thái học.
  - Sử dụng `<USkeleton>` của Nuxt UI v4 với hiệu ứng pulse mượt mà (`bg-slate-800 animate-pulse rounded`).
  - Thiết kế Skeleton bám sát 100% bố cục thực của component:
    - *Bảng dữ liệu (Data Table Skeleton):* 5 hàng placeholder với các ô chữ nhật chiều cao `h-5`, chiều rộng so le `w-1/4`, `w-1/3`, `w-1/6` và một ô nút tròn hành động ở cuối.
    - *Thẻ khóa học (Course Card Skeleton):* Khung ảnh 16:9 bo tròn `h-48 w-full`, 2 thanh tiêu đề `h-5 w-3/4` và `h-4 w-1/2`, thanh giá tiền và thanh nút mua.
    - *Cockpit Ca học:* Avatar tròn `h-10 w-10 rounded-full`, 2 dòng thông tin giảng viên và khung đếm thời gian.
- **Empty States (Trạng thái rỗng / Không có dữ liệu):**
  - Áp dụng khi danh sách học viên rỗng, chưa có lịch dạy hôm nay, hoặc kết quả tìm kiếm/lọc không có dữ liệu.
  - Cấu trúc 4 tầng bắt buộc:
    1. *Biểu tượng minh họa:* Icon nét vẽ đơn sắc kích thước lớn `w-16 h-16 text-slate-500` đặt trong vòng tròn nền mờ (`bg-slate-800/60 p-4 rounded-full`).
    2. *Tiêu đề ngắn gọn:* `text-base font-semibold text-slate-200` (ví dụ: "Chưa có bài tập nào cần nộp" hoặc "Không tìm thấy học viên phù hợp").
    3. *Mô tả chỉ dẫn:* `text-sm text-slate-400 max-w-sm text-center` giải thích lý do hoặc hướng dẫn người dùng bước tiếp theo.
    4. *Nút hành động chính (Primary CTA):* Kêu gọi hành động trực tiếp (ví dụ: `[ + Thêm Học Viên Ngay ]`, `[ Đặt Lại Bộ Lọc ]`, `[ Tạo Bài Tập Mới ]`).
- **Error Handling & Error States (Quy chuẩn xử lý lỗi):**
  - *Lỗi Toàn Trang (App Crash / 404 / 500):* Bắt qua trang chuẩn `app/error.vue` của Nuxt 4; hiển thị mã lỗi HTTP to rõ, lời giải thích thân thiện bằng tiếng Việt và nút bấm `[ 🔄 Thử Lại ]` hoặc `[ 🏠 Trở Về Trang Chủ ]`.
  - *Lỗi Tải Khối Component (Error Boundary):* Hiển thị Card cảnh báo nền đỏ nhạt (`bg-rose-950/40 border border-rose-600/40 text-rose-200 p-4 rounded-xl flex items-center justify-between`) kèm nút `[ Tải lại ]` cục bộ mà không bắt người dùng F5 toàn trang.
  - *Thông báo nhanh (Toast Notifications):* Tích hợp `useToast()` của Nuxt UI v4 ở góc trên bên phải; màu xanh cho thành công (`success`), màu đỏ cho thất bại (`danger`), màu cam cho cảnh báo (`warning`); tự động biến mất sau 4 giây.
  - *Lỗi Form Validation:* Bắt qua Zod Schema và hiển thị chữ đỏ nhỏ `text-xs text-rose-400 font-medium mt-1` ngay dưới từng ô input có lỗi.

### 5.10 Dải Chữ Chạy Thông Báo (Marquee Announcement Banner)
- **Vị trí:** Gắn cố định trên đầu trang (ngay trên Topbar) hoặc vị trí nổi bật tại Dashboard ca học khi có thông báo khẩn.
- **Quy chuẩn kỹ thuật:**
  - Sử dụng CSS animation chạy chữ mượt mà liên tục từ phải sang trái (infinite linear marquee) đảm bảo hiệu năng 60 FPS, không gây giật lag CPU.
  - **Tự động dừng khi rê chuột (Pause on Hover):** Khi người dùng di chuyển con trỏ chuột vào dải chữ, chuyển động sẽ tạm dừng để người dùng đọc thông tin hoặc nhấp vào liên kết đính kèm.
  - **Phân loại màu sắc nhận diện nghiệp vụ:**
    - *Khẩn cấp (Sự cố mất điện, rớt mạng, đổi phòng học, GV muộn):* Nền đỏ đậm `bg-rose-950 border-b border-rose-600 text-rose-100 font-medium`.
    - *Lịch đào tạo (Lịch thi, Khai giảng khóa mới, Hạn chót học phí):* Nền xanh dương `bg-sky-950 border-b border-sky-600 text-sky-100 font-medium`.
    - *Bảo trì kỹ thuật (Bảo trì máy chủ, Nâng cấp hệ thống):* Nền hổ phách `bg-amber-950 border-b border-amber-600 text-amber-100 font-medium`.
  - Nút đóng `[ ✕ ]` bên phải cho phép người dùng tắt thông báo; lưu trạng thái đã đóng vào `localStorage` theo `announcement_id` để không hiển thị lại gây phiền toái.

### 5.11 Trình Soạn Thảo Văn Bản Giàu Định Dạng (Tiptap Rich Text Editor Component)
- **Thư viện chuẩn:** Sử dụng **Tiptap Editor (`@tiptap/vue-3` & `@tiptap/starter-kit`)** chuẩn mực cho Nuxt 4 / Vue 3.
- **Phạm vi ứng dụng:**
  - Giáo viên soạn thảo đề bài tập, tài liệu hướng dẫn và giáo án buổi học.
  - Quản nhiệm lớp soạn thảo báo cáo buổi học (`coordinatorNote`) gửi phụ huynh.
  - Học viên & Trợ giảng trao đổi mã nguồn và giải thích trong Hộp thư Q&A.
  - Quản trị viên viết bài thông báo trung tâm hoặc bài viết cẩm nang học tập.
- **Thanh công cụ Toolbar công thái học (Floating / Sticky Menu):**
  - *Nhóm Tiêu Đề:* Dropdown chọn Heading (Văn bản thường, Tiêu đề 1 `H1`, Tiêu đề 2 `H2`, Tiêu đề 3 `H3`).
  - *Nhóm Định Dạng Ký Tự:* Nút In đậm (`Bold`), In nghiêng (`Italic`), Gạch chân (`Underline`), Gạch ngang (`Strike`), Khối mã inline (`` `code` ``), Đánh dấu tô sáng (`Highlight`).
  - *Nhóm Danh Sách:* Danh sách đầu dòng tròn (Bullet list), Danh sách số thứ tự (Ordered list), Danh sách công việc có ô tích chọn (Task List với checkbox `[ ]`).
  - *Nhóm Khối Nâng Cao:* Trích dẫn (`Blockquote`), Khối mã lập trình đa ngôn ngữ (`CodeBlock` với tô màu cú pháp syntax highlighting), Bảng dữ liệu (`Table` hỗ trợ thêm/xóa cột, hàng và gộp ô).
  - *Nhóm Đính Kèm Phương Tiện (Media):* Nút chèn liên kết URL, Nút tải ảnh trực tiếp (tự động upload lên MinIO/S3 và chèn link Markdown/HTML), Nút nhúng video YouTube.
- **Bảo mật:** Toàn bộ nội dung HTML do Tiptap sinh ra trước khi render hoặc lưu trữ đều bắt buộc chạy qua bộ lọc sạch mã độc `DOMPurify` để ngăn chặn triệt để lỗ hổng XSS (Cross-Site Scripting).

### 5.12 Thanh Phân Trang Toàn Năng (Advanced 3-Section Pagination Bar Component)
- **Mô tả:** Thành phần phân trang chuẩn hóa cho toàn bộ bảng biểu dữ liệu trong hệ thống (Quản lý học viên, Danh sách lớp học, Sổ điểm, Báo cáo tài chính, Hộp thư thắc mắc). Thiết kế mô phỏng chính xác 100% quy chuẩn giao diện điều hướng hiện đại.
- **Bố cục khung chứa (Card Container):**
  - Khung chữ nhật bo góc mềm mại `rounded-xl`, đường viền thanh lịch `border border-slate-700 bg-slate-900/60 dark:bg-slate-900/80 shadow-sm`.
  - Căn chỉnh Flexbox 3 cụm dàn đều theo chiều ngang (`flex items-center justify-between px-4 py-2.5 text-sm`).
- **Chi tiết 3 phân vùng chức năng:**
  1. **Cụm Bên Trái - Nhập trang trực tiếp (Quick Jump Page Input):**
     - Dòng chữ cố định: `Showing page`
     - Ô nhập số trang: `[ 1 ]` (Input nhỏ gọn `w-12 h-8 text-center rounded-lg border border-slate-600 bg-slate-800 font-semibold text-slate-100 focus:border-sky-500 focus:ring-1 focus:ring-sky-500 transition-colors`).
     - Người dùng có thể gõ trực tiếp bất kỳ số trang nào (ví dụ: gõ `8` và nhấn Enter) để nhảy ngay đến trang đó mà không cần bấm liên tiếp.
     - Dòng chữ tổng số trang: `of {totalPages}` (ví dụ: `of 10`).
  2. **Cụm Ở Giữa - Bộ nút điều hướng & Dải số trang (Navigation Controls & Number Pills):**
     - Nút `[ << ]`: Nhảy về trang đầu tiên (`page = 1`), bị disable và làm mờ khi đang ở trang 1.
     - Nút `[ < ]`: Lùi lại 1 trang (`page - 1`), bị disable khi đang ở trang 1.
     - Dải số trang tương tác:
       - *Trang đang chọn (Active Page):* Nút bấm nền xám bạc sáng `bg-slate-100 text-slate-900 dark:bg-slate-700 dark:text-sky-300 font-bold rounded-lg px-3 py-1 shadow-sm`.
       - *Trang lân cận:* Chữ số click được `text-slate-400 hover:text-slate-100 hover:bg-slate-800 rounded-lg px-3 py-1 transition-colors`.
       - *Dấu chấm rút gọn `...`:* Tự động xuất hiện khi tổng số trang vượt quá 7 trang để giữ kích thước thanh phân trang gọn gàng.
     - Nút `[ > ]`: Tiến lên 1 trang (`page + 1`), bị disable khi đang ở trang cuối cùng.
     - Nút `[ >> ]`: Nhảy tới trang cuối cùng (`page = totalPages`), bị disable khi đang ở trang cuối cùng.
  3. **Cụm Bên Phải - Chọn số lượng bản ghi mỗi trang (Rows Per Page Dropdown):**
     - Dòng chữ cố định: `Rows per page`
     - Menu thả xuống (Select Box): `[ 10 ⌄ ]` (Các tùy chọn: `10`, `20`, `50`, `100` dòng/trang).
     - Khi người dùng thay đổi số dòng hiển thị, hệ thống tự động thiết lập lại `page = 1` và gọi API tải lại dữ liệu với `perPage` mới.
- **Tương thích DTO Metadata Backend (`gmhafiz/go8`):**
  - Thành phần nhận props phản ánh 100% cấu trúc `Envelope` phân trang từ Go API:
    ```json
    {
      "meta": {
        "page": 1,
        "perPage": 10,
        "total": 95,
        "totalPages": 10
      }
    }
    ```

---

## 6. Nguyên Tắc Trải Nghiệm (Do's & Don'ts)

| Nên Làm (Do's) | Tuyệt Đối Tránh (Don'ts) |
| :--- | :--- |
| Hiển thị trạng thái tải (Skeleton loaders) khi đang fetch bài giảng, lịch dạy | Không để màn hình trắng hoặc giật lag khi đang tải video YouTube |
| Cho phép học viên lưu nháp (Auto-save) bài tập code trên Monaco Editor | Không làm mất code của học viên khi vô tình refresh trang |
| Xác nhận rõ ràng (Confirmation Dialog) trước khi nộp bài tập hoặc xóa khóa học | Không tự ý submit bài tập khi chưa có sự xác nhận của học viên |
| Thiết kế nút bấm và mục chọn điểm danh kích thước tối thiểu `44x44px` trên điện thoại | Không để các nút bấm quá nhỏ gây ấn nhầm khi điểm danh trên mobile |
| Tự động sao chép mã chuyển khoản học phí chỉ bằng 1 cú nhấp chuột | Không bắt phụ huynh phải gõ tay cú pháp chuyển khoản dài dòng |

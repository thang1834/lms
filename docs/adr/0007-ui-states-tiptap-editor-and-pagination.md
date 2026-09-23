# ADR-0007: Chuẩn Hóa Trạng Thái UI (Error, Empty, Skeleton, Loading, Marquee), Trình Soạn Thảo Tiptap và Thanh Phân Trang Toàn Năng Nuxt UI v4

## Status
**Accepted** (2026-09-23)

## Context
Khi phát triển hệ thống LMS với hơn 100+ endpoints và 4 cổng Portal (`/admin`, `/teacher`, `/student`, `/parent`), trải nghiệm người dùng đối mặt với các thách thức lớn nếu thiếu các quy chuẩn giao diện thống nhất:
1. **Trạng thái tải & Rỗng (Loading & Empty States):** Nếu chỉ dùng icon quay vòng (full-page spinner) hoặc để màn hình trắng, người dùng sẽ cảm giác ứng dụng chậm chạp và giật lag. Khi bảng dữ liệu rỗng (không có học viên, không có ca học), nếu không có hướng dẫn hành động tiếp theo, người dùng sẽ hoang mang.
2. **Xử lý lỗi (Error Handling):** Lỗi API 4xx/5xx cần được bóc tách và phân loại rõ ràng (lỗi toàn trang, lỗi khối component hay lỗi form) thay vì alert popup khó chịu.
3. **Thông báo khẩn cấp (Emergency Broadcast):** Các sự cố đổi phòng học, giảng viên muộn, mất điện cần một dải chữ chạy thông báo mượt mà (Marquee) trên đầu trang không gây cản trở thao tác.
4. **Soạn thảo nội dung phong phú (Rich Text Editing):** Giáo viên và quản nhiệm cần soạn giáo án, đề bài tập, ghi chú sư phạm có bảng biểu, code block và hình ảnh. Trình soạn thảo văn bản thuần textarea không đáp ứng được, còn nhúng các trình soạn thảo cũ (như TinyMCE/CKEditor) làm tăng gánh nặng bundle và khó tương thích với Nuxt 4 / Vue 3.
5. **Điều hướng phân trang dữ liệu lớn (Pagination Bar):** Bảng danh sách hàng nghìn học viên, lớp học cần một thanh phân trang chuyên nghiệp cho phép: xem số trang hiện tại, nhập số trang nhảy cóc trực tiếp, bộ nút tới/lui/đầu/cuối và dropdown chọn số lượng dòng mỗi trang (`10/20/50/100`).

## Decision
Chúng tôi quyết định chuẩn hóa toàn bộ các quy chuẩn trạng thái giao diện và các linh kiện cốt lõi trên nền tảng **Nuxt 4 + Nuxt UI v4**:

### 1. Chuẩn Hóa Trạng Thái UI (Loading, Skeleton, Empty, Error)
- **Global Loading:** Tích hợp `<NuxtLoadingIndicator color="#0284c7" :height="3" />` tại `app/app.vue`.
- **Button Loading:** Nút bấm submit form tự động chuyển sang `disabled`, thay icon bằng Spinner xoay vòng (`animate-spin`) và hiển thị nhãn tiếp diễn (ví dụ: `[ Đang lưu... ]`).
- **Skeleton Loaders:** Sử dụng `<USkeleton>` của Nuxt UI v4 với hiệu ứng `animate-pulse` tái tạo 100% hình dáng thực tế của bảng dữ liệu (5 dòng so le), thẻ khóa học (khung ảnh 16:9 + text lines) và Cockpit giảng viên.
- **Empty States Chuẩn 4 Tầng:**
  1. *Illustration Icon* kích thước lớn `w-16 h-16 text-slate-500` đặt trong vòng tròn mờ.
  2. *Tiêu đề ngắn gọn* (`text-slate-200 font-semibold`).
  3. *Mô tả chỉ dẫn* (`text-slate-400 text-sm`).
  4. *Nút hành động chính (Primary CTA)* (`[ + Thêm Học Viên Ngay ]`, `[ Đặt Lại Bộ Lọc ]`).
- **Xử Lý Lỗi 3 Cấp Độ:**
  - *Cấp 1 (Lỗi toàn trang / App Crash):* Bắt qua `app/error.vue` kèm nút `[ 🔄 Thử Lại ]` hoặc `[ 🏠 Về Trang Chủ ]`.
  - *Cấp 2 (Lỗi khối Component):* Error Boundary Card nền đỏ nhạt (`bg-rose-950/40 border border-rose-500/50`) kèm nút `[ Tải lại ]` cục bộ.
  - *Cấp 3 (Lỗi Form):* Chữ đỏ nhỏ `text-rose-400 text-xs` ngay dưới từng ô input vi phạm Zod Schema.
  - *Toast Notifications:* Sử dụng `useToast()` hiển thị góc trên bên phải, tự đóng sau 4 giây.

### 2. Dải Chữ Chạy Thông Báo (Marquee Announcement Banner)
- Component `AppMarquee.vue` sử dụng thuần CSS Animation liên tục từ phải sang trái đạt chuẩn 60 FPS.
- Hỗ trợ tính năng **Pause on Hover:** Dừng chạy khi người dùng rê chuột vào để đọc nội dung hoặc bấm link.
- Phân loại màu sắc: Đỏ đậm (`bg-rose-950` - Sự cố khẩn cấp), Xanh dương (`bg-sky-950` - Lịch thi/Khai giảng), Hổ phách (`bg-amber-950` - Bảo trì hệ thống).
- Nút đóng `[ ✕ ]` lưu trạng thái vào `localStorage` theo `announcement_id`.

### 3. Trình Soạn Thảo Văn Bản Giàu Định Dạng Tiptap Editor
- **Thư viện chuẩn:** Sử dụng **Tiptap Editor (`@tiptap/vue-3` & `@tiptap/starter-kit`)** thuần Vue 3 / Nuxt 4.
- **Phạm vi:** Soạn giáo án, đề bài Monaco, `coordinatorNote` gửi phụ huynh, Hộp thư Q&A, Thông báo trung tâm.
- **Tính năng Toolbar:**
  - Tiêu đề (`H1`, `H2`, `H3`, Paragraph).
  - Định dạng ký tự (`Bold`, `Italic`, `Underline`, `Strike`, `Highlight`, Inline Code).
  - Danh sách (`BulletList`, `OrderedList`, `TaskList` kèm checkbox).
  - Khối mã lập trình đa ngôn ngữ (`CodeBlockLowlight` với syntax highlighting).
  - Bảng dữ liệu (`Table` thêm/xóa dòng/cột, gộp ô).
  - Tải ảnh trực tiếp tự động upload lên MinIO/S3 và nhúng link Markdown/HTML.
- **An toàn bảo mật:** 100% nội dung HTML đều chạy qua bộ lọc sạch mã độc `DOMPurify` trước khi render để triệt tiêu nguy cơ tấn công XSS.

### 4. Thanh Phân Trang Toàn Năng (Advanced 3-Section Pagination Bar)
- Component `AppPagination.vue` chuẩn hóa thanh điều hướng phân trang trong container bo góc tròn `rounded-xl`, viền `border border-slate-700 bg-slate-900/60`, căn chỉnh Flexbox 3 phân vùng:
  1. **Cụm Trái (Quick Jump Input):** Dòng chữ `Showing page`, ô nhập `[ PageInput ]` (gõ số trang và bấm Enter nhảy trực tiếp), dòng chữ `of {totalPages}`.
  2. **Cụm Giữa (Navigation Buttons & Pills):**
     - Nút `[ << ]` (Về trang 1), `[ < ]` (Lùi 1 trang).
     - Dải số trang tương tác với pill trang hiện tại nổi bật, các trang lân cận và dấu `...` rút gọn khi $> 7$ trang.
     - Nút `[ > ]` (Tiến 1 trang), `[ >> ]` (Tới trang cuối cùng).
  3. **Cụm Phải (Rows Per Page Dropdown):** Dòng chữ `Rows per page` kèm dropdown chọn `[ 10 / 20 / 50 / 100 ⌄ ]`. Tự động reset về trang 1 khi đổi số dòng.
- **Tương thích Metadata Backend:** Khớp 100% với cấu trúc Envelope của Go backend:
  ```json
  "meta": {
    "page": 1,
    "perPage": 10,
    "total": 95,
    "totalPages": 10
  }
  ```

## Consequences

### Thuận lợi (Pros):
- **Trải nghiệm người dùng đồng nhất 100%:** Mọi bảng dữ liệu và form nhập liệu trên toàn hệ thống đều vận hành theo một chuẩn mực cao cấp.
- **Tốc độ làm việc tối ưu:** Phân trang cho phép nhảy thẳng đến trang cần tìm mà không phải bấm liên tục hàng chục lần.
- **Bảo mật & Ổn định:** Tiptap được cô lập và lọc XSS; Skeleton loaders loại bỏ triệt để hiện tượng giật cục Layout Shift (CLS).

### Hạn chế (Cons) & Biện pháp khắc phục:
- Kích thước của Tiptap Editor có thể tăng nhẹ nếu import toàn bộ extensions.
- Khắc phục bằng cách lazy load component Tiptap Editor qua `<ClientOnly>` chỉ tại các trang soạn thảo nội dung.

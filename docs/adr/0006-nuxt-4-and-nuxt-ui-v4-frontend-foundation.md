# ADR-0006: Nền Tảng Frontend Chuẩn Hóa Với Nuxt 4 và Nuxt UI v4

## Status
**Accepted** (2026-09-23)

## Context
Dự án LMS Center Platform vận hành với 4 cổng người dùng (`/admin`, `/teacher`, `/student`, `/parent`), đòi hỏi đồng thời:
1. **Khả năng SEO vượt trội & Hiệu năng tải trang cao (Core Web Vitals):** Dành cho Landing Page, trang giới thiệu khóa học, trang xác minh chứng chỉ công khai (`/verify/[code]`).
2. **Trải nghiệm ứng dụng thời gian thực SPA mượt mà:** Dành cho Khoang lái lớp học Live Class Cockpit, Trình soạn thảo Monaco Editor, Sổ chăm sóc học sinh CRM và Hộp thư Q&A.
3. **Bộ linh kiện UI hiện đại, chuẩn công thái học & Đổi Theme tốc độ cao:** Cần hệ thống component phong phú (Buttons, Modals, Drawers, DataTables, Badges, Tabs, Popovers) đồng bộ với Tailwind CSS v4, hỗ trợ Dark Mode mặc định, không sử dụng màu tím (theo `DESIGN.md`) và tuân thủ tiêu chuẩn tiếp cận **WCAG 2.1 AA**.

Nếu sử dụng Vue 3 thuần (Vite SPA), chúng ta sẽ mất hoàn toàn khả năng SEO và server-rendering cho các trang bán khóa học. Nếu tự xây dựng lại từ đầu toàn bộ UI components, chi phí phát triển và kiểm thử accessibility sẽ rất lớn.

## Decision
Chúng tôi quyết định chuẩn hóa toàn bộ tầng Frontend của dự án trên nền tảng **Nuxt 4** kết hợp với **Nuxt UI v4**:

1. **Framework Cốt Lõi: Nuxt 4 (Vue 3.5+, Vite 6, Nitro 3 Engine)**
   - Cấu trúc thư mục mới của Nuxt 4 với thư mục `app/` tập trung (`app/components`, `app/composables`, `app/pages`, `app/layouts`).
   - Chế độ **Hybrid Rendering (`routeRules`)**:
     - Pre-rendering (SSG/ISR) & SSR cho các trang công khai: `/`, `/courses/**`, `/verify/**`.
     - SPA Mode (`ssr: false`) cho toàn bộ các trang Dashboard nội bộ: `/admin/**`, `/teacher/**`, `/student/**`, `/parent/**` để đạt tốc độ tương tác cao nhất và không bị overhead SSR.

2. **Hệ Thống Thành Phần Giao Diện: Nuxt UI v4**
   - Xây dựng trên nền tảng **Reka UI (Radix Vue)** mang lại khả năng tiếp cận (A11y), phím tắt bàn phím và ARIA attributes hoàn hảo.
   - Tích hợp sâu với **Tailwind CSS v4** thông qua CSS-first configuration (`@theme` trong `assets/css/main.css`).
   - Kế thừa và thực thi triệt để bộ Design Tokens từ `DESIGN.md`:
     - Màu Primary: Ocean Blue (`#0284c7`).
     - Màu Success: Emerald (`#10b981`).
     - Nghiêm cấm sử dụng màu tím neon / purple làm màu thương hiệu chính (`Purple Ban`).
   - Quản lý Dark/Light mode tự nhiên thông qua `@nuxtjs/color-mode` tích hợp sẵn trong Nuxt UI v4.

3. **Cấu Trúc Tách Lớp Trách Nhiệm (Separation of Concerns):**
   - **UI Layer (`app/components/`):** Chỉ chịu trách nhiệm render template và nhận props/emits.
   - **Logic Layer (`app/composables/`):** Chứa các hàm composable tái sử dụng (`useCockpit`, `useStudentCare`, `useVietQR`).
   - **Data Layer (`app/services/` hoặc composable `$fetch`):** Xử lý API calls và cache keys.
   - **Type & Schema Layer (`app/types/` & `app/schemas/`):** Khai báo TypeScript types và Zod schemas đồng bộ với Backend DTOs.

## Consequences

### Thuận lợi (Pros):
- **Tốc độ phát triển thần tốc:** Sở hữu ngay hơn 50+ components cao cấp (Table, Modal, Slideover, Form, Dropdown, Command Palette) viết riêng cho Vue 3.
- **Tương thích hoàn hảo với Tailwind CSS v4:** Không cần file cấu hình `tailwind.config.js` rườm rà, token hóa trực tiếp trong CSS.
- **Tiêu chuẩn Accessibility cao:** 100% components của Nuxt UI v4 tuân thủ WAI-ARIA, thân thiện với screen reader.
- **Zero Conflict:** Hoàn toàn thuần Vue 3 / TypeScript, không bị xung đột phụ thuộc React như khi nhúng thư viện ngoại lai.

### Hạn chế (Cons) & Biện pháp khắc phục:
- Nuxt UI v4 là phiên bản hiện đại, một số cú pháp có thể thay đổi so với v2/v3 cũ.
- Khắc phục bằng cách tham chiếu chính thức tài liệu [Nuxt UI Docs](https://ui.nuxt.com) và tuân thủ chặt chẽ design tokens trong `DESIGN.md`.

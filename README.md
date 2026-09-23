# LMS Center Platform (Enterprise Multi-Disciplinary Learning Management System)

> **Hệ thống Quản lý Học tập, Giảng viên & Vận hành Đào tạo Đa hình thức (Offline, Online & Hybrid)**  
> Thiết kế chuẩn hóa cho đào tạo **Công nghệ Thông tin (CNTT)** và mở rộng linh hoạt cho **Ngoại ngữ, Thiết kế, Quản trị**.

[![Backend Blueprint](https://img.shields.io/badge/backend-gmhafiz%2Fgo8-00ADD8?style=flat-square&logo=go)](https://github.com/gmhafiz/go8)
[![Frontend](https://img.shields.io/badge/frontend-Nuxt_UI_v3-00DC82?style=flat-square&logo=nuxtdotjs)](https://nuxt.com)
[![Database](https://img.shields.io/badge/database-PostgreSQL_16-336791?style=flat-square&logo=postgresql)](https://www.postgresql.org)
[![Observability](https://img.shields.io/badge/observability-OpenTelemetry-F5A800?style=flat-square&logo=opentelemetry)](https://opentelemetry.io)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=flat-square)](LICENSE)

---

## ⚡ Quick Start (< 5 Phút)

### Yêu Cầu Môi Trường
- **Go** $\ge$ 1.23
- **Node.js** $\ge$ 20.x & **pnpm** / **npm**
- **Docker** & **Docker Compose**
- **Taskfile** (`go install github.com/go-task/task/v3/cmd/task@latest`)

### 1. Khởi Chạy Hạ Tầng Dịch Vụ (Infra + OpenTelemetry)
```bash
# Khởi chạy PostgreSQL, Redis, MinIO, Jaeger, Prometheus, Loki, Grafana
cd backend
task infra:up

# Kiểm tra trạng thái container:
docker ps
```
- **PostgreSQL 16:** `localhost:5432` (`lms_db` / `postgres` / `postgres`)
- **Jaeger UI (Tracing):** [http://localhost:16686](http://localhost:16686)
- **Grafana Dashboard (Metrics & Logs):** [http://localhost:3300](http://localhost:3300) (`admin` / `admin`)
- **MinIO Console (Object Storage):** [http://localhost:9001](http://localhost:9001) (`minioadmin` / `minioadmin`)

### 2. Chạy CSDL Migrations (Goose)
```bash
task migrate
# Hoặc: goose -dir database/migrations postgres "user=postgres password=postgres dbname=lms_db sslmode=disable" up
```

### 3. Khởi Chạy Backend API (Go-chi + Hot reload Air)
```bash
task dev
# Backend phục vụ tại: http://localhost:8080
# Tài liệu Swagger UI: http://localhost:8080/swagger/index.html
# Health check: http://localhost:8080/health
```

### 4. Khởi Chạy Frontend (Nuxt UI + Vue 3)
```bash
cd ../frontend
npm install
npm run dev
# Giao diện người dùng: http://localhost:3000
```

---

## 🌟 19 Trụ Cột Nghiệp Vụ Cốt Lõi (Core Pillars)

1. **Chuẩn Hóa Phân Quyền Hạt Nhân Dynamic RBAC:** 8 vai trò chuyên biệt (`SUPER_ADMIN`, `ACADEMIC_MANAGER`, `CLASS_COORDINATOR`, `TEACHER`, `TA`, `EXAMINER`, `STUDENT`, `PARENT`) quản lý qua bảng `roles`, `permissions`, `role_permissions`.
2. **Cơ Chế Xếp Lịch Học Linh Hoạt (Flexible Scheduling Engine):** Thiết lập lịch định kỳ 1 tuần 1 hoặc nhiều buổi; dời ngày học, đổi ca, đổi phòng học hoặc đổi link Meet/Zoom từng buổi.
3. **Khoang Lái Lớp Học Trực Tuyến & Bảng Vẽ Kỹ Thuật Số (Live Class Cockpit & Digital Whiteboard):** 1-Click mở Meet/Zoom, tự động check-in GV, bảng vẽ chuẩn Excalidraw thời gian thực, Mini Poll 2 phút.
4. **Trợ Lý AI Chấm Bài Monaco & Nhận Xét Giọng Nói (AI Code Review & Voice Note):** Trợ lý AI phân tích Clean Code, phát hiện lỗi; ghi âm nhận xét giọng nói 30s-2m trực tiếp trên web.
5. **Hộp Thư Q&A Tập Trung & Phân Công Trợ Giảng (Unified Q&A Inbox & TA Delegation):** Gom thắc mắc từ video bài giảng (kèm timestamp) và bài tập; kho câu trả lời mẫu (Snippets); SLA 2 giờ cho Trợ giảng.
6. **Chợ Dạy Thay, Lịch Rảnh & Sổ Tay Sư Phạm (Substitute Marketplace & Pedagogy):** Quản lý lịch rảnh tuần, khớp nối tự động ca dạy thay chuyển đổi thù lao, sổ tay ghi chú sư phạm bí mật giữa GV & TA.
7. **Bảng Điều Hành Ca Học Hôm Nay & Giám Sát Check-in Watchdog (`/coordinator/today`):** Quản lý ca học theo cơ sở, cờ đỏ cảnh báo khi GV trễ 10 phút, 1-Click Class Broadcast phát thông báo khẩn qua Zalo/Push/SMS.
8. **Hồ Sơ Chăm Sóc Học Viên & Can Thiệp Churn Radar (Student Retention & Care CRM):** Radar cảnh báo học viên vắng 2 buổi liên tiếp hoặc thiếu 3 BTVN, lưu nhật ký chăm sóc và đặt lịch follow-up.
9. **Quy Trình Chuyển Lớp & Bảo Lưu Khóa Học Phê Duyệt 2 Cấp (`class_transfers`):** Tính toán học phí bảo lưu tự động (`reservedCredit`), giải phóng ghế ngồi, Coordinator đề xuất $\to$ Academic Manager duyệt.
10. **Quản Lý Sự Cố Vận Hành & Khắc Phục Tức Thời (`session_incidents`):** Ghi nhận sự cố điện nước/phòng học/GV ốm, tự động cấp link Meet chuyển sang học Online trong 60 giây.
11. **Sổ Liên Lạc Điện Tử & Cấp Phát Học Liệu (`student_materials`):** Soạn báo cáo ca học (`coordinatorNote`) gửi phụ huynh, theo dõi bàn giao giáo trình, áo đồng phục, welcome kit.
12. **Điểm Danh Hai Chiều & Đánh Giá Giảng Viên:** Điểm danh học sinh lớp Hybrid (Offline/Online/Late/Absent); GV check-in/out tính công; Học viên vote sao đánh giá buổi học.
13. **Hệ Thống Tin Nhắn & Cảnh Báo Tự Động (Automated Notification Engine):** Quét vắng sau 15p gửi phụ huynh; Cảnh báo GV trễ sau 10p; Gửi nhận xét sau buổi học; Nhắc lịch trước 24h & 2h.
14. **Mua Trực Tiếp Khóa Học (Course Store & VietQR):** Khóa học Free 1-click học ngay; Khóa học trả phí sinh mã VietQR Napas247 động, kích hoạt tự động qua Webhook trong 3 giây.
15. **Hội Đồng Giám Khảo & Chấm Đồ Án Tốt Nghiệp (Capstone Project Defense):** Examiner duyệt GitHub, Live Demo, Slide thuyết trình và chấm điểm Rubric 4 tiêu chí.
16. **Trình Phát Video Chống Tua Cho Khóa Học Tự Học (Anti-Seeking Video Player):** Cấm tua vượt mốc đã xem, heartbeat 5s chống hack DevTools; mở khóa tua tự do khi hoàn thành 100%.
17. **Quản Lý Đa Cơ Sở & Chống Trùng Phòng Học (Multi-Campus & Room Conflict Guard):** Mạng lưới cơ sở chi nhánh, bộ lọc `OVERLAPS` ngăn chặn trùng phòng cùng khung giờ.
18. **Lên Lịch Học Bù & Dạy Bù (Make-up Class Engine):** Ghép lớp song song hoặc kèm 1-1 với TA; tự động đồng bộ chuyên cần khi hoàn thành ca học bù.
19. **Khảo Thí & Ngân Hàng Câu Hỏi Tự Động (Question Bank & Auto-Quiz):** Sinh đề ngẫu nhiên theo tỷ lệ Dễ/TB/Khó, xáo trộn câu hỏi và đáp án, chấm điểm tức thì và đồng bộ Sổ điểm.

---

## 🛠️ Cấu Hình Môi Trường (Configuration)

### Backend Environment Variables (`backend/.env`)

| Biến Môi Trường | Mô Tả | Mặc Định |
| :--- | :--- | :--- |
| `SERVER_PORT` | Cổng HTTP Server Backend | `8080` |
| `SERVER_ENV` | Môi trường triển khai (`development`, `staging`, `production`) | `development` |
| `DB_HOST` | Địa chỉ máy chủ PostgreSQL | `localhost` |
| `DB_PORT` | Cổng kết nối PostgreSQL | `5432` |
| `DB_NAME` | Tên cơ sở dữ liệu | `lms_db` |
| `DB_USER` | Tên đăng nhập cơ sở dữ liệu | `postgres` |
| `DB_PASSWORD` | Mật khẩu kết nối CSDL | `postgres` |
| `REDIS_ADDR` | Địa chỉ kết nối Redis Cache | `localhost:6379` |
| `JWT_SECRET` | Khóa bí mật ký mã xác thực JWT | `secret-key-change-in-prod` |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | Địa chỉ máy chủ thu thập OpenTelemetry Collector | `http://localhost:4318` |
| `MINIO_ENDPOINT` | Địa chỉ máy chủ lưu trữ file MinIO/S3 | `localhost:9000` |
| `MINIO_ACCESS_KEY` | Access key MinIO | `minioadmin` |
| `MINIO_SECRET_KEY` | Secret key MinIO | `minioadmin` |

### Frontend Environment Variables (`frontend/.env`)

| Biến Môi Trường | Mô Tả | Mặc Định |
| :--- | :--- | :--- |
| `NUXT_PUBLIC_API_BASE` | URL gốc của Backend REST API | `http://localhost:8080/api/v1` |
| `NUXT_PUBLIC_WS_BASE` | URL kết nối WebSocket Hub | `ws://localhost:8080/ws` |

---

## 📖 Hệ Thống Tài Liệu Kỹ Thuật (Architecture & Documentation)

Dự án duy trì hệ thống tài liệu kỹ thuật toàn diện, đồng bộ tại thư mục `docs/`:

- [**`DESIGN.md`**](DESIGN.md): Quy chuẩn Design Tokens, UI Theme (Sky/Ocean Blue & Emerald, cấm màu tím), Typography, Monaco Editor & Bảng vẽ.
- [**`docs/requirements.md`**](docs/requirements.md): Đặc tả yêu cầu phần mềm SRS v1.8.0 (31 phân hệ, 46 User Stories chi tiết Acceptance Criteria).
- [**`docs/architecture.md`**](docs/architecture.md): Kiến trúc Clean Go8 v2.4.0 (ERD 59 bảng CSDL, 19 Goose Migrations, RBAC Matrix, Excalidraw Reference).
- [**`docs/api-specification.md`**](docs/api-specification.md): Đặc tả chi tiết 105+ REST API Handlers, 100% Swagger annotations & Go DTO structs.
- [**`docs/implementation-plan.md`**](docs/implementation-plan.md): Kế hoạch triển khai kỹ thuật v2.3.0 (24 Go Domains, OTel Stack, 8 Pha Roadmap).
- [**`docs/user-flows.md`**](docs/user-flows.md): 20 Sơ đồ tuần tự tương tác (Mermaid) & 6 Wireframes màn hình chính.
- [**`docs/test-plan.md`**](docs/test-plan.md): Kế hoạch QA, Ma trận 11 nhóm Test Cases (45+ kịch bản kiểm thử P0/P1).
- [**`lms-center-platform.md`**](lms-center-platform.md): Tài liệu tổng thể 19 trụ cột nghiệp vụ & Khai báo mô hình thực thể Go Domain Models.
- [**`docs/adr/`**](docs/adr/): Hồ sơ các quyết định kiến trúc quan trọng (Architecture Decision Records).

---

## 🧪 Kiểm Thử & Kiểm Soát Chất Lượng (Quality Assurance)

```bash
# 1. Kiểm tra Backend Go (Syntax, vet, unit tests)
cd backend
go vet ./...
go test -v -cover ./...

# 2. Kiểm tra Frontend Nuxt UI (TypeScript typecheck & ESLint)
cd ../frontend
npm run typecheck
npm run lint

# 3. Quét bảo mật toàn diện bằng công cụ AG Kit
cd ..
python .agents/skills/vulnerability-scanner/scripts/security_scan.py .

# 4. Kiểm tra tính toàn vẹn cấu trúc dự án AG Kit
python .agents/scripts/validate_kit.py
```

---

## 📄 Bản Quyền & Giấy Phép (License)

Dự án được phân phối dưới giấy phép **MIT License**. Chi tiết xem tại tập tin `LICENSE`.

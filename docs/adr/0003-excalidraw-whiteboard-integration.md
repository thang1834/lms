# ADR-0003: Tích Hợp Bảng Vẽ Kỹ Thuật Số Excalidraw qua Standalone IFrame và Go WebSocket Hub

## Status
**Deferred (Tạm hoãn)** (2026-09-23)

> **Ghi chú cập nhật:** Tạm thời hoãn việc tích hợp Excalidraw (React/IFrame) để tìm kiếm và đánh giá giải pháp bảng vẽ thuần **Vue 3 / Nuxt UI (Vue-native)** phù hợp hơn với kiến trúc frontend của dự án (ví dụ: `vue-konva`, `fabric.js`, `perfect-freehand` với SVG canvas), tránh sự cồng kềnh của IFrame bridge và phụ thuộc vào hệ sinh thái React.

## Context
Trong các lớp học CNTT trực tuyến và Hybrid, giảng viên cần bảng vẽ kỹ thuật số trực quan để phác thảo sơ đồ kiến trúc hệ thống, luồng dữ liệu ERD và giải thích thuật toán trực tiếp cho học sinh. Dự án chọn tham khảo thư viện mã nguồn mở hàng đầu thế giới là [Excalidraw](https://github.com/excalidraw/excalidraw).  
Tuy nhiên, Excalidraw được viết bằng React trong khi Frontend của hệ thống LMS được xây dựng bằng **Nuxt UI (Vue 3 / Vite)**. Nếu cố gắng nhúng trực tiếp React component vào cây component Vue qua các wrapper như `veaury`, sẽ dẫn đến xung đột phiên bản React/Vue, làm phình to kích thước bundle chính (thêm >1.5MB), và dễ gây memory leak khi chuyển trang.

## Decision
Chúng tôi quyết định áp dụng giải pháp **Kiến trúc Cô Lập IFrame Container Độc Lập (Isolated Standalone Container Architecture)**:
1. **Container độc lập:** Đặt bản dựng Excalidraw độc lập trong thư mục `frontend/public/whiteboard/index.html`.
2. **Giao tiếp liên ngữ cảnh (Cross-context IPC):** Sử dụng `window.postMessage` với cơ chế xác thực nguồn (`origin verification`) nghiêm ngặt để trao đổi sự kiện giữa Nuxt UI và Excalidraw container (các lệnh `INIT_SCENE`, `SCENE_CHANGED`, `EXPORT_REQUEST`, `EXPORT_RESPONSE`).
3. **Lưu trữ chuẩn Excalidraw Scene JSON:** Dữ liệu bảng vẽ lưu trữ tại trường `class_whiteboards.board_data` tuân thủ 100% cấu trúc schema gốc của Excalidraw gồm 3 trường khóa:
   - `elements`: Mảng các phần tử vector (hình hộp, mũi tên, chữ, đường cong tự do).
   - `appState`: Trạng thái hiển thị (zoom, viewBackgroundColor, scrollX, scrollY).
   - `files`: Bảng ánh xạ nhị phân các hình ảnh nhúng trên bảng vẽ.
4. **Cơ chế Autosave Debounce 3s:** Khi có thao tác vẽ mới, client debounce 3 giây trước khi gọi API `PUT /api/v1/cockpit/whiteboards/{id}/autosave`.
5. **Cộng tác nhiều người (Multiplayer Collaboration):** Sử dụng Go WebSocket Hub (`internal/domain/cockpit/ws_hub.go`) để broadcast các delta thay đổi nét vẽ và tọa độ con trỏ chuột thời gian thực giữa giảng viên và các học viên cùng ca học.
6. **Tự động xuất PDF kết thúc ca học:** Khi buổi học hoàn tất, client sử dụng `@excalidraw/utils` (`exportToBlob`) kết xuất hình ảnh/PDF vector $\to$ upload lên MinIO/S3 $\to$ lưu đường dẫn tại `exportPdfUrl` để học sinh xem lại sau giờ học.

## Consequences

### Thuận lợi (Pros):
- **Zero Conflict:** Hoàn toàn không gây xung đột giữa React và Vue 3 / Nuxt UI.
- Bundle của ứng dụng Nuxt UI vẫn siêu nhẹ, tốc độ tải trang ban đầu không bị ảnh hưởng.
- Tận dụng 100% sức mạnh, giao diện tay vẽ tự nhiên (hand-drawn style) và tính năng xuất sắc của Excalidraw mà không phải viết lại canvas từ đầu.

### Hạn chế (Cons) & Biện pháp khắc phục:
- Cần quản lý giao tiếp hai chiều qua `postMessage` cẩn thận để tránh mất đồng bộ.
- Khắc phục bằng việc định nghĩa bộ Type và Message Protocol tường minh giữa Nuxt UI và IFrame.

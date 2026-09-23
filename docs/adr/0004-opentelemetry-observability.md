# ADR-0004: Chuẩn Hóa Ngăn Xếp Giám Sát Toàn Diện OpenTelemetry (Jaeger, Prometheus, Loki, Grafana)

## Status
**Accepted** (2026-09-22)

## Context
Khi số lượng học viên và lớp học trực tuyến tăng cao, việc giám sát hiệu năng hệ thống, phát hiện sớm các API phản hồi chậm (slow queries), nghẽn cổ chai I/O hoặc truy vết nguyên nhân gây lỗi 500 giữa hàng nghìn request là tối quan trọng. Nếu chỉ sử dụng log file thông thường phân tán trên máy chủ, việc điều tra lỗi sẽ mất hàng giờ đồng hồ và không đo lường được các chỉ số chất lượng dịch vụ (SLIs/SLOs).

## Decision
Chúng tôi quyết định triển khai ngăn xếp giám sát chuẩn công nghiệp **OpenTelemetry (OTel)** tích hợp chặt chẽ theo blueprint `gmhafiz/go8`:
1. **Traces (Truy vết phân tán):**
   - Sử dụng **Jaeger** nhận dữ liệu trace qua chuẩn OTLP gRPC (`:4317`) / HTTP (`:4318`).
   - Go-chi Middleware tự động khởi tạo trace span cho mỗi incoming request, đính kèm các thuộc tính: `http.method`, `http.route`, `http.status_code`, `user.id`, `tenant.id`.
   - W3C Trace Context propagation đảm bảo liên kết trace xuyên suốt giữa các microservices hoặc background workers.
2. **Metrics (Chỉ số thời gian thực):**
   - Sử dụng **Prometheus** định kỳ scrape endpoint `/metrics` của Backend Go.
   - Theo dõi các metric vàng: `http_requests_total`, `http_request_duration_seconds`, `database_connections_open`, `goroutines_count`, `active_websocket_connections`.
3. **Logs & Correlations (Nhật ký gắn vết):**
   - Zap Logger cấu hình structured JSON tự động tiêm `trace_id` và `span_id` vào mọi dòng log.
   - **Loki** thu gom log và liên kết trực tiếp với Traces trên Grafana.
4. **Dashboards (Giao diện hiển thị tập trung):**
   - **Grafana** (cổng `3300`) cung cấp dashboard điều hành trực quan hiển thị đồng thời Traces, Metrics và Logs trong cùng một giao diện.

## Consequences

### Thuận lợi (Pros):
- Quan sát toàn diện (Full Observability) 3 trụ cột Traces, Metrics, Logs theo chuẩn OpenTelemetry không bị vendor lock-in.
- Kỹ sư có thể nhấp vào một dòng log lỗi và xem ngay toàn bộ hành trình phân tán của request đó trên Jaeger chỉ trong 1 click.
- Tự động cảnh báo khi p99 latency vượt ngưỡng 500ms.

### Hạn chế (Cons) & Biện pháp khắc phục:
- Tiêu tốn thêm tài nguyên RAM/CPU cho OTel Collector và Prometheus.
- Khắc phục bằng cách cấu hình tỷ lệ lấy mẫu trace (Trace Sampling Rate) ở mức 10% trong môi trường production có lưu lượng truy cập lớn.

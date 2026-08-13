# campe

Tài liệu thiết kế cho hệ thống tìm kiếm và tracking hành vi người dùng của Campe.

## Mục tiêu tracking

- Không bỏ sót từ khóa người dùng tìm kiếm.
- Lưu cả từ khóa gốc và từ khóa đã chuẩn hóa.
- Biết truy vấn nào không có kết quả để bổ sung linh kiện vào database.
- Đếm số lần mỗi sản phẩm được hiển thị trong kết quả tìm kiếm.
- Đếm click vào từng sản phẩm.
- Đếm click sang Shopee/affiliate riêng biệt với click xem sản phẩm.
- Liên kết hành vi theo `visitor_id`, `session_id` và `user_id` khi có đăng nhập.
- Dữ liệu gốc phải thuộc database của Campe; công cụ analytics bên ngoài chỉ là lớp phân tích phụ.

## Tài liệu cho Codex

1. [`docs/01-architecture.md`](docs/01-architecture.md) — kiến trúc tổng thể và luồng dữ liệu.
2. [`docs/02-tracking-spec.md`](docs/02-tracking-spec.md) — đặc tả event tracking chi tiết.
3. [`docs/03-database-schema.md`](docs/03-database-schema.md) — schema PostgreSQL đề xuất.
4. [`docs/04-codex-implementation-plan.md`](docs/04-codex-implementation-plan.md) — thứ tự triển khai dành cho Codex.
5. [`docs/05-dashboard-metrics.md`](docs/05-dashboard-metrics.md) — các chỉ số và truy vấn cần có trong dashboard.

## Repo mẫu cần đối chiếu

- Umami browser tracker: https://github.com/umami-software/umami/blob/de474a1da915d4666768cad0af0bed5b3449d661/src/tracker/index.ts
- Umami server event route: https://github.com/umami-software/umami/blob/de474a1da915d4666768cad0af0bed5b3449d661/src/app/api/send/route.ts
- PostHog JS autocapture: https://github.com/PostHog/posthog-js/blob/48b1a0ce7f914a8dc15dc2c58fe6802954a917bb/packages/browser/src/autocapture.ts

Codex phải đọc các source trên để đối chiếu cách gửi custom event, event delegation, navigation-safe delivery, validate event và event capture pipeline trước khi triển khai.

## Nguyên tắc quan trọng

Các event kinh doanh quan trọng như `search`, `product_click` và `affiliate_click` phải được ghi nhận ở backend. Không dùng PostHog/JavaScript frontend làm nguồn dữ liệu chuẩn duy nhất.

# Kế hoạch triển khai cho Codex

Tài liệu này mô tả thứ tự làm feature tracking. Không nhảy cóc sang dashboard trước khi event pipeline ổn định.

## Phase 0 — Đọc codebase

Codex phải xác định trước:

- framework frontend;
- framework backend;
- ORM/database layer;
- auth/session hiện tại;
- model `Product`, listing/shop/platform;
- route tìm kiếm hiện tại;
- route mở trang sản phẩm hiện tại;
- cách tạo affiliate URL hiện tại nếu đã có.

Không tự tạo hệ auth/database thứ hai nếu repo đã có sẵn.

## Phase 1 — Visitor và session identity

### Mục tiêu

Mỗi browser có `visitor_id`, mỗi phiên có `session_id`.

### Yêu cầu

- Dùng first-party cookie.
- UUID ngẫu nhiên.
- Cookie phải có cấu hình bảo mật phù hợp môi trường production.
- Nếu user login, backend tự lấy `user_id` từ auth context.
- Không nhận `user_id` do frontend khai báo làm dữ liệu tin cậy.

### Acceptance test

1. Browser mới nhận visitor ID.
2. Refresh vẫn cùng visitor ID.
3. Session ID ổn định trong phiên.
4. Login không làm đổi visitor ID.

---

## Phase 2 — Search tracking

### Mục tiêu

Không bỏ sót bất kỳ search thực tế nào của user.

### Backend

Trong search handler/service:

1. nhận `query_raw`;
2. normalize thành `query_normalized`;
3. tạo `search_event_id`;
4. chạy search engine;
5. lưu `result_count`;
6. batch insert impression;
7. trả kết quả + `search_event_id`.

Response gợi ý:

```json
{
  "search_event_id": "uuid",
  "query": "buck 5v",
  "total": 37,
  "items": [
    {
      "product_id": 1827,
      "position": 1
    }
  ]
}
```

### Cần phân biệt

- `result_count = 0`: search thành công nhưng không có kết quả;
- search engine exception/timeout: lỗi hệ thống, không được giả thành zero result.

### Acceptance test

- Search `LM2596` tạo event.
- Search từ khóa tiếng Việt có dấu giữ nguyên raw.
- Search 0 result vẫn tạo event.
- 24 results tạo đúng 24 impression.
- Pagination position đúng.

---

## Phase 3 — Product click tracking

### Mục tiêu

Biết user click sản phẩm nào, từ đâu, từ search nào và vị trí bao nhiêu.

### Endpoint gợi ý

```text
POST /api/events/product-click
```

Body:

```json
{
  "event_id": "uuid",
  "product_id": 1827,
  "search_event_id": "uuid",
  "position": 3,
  "surface": "search_results"
}
```

Backend tự thêm:

- visitor_id;
- session_id;
- user_id;
- created_at;
- bot/internal flags nếu có.

### Frontend

Khi user click product card:

1. tạo `event_id`;
2. gửi event;
3. không làm UX chậm đáng kể;
4. có thể dùng `sendBeacon`/`fetch keepalive` nếu route chuyển trang ngay, nhưng backend vẫn là nơi lưu chuẩn.

### Acceptance test

- Một click tạo một record.
- Retry cùng event_id không tạo duplicate.
- Hai click thật khác nhau tạo hai record.
- Click từ homepage không bắt buộc có search_event_id.

---

## Phase 4 — Affiliate click tracking

### Mục tiêu

Mọi click rời Campe sang Shopee/platform affiliate đều đi qua backend redirect.

### Route gợi ý

```text
GET /r/:listing_id
```

### Backend flow

```text
validate listing
    ↓
resolve affiliate URL
    ↓
create affiliate_click_event
    ↓
return HTTP 302/303 redirect
```

### Bảo mật

- Không nhận `target_url` tùy ý từ query string.
- `listing_id` phải map tới URL đã biết trong database.
- Không trở thành open redirect.

### Acceptance test

- Click nút mua tạo event rồi mới redirect.
- URL invalid không redirect ra domain tùy ý.
- Refresh product detail không tăng affiliate click.
- Affiliate click tách biệt product click.

---

## Phase 5 — Bot/internal filtering

Ban đầu không cần hệ anti-fraud phức tạp.

Thêm các trường và rule cơ bản:

```text
is_bot
is_internal
risk_score
```

Ví dụ rule:

- known bot user-agent;
- admin/dev IP hoặc account;
- tốc độ request bất thường;
- cùng visitor spam affiliate redirect quá nhanh.

Không xóa dữ liệu ngay. Đánh dấu rồi loại khỏi dashboard mặc định.

---

## Phase 6 — Analytics queries

Tạo repository/service cho analytics, không nhét raw SQL rải rác trong controller.

Các query tối thiểu:

1. top searched keywords;
2. top zero-result keywords;
3. search volume theo ngày;
4. impressions theo product;
5. product clicks theo product;
6. affiliate clicks theo product/listing;
7. CTR search result;
8. affiliate CTR;
9. unique visitors;
10. keyword → clicked products.

---

## Phase 7 — Admin dashboard

Chỉ làm sau khi test event pipeline.

Dashboard MVP:

```text
Overview
Search Keywords
Zero Results
Products
Affiliate Clicks
```

### Overview

- searches today;
- unique visitors;
- zero-result searches;
- product clicks;
- affiliate clicks.

### Search Keywords

- raw/normalized query;
- search count;
- unique visitors;
- avg result count;
- product click-through rate.

### Products

- impressions;
- product clicks;
- CTR;
- affiliate clicks;
- affiliate CTR.

---

## Phase 8 — PostHog (optional)

Chỉ tích hợp sau khi database tracking của Campe hoạt động.

Có thể mirror các event:

```text
search
product_clicked
affiliate_clicked
```

PostHog không thay thế các bảng event chính.

---

# Thứ tự commit đề xuất

Codex nên chia commit nhỏ:

```text
feat: add visitor and session tracking
feat: record search events and impressions
feat: record product click events
feat: add affiliate redirect tracking
feat: add analytics queries
feat: add tracking admin dashboard
test: cover tracking event pipeline
```

# Checklist hoàn tất feature

- [ ] Raw keyword không bị mất.
- [ ] Zero-result keyword được lưu.
- [ ] Search impressions được lưu theo ranking position.
- [ ] Product click có event_id chống duplicate kỹ thuật.
- [ ] Affiliate click ghi ở redirect backend.
- [ ] Anonymous visitor được tracking.
- [ ] Logged-in user được link vào event.
- [ ] Internal/bot traffic có thể lọc.
- [ ] Có test cho event pipeline.
- [ ] Dashboard không tính bot/internal mặc định.
- [ ] Không có open redirect.
- [ ] Không dùng PostHog làm source of truth.

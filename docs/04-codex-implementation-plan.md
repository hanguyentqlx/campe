# Kế hoạch triển khai cho Codex

Tài liệu này mô tả thứ tự làm feature tracking. Không nhảy cóc sang dashboard trước khi event pipeline ổn định.

## Source tham khảo bắt buộc

Codex phải đọc các source dưới đây trước khi code:

1. Umami browser tracker:
   https://github.com/umami-software/umami/blob/de474a1da915d4666768cad0af0bed5b3449d661/src/tracker/index.ts

   Dùng để đối chiếu:
   - custom event `name` + `data`;
   - click event delegation;
   - xử lý anchor navigation sau khi event được gửi;
   - `fetch(..., { keepalive: true })`;
   - visitor identity và payload browser.

2. Umami event ingest route:
   https://github.com/umami-software/umami/blob/de474a1da915d4666768cad0af0bed5b3449d661/src/app/api/send/route.ts

   Dùng để đối chiếu:
   - validate payload ở backend;
   - tách event / identify;
   - tạo và cập nhật session;
   - lưu distinct ID;
   - bot detection;
   - server-side enrichment browser/device/location;
   - lưu event vào storage ở backend.

3. PostHog JS autocapture:
   https://github.com/PostHog/posthog-js/blob/48b1a0ce7f914a8dc15dc2c58fe6802954a917bb/packages/browser/src/autocapture.ts

   Dùng để đối chiếu:
   - listener `click`, `change`, `submit` theo capture phase;
   - lấy target và ancestor chain;
   - event properties;
   - external click URL;
   - tránh capture dữ liệu nhạy cảm;
   - cuối pipeline gọi `capture(eventName, props)`.

Các source này là reference pattern, không phải yêu cầu copy nguyên kiến trúc của Umami/PostHog.

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

### Dẫn chứng repo mẫu

Umami `src/tracker/index.ts` đưa identity vào payload qua trường `id`. Backend `src/app/api/send/route.ts` tạo `sessionId`, lưu `distinctId` và có luồng `identify` để link session với identity. Campe dùng ý tưởng tương tự nhưng giữ riêng `visitor_id`, `session_id`, `user_id`.

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

### Dẫn chứng repo mẫu

Umami ingest route validate payload bằng schema trước khi xử lý rồi mới phân loại và lưu event. Campe áp dụng cùng nguyên tắc: search event phải được validate và ghi ở backend, không phụ thuộc event JS analytics.

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
4. có thể dùng `sendBeacon` hoặc `fetch keepalive` nếu route chuyển trang ngay, nhưng backend vẫn là nơi lưu chuẩn.

### Dẫn chứng repo mẫu

- Umami `handleClicks()` dùng event delegation qua `document.addEventListener('click', ..., true)` và tìm phần tử gần nhất có tracking attribute.
- Khi click link nội bộ, Umami tạm ngăn navigation, gửi event, rồi mới chuyển trang.
- Umami `send()` dùng `fetch` với `keepalive: true` để tăng khả năng event được gửi khi trang chuẩn bị unload.
- PostHog `autocapture.ts` cũng gắn listener ở capture phase và cuối pipeline gọi `capture(eventName, props)`.

Campe không cần autocapture toàn DOM. Chỉ track product card, compare action và affiliate action có chủ đích.

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

### Dẫn chứng repo mẫu

PostHog autocapture có logic nhận biết external URL và đưa URL đó vào event properties. Umami tracker cũng xử lý riêng click vào anchor để đảm bảo event được gửi trước navigation. Campe dùng cùng tư tưởng nhưng với affiliate thì mạnh hơn: redirect phải đi qua backend để event được ghi trước khi rời site.

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

### Dẫn chứng repo mẫu

Umami backend dùng `isbot(userAgent)` trước khi ghi analytics event và có cả IP block check. Campe có thể học pattern này nhưng không nên xóa raw event ngay: nên đánh dấu `is_bot`/`is_internal` để vẫn audit được.

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

- [ ] Codex đã đọc 3 source repo mẫu ở đầu tài liệu.
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

# Đặc tả event tracking

Tài liệu này là hợp đồng dữ liệu để Codex triển khai tracking thống nhất giữa frontend, backend và database.

## 1. Event: search_event

Tạo đúng một record mỗi lần backend thực hiện một search thực tế.

```json
{
  "event": "search",
  "search_event_id": "uuid",
  "visitor_id": "uuid",
  "session_id": "uuid",
  "user_id": 102,
  "query_raw": "Mạch buck 5V",
  "query_normalized": "mach buck 5v",
  "result_count": 37,
  "page": 1,
  "page_size": 24,
  "created_at": "2026-08-12T20:00:00+07:00"
}
```

### Bắt buộc

- Không debounce ở backend theo kiểu làm mất search thực tế.
- Không bỏ qua search có `result_count = 0`.
- Không ghi search từ health check, crawler nội bộ hoặc job hệ thống vào cùng bảng user analytics.

## 2. Event: search_result_impression

Một record cho mỗi sản phẩm được backend trả về trong một search.

```json
{
  "search_event_id": "uuid",
  "product_id": 1827,
  "position": 3,
  "page": 1,
  "score": 12.883,
  "created_at": "2026-08-12T20:00:00+07:00"
}
```

Nếu search trả 24 sản phẩm thì ghi 24 impression.

`position` là vị trí tuyệt đối trong ranking. Ví dụ page 2, page_size 24, phần tử đầu tiên có position = 25.

## 3. Event: product_click

Ghi khi user click vào một card/result sản phẩm.

```json
{
  "visitor_id": "uuid",
  "session_id": "uuid",
  "user_id": 102,
  "search_event_id": "uuid",
  "product_id": 1827,
  "position": 3,
  "surface": "search_results",
  "created_at": "2026-08-12T20:00:08+07:00"
}
```

`search_event_id` có thể nullable nếu sản phẩm được mở từ homepage, recommendation hoặc link trực tiếp.

`surface` nên dùng enum/string có kiểm soát:

```text
search_results
homepage
category
comparison
recommendation
community
related_products
direct
```

## 4. Event: affiliate_click

Ghi tại backend redirect endpoint.

```json
{
  "visitor_id": "uuid",
  "session_id": "uuid",
  "user_id": 102,
  "product_id": 1827,
  "listing_id": 99182733,
  "seller_id": 556677,
  "platform": "shopee",
  "affiliate_sub_id": "u102",
  "source_surface": "product_detail",
  "search_event_id": "uuid",
  "created_at": "2026-08-12T20:00:20+07:00"
}
```

Không dùng frontend-only event làm số click affiliate chính thức.

## 5. Visitor và session

### visitor_id

- UUID random do Campe phát sinh.
- Lưu cookie first-party.
- Không dùng fingerprint bí mật để cố nhận diện người dùng đã xóa cookie.

### session_id

- UUID random.
- Có thể rotate sau khoảng thời gian inactivity phù hợp.
- Backend phải chấp nhận session mới nếu client hết phiên.

### user_id

- Nullable.
- Chỉ set khi user đăng nhập/xác thực.
- Không dùng email/số điện thoại trực tiếp trong analytics tables.

## 6. Deduplication

Không được loại bỏ click chỉ vì cùng user click lại. Click lặp có ý nghĩa hành vi.

Nhưng cần chống duplicate kỹ thuật do:

- double submit;
- retry request;
- browser gửi lại request;
- mobile double tap.

Khuyến nghị client tạo `event_id` UUID cho các event gửi qua POST và database đặt unique constraint trên `event_id`.

Ví dụ:

```json
{
  "event_id": "c1bd3e55-7d42-47eb-b96c-56a85c36b307",
  "product_id": 1827
}
```

Hai click thực sự khác nhau phải có hai `event_id` khác nhau.

## 7. Bot / internal traffic

Không xóa raw event ngay khi nghi bot. Nên đánh dấu:

```text
is_bot
is_internal
risk_score
```

Sau đó dashboard mặc định lọc traffic không hợp lệ, nhưng vẫn giữ dữ liệu để điều tra.

## 8. Time

Database lưu `timestamptz`/UTC hoặc timestamp có timezone chuẩn. UI chuyển sang timezone người xem.

Không dùng timestamp client làm thời gian chuẩn cho event kinh doanh; backend set `created_at`.

## 9. Quy tắc privacy tối thiểu

- Không lưu query cùng plaintext email/phone trong analytics tables.
- IP nếu cần chống abuse nên lưu ngắn hạn hoặc hash/pseudonymize theo chính sách hệ thống.
- Không thu thập dữ liệu không cần thiết cho mục tiêu search/affiliate analytics.
- Có cơ chế loại traffic của admin/dev khỏi số liệu sản phẩm.

## 10. Acceptance criteria

Codex chỉ coi feature tracking hoàn tất khi chứng minh được các case sau:

1. Search có kết quả tạo `search_event` và đúng số impression.
2. Search 0 kết quả vẫn tạo `search_event`.
3. Search giữ nguyên `query_raw`.
4. Click product tham chiếu được `search_event_id` và `position`.
5. Affiliate redirect tạo event trước khi redirect.
6. Refresh trang không tự tạo affiliate click.
7. Retry cùng `event_id` không tạo duplicate record.
8. User chưa login vẫn theo dõi bằng visitor/session.
9. Sau login event mới có `user_id` mà không làm mất visitor history.
10. Dashboard có thể tính impressions, product clicks và affiliate clicks độc lập.

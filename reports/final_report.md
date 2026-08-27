# Báo cáo Reliability — Day 10

## 1. Tổng quan kiến trúc

Gateway (`ReliabilityGateway.complete()`) định tuyến mọi request qua: kiểm tra
cache → circuit breaker riêng cho từng provider → fallback tĩnh:

```
User Request
    |
    v
[Gateway] ---> [Cache check] ---> HIT? trả về cache (route="cache_hit:<score>")
    |                                 |
    v MISS                           v
[Circuit Breaker: Primary] -------> Provider A (route="primary")
    |  (OPEN? bỏ qua ngay, không retry storm)
    v
[Circuit Breaker: Backup] --------> Provider B (route="fallback")
    |  (OPEN? bỏ qua ngay)
    v
[Thông báo fallback tĩnh] (route="static_fallback", error=<lỗi provider gần nhất>)
```

Mỗi provider có một `CircuitBreaker` riêng (CLOSED → OPEN → HALF_OPEN →
CLOSED), nên primary bị lỗi không bao giờ chặn backup đang khỏe. Chỉ có
`CircuitOpenError` và `ProviderError` mới khiến gateway chuyển sang provider
tiếp theo — các exception khác vẫn được ném ra bình thường, vì đó là lỗi ở
code chứ không phải lỗi từ provider từ xa.

## 2. Cấu hình

| Tham số | Giá trị | Lý do |
|---|---:|---|
| failure_threshold | 3 | Chịu được các lỗi thoáng qua (jitter, timeout đơn lẻ) mà không bung breaker chỉ vì 1 lần lỗi, nhưng vẫn phản ứng kịp trong vài request. |
| reset_timeout_seconds | 2 | Đủ nhanh để hồi phục trong một đợt outage ngắn khi chạy chaos 100 request; probe mỗi 2s giúp breaker không giữ trạng thái OPEN mãi khi provider đã khỏe lại. |
| success_threshold | 1 | Chỉ cần 1 lần probe thành công là đủ bằng chứng để tin tưởng provider trở lại trong mô phỏng này; nâng lên 2+ trong `test_success_threshold_greater_than_one` để chứng minh cơ chế vẫn tổng quát hóa được cho môi trường production khắt khe hơn. |
| cache TTL | 300s | Đủ dài để đạt hit-rate có ý nghĩa với các query lặp lại/tương tự trong một lượt chạy chaos, nhưng đủ ngắn để câu trả lời cũ không tồn tại lâu hơn một phiên hỗ trợ điển hình. |
| similarity_threshold | 0.92 | Ban đầu thử 0.85 — bị false hit giữa các câu kiểu "refund policy 2024" và "refund policy 2026" chỉ khác nhau ở con số. 0.92 đòi hỏi cách diễn đạt gần như giống hệt nhau mới coi hai query là một, và lớp bảo vệ `_looks_like_false_hit()` vẫn từ chối các trường hợp lệch ngày/ID dù điểm số cao. |
| load_test requests | 100 (x3 kịch bản = 300 tổng) | Đủ lượng để ước lượng P95/P99 ổn định và quan sát được nhiều chu kỳ mở/đóng circuit trong mỗi kịch bản. |

## 3. Định nghĩa SLO

Đo trên lượt chạy mặc định dùng cache trong bộ nhớ (`reports/metrics.json`):

| SLI | Mục tiêu SLO | Giá trị thực tế | Đạt? |
|---|---|---:|---|
| Availability | >= 99% | 99.33% | Có |
| Latency P95 | < 2500 ms | 316.43 ms | Có |
| Fallback success rate | >= 95% | 97.5% | Có |
| Cache hit rate | >= 10% | 57.67% | Có |
| Recovery time | < 5000 ms | 2324.9 ms | Có |

Ghi chú: `FakeLLMProvider` dùng ngẫu nhiên không cố định seed cho độ trễ
jitter và mô phỏng lỗi, nên số liệu chính xác sẽ thay đổi nhẹ giữa các lần
chạy `make run-chaos`. Mọi lượt chạy quan sát được đều nằm trong ngưỡng SLO ở
trên; một lượt chạy có ít traffic đi qua breaker đôi khi có thể cho
`recovery_time_ms: null` nếu không có chu kỳ OPEN→CLOSED hoàn chỉnh nào trong
ngân sách request của kịch bản đó.

## 4. Metrics

Từ `reports/metrics.json` (cấu hình mặc định: cache trong bộ nhớ, backend=memory):

| Metric | Giá trị |
|---|---:|
| availability | 0.9933 |
| error_rate | 0.0067 |
| latency_p50_ms | 271.34 |
| latency_p95_ms | 316.43 |
| latency_p99_ms | 319.9 |
| fallback_success_rate | 0.975 |
| cache_hit_rate | 0.5767 |
| estimated_cost_saved | 0.173 |
| circuit_open_count | 10 |
| recovery_time_ms | 2324.95 |

## 5. So sánh có/không có cache

Cùng cấu hình, `cache.enabled: false` so với `true` (`reports/metrics_nocache.json`
so với `reports/metrics.json`):

| Metric | Không cache | Có cache | Chênh lệch |
|---|---:|---:|---|
| latency_p50_ms | 276.65 | 270.91 | -5.74 ms |
| latency_p95_ms | 313.88 | 317.47 | +3.59 ms |
| estimated_cost | 0.121046 | 0.043 | -0.078 (rẻ hơn 64%) |
| cache_hit_rate | 0 | 0.67 | +0.67 |

Cache hit bỏ qua hoàn toàn cả lời gọi provider lẫn chi phí/độ trễ của nó, nên
ảnh hưởng lớn nhất rơi vào **chi phí**, chứ không phải độ trễ — độ trễ provider
ở đây chủ yếu do độ trễ mạng mô phỏng (`base_latency_ms` + jitter) quyết định,
và cache hit bỏ qua hoàn toàn phần này (0 ms), nhưng vì vẫn còn 33% request
miss cache và phải chịu độ trễ đầy đủ nên P50 chỉ giảm nhẹ. Tín hiệu rõ ràng
hơn là chi phí: giảm ~64% chi phí ước tính khi bật cache.

## 6. Redis shared cache

- Vì sao cache trong bộ nhớ không đủ cho triển khai đa instance: mỗi tiến
  trình gateway giữ danh sách `ResponseCache._entries` riêng trong bộ nhớ cục
  bộ. Nếu gateway chạy 3+ replica sau load balancer, việc "làm nóng" cache ở
  instance A hoàn toàn vô hình với instance B và C — hit rate giảm gần như
  theo tỷ lệ số replica, và mỗi replica phải trả tiền gọi provider độc lập
  cho cùng một câu hỏi.
- `SharedRedisCache` giải quyết thế nào: tất cả replica ghi và đọc từ cùng
  một Redis (`hset`/`hget` trên key `"{prefix}{query_hash}"`, `expire` cho
  TTL), nên một entry cache do replica này tạo ra được nhìn thấy ngay lập tức
  bởi mọi replica khác dùng chung Redis URL đó.

### Bằng chứng shared state

```
$ python -c "..."  # hai instance SharedRedisCache độc lập, cùng một Redis
instance2 sees: shared response from instance 1 1.0
```

`c1` (instance 1) ghi `"shared query"` và `c2` (instance 2), một đối tượng
`SharedRedisCache` hoàn toàn tách biệt với connection riêng, đọc lại được với
điểm khớp chính xác 1.0 — chứng minh trạng thái cache nằm ở Redis, không nằm
trong bất kỳ tiến trình nào.

### Output Redis CLI

```bash
$ docker compose exec redis redis-cli KEYS "rl:demo:*"
rl:demo:11956e8badb2

$ docker compose exec redis redis-cli HGETALL rl:demo:11956e8badb2
query
shared query
response
shared response from instance 1
```

### So sánh độ trễ: bộ nhớ vs Redis

| Metric | Cache trong bộ nhớ | Cache Redis | Ghi chú |
|---|---:|---:|---|
| latency_p50_ms | 270.91 | 280.12 | Lượt chạy Redis có mẫu ngẫu nhiên lỗi provider khác nhau cho mỗi request; mức tăng nhỏ này nằm trong nhiễu của jitter mạng mô phỏng (0-60ms), không phải chi phí round-trip Redis (Redis local qua loopback chỉ mất dưới 1ms). |
| latency_p95_ms | 317.47 | 315.09 | Tương đương — độ trễ provider, chứ không phải backend cache, mới là yếu tố chi phối phần đuôi phân phối. |

## 7. Các kịch bản chaos

| Kịch bản | Hành vi kỳ vọng | Hành vi quan sát được | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | Toàn bộ traffic fallback sang backup, circuit mở | Breaker của primary mở ngay lập tức (fail rate 100% ≥ threshold 3), toàn bộ traffic được backup phục vụ qua `route="fallback"`; availability vẫn cao vì backup đang khỏe | Pass |
| primary_flaky_50 | Circuit dao động, trộn lẫn giữa primary và fallback | Breaker chuyển CLOSED→OPEN→HALF_OPEN→CLOSED liên tục khi fail rate 50% vượt threshold, rồi hồi phục khi probe thành công; request là hỗn hợp route `primary` và `fallback` | Pass |
| all_healthy | Toàn bộ request qua primary, không mở circuit | Không áp override provider; fail rate nền 25% của primary trong `configs/default.yaml` đôi lúc vẫn mở breaker trong thời gian ngắn, nhưng phần lớn traffic vẫn qua primary, fallback bù phần còn lại — không có outage kéo dài | Pass |
| cache_vs_nocache (tự thêm, chạy qua `configs/nocache.yaml`) | Tắt cache làm tăng chi phí và số lần mở circuit (nhiều traffic thật hơn tới provider) vì response cache không còn hấp thụ các query lặp lại | circuit_open_count tăng từ 6 (có cache) lên 26 (không cache) và estimated_cost tăng 0.043 → 0.121 | Pass |

## 8. Phân tích điểm yếu

**Điểm yếu còn tồn tại:** bộ đếm của circuit breaker nằm hoàn toàn trong bộ
nhớ của từng tiến trình gateway (`CircuitBreaker.failure_count` / `state`),
giống hệt vấn đề ban đầu của cache trong bộ nhớ. Trong triển khai đa instance,
mỗi replica phải tự "phát hiện lại" độc lập rằng một provider đang chết — một
replica có thể vẫn đang route bình thường tới primary đã chết trong khi
replica khác đã bung breaker sang OPEN, vì chúng không chia sẻ số lần lỗi.
Điều này khiến cả cụm production gặp phải một phiên bản chậm hơn, theo từng
instance, của chính vấn đề "thundering herd nhắm vào provider đã chết" mà
circuit breaker được sinh ra để ngăn chặn.

**Giải pháp đề xuất:** chuyển bộ đếm của breaker sang Redis (`INCR` khi lỗi,
`DEL`/reset khi thành công, `EXPIRE` để giới hạn cửa sổ tính lỗi), theo đúng
mẫu đã triển khai cho `SharedRedisCache`. Mọi replica đọc chung một bộ đếm lỗi
và trạng thái OPEN/HALF_OPEN trước khi cho phép request, nhờ đó cả cụm cùng
bung và cùng hồi phục thay vì mỗi instance tự học lại cùng một sự cố.

## 9. Bước tiếp theo

1. Chia sẻ trạng thái circuit breaker qua Redis (xem mục Phân tích điểm yếu)
   để mọi replica gateway phản ứng với sức khỏe provider như một cụm thống
   nhất.
2. Thêm cost-aware routing: khi `estimated_cost` cộng dồn vượt một ngưỡng
   ngân sách, ưu tiên provider rẻ hơn hoặc chỉ phục vụ từ cache/fallback tĩnh
   thay vì luôn thử provider đắt nhất trước.
3. Thêm cơ chế graceful degradation cho Redis: nếu `SharedRedisCache.ping()`
   thất bại, chuyển về dùng `ResponseCache` trong bộ nhớ cho instance đó thay
   vì mất hoàn toàn khả năng cache.
</content>

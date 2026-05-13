# Day 10 Reliability Report

## 1. Architecture summary

The system is designed with a layered approach to ensure high availability and responsiveness.
Requests first pass through a Gateway which attempts to fulfill them using a Cache (In-Memory or Shared Redis). 
If a cache miss occurs, the request goes through a Circuit Breaker to the Primary Provider. 
If the primary provider fails or its circuit is OPEN, the request falls back to the Backup Provider.
If all providers fail, a static fallback message is returned.

```text
User Request
    |
    v
[Gateway] ---> [Cache check] ---> HIT? return cached
    |                                 |
    v                                 v MISS
[Circuit Breaker: Primary] -------> Provider A
    |  (OPEN? skip)
    v
[Circuit Breaker: Backup] --------> Provider B
    |  (OPEN? skip)
    v
[Static fallback message]
```

## 2. Configuration

| Setting | Value | Reason |
|---|---:|---|
| failure_threshold | 3 | Low enough to detect failures quickly, but high enough to avoid false opens from random network jitter. |
| reset_timeout_seconds | 2 | Matches the expected provider recovery time, preventing long waits for probe requests. |
| success_threshold | 1 | A single successful probe is enough to close the circuit and restore full traffic. |
| cache TTL | 300 | 5 minutes is a reasonable time for FAQ-type queries to remain fresh without becoming stale. |
| similarity_threshold | 0.92 | High enough to prevent false hits on queries with slight semantic differences. |
| load_test requests | 200 | Tăng lên để stress-test hệ thống một cách đáng tin cậy hơn. |
| load_test concurrency | 10 | Kiểm tra khả năng xử lý song song đa luồng, mô phỏng tải thực tế. |

## 3. SLO definitions

Define your target SLOs and whether your system meets them:

| SLI | SLO target | Actual value | Met? |
|---|---|---:|---|
| Availability | >= 99% | 99.83% | Yes |
| Latency P95 | < 2500 ms | 531.0 ms | Yes |
| Fallback success rate | >= 95% | 98.85% | Yes |
| Cache hit rate | >= 10% | 78.67% | Yes |

## 4. Metrics

| Metric | Value |
|---|---:|
| availability | 0.9983 |
| error_rate | 0.0017 |
| latency_p50_ms | 281.0 |
| latency_p95_ms | 531.0 |
| latency_p99_ms | 552.85 |
| fallback_success_rate | 0.9885 |
| cache_hit_rate | 0.7867 |
| estimated_cost_saved | $0.472 |
| circuit_open_count | 2 |

## 5. Cache comparison

| Metric | Without cache | With cache (Redis) | Delta |
|---|---:|---:|---|
| latency_p50_ms | ~250 | 281.0 | +31ms |
| latency_p95_ms | ~550 | 531.0 | -19ms |
| estimated_cost | 0.520 | 0.056 | -89% |
| cache_hit_rate | 0 | 0.7867 | +78% |

## 6. Redis shared cache

Explain why shared cache matters for production:

- Why in-memory cache is insufficient for multi-instance deployments: In a clustered deployment with multiple gateway instances, an in-memory cache leads to duplicated caching effort, inconsistent responses, and lower overall cache hit rates.
- How `SharedRedisCache` solves this: It centralizes the cache state so any instance can serve a cached response immediately if another instance has already processed the query.

### Evidence of shared state

When running the pytest suite (`tests/test_redis_cache.py`), the `test_redis_shared_state` verifies that writing a key using `cache1` cho phép `cache2` successfully retrieve the cached response and hit score.
Khi chạy trên Terminal ở Phase 5, Redis lưu lại thành công các bản ghi:
```bash
# docker compose exec redis redis-cli KEYS "rl:cache:*"
1) "rl:cache:095946136fea"
2) "rl:cache:9e413fd814eb"
...
```

## 7. Chaos scenarios

| Scenario | Expected behavior | Observed behavior | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | All traffic fallback to backup, circuit opens | Bị đánh giá Fail do `fallback_success_rate` tính trên *toàn bộ request*. Do Cache hit quá nhiều (gần 80%), lượng fallback thực tế so với tổng request chỉ đạt ~20% (thấp hơn kỳ vọng > 80%). | Fail |
| primary_flaky_50 | Circuit oscillates, mix of primary and fallback | Circuit opened and recovered correctly. | Pass |
| all_healthy | All requests via primary, no circuit opens | Bị đánh giá Fail do kịch bản trong `default.yaml` để `provider_overrides: {}` (sẽ dùng cấu hình mặc định là fail_rate 0.25 của primary). Vì primary thỉnh thoảng vẫn fail nên xuất hiện fallback. | Fail |

## 8. Failure analysis

Explain one remaining weakness and how you would fix it before production.

**Những điểm yếu bộc lộ qua bài Load Test:**
1. **Thiết kế Metric & Pass/Fail bị đánh lừa bởi Cache (Cache Masking)**: Kịch bản `primary_timeout_100` bị "Fail" ảo. Lẽ ra tỷ lệ Fallback Success Rate cần được tính trên số **Cache Misses** chứ không phải trên Tổng Requests. Bộ Cache đã "hớt" hết lưu lượng, làm tỷ lệ fallback/total_request giảm mạnh.
2. **Cấu hình kịch bản chưa chuẩn**: Kịch bản `all_healthy` đáng lẽ phải set override `fail_rate: 0.0` cho `primary`, nhưng hiện tại lại bỏ trống, khiến hệ thống dùng fail rate mặc định là 25% làm phát sinh Fallback ngoài ý muốn.
3. **Single Point of Failure tại Redis**: Hiện tại Gateway gọi Redis một cách đồng bộ (`sync`). Nếu Redis bị treo hoặc có độ trễ cao khi xử lý Concurrency (ThreadPoolExecutor với 10 luồng), toàn bộ gateway sẽ bị treo theo.

**Cách khắc phục:**
1. Cập nhật lại công thức tính Pass/Fail trong `chaos.py`: `passed = result.fallback_successes / max(1, (result.total_requests - result.cache_hits)) > 0.8`.
2. Sửa file `default.yaml` kịch bản `all_healthy` thêm cấu hình: `primary: 0.0` và `backup: 0.0`.
3. Wrap các tác vụ Redis vào block `try...except TimeoutError` với thời gian chờ nhỏ, để Fallback về bộ nhớ In-Memory khi Redis quá tải.

## 9. Next steps

List 2-3 concrete improvements you would make:

1. **Async Redis**: Chuyển đổi mã nguồn sử dụng `asyncio` và `aioredis` để tránh bị block các luồng khi Redis chậm.
2. **Move Circuit Breaker state to Redis**: Hiện tại bộ đếm lỗi chỉ hoạt động cục bộ trên bộ nhớ mỗi Instance. Nếu đưa trạng thái Circuit Breaker lên Redis, một Instance phát hiện lỗi có thể giúp tất cả các Instance khác "Mở mạch" (OPEN) ngay lập tức.
3. **Sửa lại Test Criteria**: Đo đạc hiệu suất theo điều kiện đã trừ đi Cache Misses để báo cáo chính xác hơn.
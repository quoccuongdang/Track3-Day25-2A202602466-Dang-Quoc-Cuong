# Lab 25B — Report

**Họ tên / nhóm:** Đặng Quốc Cường (2A202602466)  **Commit:** `01256ae`  **Điểm tự động (`make grade`):** 88 / 88 (84/88 public + 4/4 hidden) 

> Mọi nhận định phải có số. Dán số từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`.

---

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | yours | 1.0000 | 4.776 | 1.062 | 0.002 | 2.101 | 0.0 | PASS |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | yours | 1.0000 | 11.217 | 1.044 | 0.005 | 2.129 | 0.0 | PASS |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL |
| G3 | yours | 1.0000 | 4.718 | 1.065 | 0.035 | 2.146 | 0.0 | PASS |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL |
| G4 | yours | 1.0000 | 5.684 | 1.013 | 0.002 | 2.069 | 0.0 | PASS |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | N/A | FAIL |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | N/A | PASS |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0000 | 5.958 | 1.053 | 0.019 | 2.203 | 0.0 | PASS |

---

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| `attempt_timeout_s` | 8.0 | Dựa trên băng **G6**: Nếu timeout quá lớn (như 30s của naive), khi `alpha-eu` bị treo hoàn toàn, 1 request ngốn hết deadline 30s khiến 107 lượt bị fail vì deadline (availability tụt còn 94.88%). Đặt 8.0s giúp client kịp phát hiện treo và failover sang `beta` mà vẫn đủ thời gian cho các call hợp lệ (median service của các provider là 0.6s–1.8s). |
| `max_attempts_per_provider` | 2 | Dựa trên băng **G2, G3**: Cho phép tối đa 1 lần retry trên cùng provider (tổng 2 lần thử) để vượt qua lỗi thoáng qua (transient error). Giới hạn này chặn đứng bão retry dây chuyền (amplification ở G2 giảm từ 9.904 của naive xuống 1.044). |
| `backoff_base_s` / `cap_s` | 0.25 / 4.0 | Dựa trên băng **G1, G2**: Full jitter với base 0.25s và cap 4.0s giúp phân tán đều các request retry, tránh hiện tượng thundering herd tái quá tải lên provider đang nghẽn. |
| `retry_budget_ratio` | 0.1 | Dựa trên băng **G3**: Quy định số lần retry không vượt quá 10% tổng request. Giữ `retry_ratio` của G3 ở mức 0.035 (thấp hơn nhiều so với trần 0.10, trong khi naive là 0.164). |
| `breaker.window_size` / `min_calls` | 20 / 10 | Dựa trên băng **G2, G4**: Thu thập đủ 10–20 mẫu để tính xác suất suy giảm một cách tin cậy, tránh việc breaker bị mở nhầm vào ban đêm hoặc khi lưu lượng thấp do 1–2 lỗi ngẫu nhiên. |
| `breaker.failure_rate_threshold` | 0.5 | Dựa trên băng **G2**: Khi tỷ lệ lỗi hạ tầng (529, 500, timeout) chạm 50%, ngắt ngay mạch provider để chuyển tải sang `alpha-eu` và `beta`, bảo vệ server chính khỏi sập sâu. |
| `breaker.slow_call_s` | 5.0 | Dựa trên băng **G4**: Median latency của alpha là 1.2s. Khi mạng chậm thêm 5s, latency vọt lên ~6.2s. Ngưỡng 5.0s giúp phát hiện mạng chậm trước khi xảy ra timeout. |
| `breaker.slow_rate_threshold` | 0.6 | Dựa trên băng **G4**: Khi >60% request bị chậm $\ge 5.0\text{s}$, breaker mở (`slow_call_rate`), ép traffic chuyển sang `alpha-eu`, giảm p95 từ 13.764s (naive) xuống còn 5.684s. |
| `breaker.open_duration_s` | 10.0 | Dựa trên băng **G2**: Cho provider 10 giây nghỉ ngơi để xả hàng đợi GPU trước khi cho phép probe thăm dò lại. |
| `breaker.half_open_max_probes` | 2 | Dựa trên băng **G2**: Chỉ cho phép đúng 2 request thăm dò cùng lúc ở trạng thái HALF-OPEN. Các request khác tiếp tục fail-fast, ngăn chặn bão request ập vào khi provider vừa gượng dậy. |

### Sơ đồ quyết định của `on_error`

```
                          [ Nhận ProviderError ]
                                     |
                          +----------v----------+
                          |   classify(error)   |
                          +----------+----------+
                                     |
         +-----------------+---------+---------+-----------------+
         |                 |                   |                 |
  [CLIENT_ERROR]   [QUOTA_EXHAUSTED]    [RATE_LIMITED]     [OVERLOADED /
  (400, 422...)       (429 quota)        (429 limit)       SERVER_ERROR /
         |                 |                   |             TIMEOUT]
   Release Permit    Release Permit      Release Permit          |
         |                 |                   |          Ghi nhận lỗi vào
      GiveUp       Cooldown 300s        Cooldown          Breaker (on_failure)
  ("client_error")         |            now+retry_after          |
  *Không retry*            +--------+----------+          Breaker còn đóng?
  *Không failover*                  |                     tries < max_tries?
                               Failover ngay              Còn kịp deadline?
                             (delay = 0.0s sang           Budget còn token?
                             provider kế tiếp)                   |
                                                        +--------+--------+
                                                        |                 |
                                                      (YES)              (NO)
                                                        |                 |
                                                  Retry cùng        Failover ngay
                                                  provider với       sang provider
                                                  Full Jitter       kế tiếp (delay 0s)
```

---

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối:**
   * Khi giả lập breaker bị hỏng (`min_calls: 1000000000`, `max_attempts_per_provider: 5`), nếu **bật retry budget**, trên băng G2 `amplification` chỉ là **1.378**, `retry_ratio` là **0.058**, `recovery` đạt **0.0s**, `availability` đạt **99.83%**; trên G3 `retry_ratio` là **0.068**.
   * Khi **tắt retry budget**, trên G2 `amplification` tăng vọt lên **1.675**, `retry_ratio` lên **0.495**, thời gian hồi phục bị trễ mất **25.0s** và availability tụt xuống **96.66%**; trên G3 `retry_ratio` nhảy lên **0.159** (vi phạm tiêu chuẩn $\le 0.10$).
   * *Ý nghĩa:* Kể cả khi breaker hoạt động chuẩn xác, retry budget vẫn là chốt chặn an toàn thiết yếu để giới hạn tổng lượng tài nguyên retry của toàn hệ thống ở mức an toàn ($\le 10\%$), ngăn ngừa bão tải lan truyền trên diện rộng khi có sự cố bất ngờ.

2. **`attempt_timeout_s: 30`:**
   * Khi tăng timeout lên 30.0s, trên băng **G6**, `availability` tụt từ **100% xuống 94.88%** (thất bại so với yêu cầu $\ge 95\%$), với **107 lượt người dùng bị bỏ rơi vì hết deadline** (`give_up_reasons: {'deadline': 107}`). Trên băng G4, p95 giữ ở mức 5.608s do breaker phát hiện slow call và ngắt mạch kịp thời.
   * *Kết luận:* Timeout "đúng" không được dựa trên hạn chót của người dùng (30s) mà phải dựa trên **SLO độ trễ và phân phối thời gian xử lý bình thường của dịch vụ** (ở đây là $3\times$ đến $4\times$ median latency $\approx 5\text{s} - 8\text{s}$), cộng thêm biên an toàn để kịp failover sang nhà cung cấp dự phòng trước khi người dùng rời đi.

3. **Bỏ lọc tool calling khỏi `ladder()`:**
   * Khi bỏ lọc khả năng tool calling trên băng **G6**, hệ thống phát sinh **244 vi phạm `incompatible_routes`**, khiến `availability` sụt giảm nghiêm trọng từ **100% xuống 89.77%**, trong đó `gamma-mini` nhận tới 451 lượt gọi vô ích.
   * *Góc nhìn Product Manager:* Điều này tương đương với việc trong lúc nhà cung cấp chính gặp sự cố, hệ thống đã chuyển hướng 244 khách hàng đang cần hoàn tiền sang một mô hình không có khả năng kích hoạt nghiệp vụ hoàn tiền, khiến khách hàng nhận được câu trả lời lỗi/từ chối ngớ ngẩn, làm sụt giảm niềm tin vào chất lượng dịch vụ của ShopAgent.

---

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0000 | 81 | 128 | 81 | 0 | 1,574,200 cents | 1.123 |
| yours | 1.0000 | 66 | 0 | 0 | 0 | 0 cents | 0.303 |

**Trả lời:**
* **Vì sao chỉ idempotency key là chưa đủ?** Nếu chỉ có idempotency key mà không có checkpoint, khi worker bị crash sau bước thanh toán, worker mới khởi động lại sẽ phải chạy lại từ đầu toàn bộ các bước LLM tốn kém (`classify_request`), làm lãng phí chi phí tính toán AI (`wasted_llm_per_crash > 1.0`) và gia tăng độ trễ cho khách hàng.
* **Vì sao chỉ checkpoint cũng chưa đủ?** Nếu chỉ có checkpoint mà không có idempotency key, nếu worker bị crash hoặc mạng bị timeout *ngay sau khi cổng thanh toán đã trừ/hoàn tiền nhưng trước khi kịp ghi nhận kết quả*, worker mới chạy lại bước thanh toán mà không có khóa chống trùng lặp sẽ ra lệnh hoàn tiền lần thứ 2, dẫn đến thất thoát tài chính nghiêm trọng.
* **Điều gì xảy ra nếu ghi checkpoint TRƯỚC khi gọi payment?** Nếu ghi checkpoint hoàn tiền thành công trước khi thực sự gọi API thanh toán, khi worker bị crash ngay tại thời điểm đó, worker mới hồi phục sẽ thấy checkpoint đã tồn tại và bỏ qua bước hoàn tiền. Kết quả là khách hàng không bao giờ nhận được tiền hoàn (`lost_refunds > 0`), vi phạm nghiêm trọng cam kết dịch vụ.

---

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | page | page | 27 phút |
| provider_outage | page | page | page | 4 phút |
| deploy_blip | none | ❌ page | **none** | Không báo |
| slow_leak | ticket | ❌ không thấy | **ticket** | 1,337 phút |
| degraded_evening | page | page | page | 291 phút |
| night_spike | none | ❌ page | **none** | Không báo |
| small_bump | none | ❌ page | **none** | Không báo |

* **Error budget 30 ngày:** Đã tiêu **42.95%**, còn dư **57.05%**.
* **Đóng băng release:** **Không đóng băng** (`freeze_releases: False`) vì ngân sách lỗi vẫn còn hơn 57%.
* **Vì sao `small_bump` (20 phút ở 12% lỗi) KHÔNG nên đánh thức ai lúc 11 giờ trưa?** Vì sự cố chỉ kéo dài 20 phút và tiêu hao chưa đầy 0.3% error budget của cả tháng. Nó không đe dọa đến mục tiêu SLO 99.5%, và tự động kết thúc trước khi kịp gây thiệt hại đáng kể. Đánh thức kỹ sư trực lúc này chỉ gây mệt mỏi cảnh báo (alert fatigue) không cần thiết.

---

## 6. Postmortem — Băng G2 với client naive (blameless)

*Dùng `make plot` (hình `reports/gameday/G2_overload_storm.png`) và CSV `reports/gameday/naive_G2_overload_storm_timeseries.csv`.*

**Tóm tắt (3 câu):**
Vào giây 120 của kịch bản mô phỏng, nhà cung cấp alpha bị sụt giảm 75% năng lực xử lý phần cứng trong vòng 180 giây. Do phiên bản client naive thực hiện retry 3 tầng cố định, không có circuit breaker và không quan sát header `retry-after`, các request dồn ứ đã tạo thành một cơn bão retry khuếch đại tải lên gấp gần 10 lần. Hệ thống rơi vào trạng thái hỏng hóc tự duy trì (metastable failure), khiến dịch vụ tiếp tục tê liệt thêm 155 giây sau khi hạ tầng phần cứng đã hồi phục hoàn toàn.

**Ảnh hưởng:**
* **1,285 lượt người dùng thất bại** (availability sụt giảm nghiêm trọng xuống còn **45.60%**, trong khi tiêu chuẩn là $\ge 95\%$).
* Sự cố kéo dài tổng cộng **335 giây** (từ $t=120$ đến $t=455$).
* Tiêu tốn **108.7% error budget của cả tháng** chỉ trong duy nhất một sự cố kéo dài hơn 5 phút này.

**Timeline (giây ảo — lấy từ CSV `naive_G2_overload_storm_timeseries.csv`):**

| t | Sự kiện | Bằng chứng (số) |
|---:|---|---|
| 119 | Hệ thống hoạt động bình thường | `calls_ok = 7`, `calls_done = 7`, `attempts_alpha = 9` attempts/s |
| 120 | Alpha mất 75% GPU (bắt đầu sự cố phần cứng) | `calls_ok = 7`, hàng đợi server bắt đầu ứ đọng |
| 150 | Bão retry bùng nổ, hệ thống mất tính sẵn sàng | `calls_ok = 0`, `calls_done = 3`, `attempts_alpha = 163` attempts/s (tải tăng gấp 18 lần) |
| 180 | Quá tải trầm trọng, hàng loạt client timeout | `calls_ok = 0`, `calls_done = 5`, `attempts_alpha = 138` attempts/s, server trả lỗi 529 hàng loạt |
| 300 | Alpha hồi phục 100% GPU về mặt hạ tầng | `calls_ok = 0`, `calls_done = 5`, `attempts_alpha = 115` attempts/s (server vẫn quá tải hoàn toàn) |
| 360 | Hạ tầng đã tốt 60s nhưng dịch vụ vẫn sập | `calls_ok = 0`, `calls_done = 1`, `attempts_alpha = 50` attempts/s (metastable state kéo dài) |
| 455 | Hàng đợi retry bắt đầu được giải tỏa | `calls_ok = 4`, `calls_done = 5`, `attempts_alpha = 13` attempts/s (sau 155s kể từ khi khôi phục GPU) |
| 460 | Hệ thống hồi phục hoàn toàn | `calls_ok = 9`, `calls_done = 12`, `attempts_alpha = 14` attempts/s, latency trở lại bình thường |

**Trigger vs root cause:**
* **Trigger (Ngòi nổ):** Sự cố phần cứng làm mất 75% năng lực GPU của provider alpha tại giây 120.
* **Root cause (Nguyên nhân gốc rễ):** Cơ chế client naive gồm 3 tầng retry chồng chéo (tối đa 36 lần thử/call), retry với delay cố định không có Jitter, không tuân thủ `retry-after`, không có Retry Budget và thiếu Circuit Breaker.
* **Vòng lặp tự duy trì (Metastable Failure):** Khi GPU giảm, hàng đợi server bị đầy $\rightarrow$ server trả 529 và client bị timeout $\rightarrow$ client naive retry ồ ạt $\rightarrow$ tải gửi đến tăng vọt lên gấp 10 lần $\rightarrow$ server tiếp tục xử lý các request cũ mà client đã timeout bỏ cuộc trong khi các request mới tiếp tục kẹt. Khi GPU hồi phục ở $t=300$, lượng request retry tồn đọng khổng lồ vẫn vượt quá 100% năng lực phục vụ, duy trì tình trạng sập thêm 155 giây.

**Điều gì giúp / điều gì làm tệ hơn:**
* *Làm tệ hơn:* Việc retry mù quáng với khoảng thời gian cố định và không có giới hạn trần ngân sách retry; bỏ qua thông điệp `retry-after` từ server.
* *Giúp ích:* Cơ chế Circuit Breaker cắt tải ngay sang deployment dự phòng khi tỷ lệ lỗi vượt 50%; Full Jitter giúp dàn trải lưu lượng; Retry Budget khống chế lượng retry dưới 10%; và cơ chế HALF-OPEN thăm dò dè dặt khi server hồi phục.

**Action items:**

| # | Hành động | Loại (prevent / detect / mitigate) | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Cài đặt Circuit Breaker sliding window và Retry Budget cho toàn bộ client | **Prevent** | Tech Lead (Client SDK) | Chạy kiểm tra kịch bản G2 trong CI: `amplification <= 1.2` và `recovery_s == 0.0s`. |
| 2 | Thiết lập hệ thống cảnh báo đa cửa sổ dựa trên Burn Rate (Google SRE) | **Detect** | SRE Team | Bắt chính xác sự cố suy giảm trong vòng $\le 5$ phút, 0 false page trên dữ liệu lịch sử. |
| 3 | Tự động điều hướng lưu lượng (Failover) sang vùng phụ (`alpha-eu`, `beta`) | **Mitigate** | Platform Team | Diễn tập Game Day định kỳ: Availability duy trì $\ge 98\%$ khi primary mất 75% năng lực. |

---

## 7. Một điểm yếu còn lại

* **Điểm yếu còn lại:** Khi xảy ra sự cố diện rộng trên toàn bộ nhà cung cấp thuộc nhóm chính (`alpha` và `alpha-eu`), client phải chuyển tải sang `beta`. Mặc dù `beta` hỗ trợ đầy đủ công cụ và duy trì được availability 100%, chi phí của `beta` đắt hơn 40% (cost 1.4 so với 1.0) và năng lực rate limit chỉ có 16 rps. Nếu sự cố kéo dài nhiều ngày, chi phí vận hành sẽ tăng đột biến và `beta` có nguy cơ bị quá tải thứ cấp.
* **Giải pháp khắc phục:** 
  1. Áp dụng kỹ thuật suy giảm duyên dáng (Graceful Degradation): Đối với các lượt chat thông thường không cần gọi công cụ, ưu tiên định tuyến sang `gamma-mini` (chi phí chỉ 0.2) thay vì đổ dồn vào `beta`.
  2. Kết hợp Semantic Cache (Part G) để chặn đứng tối đa các câu hỏi lặp lại ngay từ biên mạng trước khi gửi tới `beta`.

---

## 8. Cache (Part G)

Số từ `reports/cache/*.json` (`make cache`):

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của bạn) | 0.9120 | 0.9645 | 0.627 | 186 | 0 | 6 | 0.933 | PASS |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| `similarity_threshold` | 0.90 | Ngưỡng mặc định 0.80 làm nhầm lẫn giữa các câu hỏi có ngữ nghĩa gần nhau nhưng nội dung khác nhau (như hỏi quy định hoàn tiền vs thời gian hoàn tiền), tạo ra **112 câu trả lời sai** (`other_question`). Nâng lên 0.90 triệt tiêu hoàn toàn lỗi này (`wrong_answers = 0`) mà vẫn giữ được tỷ lệ trúng cache cao (62.7%). |
| `ttl_s` | 40 | Dựa trên sự kiện cập nhật bảng giá cước vận chuyển ngầm ở giây 480 (không có webhook). Nếu để TTL 300s, có tới 95 câu trả lời bị lỗi thời; để TTL 40s giúp số câu trả lời lỗi thời chỉ còn **6 câu** ($\le 15$), trong khi vẫn duy trì được hit ratio $\ge 50\%$. |
| `max_stale_s` | 300 | Dựa trên sự cố sập toàn bộ provider kéo dài 120s (từ t=240 đến t=360). Nếu để bằng 0, không có câu trả lời stale nào được phục vụ, faq_availability sụt xuống 85.64% (FAIL). Đặt 300s giúp phục vụ an toàn **186 câu trả lời stale**, kéo faq_availability lên **96.45%** (PASS). |

### Thí nghiệm:

1. **Ngưỡng tương đồng (`similarity_threshold`):**
   * Với ngưỡng **0.80**: `cache_hit_ratio = 0.627`, phát sinh **112 câu trả lời sai** (`wrong_answers = 112`, toàn bộ là `other_question`), chi phí là **0.975**. Các cặp câu hỏi bị trả lời nhầm là những câu có embedding centroid gần nhau ($\approx 0.86$) như: hỏi *chính sách hoàn tiền* (`refund_policy`) bị nhầm sang *thời gian nhận tiền hoàn* (`refund_time`), hoặc *phí ship nội thành* bị nhầm sang *phí ship đi tỉnh*.
   * Với ngưỡng **0.90**: `cache_hit_ratio = 0.627`, **0 câu trả lời sai**, chi phí tối ưu nhất ở mức **0.933**.
   * Với ngưỡng **0.97**: Quá khắt khe khiến nhiều câu hỏi cùng ý nghĩa bị bỏ lỡ, `cache_hit_ratio` tụt xuống **0.538**, `wrong_answers = 0`, chi phí tăng lên **0.992** do phải gọi provider nhiều hơn.

2. **TTL (`ttl_s`):**
   * Khi thử nghiệm với TTL = 30s / 60s / 300s, số câu trả lời lỗi thời (`outdated_answers`) lần lượt là **2** / **3** / **95 câu** (tương ứng hit ratio là 59.2%, 65.2%, 73.1%).
   * *Giải thích:* Webhook chỉ phát huy tác dụng khi CMS chủ động gửi thông báo thay đổi nội dung. Với những thay đổi âm thầm từ bên thứ ba (như đơn vị vận chuyển tự đổi giá ship ở $t=480$ mà không gửi webhook), hệ thống không có cách nào biết để xóa cache. Do đó, **TTL là cơ chế tự vệ duy nhất** buộc cache phải hết hạn định kỳ để kéo dữ liệu mới về.

3. **Stale-if-error (`max_stale_s: 0`):**
   * Khi tắt stale-if-error (`max_stale_s: 0`), trong 120 giây toàn bộ 4 provider bị sập (t=240 đến t=360), số câu trả lời stale phục vụ được là **0**, khiến `faq_availability` tụt thảm hại từ **96.45% xuống 85.64%** (không đạt chỉ tiêu $\ge 93\%$).

**Trả lời câu hỏi:**
* **Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") KHÔNG được cache, kể cả khi cache theo từng user?** Vì dữ liệu trạng thái đơn hàng và điểm thưởng thay đổi liên tục theo thời gian thực (vừa giao hàng, vừa thanh toán). Nếu cache, người dùng sẽ nhận được thông tin cũ. Ngoài ra, câu hỏi cá nhân của hai người dùng khác nhau chỉ khác nhau mỗi mã đơn hàng nên vector embedding rất giống nhau ($\sim 0.99$), nếu sơ suất lưu vào cache chung sẽ gây rò rỉ dữ liệu nghiêm trọng (`other_user`).
* **Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố?** Cần tạo alert dựa trên chỉ số **tỷ lệ phục vụ dữ liệu cũ (`stale_served`)** hoặc **tỷ lệ gọi provider thất bại ở tầng gateway backend** (upstream provider error rate). Khi `stale_served > 0` hoặc tỷ lệ lỗi của các provider chạm ngưỡng burn rate, hệ thống phải kích hoạt Page ngay lập tức cho đội on-call, không để tỷ lệ thành công ở tầng giao diện người dùng che giấu sự cố hạ tầng.


# Q4: Performance Troubleshooting Stories

## 📋 Question
**"Describe a performance issue you identified and resolved. What was your methodology, and how did you measure the improvement?"**

## Difficulty: 🟠 Medium-Hard

---

## 💡 How to Answer

### The STAR Framework

- **Situation**: Describe the system under load, the performance symptoms (latency
  spikes, throughput degradation, resource exhaustion), and the business impact.
  Include numbers — response times, error rates, user complaints, SLA breaches.
  Be specific about what "slow" meant: a p95 of 1200ms is very different from
  a p50 of 1200ms.
- **Task**: Clarify what "good" performance looked like (the target/SLA) and why
  you were the one to investigate. Was this proactive (you noticed a trend) or
  reactive (alerts fired, users complained)?
- **Action**: Walk through your profiling and analysis methodology step by step.
  Name the tools you used, the hypotheses you formed and tested, the dead ends
  you hit, and the specific bottleneck you identified. Then describe the fix
  and how you validated it before deploying to production.
- **Result**: Quantify the before and after. Show the improvement in concrete
  terms — p95 latency, throughput, resource utilization, cost savings. Describe
  any systemic improvements you made to prevent similar issues.

### Sample Answer

> **Situation**: Our Spring Boot order management API had a p95 response time of
> 1,200ms on the "get order history" endpoint, which was well above our SLA of
> 500ms. The endpoint served the customer-facing order dashboard and was hit
> roughly 50,000 times per hour during peak. Customer complaints had increased
> 40% over the past month, and the product team was considering adding a
> "loading" spinner — essentially giving up on performance. Our monitoring
> showed the degradation had been gradual over three months, correlating with
> growth in average customer order history size. The database was PostgreSQL,
> and we used Hibernate as the ORM with Spring Data JPA repositories.
>
> **Task**: As a senior developer on the team, I volunteered to investigate before
> we accepted the performance regression as permanent. My goal was to bring p95
> below 300ms without major architectural changes (no caching layer, no read
> replicas — those were options but I wanted to find the root inefficiency first).
> I allocated myself three days for investigation and implementation.
>
> **Action**: I followed a systematic profiling methodology — measure, locate,
> analyze, fix, validate:
>
> 1. **Established a baseline**: I used Micrometer metrics already instrumented
>    in the application to confirm the p95 was consistently at 1,200ms. I also
>    captured the baseline CPU utilization (45%), heap usage (60%), and database
>    connection pool metrics (HikariCP active connections frequently at 8/10
>    maximum). I set up a Grafana dashboard with these metrics so I could watch
>    them in real-time during testing.
>
> 2. **Identified the slow layer**: I added timing breakdowns using Spring AOP
>    to measure time spent in the controller, service, and repository layers
>    independently. I created a simple aspect:
>    ```java
>    @Around("execution(* com.example.order..*(..))")
>    public Object timeMethod(ProceedingJoinPoint pjp) throws Throwable {
>        Timer.Sample sample = Timer.start(meterRegistry);
>        try {
>            return pjp.proceed();
>        } finally {
>            sample.stop(Timer.builder("method.execution")
>                .tag("class", pjp.getTarget().getClass().getSimpleName())
>                .tag("method", pjp.getSignature().getName())
>                .register(meterRegistry));
>        }
>    }
>    ```
>    Results: controller overhead was negligible (5ms), service layer processing
>    was 100ms, and the repository layer accounted for 1,095ms. The database was
>    clearly the bottleneck.
>
> 3. **Analyzed the queries**: I enabled Hibernate statistics
>    (`spring.jpa.properties.hibernate.generate_statistics=true`) and used
>    `p6spy` for detailed query analysis in a staging environment with
>    production-volume data. I discovered a classic **N+1 query problem**: the
>    `findOrdersByCustomerId` method returned orders, but each order's line
>    items, shipping details, and payment information were lazily loaded in
>    separate queries. For a customer with 30 orders, this generated **91 SQL
>    queries** per request:
>    - 1 query for orders
>    - 30 queries for line items (one per order)
>    - 30 queries for shipping details
>    - 30 queries for payment records
>
>    Each individual query was fast (5-15ms), but 91 sequential round trips to
>    the database added up to 1,095ms. The query count scaled linearly with
>    order history size, explaining the gradual degradation.
>
> 4. **Implemented the fix in stages**:
>
>    - **Stage 1 — Query optimization (Day 1)**: I replaced the lazy loading
>      with a single `@Query` annotation using `JOIN FETCH`:
>      ```java
>      @Query("SELECT o FROM Order o " +
>             "JOIN FETCH o.lineItems " +
>             "JOIN FETCH o.shippingDetails " +
>             "JOIN FETCH o.payment " +
>             "WHERE o.customerId = :customerId " +
>             "ORDER BY o.createdAt DESC")
>      List<Order> findOrdersWithDetailsByCustomerId(
>          @Param("customerId") Long customerId);
>      ```
>      This brought the query count from 91 down to 1, and the p95 dropped
>      from 1,200ms to 450ms. However, I noticed the single query was now
>      returning a large result set for customers with 100+ orders.
>
>    - **Stage 2 — Pagination (Day 1)**: I added pagination to the query,
>      returning only the most recent 20 orders by default with a cursor-based
>      "load more" option. This kept the result set bounded regardless of
>      history size.
>
>    - **Stage 3 — Connection pool tuning (Day 2)**: With the N+1 fix in place,
>      I noticed the connection pool was still a bottleneck. HikariCP was
>      configured with only 10 connections, but the service had 200 concurrent
>      request threads. Under load, threads were waiting for connections. I
>      right-sized the pool to 30 connections using the HikariCP formula:
>      `connections = (core_count * 2) + effective_spindle_count`, verified with
>      load testing that this eliminated connection wait times. I also set
>      `connectionTimeout` to 5 seconds (from the 30-second default) so slow
>      connection acquisition would fail fast instead of blocking. The p95
>      dropped further to 280ms.
>
>    - **Stage 4 — Response caching (Day 2)**: For customers with stable order
>      history (no new orders in the last hour), I added a Caffeine cache with
>      15-minute TTL, reducing database hits for repeat page views by 60%.
>      I also added cache invalidation triggered by new order events to ensure
>      consistency.
>
> 5. **Validated the fix (Day 3)**: I ran load tests with Gatling simulating
>    1,000 concurrent users over 30 minutes with a realistic mix of customers
>    (varying order history sizes from 5 to 500 orders). The p95 stayed below
>    200ms. I also added a Hibernate query count assertion in integration tests:
>    ```java
>    @Test
>    void shouldFetchOrderHistoryInSingleQuery() {
>        Statistics stats = entityManagerFactory
>            .unwrap(SessionFactory.class).getStatistics();
>        stats.clear();
>
>        orderRepository.findOrdersWithDetailsByCustomerId(customerId);
>
>        assertThat(stats.getQueryExecutionCount()).isEqualTo(1);
>    }
>    ```
>    This would catch any future N+1 regression. I deployed incrementally
>    using feature flags, monitoring each stage for a full day before enabling
>    the next optimization.
>
> **Result**: The p95 response time dropped from 1,200ms to 180ms — an 85%
> improvement. Database CPU utilization fell from 78% to 35% during peak hours.
> Connection pool wait time was virtually eliminated. The product team canceled
> the "loading spinner" ticket. Customer satisfaction scores for the order
> dashboard improved by 22% in the following month's survey. I documented the
> N+1 detection and resolution pattern in our team wiki, and we added Hibernate
> query count assertions in integration tests across all repositories to prevent
> future N+1 regressions. The p6spy configuration was standardized so any
> developer could enable query logging in staging with a single config change.

### Key Points to Emphasize
- You followed a structured profiling methodology — measure, identify, fix, validate
- You used specific tools: Micrometer, Grafana, p6spy, Hibernate statistics, Gatling, Caffeine
- You identified the root cause (N+1 queries) rather than guessing
- You fixed the problem in stages, each with measurable improvement
- You validated with load testing before and after
- You added preventive measures (query count assertions, documentation)
- You quantified everything — before/after numbers strengthen your story significantly
- You considered proportionality — fixed the root issue before adding caching

### What NOT to Say
- "I just added an index and it got faster" — too shallow, no methodology
- "I increased the heap size / added more instances" — throwing resources at a problem
  isn't engineering
- "I rewrote the whole service in a faster language" — disproportionate response
- Don't describe a fix without explaining how you found the bottleneck
- Avoid answers where you only describe what you did, not why each step was right
- "It was slow, so I cached everything" — caching without understanding root cause
  hides problems and adds complexity

---

## 🎯 Interviewer Insight

Performance troubleshooting is one of the highest-signal interview topics.
Interviewers evaluate:

1. **Methodology** — Do you have a systematic approach? The best candidates follow:
   measure → hypothesize → test → fix → validate → prevent.
2. **Tool Knowledge** — Do you know the Java/JVM performance toolkit? Profilers
   (async-profiler, YourKit), APM tools (Datadog, New Relic), JVM flags, database
   analysis tools (EXPLAIN ANALYZE, pg_stat_statements).
3. **Root Cause Depth** — Can you go beyond "the database was slow" to explain
   exactly why (N+1 queries, missing index, lock contention, full table scan)?
4. **Proportionality** — Is your fix appropriate to the problem? Over-engineering
   is a red flag. Under-engineering (just restart) is also a flag.
5. **Prevention** — Do you stop at fixing, or do you also prevent recurrence?
6. **Communication** — Can you explain the investigation clearly to a non-expert?

### Performance Troubleshooting Checklist

| Layer | What to Check | Tools |
|-------|--------------|-------|
| Application | N+1 queries, inefficient algorithms, blocking calls | p6spy, Hibernate stats, profiler |
| JVM | GC pauses, heap pressure, thread contention | GC logs, JFR, jstat, async-profiler |
| Database | Slow queries, missing indexes, lock waits | EXPLAIN ANALYZE, pg_stat_statements |
| Network | Latency to dependencies, DNS resolution | curl timing, tcpdump, Wireshark |
| OS | CPU saturation, disk I/O, memory swapping | top, vmstat, iostat |
| Connection pools | Exhaustion, leak, wrong sizing | HikariCP metrics, leak detection |

## ⚡ Follow-up Questions

1. "How did you decide which optimization to try first?"
2. "What trade-offs did the caching layer introduce, and how did you handle
   cache invalidation?"
3. "How did you determine the right connection pool size?"
4. "What if the N+1 fix had made the single query too complex or slow due to
   large Cartesian product from multiple joins?"
5. "How do you proactively prevent performance regressions in your development
   process?"

## 📚 Related Scenarios
- [Q3: Debugging Complex Issues](./q03-debugging-complex-issues.md) — overlapping
  diagnostic techniques
- [Q6: Architectural Decisions](./q06-architectural-decisions.md) — when performance
  requires architectural change
- [Q9: Production Incident Response](./q09-production-incident-response.md) —
  performance degradation as a production incident

---

## 🔑 Additional Examples You Can Adapt

### GC Tuning Under Load
- **Situation**: Service experienced 2-3 second latency spikes every few minutes
  during peak traffic. Application logs showed no errors, no slow queries, no
  external dependency issues. The spikes were completely invisible to application-level
  monitoring — they only appeared in the load balancer's response time metrics.
- **Action**: Enabled GC logging with `-Xlog:gc*:file=/var/log/gc.log:time,uptime,level,tags`.
  Discovered long Stop-the-World pauses during G1 GC Full GC events — each pause was
  2-3 seconds. The old generation was filling because short-lived request objects were
  being promoted prematurely due to an undersized young generation. The default
  `-XX:NewRatio=2` gave young gen only 33% of the 4GB heap. Analyzed allocation rates
  with `jstat -gcutil <pid> 1000` and confirmed that young gen was filling every
  200ms, causing extremely frequent minor GCs that promoted objects before they died.
- **Result**: Increased young generation to 40% of heap with `-XX:NewRatio=1`,
  tuned `-XX:MaxGCPauseMillis=100` to guide the GC toward shorter pauses, and
  switched to ZGC for the most latency-sensitive service (ZGC pause times are
  sub-millisecond regardless of heap size). p99 spikes dropped from 3 seconds
  to 200ms. Added GC pause time as a standard Grafana dashboard metric for all
  services.

### Thread Contention in Synchronized Block
- **Situation**: API throughput plateaued at 500 req/s despite CPU being at only
  30% utilization. Adding more Tomcat threads from 200 to 400 didn't help — throughput
  stayed the same while CPU dropped further. This was counterintuitive: more threads
  should have increased throughput for I/O-bound work, but something was serializing
  the requests.
- **Action**: Used async-profiler in lock contention mode (`-e lock -d 30 -o flamegraph`).
  The resulting flame graph showed that 80% of thread time was spent waiting to enter
  a `synchronized` block in `LegacyDateUtils.formatTimestamp()`. This utility class
  used a shared `SimpleDateFormat` instance variable (not thread-safe), and every
  single request called it to format the response timestamp. With 200 threads, only
  one could enter the `synchronized` block at a time — the rest waited in line. The
  lock graph showed an average wait time of 3.8ms per acquisition.
- **Result**: Replaced `SimpleDateFormat` with `DateTimeFormatter` from `java.time`
  (thread-safe, immutable, no synchronization needed). Throughput immediately jumped
  from 500 to 3,200 req/s on the same hardware. CPU utilization rose to 75%,
  indicating the CPUs were finally doing useful work instead of context-switching
  between waiting threads. Added a static analysis rule (using ArchUnit) to flag
  any usage of `SimpleDateFormat` in the codebase.

### Database Connection Leak
- **Situation**: Service gradually degraded over 6-8 hours until it stopped
  responding entirely. All requests would hang waiting for a database connection.
  Restarting temporarily fixed it for another 6-8 hours. The pattern was
  extremely consistent: exactly 6-8 hours from restart to failure.
- **Action**: Monitored HikariCP connection pool metrics via Micrometer. Active
  connections grew linearly (about 2 per hour) and never returned to the pool.
  This was a classic connection leak pattern. Enabled HikariCP's leak detection
  with `leakDetectionThreshold=30000` (30 seconds — any connection held longer
  than 30s triggers a warning with a stack trace). The stack traces pointed to a
  code path where `dataSource.getConnection()` was called manually for a bulk
  CSV export operation, but the `finally` block was missing. When the CSV
  generation threw a `RuntimeException` (which happened for certain data
  patterns), the connection was never returned to the pool. Each invocation of
  the export feature leaked one connection.
- **Result**: Fixed the resource management with try-with-resources. Added
  HikariCP connection pool monitoring dashboard with alerts at 70% and 90%
  utilization. Added a code review checklist item for manual connection handling.
  Conducted a codebase audit for all `getConnection()` calls and found two more
  paths with potential leaks. Added an integration test that verified connection
  pool size returned to baseline after each test method execution.

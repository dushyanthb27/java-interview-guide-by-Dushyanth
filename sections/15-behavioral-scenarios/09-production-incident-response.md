# Q9: Production Incident Response

## 📋 Question
**"Tell me about a production incident you managed. Walk me through the triage, resolution, communication, and post-mortem process."**

## Difficulty: 🟠 Medium-Hard

---

## 💡 How to Answer

### The STAR Framework

- **Situation**: Describe the incident — what broke, what were the symptoms from the
  user's perspective, and what was the blast radius? Include the time of day (incidents
  at 2 AM are different from incidents at 2 PM), the severity level, and the business
  impact in concrete terms (revenue loss, users affected, SLA breach). Set the stakes
  clearly — production incidents are high-pressure, and interviewers want to see how
  you operate when it matters most.
- **Task**: What was your role? Were you the on-call engineer who got paged? The
  incident commander who coordinated the response? A subject-matter expert who was
  pulled in? What was expected of you — fix it, coordinate, communicate, or all three?
- **Action**: Walk through the incident timeline in order: detection → triage →
  mitigation → diagnosis → resolution → communication → post-mortem. Show both
  technical skill (diagnostic commands, root cause analysis) and process discipline
  (structured communication, stakeholder updates, incident management). Demonstrate
  that you can think clearly under pressure, communicate proactively, and coordinate
  with others effectively.
- **Result**: Describe the resolution, the post-mortem findings, and the preventive
  measures implemented. Show that the incident led to systemic improvement in
  reliability, monitoring, or process — not just a one-time fix.

### Sample Answer

> **Use this as a structure, not a story to memorize.** This is an illustrative
> composite scenario, not a claim about the reader's experience. Replace every
> detail with a real incident you can discuss truthfully; anonymize company and
> customer information, and use approximate impact figures only when you can
> explain how they were measured. If you have not led an incident, say so and
> describe your actual role in one.

> **Situation**: At 2:15 AM on a Tuesday, I received a PagerDuty alert: our order
> processing service was returning HTTP 503 errors for 60% of requests. The service
> was a Spring Boot application running on four instances behind an AWS ALB, handling
> 3,000 orders per minute during our overnight batch processing window for
> international customers across European and Asian time zones. The Grafana dashboard
> showed response times had spiked from a normal 200ms to 30+ seconds, and the error
> rate had gone from our baseline 0.1% to 62% in under 10 minutes. Two of the four
> instances were showing extreme GC activity. Revenue impact was approximately $15K
> per minute of degraded service.
>
> **Task**: I was the primary on-call engineer for the order platform. My defined
> responsibilities during an incident were:
> 1. Triage severity and mitigate impact as quickly as possible
> 2. Communicate status to the incident Slack channel with regular updates
> 3. Escalate to additional engineers or teams if needed
> 4. Drive to root cause resolution
> 5. Lead the post-mortem process within 48 hours
>
> **Action**: I followed our incident response framework methodically:
>
> 1. **Detection and initial triage (2:15 - 2:25 AM)**:
>
>    First, I opened the `#incident-orders` Slack channel and posted the initial
>    status message:
>    > "🔴 **SEV-1 INCIDENT** — Investigating elevated 503s on order-service.
>    > Impact: ~60% of orders failing. Revenue impact: ~$15K/min.
>    > I'm acting as incident commander. Next update in 10 minutes."
>
>    Then I started systematic triage — checking the most likely causes first:
>    - **Recent deployments**: Checked the deployment log — no deployments in the
>      last 18 hours. Ruled out code change as trigger.
>    - **Dependent services**: Checked the payment gateway and inventory service
>      dashboards — both healthy, normal latency. Not an upstream issue.
>    - **Infrastructure**: AWS health dashboard showed no issues. ALB was healthy.
>      Two of four instances were responding, two were not.
>    - **Application metrics**: This is where the signal was. On the two unhealthy
>      instances: CPU at 40% (normal), but heap usage at 92% and climbing. GC
>      activity was extreme — the GC log showed Full GC events every 15 seconds,
>      each taking 3-5 seconds. The application was spending more time in GC than
>      processing requests.
>
> 2. **Immediate mitigation (2:25 - 2:35 AM)**:
>
>    I suspected `OutOfMemoryError` was imminent on both unhealthy instances.
>    Before restarting, I needed diagnostic data:
>
>    ```bash
>    # Capture heap dump from one unhealthy instance before it crashes
>    jmap -dump:live,format=b,file=/tmp/heap_incident_20240115.hprof <pid>
>
>    # Capture thread dump for good measure
>    jstack <pid> > /tmp/threads_incident_20240115.txt
>    ```
>
>    Then I restarted the two unhealthy instances (rolling restart, one at a time)
>    while the two healthy instances continued serving traffic at reduced capacity.
>
>    Slack update at 2:35 AM:
>    > "🟡 **UPDATE** — Mitigated with rolling restart of 2 unhealthy instances.
>    > All 4 instances now healthy. Error rate dropping to baseline. Captured
>    > heap dump and thread dump before restart. Investigating root cause.
>    > Suspected: heap exhaustion / memory leak. Next update in 15 minutes."
>
> 3. **Root cause analysis (2:35 - 3:30 AM)**:
>
>    With the service stabilized, I moved to root cause analysis:
>
>    - **Heap dump analysis**: Opened the 2.4GB heap dump in Eclipse MAT. The
>      "Leak Suspects" report immediately flagged one issue: 1.2 million
>      `OrderAuditEvent` objects consuming 1.8GB of the 4GB heap. These were
>      audit log entries that should have been flushed to Kafka asynchronously
>      every 100ms, not accumulated in memory.
>
>    - **Follow the reference chain**: MAT's "Path to GC Roots" showed the
>      `OrderAuditEvent` objects were held by an `ArrayList` inside the
>      `AsyncAuditProducer` class. This was an in-memory buffer that the
>      producer drained to Kafka. But the buffer was unbounded — it had no
>      size limit.
>
>    - **Why was the buffer growing?**: Checked the Kafka producer metrics
>      in Grafana. The producer was blocked — `record-send-rate` had dropped
>      to zero at 1:45 AM (30 minutes before the alert). The Kafka broker
>      logs (requested from the platform team, who I paged at 2:45 AM)
>      showed a **disk full condition** on broker-2. The partition leader for
>      the `order-audit` topic was on broker-2, and when its disk filled up,
>      the leader became unavailable.
>
>    - **Root cause chain**:
>      ```
>      Kafka broker disk full (root cause)
>        → Partition leader unavailable
>          → Kafka producer send() calls blocked (max.block.ms=60000)
>            → AsyncAuditProducer's unbounded ArrayList kept accumulating events
>              → Events generated faster than they could be sent
>                → Heap exhausted (1.2M events × 1.5KB each ≈ 1.8GB)
>                  → Full GC storms (every 15 seconds, 3-5s each)
>                    → Request processing starved of CPU and memory
>                      → HTTP 503 errors (60% of requests)
>      ```
>
> 4. **Full resolution (3:30 - 4:00 AM)**:
>
>    - **Immediate fix**: The Kafka platform team freed disk space on broker-2
>      by purging expired topic partitions and increasing the disk volume. The
>      producer unblocked, the in-memory buffer drained, and heap usage
>      normalized.
>
>    - **Short-term fix** (applied same night): I changed the `AsyncAuditProducer`
>      to use a bounded `ArrayBlockingQueue` (capacity: 10,000) instead of
>      an `ArrayList`. When the queue was full, new events would be dropped
>      with a warning log and a metric increment (`audit.events.dropped`).
>      This ensured a Kafka outage could never again cause heap exhaustion —
>      at worst, we'd lose some audit events temporarily.
>
>    - **Added monitoring**: Created a Grafana alert for `audit.queue.size > 5000`
>      (50% capacity) to give early warning of Kafka backpressure.
>
> 5. **Communication throughout the incident**:
>
>    - Sent Slack updates every 10-15 minutes, even when there was no new
>      information: "Still analyzing heap dump, ETA 15 min for initial findings."
>      Silence during an incident is worse than "no new info yet."
>    - Paged the Kafka platform team at 2:45 AM to assist with broker recovery
>    - At 4:30 AM, sent an executive summary email to VP of Engineering, Director
>      of Product, and the on-call manager:
>
>      > **Subject: SEV-1 Resolved — Order Service Degradation (2:15 - 4:00 AM)**
>      > - **Duration**: 1 hour 45 minutes
>      > - **Impact**: ~60% of orders failed for 20 minutes; reduced to 0% after
>      >   mitigation at 2:35 AM
>      > - **Root Cause**: Kafka broker disk full → audit event backpressure → heap
>      >   exhaustion
>      > - **Resolution**: Broker disk freed, bounded event buffer deployed
>      > - **Customer Impact**: No orders permanently lost (retried automatically)
>      > - **Post-mortem**: Scheduled for Wednesday 2 PM
>
> 6. **Post-mortem (Wednesday, 2 days later)**:
>
>    Led a **blameless** post-mortem with 8 participants (order team, Kafka platform
>    team, SRE team). Key principles: no blame, focus on systems and processes,
>    identify actionable improvements.
>
>    **Timeline reviewed**: Every action from detection to resolution with timestamps.
>
>    **Contributing factors identified** (not "root cause" — incidents usually have
>    multiple contributing factors):
>    - Kafka broker disk monitoring alert was set at 95%, too late to act
>    - Audit producer used unbounded in-memory buffer with no backpressure strategy
>    - `max.block.ms=60000` on the Kafka producer meant threads blocked for 60 seconds
>      before failing — amplifying the memory pressure
>    - No monitoring on audit event queue depth or Kafka producer blocked time
>
>    **Five action items assigned with owners and due dates**:
>    1. Add Kafka broker disk usage alerts at 70% and 85% thresholds (Platform team,
>       1 week)
>    2. Bound all in-memory queues in the order service with configurable limits and
>       backpressure handling (Order team, 1 sprint)
>    3. Add `max.block.ms` timeout handling in the Kafka producer — if send blocks
>       for more than 5 seconds, switch to fallback local file logging (Order team,
>       1 sprint)
>    4. Implement a circuit breaker on the audit service — if Kafka is unavailable
>       for 30+ seconds, switch to async file-based logging until Kafka recovers
>       (Order team, 2 sprints)
>    5. Add a runbook entry for "order-service heap exhaustion" symptoms with
>       diagnostic steps (Order team, 1 week)
>
>    Published the post-mortem document to the engineering wiki within 48 hours.
>
> **Result**: The incident lasted 1 hour 45 minutes from detection to full resolution.
> Actual customer-facing degradation was 20 minutes (from first alert to successful
> mitigation via rolling restart). No orders were permanently lost — the failed
> orders were retried successfully by the client applications after recovery.
>
> All five action items were completed within two sprints. In the six months following:
> - Zero incidents related to Kafka backpressure or unbounded queues
> - The bounded queue pattern was adopted as a team-wide standard for all async
>   producers across all services
> - Kafka broker disk alerts at 70% fired twice, allowing proactive cleanup before
>   any customer impact
> - The post-mortem was cited as an example of good incident response practice by
>   our VP of Engineering and used as a training template for new on-call engineers

### Key Points to Emphasize
- You followed a structured incident response process — triage → mitigate → diagnose
  → resolve → communicate → post-mortem
- You communicated proactively and regularly, not just when you had answers
- You assessed whether mitigation was safe and likely to reduce impact, while
  preserving useful evidence when doing so would not delay recovery; the right order
  depends on the incident
- You captured diagnostic data (heap dump) before restarting — evidence preservation
- You traced the full root cause chain, not just the immediate symptom
- Your post-mortem was blameless and produced actionable, time-boxed improvements
- You escalated appropriately (Kafka team) rather than trying to solo everything
- The incident led to systemic improvements that prevented recurrence

### What NOT to Say
- "I fixed it by restarting the server" — shows no diagnostic depth or root cause analysis
- "I spent 4 hours debugging before telling anyone" — poor communication discipline
- "It was the other team's fault — their Kafka broker was full" — blame-oriented thinking
- "I stayed up until 8 AM fixing everything by myself" — heroics suggest poor escalation
- If you made a mistake or delayed communication, explain it honestly, how you
  corrected course, and what you changed afterward; accountability is stronger than
  a polished but implausible story
- "We don't do post-mortems" — suggests a team that doesn't learn from incidents

---

## 🎯 Interviewer Insight

Production incident response reveals critical senior engineering skills:

1. **Composure Under Pressure** — Can you think clearly and make good decisions when
   revenue is at stake and people are watching?
2. **Systematic Diagnosis** — Do you follow the evidence methodically, or guess randomly
   and make things worse?
3. **Communication Discipline** — Do you keep stakeholders informed proactively with
   structured updates?
4. **Mitigation vs Root Cause** — Can you balance restoring service with preserving
   evidence, and explain why the chosen action was appropriate for this incident?
5. **Post-Incident Learning** — Do incidents under your watch lead to systemic
   improvements, or just one-time fixes?
6. **Escalation Judgment** — Do you know when to bring in help versus trying to
   solve everything alone?

### Incident Communication Template

```
🔴/🟡/🟢 [STATUS] — [Service] [Issue Type]
- Impact: [What users see, how many affected]
- Current theory: [What you think is happening]
- Actions: [What you're doing right now]
- Next update: [Specific time, e.g., "in 15 minutes"]
```

## ⚡ Follow-up Questions

1. "How do you decide when to mitigate (restart) versus continuing to diagnose?"
2. "How did you handle the stress of a 2 AM incident with revenue impact?"
3. "How do you ensure post-mortem action items actually get completed?"
4. "What's the difference between a good post-mortem and a bad one?"
5. "How do you prepare your team for on-call responsibilities?"

## 📚 Related Scenarios
- [Q3: Debugging Complex Issues](./q03-debugging-complex-issues.md) — similar diagnostic
  techniques applied in a calmer, non-incident setting
- [Q4: Performance Troubleshooting](./q04-performance-troubleshooting.md) — performance
  degradation can escalate to a production incident
- [Q10: Cross-Team Communication](./q10-cross-team-communication.md) — incident
  communication with non-technical stakeholders

---

## 🔑 Additional Examples You Can Adapt

### Thread Pool Exhaustion
- **Situation**: Service stopped accepting new requests — all API endpoints returned
  504 Gateway Timeout. Health check endpoint returned 200 (it ran on a separate
  thread pool). No errors in application logs. CPU at 5%. Memory stable. Everything
  looked "fine" except the service was completely unresponsive.
- **Action**: Captured a thread dump with `jstack <pid>`. Found all 200 Tomcat
  request threads were in `WAITING` state, blocked on `SocketInputStream.read()` —
  they were all waiting for responses from a third-party fraud detection API. The
  fraud API was experiencing a slowdown (responding in 45 seconds instead of 200ms),
  but it wasn't completely down, so the circuit breaker hadn't tripped (it was
  configured for failure rate, not slow call rate). Each request thread was tied up
  waiting for the fraud API, and with all 200 threads occupied, no new requests
  could be accepted.
- **Result**: Immediate: set `connectTimeout=3000` and `readTimeout=5000` on the
  fraud API HTTP client to free threads faster. Long-term: configured Resilience4j
  `TimeLimiter` with 5-second maximum per call; added a `slowCallRateThreshold=50`
  and `slowCallDurationThreshold=3s` to the circuit breaker configuration; created
  a runbook for "thread pool exhaustion" symptoms with diagnostic steps.

### Deadlock in Production
- **Situation**: Two instances of a payment service froze simultaneously during
  end-of-month batch processing. No CPU usage, no GC activity, no log output,
  process alive but completely unresponsive. Health checks failed, load balancer
  drained both instances, all requests failed.
- **Action**: Thread dump from both instances revealed identical deadlocks: Thread A
  held the monitor on `AccountService.debit()` and waited for the monitor on
  `TransactionService.commit()`. Thread B held `TransactionService.commit()` and
  waited for `AccountService.debit()`. Root cause: two users transferring money to
  each other simultaneously. Transfer A→B locked Account A then Transaction B.
  Transfer B→A locked Account B then Transaction A. Different lock ordering created
  the classic deadlock condition.
- **Result**: Refactored to consistent lock ordering — always lock by the lower
  account ID first, regardless of transfer direction. Replaced `synchronized` with
  `ReentrantLock.tryLock(5, TimeUnit.SECONDS)` so even if a deadlock somehow occurred,
  it would time out instead of freezing forever. Added `ThreadMXBean.findDeadlockedThreads()`
  to the health check endpoint — if a deadlock is detected, the health check fails
  and the load balancer removes the instance. Added a concurrent load test exercising
  bidirectional transfers.

### Connection Leak Causing Cascading Failure
- **Situation**: Service degraded gradually over 6-8 hours until it stopped responding.
  Pattern was perfectly predictable: deploy at 9 AM, first symptoms at 3 PM, full
  outage by 5 PM. Restarting bought another 6-8 hours. The team had been restarting
  twice daily for a week.
- **Action**: Monitored HikariCP connection pool metrics via Micrometer. Active
  connections grew linearly (about 2 per hour) and never returned to the pool — the
  classic "sawtooth that doesn't come back down" pattern. Enabled HikariCP leak
  detection with `leakDetectionThreshold=60000` (60 seconds — any connection held
  longer triggers a stack trace warning). The logged stack traces pointed to a code
  path in `BulkExportService.generateReport()` where `dataSource.getConnection()`
  was called manually. The method had a `try/catch` but no `finally` block. When
  the CSV generation threw a `NumberFormatException` for certain currency formats,
  the connection was never returned.
- **Result**: Wrapped the connection in try-with-resources (always the right approach
  for manual connection management). Added HikariCP monitoring dashboard with alerts
  at 70% and 90% pool utilization. Added a code review checklist item: "All manual
  `getConnection()` calls must use try-with-resources." Conducted a codebase-wide
  audit for `getConnection()` — found two more paths with potential leaks and fixed
  them preventively.

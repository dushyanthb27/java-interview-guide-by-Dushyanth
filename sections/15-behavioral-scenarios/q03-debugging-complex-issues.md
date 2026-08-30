# Q3: Debugging Complex Issues

## 📋 Question
**"Tell me about the most challenging bug you've ever solved. Walk me through your debugging process from discovery to resolution."**

## Difficulty: 🟠 Medium-Hard

---

## 💡 How to Answer

### The STAR Framework

- **Situation**: Describe the system, the symptoms, and why the bug was hard to find.
  Great answers involve bugs that were intermittent, environment-specific, or involved
  subtle interactions between components. Set the stakes — what was the business impact?
  How long had the bug been occurring before you got involved?
- **Task**: Explain your responsibility. Were you the one on-call? Did you volunteer
  to investigate? Was the team stuck and you stepped in? What was the expected timeline
  for resolution?
- **Action**: This is the core of your answer — it should be the longest section.
  Walk through your systematic debugging methodology step by step. Name specific
  tools, commands, and techniques. Show that you have a repeatable process, not just
  luck. Explain your reasoning at each step — why you chose that tool, what you
  expected to find, and what you actually found.
- **Result**: Describe the root cause, the fix, and any preventive measures you put
  in place. Quantify the impact of the resolution. Explain what you learned and what
  systemic improvements resulted from the investigation.

### Sample Answer

> **Situation**: We had a Java-based payment processing service running on Spring Boot
> that was experiencing intermittent `OutOfMemoryError: Metaspace` crashes in
> production — roughly once every 3-4 days. The service would run fine under normal
> load, pass all stress tests in staging, and the heap dumps from staging showed
> nothing unusual. The operations team had been "fixing" it by increasing Metaspace
> size (from 256MB to 512MB to 1GB) and restarting, but the problem kept recurring
> with longer intervals between crashes. This had been happening for six weeks
> before I got involved.
>
> **Task**: As the senior engineer on the payments team, I was asked to investigate
> after the third escalation in two weeks. The service processed $2M+ in transactions
> daily, so unplanned restarts were a serious business risk — each restart caused a
> 90-second gap in payment processing. The previous approach of increasing Metaspace
> was clearly treating the symptom, not the cause, and was unsustainable.
>
> **Action**: I followed a systematic approach over three days:
>
> 1. **Reproduced the pattern**: I started by looking at data, not code. I pulled
>    the crash timestamps from the last month and correlated them with deployment
>    logs, traffic patterns, and scheduled jobs. The crashes didn't correlate with
>    deployments or traffic spikes, but they did correlate with a weekly batch job
>    that loaded merchant-specific pricing rules from an external configuration
>    service. Every crash happened 3-4 days after a batch rule reload.
>
> 2. **Formed a hypothesis**: Metaspace stores class metadata and compiled code.
>    If Metaspace was growing, either new classes were being loaded continuously
>    or loaded classes weren't being garbage collected. I hypothesized that the
>    pricing rule loading mechanism was somehow creating new classes on each reload
>    without releasing the old ones.
>
> 3. **Gathered diagnostic data**: I enabled JVM flags on the production instance
>    to capture evidence:
>    - `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/dumps/` to capture
>      the state at crash time automatically
>    - `-verbose:class` temporarily on a canary instance to watch class loading
>      and unloading activity in real time
>    - I also attached Java Flight Recorder (JFR) with
>      `jcmd <pid> JFR.start duration=24h filename=/var/jfr/payment.jfr` to capture
>      a full day of activity including class loading events, memory allocation,
>      and GC behavior
>
> 4. **Analyzed the evidence**: The JFR recording was the breakthrough. The class
>    loading events view revealed that class count was growing steadily — roughly
>    200 new classes loaded per hour with zero classes being unloaded. In a healthy
>    JVM, you'd expect class count to stabilize after startup. The `-verbose:class`
>    output confirmed this, showing repeated loading of dynamically generated classes
>    like `GeneratedMethodAccessor_1234`, `Proxy$_5678`, and
>    `GroovyScript_rule_pricing_<hash>`.
>
> 5. **Narrowed the root cause**: I used `jmap -clstats <pid>` to dump classloader
>    statistics, which showed thousands of instances of `groovy.lang.GroovyClassLoader`
>    — each one holding references to a small number of classes. I traced this back
>    to our pricing rules engine: it used Groovy scripts for flexible rule evaluation
>    (merchants could customize their pricing logic). The critical flaw: each time a
>    rule was reloaded, the code created a **new** `GroovyClassLoader`, compiled the
>    script in it, but **never closed** the old classloader. Because each classloader
>    held strong references to its loaded classes, those classes could never be garbage
>    collected from Metaspace — even though no application code referenced them anymore.
>    The classloader itself was kept alive by a reference in a `WeakHashMap` that
>    wasn't actually weak in the way the original developer expected (the values
>    held strong back-references to the keys).
>
> 6. **Implemented the fix**: I refactored the pricing engine in three steps:
>    - Replaced per-script `GroovyClassLoader` creation with a single cached
>      `GroovyShell` instance using a shared `CompilerConfiguration` and parent
>      classloader
>    - Added a bounded cache for compiled scripts using Caffeine with a maximum
>      size of 500 entries and a 1-hour expiry, with an eviction listener that
>      explicitly cleared classloader references
>    - Added a JMX metric (`loaded.class.count`) exposed via Micrometer to
>      monitor loaded class count in Grafana so we'd detect any future regression
>
> 7. **Validated thoroughly**: I wrote a targeted load test that simulated the batch
>    rule loading pattern — loading and reloading 200 pricing rules in a loop for
>    48 hours. Metaspace stayed flat at 180MB instead of growing unboundedly. I
>    deployed behind a feature flag, rolled out to one instance first, monitored
>    for a week, then rolled out to all instances.
>
> **Result**: The Metaspace leak was completely eliminated. The service ran for 90+
> days without restart after the fix — previously the longest run was 4 days. Loaded
> class count stabilized at ~8,000 instead of growing to 50,000+. Metaspace usage
> stabilized at 180MB instead of growing to the configured maximum. I presented the
> investigation at our engineering all-hands as a case study in systematic debugging,
> and we added Metaspace monitoring to all services as a standard. The incident also
> led us to establish a team rule: any dynamic code execution (Groovy, MVEL, SpEL
> expressions) must go through a reviewed, centralized utility class with proper
> classloader lifecycle management.

### Key Points to Emphasize
- You followed a systematic, repeatable debugging methodology — not trial and error
- You used specific JVM tools: `jcmd`, `jmap`, JFR, `-verbose:class`, heap dumps
- You correlated symptoms with environmental factors (the batch job timing)
- You formed a hypothesis before gathering data, then tested the hypothesis
- You didn't just fix the symptom — you found and fixed the root cause
- You understood the JVM internals deeply (classloader lifecycle, Metaspace GC)
- You put preventive measures in place (monitoring, coding standards)
- You shared knowledge with the broader team to prevent similar issues

### What NOT to Say
- "I just increased the memory and it worked" — that's treating symptoms, not debugging
- "I found it by adding print statements everywhere" — suggests no methodology
- "I Googled the error and found a Stack Overflow answer" — too shallow for senior roles
- Don't describe a bug that was trivially simple (typo, missing null check) unless
  you add significant complexity to the investigation story
- Avoid stories where you heroically stayed up all night — focus on process, not endurance
- "I just knew where to look" — interviewer wants to see the process, not intuition

---

## 🎯 Interviewer Insight

This question is one of the strongest signals for senior engineering capability.
Interviewers evaluate:

1. **Systematic Thinking** — Do you have a repeatable debugging process, or do you
   rely on intuition and luck? The best process: observe → hypothesize → gather data
   → analyze → fix → validate → prevent.
2. **Tool Proficiency** — Do you know the JVM diagnostic toolkit? Can you name
   specific tools and explain when to use each?
3. **Depth of Understanding** — Do you understand JVM internals (classloading, memory
   model, GC) well enough to reason about root causes?
4. **Completeness** — Do you stop at the fix, or do you also add monitoring, tests,
   and documentation to prevent recurrence?
5. **Communication** — Can you explain a complex technical investigation clearly
   and sequentially to someone who wasn't there?

### JVM Diagnostic Tool Reference

| Tool | Use Case | Example |
|------|----------|---------|
| `jstack` | Thread dumps — deadlocks, blocked threads | `jstack <pid> > threads.txt` |
| `jmap` | Heap dumps, class statistics | `jmap -dump:live,format=b,file=heap.hprof <pid>` |
| `jcmd` | JFR, GC info, VM flags | `jcmd <pid> JFR.start duration=60s` |
| `jstat` | GC statistics over time | `jstat -gcutil <pid> 1000` |
| JFR | Continuous profiling | Analyze with JDK Mission Control |
| async-profiler | CPU/allocation/lock profiling | `-e cpu` / `-e alloc` / `-e lock` |
| Eclipse MAT | Heap dump analysis | Dominator tree, leak suspects |
| VisualVM | Real-time JVM monitoring | Visual GC, thread monitoring |

## ⚡ Follow-up Questions

1. "How did you decide which diagnostic tool to use at each stage?"
2. "What would you have done if the issue only reproduced in production and never in staging?"
3. "How did you communicate progress and findings to stakeholders during the investigation?"
4. "What monitoring or alerting did you add to prevent similar issues?"
5. "How long did the investigation take, and how did you manage expectations?"

## 📚 Related Scenarios
- [Q4: Performance Troubleshooting](./q04-performance-troubleshooting.md) — performance
  issues often involve similar diagnostic techniques
- [Q9: Production Incident Response](./q09-production-incident-response.md) — debugging
  under production pressure with time constraints
- [Q1: Technical Leadership](./q01-technical-leadership.md) — turning a bug investigation
  into a systemic improvement

---

## 🔑 Additional Examples You Can Adapt

### Deadlock Investigation
- **Situation**: Service occasionally hung under load — no errors, no crashes, just stopped
  processing requests. Health checks passed (they only checked if the JVM was alive),
  so the load balancer kept routing traffic to the dead instance. It happened roughly
  once a week, always during peak hours.
- **Action**: Captured multiple thread dumps with `jstack <pid>` during a hang (3 dumps,
  5 seconds apart, to see if threads were stuck or progressing). Analyzed with
  fastThread.io. Found two threads deadlocked on `synchronized` blocks in a legacy
  billing module — Thread A held lock on `AccountLedger` monitor and waited for
  `TransactionLog` monitor, while Thread B held `TransactionLog` and waited for
  `AccountLedger`. The deadlock only manifested when two customers' payments were
  processed simultaneously for the same merchant, causing both threads to access
  the same pair of objects in opposite order.
- **Result**: Refactored to use `ReentrantLock` with `tryLock(timeout)` and consistent
  lock ordering (always lock by lower ID first). Added deadlock detection via
  `ThreadMXBean.findDeadlockedThreads()` in a health check endpoint so the load
  balancer would detect hung instances. Added a concurrent load test exercising the
  bidirectional locking scenario. Zero deadlocks in the following 6 months.

### Race Condition in Concurrent Cache
- **Situation**: Intermittent duplicate order processing — happened roughly once per
  10,000 orders (about once a day), only under high concurrency. Orders were being
  charged twice, causing customer complaints and manual refund work.
- **Action**: Reviewed the idempotency check logic. Found a classic check-then-act
  race condition: two threads could both read "not processed" from a `HashMap`
  before either wrote "processed." The `HashMap` itself was not thread-safe, meaning
  concurrent writes could corrupt the internal structure (infinite loops in bucket
  chains). The `putIfAbsent` method wasn't available on raw `HashMap`, and the
  `synchronized` block only covered the `get()` call, not the subsequent `put()`.
  Wrote a reproducer test with `CountDownLatch` to make 50 threads simultaneously
  attempt to process the same order ID.
- **Result**: Replaced with `ConcurrentHashMap.computeIfAbsent()` for atomic
  check-and-insert. The reproducer test verified the fix — 50 concurrent threads,
  exactly one processed, 49 rejected. Added the reproducer as a permanent
  integration test. Zero duplicate processing in the following quarter.

### ClassNotFoundException in Production Only
- **Situation**: Service worked perfectly in dev and staging but threw
  `ClassNotFoundException` for a custom Jackson deserializer in production. The
  exact same JAR was deployed to all environments (verified by checksum).
  The error was intermittent — it happened on about 30% of requests.
- **Action**: Compared classloader hierarchies across environments using
  `jcmd <pid> VM.classloaders` and a custom diagnostic servlet that logged the
  classloader chain. Discovered that production used a Java agent for APM
  monitoring (Elastic APM) that loaded its own Jackson version (2.13) via a
  parent-first classloader, shadowing the application's Jackson version (2.15).
  The 30% intermittency was because the APM agent's classloader won the race
  condition in parallel class loading — it depended on which classloader resolved
  first. The custom deserializer existed only in Jackson 2.15, hence the
  `ClassNotFoundException` when the agent's 2.13 was loaded first.
- **Result**: Configured the APM agent to use child-first classloading for
  application packages via the agent's `instrument_classloader_exclude` config.
  Documented the classloader interaction in the service's runbook. Added a
  startup health check that logged versions of critical libraries (Jackson,
  Spring, Hibernate) and failed fast if unexpected versions were detected.
  Shared the investigation as a team learning session on Java classloading.

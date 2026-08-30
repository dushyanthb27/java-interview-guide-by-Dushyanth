# Q6: Architectural Decision-Making

## 📋 Question
**"Describe an architectural decision you made that had significant impact on your system. How did you evaluate the trade-offs and align stakeholders?"**

## Difficulty: 🔴 Hard

---

## 💡 How to Answer

### The STAR Framework

- **Situation**: Describe the system's state — scale, pain points, and growth trajectory.
  Explain why the status quo was no longer viable. Include enough technical detail to
  show the complexity, but keep it understandable. Quantify: users, transactions per
  second, team size, deployment frequency, incident rate.
- **Task**: Define the decision that needed to be made and the constraints you were
  operating under (time, budget, team skills, legacy dependencies, regulatory
  requirements). Clarify the stakes — what would happen if you made the wrong choice?
  What would happen if you made no choice?
- **Action**: Walk through your decision-making process: how you identified options,
  evaluated trade-offs, built consensus, and managed risk. Show architectural maturity
  — great architects consider not just what's optimal today but what's maintainable
  for the team and sustainable for the organization long-term.
- **Result**: Describe both the technical outcome and the organizational impact.
  Include metrics. Be honest about any downsides, surprises, or things you'd do
  differently. Intellectual honesty about imperfect outcomes is a strong senior signal.

### Sample Answer

> **Situation**: I was a senior engineer on a fintech platform that processed loan
> applications. Our monolithic Spring Boot application had grown to 500K+ lines of
> code with 12 developers across three feature teams. Deployments took 45 minutes
> for the build alone, required full regression testing (another 2 hours), and had a
> 20% rollback rate because a change in one module often had unexpected effects on
> another. Feature development had slowed because teams were constantly dealing with
> merge conflicts in shared code, and a bug in one module (e.g., document processing)
> would delay releases for unrelated modules (e.g., credit scoring). Our business was
> scaling from 10K to 100K loan applications per month, and the monolith's single
> PostgreSQL database was becoming a bottleneck — query lock contention was causing
> intermittent timeouts during peak hours.
>
> **Task**: I was asked to propose an architecture that would allow independent team
> ownership and deployment while supporting 10x growth. The constraints were real and
> non-negotiable:
> - We couldn't stop feature development for more than a sprint — a full rewrite was
>   off the table because the business had quarterly revenue targets
> - The team had strong Spring Boot expertise but limited experience with distributed
>   systems (no one had operated Kafka, service meshes, or distributed tracing in
>   production)
> - We had regulatory requirements for data residency and audit trails that
>   constrained how we could partition data
> - Budget for new infrastructure was limited — we couldn't double our cloud spend
>
> **Action**: I evaluated three approaches systematically, documenting each in an
> Architecture Decision Record (ADR) template:
>
> 1. **Option A — Modular monolith**: Restructure the codebase into well-defined
>    modules with clear API boundaries using Java modules (JPMS) and enforced
>    dependency rules (ArchUnit). Keep a single deployable artifact, single database.
>    *Pros*: lowest risk, no operational complexity, team can execute immediately.
>    *Cons*: doesn't solve deployment coupling — all three teams still deploy together;
>    doesn't solve the database bottleneck.
>
> 2. **Option B — Full microservices**: Decompose into 8-10 independently deployable
>    services with separate databases (database-per-service pattern). Communication
>    via REST and async messaging.
>    *Pros*: maximum flexibility, independent scaling, independent deployment.
>    *Cons*: significant operational complexity (service discovery, distributed tracing,
>    saga patterns for distributed transactions) for a team with no distributed
>    systems experience; risk of "distributed monolith" if not executed well.
>
> 3. **Option C — Incremental decomposition (strangler fig)**: Start with a modular
>    monolith (Option A) to establish clean boundaries. Then extract the most painful
>    modules into independent services one at a time, starting with document processing
>    (highest change frequency, most merge conflicts, independent data model). Learn
>    distributed systems patterns on the first extraction, then apply lessons to
>    subsequent extractions.
>    *Pros*: manages risk incrementally; team learns as they go; delivers value early.
>    *Cons*: slower to reach the end state; temporary complexity of running both monolith
>    and services.
>
> I created a decision matrix scoring each option across six criteria, with weights
> agreed upon by the tech leads:
>
> | Criteria | Weight | Option A | Option B | Option C |
> |----------|--------|----------|----------|----------|
> | Dev velocity improvement | 25% | 5/10 | 9/10 | 7/10 |
> | Operational complexity | 20% | 9/10 | 3/10 | 6/10 |
> | Team skill match | 20% | 9/10 | 3/10 | 7/10 |
> | Migration risk | 15% | 9/10 | 4/10 | 7/10 |
> | Time to first value | 10% | 8/10 | 3/10 | 7/10 |
> | Reversibility | 10% | 9/10 | 4/10 | 7/10 |
> | **Weighted Score** | | **7.4** | **4.5** | **6.8** |
>
> I presented this at an architecture review with tech leads, the engineering manager,
> and the VP of Engineering. Option C won on the matrix, but the discussion surfaced
> important nuances that refined the approach.
>
> The key trade-off discussion was **synchronous REST vs asynchronous messaging**
> between the extracted services and the monolith. I advocated for different
> communication patterns depending on the use case:
>
> - **Document processing service → Kafka (async)**: Document processing was
>   naturally asynchronous (users upload documents → processing happens in the
>   background → notification sent when complete). Async communication decoupled
>   the services temporally — the monolith didn't need the document service to be
>   available to accept loan applications. It also provided a natural replay
>   mechanism for failed processing. The trade-off was eventual consistency, but
>   this was acceptable for document processing where results weren't needed in
>   real-time.
>
> - **Credit scoring service → REST with circuit breaker (sync)**: The loan
>   decision required the credit score immediately — eventual consistency was
>   unacceptable here because a loan officer was waiting for the result. I added
>   a Resilience4j circuit breaker with a fallback to a simplified scoring model
>   (based on cached data) so the application could still proceed if the scoring
>   service was temporarily unavailable, with a flag for manual review.
>
> For the data question, each extracted service got its own PostgreSQL schema
> initially (same database instance to limit cost) with a path to separate
> instances later. I implemented the **Outbox pattern** with Debezium for reliable
> event publishing — instead of dual-writing to both the database and Kafka
> (which risks inconsistency), the service writes an event record to an "outbox"
> table in the same database transaction, and Debezium captures the change and
> publishes it to Kafka.
>
> **Result**: Over six months, we extracted three services (document processing,
> credit scoring, notifications). Results:
> - Deployment frequency for the document processing team went from bi-weekly to
>   daily (10x improvement)
> - The overall rollback rate dropped from 20% to 4%
> - The monolith build time decreased from 45 to 18 minutes as code was removed
> - Database lock contention dropped significantly because document processing
>   queries moved to their own schema
> - The platform handled the 10x growth without architectural issues
>
> However, debugging cross-service issues was initially harder — we underestimated
> the observability investment needed. We had to invest two sprints in distributed
> tracing (Jaeger via OpenTelemetry) and centralized logging (ELK stack) that I
> hadn't prioritized enough upfront. If I did it again, I'd set up the
> observability stack **before** extracting the first service, not after.

### Key Points to Emphasize
- You evaluated multiple options rather than jumping to the trendy solution
- You used structured decision-making tools (decision matrix, ADR, weighted criteria)
- You considered the team's capabilities, not just the theoretical best architecture
- You made different communication choices (sync vs async) for different use cases, showing nuance
- You managed the transition incrementally to reduce risk
- You aligned stakeholders with a clear presentation of trade-offs, not just a recommendation
- You were honest about what you'd do differently — this shows maturity and self-awareness
- The outcome was measurable and tied to business impact

### What NOT to Say
- "We moved to microservices because that's the industry standard" — shows no
  independent thinking; architecture should serve your specific context
- "I made the decision and told the team what to do" — lacks stakeholder alignment
  and collaborative decision-making
- "We rewrote everything from scratch over six months" — extremely risky without
  strong justification and a track record of successful rewrites
- Don't present a decision without acknowledging trade-offs and downsides — every
  architecture has costs
- Avoid describing a decision where you never measured the outcome — if you can't
  prove it worked, did it?

---

## 🎯 Interviewer Insight

Architectural decision-making questions are the highest-signal questions for staff+
level candidates. Interviewers evaluate:

1. **Systems Thinking** — Can you reason about complex systems with many interacting
   components, constraints, and stakeholders?
2. **Trade-off Analysis** — Do you understand that every architectural choice has costs?
   Can you articulate them clearly and quantitatively?
3. **Pragmatism** — Do you choose the right solution for the context (team, timeline,
   scale) or the theoretically optimal one?
4. **Stakeholder Management** — Can you align engineering leadership, product, and
   your team on a technical direction?
5. **Long-term Thinking** — Do you consider maintainability, team growth, hiring,
   and future requirements?
6. **Intellectual Honesty** — Can you acknowledge mistakes and articulate what you'd
   change? This is the strongest signal of seniority.

A common failure mode is candidates who describe a decision that sounds impressive but
can't articulate the trade-offs, alternatives they considered, or what went wrong.

## ⚡ Follow-up Questions

1. "What was the strongest argument against your chosen approach, and how did you address it?"
2. "How did you handle data consistency between the monolith and the extracted services?"
3. "What would have happened if the team had chosen Option A (modular monolith) instead?"
4. "How did you get buy-in from team members who were skeptical about microservices?"
5. "What was the most unexpected challenge you faced during the transition?"
6. "How do you decide when a monolith is 'big enough' to justify decomposition?"

## 📚 Related Scenarios
- [Q1: Technical Leadership](./q01-technical-leadership.md) — driving architectural
  changes requires leadership skills
- [Q7: Handling Technical Debt](./q07-handling-technical-debt.md) — architectural
  evolution often addresses systemic debt
- [Q10: Cross-Team Communication](./q10-cross-team-communication.md) — architectural
  decisions affect multiple teams and require clear communication

---

## 🔑 Additional Examples You Can Adapt

### ORM vs Raw SQL Decision
- **Situation**: Building a high-throughput reporting service that needed to join
  across 8 tables with complex aggregations, window functions, and conditional
  pivots. Hibernate generated inefficient queries with multiple round trips and
  couldn't express the required window functions. The queries needed to process
  10M+ rows nightly for financial compliance reports.
- **Action**: Evaluated three approaches with production-scale data (10M rows):
  1. Hibernate with `@Query` and native SQL — worked but lost type safety and
     compile-time checking; maintenance burden of raw SQL strings in annotations
  2. Spring JDBC Template with hand-crafted queries — maximum performance control
     but verbose boilerplate for mapping results to objects; error-prone for
     complex queries
  3. jOOQ for type-safe SQL generation — provided compile-time SQL validation,
     IDE autocomplete for table/column names, and generated queries as efficient
     as hand-written SQL

  Benchmarked all three with the most complex report query: Hibernate 4.2s,
  JDBC Template 1.1s, jOOQ 1.1s (same performance as raw SQL, but type-safe).
- **Result**: Chose jOOQ — it provided type safety and composability (like an ORM)
  with the performance of raw SQL. Documented the decision in an ADR, including
  when Hibernate was still the right choice (CRUD-heavy services with simple
  queries and standard entity lifecycle management).

### Choosing a Concurrency Model
- **Situation**: Building a data aggregation service that called 6 downstream APIs
  to compose a single customer profile response. Sequential calls took 3.2 seconds
  (each API averaged 400-600ms). The SLA was 500ms.
- **Action**: Evaluated three concurrency approaches:
  1. `CompletableFuture` with a shared `ForkJoinPool` — mature, well-understood,
     but required careful thread pool sizing and error handling was verbose
  2. Spring WebFlux with Project Reactor — non-blocking I/O, excellent throughput,
     but reactive programming has a steep learning curve and debugging stack
     traces are difficult to read
  3. Java 21 virtual threads — concurrent I/O without reactive complexity;
     existing synchronous code works as-is; standard stack traces for debugging

  Considered team experience (strong imperative Java, limited reactive), latency
  requirements (sub-500ms), debugging complexity, and hiring pool.
- **Result**: Chose virtual threads — they provided concurrent I/O without the
  reactive programming learning curve. Existing synchronous HTTP client code worked
  unchanged — just ran on virtual threads instead of platform threads. Response
  time dropped from 3.2s to 480ms (limited by the slowest downstream API).
  Debugging remained straightforward with standard stack traces.

### Event-Driven vs Request-Response
- **Situation**: Order service needed to notify inventory, shipping, billing, and
  notification services after order placement. Initially done via synchronous REST
  calls in sequence: `placeOrder() → deductInventory() → chargeBilling() →
  createShipment() → sendNotification()`. If any downstream service was slow or
  down, order placement failed entirely. Average order placement time was 2.5s,
  with a 5% failure rate due to downstream issues.
- **Action**: Documented the fragility and proposed event-driven architecture:
  order service publishes an `OrderPlaced` event to Kafka; downstream services
  subscribe and process independently. Addressed concerns about eventual
  consistency by designing a state machine for order status
  (`PLACED → INVENTORY_RESERVED → BILLED → SHIPPED → NOTIFIED`) and implementing
  compensation events for rollback scenarios (`InventoryReservationFailed` →
  automatic order cancellation).
- **Result**: Order placement latency dropped from 2.5s to 200ms (just database
  write + Kafka publish). Services were independently deployable and scalable. A
  downstream outage no longer affected order placement — events queued in Kafka
  and were processed when the service recovered. Added a dead letter queue for
  failed event processing with alerting and a retry mechanism.

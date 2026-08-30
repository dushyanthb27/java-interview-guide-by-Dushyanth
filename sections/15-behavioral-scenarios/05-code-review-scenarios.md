# Q5: Code Review Scenarios

## 📋 Question
**"How do you approach code reviews? Describe a time when you provided feedback that significantly improved the code or caught a critical issue."**

## Difficulty: 🟡 Medium

---

## 💡 How to Answer

### The STAR Framework

- **Situation**: Describe the context — what kind of project, what was the PR about,
  and what was the relationship with the author (peer, junior, cross-team)? Set the
  stage for why thoughtful review mattered in this particular case.
- **Task**: Explain what you were reviewing for — correctness, performance, security,
  readability, maintainability? What was your responsibility as a reviewer? What was
  at stake if the review was superficial?
- **Action**: Describe your review process and the specific feedback you gave. Show
  how you balanced being thorough with being constructive and respectful. If there
  was disagreement, explain how you navigated it.
- **Result**: What happened after your review? Was a bug prevented? Was the design
  improved? Did the author grow from the feedback? Did the team adopt new practices?

### Sample Answer

> **Situation**: A mid-level developer on my team submitted a pull request for a new
> payment reconciliation service. The PR was about 400 lines across four files — a
> new `ReconciliationService` class, a `ReconciliationScheduler` that triggered it
> via `@Scheduled`, a `ReconciliationRepository` for database access, and associated
> unit tests. The service would run nightly at 2 AM, compare payment gateway records
> with our database, and flag discrepancies for manual review. It was a critical
> financial component — reconciliation errors could mean undetected revenue leakage
> or incorrect financial reporting.
>
> **Task**: As a senior engineer and the designated reviewer for the payments module,
> I needed to evaluate correctness (would it find real discrepancies?), thread safety
> (the scheduled job could overlap with API requests to the same data), error handling
> (financial data demands robustness — a silent failure here could mean millions in
> undetected discrepancies), and overall design quality. I also wanted to ensure the
> code was maintainable for the next person who would touch it.
>
> **Action**: I followed my standard multi-pass review process:
>
> 1. **Understand the context first (10 minutes)**: Before looking at any code, I
>    read the JIRA ticket, the linked design document, and the acceptance criteria.
>    I understood the business requirements, the expected data volumes (up to 50K
>    transactions per nightly run), and the edge cases (partial refunds, currency
>    conversions, timezone differences) before evaluating the implementation. This
>    context let me assess whether the code solved the right problem, not just
>    whether it was syntactically correct.
>
> 2. **High-level design review (5 minutes)**: I checked class responsibilities,
>    naming, and architecture before diving into line-level details. The overall
>    structure was solid — good separation between scheduling (`ReconciliationScheduler`),
>    business logic (`ReconciliationService`), and data access
>    (`ReconciliationRepository`). The naming was clear and consistent. This
>    deserved positive feedback, and I made sure to say so.
>
> 3. **Deep dive on critical paths (30 minutes)**: I focused my detailed review on
>    the code paths that handled money and data integrity. I found three significant
>    issues:
>
>    - **Thread safety (severity: high)**: The service used a `SimpleDateFormat`
>      instance variable for timestamp parsing when comparing payment gateway
>      timestamps with our database timestamps. `SimpleDateFormat` is not thread-safe
>      — if the scheduled job at 2 AM and an API-triggered manual reconciliation
>      ran simultaneously, the date parsing could produce **silently corrupt results**
>      (garbled dates, not exceptions). I wrote in my review:
>
>      > "⚠️ Thread safety concern: `SimpleDateFormat` is not thread-safe
>      > ([Javadoc link]). If `runReconciliation()` is called concurrently
>      > (scheduled + manual trigger), this can produce corrupt date comparisons
>      > without throwing an exception. Consider switching to `DateTimeFormatter`
>      > from `java.time`, which is immutable and thread-safe:
>      > ```java
>      > private static final DateTimeFormatter FORMATTER =
>      >     DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm:ss");
>      > ```"
>
>    - **Exception handling (severity: high)**: The reconciliation loop caught
>      `Exception` broadly and logged it, but continued processing:
>      ```java
>      for (Transaction tx : transactions) {
>          try {
>              reconcile(tx);
>          } catch (Exception e) {
>              log.error("Error reconciling: {}", tx.getId(), e);
>          }
>      }
>      ```
>      The problem: if a `DataAccessException` occurred mid-batch (database
>      connectivity lost), the service would log the error but continue
>      iterating — marking remaining records as "reconciled" even though it
>      couldn't actually read them from the database. The final report would
>      show "all clear" when in reality the database was unreachable.
>
>      I recommended distinguishing between recoverable errors (single record
>      parse failure — log and continue) and unrecoverable errors (database
>      down — abort and alert). I suggested:
>      ```java
>      try {
>          reconcile(tx);
>      } catch (DataIntegrityException e) {
>          // Single record issue - log and continue
>          log.warn("Skipping transaction {}: {}", tx.getId(), e.getMessage());
>          discrepancies.add(tx);
>      } catch (DataAccessException e) {
>          // Infrastructure failure - abort the batch
>          log.error("Database error during reconciliation - aborting", e);
>          alertService.sendCriticalAlert("Reconciliation aborted", e);
>          throw e;
>      }
>      ```
>
>    - **Resource leak risk (severity: medium)**: The service opened a
>      `BufferedReader` to parse a CSV file downloaded from the payment gateway
>      but didn't use try-with-resources. If parsing threw an exception on line
>      500 of a 10,000-line file, the file handle would leak. Over time, this
>      could exhaust the OS file descriptor limit.
>
> 4. **Delivered feedback constructively**: For each issue, I explained the *why* —
>    not just "this is wrong" but "here's the specific scenario where this fails
>    and the business consequence." I included code snippets showing the suggested
>    fix so the author didn't have to figure out the solution from scratch. For the
>    thread safety issue, I linked to the `SimpleDateFormat` Javadoc where it
>    explicitly states the class is not synchronized.
>
>    I also called out what was done well in a separate "Praise" comment:
>    - Clean separation of concerns between scheduling, logic, and data access
>    - Comprehensive happy-path tests with clear assertions
>    - Good variable naming that made the code self-documenting
>    - Thoughtful use of `BigDecimal` for monetary calculations (avoiding float)
>
> **Result**: The author was grateful — especially about the thread safety issue,
> which was a subtle bug that wouldn't have been caught by unit tests (since they
> run single-threaded) and might not manifest for months until someone happened to
> trigger a manual reconciliation during the nightly batch. The exception handling
> change prevented a potential data integrity issue that could have affected
> financial reporting to regulators. The developer later told me it was the most
> useful code review they'd received because they understood the reasoning behind
> each comment, not just the correction. They subsequently applied the same patterns
> (thread-safe date handling, discriminated error handling, try-with-resources) in
> their next three PRs without being asked — proof that the review was educational,
> not just correctional.

### Key Points to Emphasize
- You have a structured, multi-pass review process: context → design → critical paths → details
- You explain the "why" behind feedback, not just the "what"
- You balance critique with genuine positive feedback
- You focus on the most impactful issues (thread safety, correctness, resource management)
  rather than style nitpicks
- You frame feedback as collaborative ("consider") rather than authoritative ("you must")
- You provide concrete code examples with your suggestions
- You care about the author's growth, not just the code quality of this one PR

### What NOT to Say
- "I mostly check for formatting and naming" — too superficial for senior roles
- "I approve PRs quickly so I don't block the team" — suggests carelessness
- "I pointed out 50 issues on the PR" — suggests nitpicking or poor earlier communication
  about standards
- Don't describe a review where you were condescending or created conflict
- Avoid framing it as catching someone's "mistake" — frame it as improving the code together
- "I only review code from junior developers" — reviews are valuable at all levels

---

## 🎯 Interviewer Insight

Code review skills reveal multiple dimensions of senior engineering:

1. **Technical Depth** — Can you spot subtle issues like thread safety, resource leaks,
   and error handling gaps that automated tools (linters, SonarQube) miss?
2. **Communication** — Can you give feedback that is specific, actionable, kind, and
   educational? Do you explain reasoning?
3. **Prioritization** — Do you focus on what matters (correctness, security, performance)
   or get lost in style nitpicks that could be handled by formatters?
4. **Mentorship** — Does your review help the author grow, or just fix this one PR?
5. **Process** — Do you have a repeatable approach, or do you wing it?
6. **Handling Pushback** — How do you respond when the author disagrees with your feedback?

### Code Review Priority Hierarchy

| Priority | Category | Examples |
|----------|----------|---------|
| 🔴 Critical | Correctness bugs | Race conditions, data corruption, logic errors |
| 🔴 Critical | Security issues | SQL injection, IDOR, credential exposure |
| 🟠 Important | Error handling | Silent failures, missing edge cases |
| 🟠 Important | Performance | N+1 queries, unnecessary allocations in hot paths |
| 🟡 Moderate | Design | Coupling, single responsibility, naming clarity |
| 🟢 Minor | Style | Formatting, import ordering, comment style |

Focus your review time and attention in this order. Style issues should be
handled by automated formatters, not human reviewers.

## ⚡ Follow-up Questions

1. "How do you handle it when the PR author pushes back on your feedback?"
2. "How do you balance thoroughness in reviews with not being a bottleneck?"
3. "What do you look for first when reviewing a large PR (500+ lines)?"
4. "How do you review code in a technology or domain you're less familiar with?"
5. "How do you give feedback differently to a senior developer versus a junior developer?"
6. "What's the most common mistake you see in code reviews?"

## 📚 Related Scenarios
- [Q2: Team Collaboration](./q02-team-collaboration.md) — navigating disagreements
  that arise from reviews
- [Q8: Mentoring Juniors](./q08-mentoring-juniors.md) — code reviews as a primary
  mentorship channel
- [Q7: Handling Technical Debt](./q07-handling-technical-debt.md) — using reviews
  to prevent new debt from entering the codebase

---

## 🔑 Additional Examples You Can Adapt

### Catching a Security Issue in API Design
- **Situation**: A PR exposed a new REST endpoint `GET /api/users/{userId}/account`
  that accepted a user ID as a path parameter and returned full account details
  including balance, transaction history, and linked bank accounts. The endpoint
  had `@PreAuthorize("isAuthenticated()")` — it checked that the caller was
  logged in, but didn't verify they were requesting their own data. Any
  authenticated user could view any other user's financial information.
- **Action**: Identified this as an Insecure Direct Object Reference (IDOR)
  vulnerability — OWASP Top 10 #1 (Broken Access Control). Wrote a review
  comment explaining the attack scenario with a concrete example: "User A
  (authenticated) could call `GET /api/users/456/account` and see User B's
  bank account details." Suggested adding:
  ```java
  @PreAuthorize("isAuthenticated() and #userId == authentication.principal.id")
  ```
  Also recommended creating a reusable `@OwnerOnly` annotation for this pattern.
  Linked to OWASP documentation on IDOR prevention.
- **Result**: Prevented a potential data breach that would have exposed customer
  financial data. The team added an authorization check review as a mandatory
  item in the PR template. Created the `@OwnerOnly` annotation that was reused
  across 12 endpoints.

### Improving Exception Handling Design
- **Situation**: PR had a service method that caught all exceptions and returned
  `null`, forcing every caller to null-check and guess what went wrong:
  ```java
  public Order getOrder(Long id) {
      try {
          return orderRepository.findById(id).orElse(null);
      } catch (Exception e) {
          log.error("Error", e);
          return null;
      }
  }
  ```
  Callers had no way to distinguish between "order not found" (expected),
  "database is down" (infrastructure failure), and "invalid data" (bug).
- **Action**: Suggested using a custom exception hierarchy that communicates intent:
  `OrderNotFoundException extends BusinessException` for expected "not found" cases,
  `DataAccessException` propagating naturally for infrastructure failures, and
  letting unexpected exceptions reach the global `@ControllerAdvice` handler.
  Provided code examples showing how this simplified caller code from 8 lines
  of null-checking to 1 line of direct usage.
- **Result**: Author refactored the entire service layer to use this pattern.
  Null-return methods were eliminated. API error responses became consistent
  and informative (proper HTTP status codes: 404 for not found, 500 for
  infrastructure failures). Error monitoring improved because exceptions
  carried meaningful context.

### Reviewing for Performance in a Hot Path
- **Situation**: PR added logging and string formatting inside a loop that processed
  100K+ records for a daily report generation job. The code used string concatenation
  with `+` operator inside the loop to build a summary:
  ```java
  String summary = "";
  for (Record record : records) {
      summary = summary + record.getId() + ": " + record.getStatus() + "\n";
  }
  ```
- **Action**: Explained that `+` in a loop creates a new `StringBuilder` and a new
  `String` object on each iteration. For 100K iterations, this means 100K temporary
  StringBuilder objects and 100K intermediate String objects, with each intermediate
  string being larger than the last (O(n²) total memory). Shared a JMH benchmark
  showing the difference at 100K iterations: string concatenation took 12 seconds
  and allocated 4.8GB of garbage, while `StringBuilder` took 15 milliseconds and
  allocated 2MB. Suggested:
  ```java
  StringBuilder summary = new StringBuilder(records.size() * 40);
  for (Record record : records) {
      summary.append(record.getId()).append(": ")
             .append(record.getStatus()).append('\n');
  }
  ```
- **Result**: Author made the change and started proactively considering performance
  in loops. The team added an ArchUnit rule to flag string concatenation in loops
  during CI builds.

### Handling Pushback Gracefully
- **Situation**: You suggested restructuring a 300-line method into smaller methods
  for readability and testability. The author argued that a single method was easier
  to follow top-to-bottom and that extracting methods just scattered the logic
  across multiple places. They had a reasonable point — excessive extraction can
  hurt readability too.
- **Action**: Acknowledged their perspective genuinely: "You're right that extracting
  too many tiny methods can make code harder to follow — I've seen that anti-pattern
  too." Offered a focused compromise — extract only the two most complex and
  independent blocks (40-line validation logic and 35-line retry logic) into
  well-named private methods, keeping the main flow readable as a sequential
  narrative. Pointed out the testing benefit: the validation logic had three
  edge cases that were currently untestable because they were buried inside
  a 300-line method.
- **Result**: Author agreed to the compromise. The extracted validation method
  immediately caught a boundary condition bug when it was unit-tested in isolation —
  a bug that the existing integration test hadn't covered. The author came back
  in the next PR and proactively extracted a similar block, having seen the benefit
  firsthand.

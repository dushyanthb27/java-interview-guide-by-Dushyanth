# Q7: What's New in the Date/Time API?

## 📋 Question
What are the key classes in the Java 8 Date/Time API? How do they compare to the old Date/Calendar API, and how do you handle time zones, formatting, and temporal adjustments?

## Difficulty: 🟡 Medium

---

## 💡 Answer

The `java.time` package (Java 8, JSR-310) provides an **immutable, thread-safe, and fluent** date/time API that replaces the deeply flawed `java.util.Date` and `java.util.Calendar`. Inspired by Joda-Time, it cleanly separates concepts like dates without times, times without dates, instants on the timeline, and durations/periods.

### Core Classes Overview

| Class | Represents | Example | Time Zone? |
|-------|-----------|---------|------------|
| `LocalDate` | Date only | `2026-08-30` | No |
| `LocalTime` | Time only | `14:30:45` | No |
| `LocalDateTime` | Date + time | `2026-08-30T14:30:45` | No |
| `ZonedDateTime` | Date + time + zone | `2026-08-30T14:30:45+05:30[Asia/Kolkata]` | Yes |
| `Instant` | Machine timestamp | `1725018645` (epoch seconds) | UTC |
| `Duration` | Time-based amount | `PT2H30M` (2h 30m) | — |
| `Period` | Date-based amount | `P1Y2M3D` (1y 2m 3d) | — |
| `OffsetDateTime` | Date + time + offset | `2026-08-30T14:30:45+05:30` | Offset only |

### LocalDate, LocalTime, LocalDateTime

```java
package com.interview.java8;

import java.time.*;

public class LocalDateTimeDemo {

    public static void main(String[] args) {

        // ═══════════════════════════════════════════════════
        // LocalDate — date without time or zone
        // ═══════════════════════════════════════════════════
        LocalDate today = LocalDate.now();
        LocalDate birthday = LocalDate.of(1990, Month.JUNE, 15);
        LocalDate parsed = LocalDate.parse("2026-08-30");

        System.out.println("Today: " + today);
        System.out.println("Day of week: " + today.getDayOfWeek());
        System.out.println("Day of year: " + today.getDayOfYear());
        System.out.println("Is leap year: " + today.isLeapYear());

        // ✅ Immutable — every operation returns a NEW instance
        LocalDate nextWeek = today.plusWeeks(1);
        LocalDate lastMonth = today.minusMonths(1);
        // today is unchanged!

        // ═══════════════════════════════════════════════════
        // LocalTime — time without date or zone
        // ═══════════════════════════════════════════════════
        LocalTime now = LocalTime.now();
        LocalTime lunch = LocalTime.of(12, 30);
        LocalTime precise = LocalTime.of(14, 30, 45, 123_000_000); // nano precision

        LocalTime meetingEnd = lunch.plusHours(1).plusMinutes(30);
        System.out.println("Meeting ends: " + meetingEnd);  // 14:00

        // ═══════════════════════════════════════════════════
        // LocalDateTime — date + time, no zone
        // ═══════════════════════════════════════════════════
        LocalDateTime dateTime = LocalDateTime.now();
        LocalDateTime specific = LocalDateTime.of(2026, 8, 30, 14, 30);
        LocalDateTime combined = LocalDate.now().atTime(LocalTime.NOON);

        System.out.println("DateTime: " + dateTime);
    }
}
```

### ZonedDateTime, Instant, and Time Zones

```java
package com.interview.java8;

import java.time.*;
import java.util.Set;

public class TimeZoneDemo {

    public static void main(String[] args) {

        // ═══════════════════════════════════════════════════
        // ZonedDateTime — full date-time with time zone
        // ═══════════════════════════════════════════════════
        ZonedDateTime nowIndia = ZonedDateTime.now(ZoneId.of("Asia/Kolkata"));
        ZonedDateTime nowNY = ZonedDateTime.now(ZoneId.of("America/New_York"));
        ZonedDateTime nowUTC = ZonedDateTime.now(ZoneOffset.UTC);

        System.out.println("India: " + nowIndia);
        System.out.println("New York: " + nowNY);
        System.out.println("UTC: " + nowUTC);

        // ✅ Convert between zones
        ZonedDateTime indiaToNY = nowIndia.withZoneSameInstant(ZoneId.of("America/New_York"));
        System.out.println("India time in NY: " + indiaToNY);

        // ═══════════════════════════════════════════════════
        // Instant — machine-readable timestamp (epoch-based)
        // ═══════════════════════════════════════════════════
        Instant now = Instant.now();
        Instant epoch = Instant.EPOCH;  // 1970-01-01T00:00:00Z
        Instant fromEpoch = Instant.ofEpochSecond(1_000_000_000L);

        System.out.println("Now: " + now);
        System.out.println("Epoch seconds: " + now.getEpochSecond());

        // ✅ Convert Instant to ZonedDateTime
        ZonedDateTime zoned = now.atZone(ZoneId.of("Asia/Kolkata"));

        // ✅ Convert LocalDateTime to Instant (requires zone)
        LocalDateTime local = LocalDateTime.of(2026, 8, 30, 14, 30);
        Instant instant = local.toInstant(ZoneOffset.of("+05:30"));

        // ═══════════════════════════════════════════════════
        // Clock — injectable time source for testing
        // ═══════════════════════════════════════════════════
        Clock fixed = Clock.fixed(Instant.parse("2026-01-01T00:00:00Z"), ZoneOffset.UTC);
        LocalDate fixedDate = LocalDate.now(fixed);  // Always 2026-01-01
        System.out.println("Fixed date: " + fixedDate);

        // Available time zones
        Set<String> zones = ZoneId.getAvailableZoneIds();
        System.out.println("Total zones: " + zones.size());  // ~600
    }
}
```

### Duration and Period

```java
package com.interview.java8;

import java.time.*;

public class DurationPeriodDemo {

    public static void main(String[] args) {

        // ═══════════════════════════════════════════════════
        // Duration — time-based (hours, minutes, seconds, nanos)
        // ═══════════════════════════════════════════════════
        Duration twoHours = Duration.ofHours(2);
        Duration thirtyMinutes = Duration.ofMinutes(30);
        Duration combined = twoHours.plus(thirtyMinutes);  // PT2H30M

        Duration between = Duration.between(
                LocalTime.of(9, 0), LocalTime.of(17, 30));
        System.out.println("Work hours: " + between);     // PT8H30M
        System.out.println("In minutes: " + between.toMinutes());  // 510

        // ═══════════════════════════════════════════════════
        // Period — date-based (years, months, days)
        // ═══════════════════════════════════════════════════
        Period sixMonths = Period.ofMonths(6);
        Period oneYearThreeMonths = Period.of(1, 3, 0);

        LocalDate start = LocalDate.of(2024, 1, 1);
        LocalDate end = LocalDate.of(2026, 8, 30);
        Period age = Period.between(start, end);
        System.out.println("Period: " + age);  // P2Y7M29D
        System.out.println("Years: " + age.getYears() + ", Months: " + age.getMonths());

        // ✅ Adding period to a date
        LocalDate future = LocalDate.now().plus(Period.ofMonths(3));
        System.out.println("3 months from now: " + future);
    }
}
```

### DateTimeFormatter and Temporal Adjusters

```java
package com.interview.java8;

import java.time.*;
import java.time.format.*;
import java.time.temporal.*;

public class FormatterAndAdjusterDemo {

    public static void main(String[] args) {

        // ═══════════════════════════════════════════════════
        // DateTimeFormatter — formatting and parsing
        // ═══════════════════════════════════════════════════
        LocalDateTime now = LocalDateTime.now();

        // Built-in formatters
        String iso = now.format(DateTimeFormatter.ISO_LOCAL_DATE_TIME);

        // Custom patterns
        DateTimeFormatter custom = DateTimeFormatter.ofPattern("dd-MMM-yyyy HH:mm:ss");
        String formatted = now.format(custom);
        System.out.println("Formatted: " + formatted);  // 30-Aug-2026 14:30:45

        // Parsing with custom format
        LocalDate parsed = LocalDate.parse("30-Aug-2026",
                DateTimeFormatter.ofPattern("dd-MMM-yyyy"));
        System.out.println("Parsed: " + parsed);

        // ✅ Thread-safe — can be shared as static final (unlike SimpleDateFormat)
        // static final DateTimeFormatter FORMATTER = DateTimeFormatter.ofPattern("...");

        // ═══════════════════════════════════════════════════
        // TemporalAdjusters — complex date calculations
        // ═══════════════════════════════════════════════════
        LocalDate today = LocalDate.now();

        LocalDate firstDay = today.with(TemporalAdjusters.firstDayOfMonth());
        LocalDate lastDay = today.with(TemporalAdjusters.lastDayOfMonth());
        LocalDate nextMonday = today.with(TemporalAdjusters.next(DayOfWeek.MONDAY));
        LocalDate firstDayNextYear = today.with(TemporalAdjusters.firstDayOfNextYear());

        System.out.println("First of month: " + firstDay);
        System.out.println("Last of month: " + lastDay);
        System.out.println("Next Monday: " + nextMonday);

        // ✅ Custom temporal adjuster
        TemporalAdjuster nextWorkday = temporal -> {
            LocalDate date = LocalDate.from(temporal);
            DayOfWeek dow = date.getDayOfWeek();
            if (dow == DayOfWeek.FRIDAY) return date.plusDays(3);
            if (dow == DayOfWeek.SATURDAY) return date.plusDays(2);
            return date.plusDays(1);
        };
        System.out.println("Next workday: " + today.with(nextWorkday));
    }
}
```

### Old API vs New API Comparison

| Feature | Old (`Date`/`Calendar`) | New (`java.time`) |
|---------|------------------------|-------------------|
| **Mutability** | Mutable (thread-unsafe) | Immutable (thread-safe) |
| **Thread Safety** | `SimpleDateFormat` not thread-safe | `DateTimeFormatter` is thread-safe |
| **Month indexing** | 0-based (January = 0) | 1-based (January = 1) |
| **API clarity** | Confusing (`Date` has time, too) | Clean separation of concerns |
| **Time zones** | Weak `TimeZone` class | Comprehensive `ZoneId` |
| **Null safety** | Returns null on errors | Throws descriptive exceptions |
| **Design** | Procedural | Fluent, functional |

```java
// ❌ Old API: Mutable, confusing, error-prone
Date oldDate = new Date();               // Includes time — misleading
Calendar cal = Calendar.getInstance();
cal.set(Calendar.MONTH, 0);              // 0 = January — easy to get wrong

// ✅ New API: Immutable, clear, type-safe
LocalDate newDate = LocalDate.now();
LocalDate jan = LocalDate.of(2026, Month.JANUARY, 1);  // 1 = January
```

### ⚠️ Common Mistake
**Using `LocalDateTime` to store timestamps in databases**: `LocalDateTime` has **no time zone information** — it represents a "wall clock" reading. If your server is in UTC and a user is in IST, `LocalDateTime.now()` gives different values on each machine. For timestamps (event occurred at), use `Instant` or `ZonedDateTime`. For human-readable dates (birthday, appointment), use `LocalDate` or `LocalDateTime`. Mixing these up causes bugs when servers are in different time zones.

### ⚡ Performance Tip
`DateTimeFormatter` is **immutable and thread-safe** — create it once as a `static final` field and reuse it. Unlike the old `SimpleDateFormat` (which required `ThreadLocal` or synchronization), `DateTimeFormatter` can safely be shared across all threads. This eliminates both the performance overhead of creating new formatters and the risk of thread-safety bugs.

### 🔧 JVM Internals
The `java.time` classes store dates as **epoch-day counts** (`LocalDate` stores a single `long` for days since epoch) and times as **nanosecond-of-day** (`LocalTime` stores two fields: second-of-day and nano-of-second). This representation is compact and makes arithmetic operations (addition, subtraction, comparison) extremely fast — they reduce to simple long arithmetic. `Instant` stores epoch-seconds + nano adjustment, making it directly convertible to/from `System.currentTimeMillis()` and database timestamps.

---

## 🎯 Interviewer Insight
Interviewers test whether you know **which class to use for which scenario**: `LocalDate` for birthdays, `Instant` for event timestamps, `ZonedDateTime` for scheduling across time zones. Knowing the problems with the old API (mutability, 0-based months, thread-unsafe formatting) and how the new API solves them is essential. Mentioning `Clock` injection for testability shows production experience.

## ⚡ Follow-up Questions
1. How do you convert between `Date` (old) and `Instant` (new)?
2. What happens to `ZonedDateTime` during daylight saving time transitions?
3. Why should you use `Instant` instead of `System.currentTimeMillis()`?

## 📚 Related Topics
- [Lambda Expressions](./q01-lambda-expressions.md)
- [Optional API](./q06-optional-api.md)
- [Records](./q14-records-detail.md)

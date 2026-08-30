# Q10: Private Methods in Interfaces (Java 9)

## 📋 Question
What are private methods in interfaces? How do they enable code reuse between default methods, and what are the rules for private static methods in interfaces?

## Difficulty: 🟡 Medium

---

## 💡 Answer

**Private interface methods** (Java 9) allow interfaces to define **private helper methods** that can be called by default methods and other private methods within the same interface. They solve the problem of **code duplication** between default methods — before Java 9, if two default methods shared common logic, you had to either duplicate the code or expose a default helper method (which pollutes the public API).

### Private Methods — Reducing Duplication

```java
package com.interview.java9;

import java.util.*;
import java.util.stream.*;

public class PrivateInterfaceMethodDemo {

    // ❌ BEFORE Java 9: Shared logic exposed as default (leaks implementation)
    interface LoggerBefore {
        default void logInfo(String msg) {
            // Duplicated formatting logic
            System.out.println("[INFO] " + java.time.LocalDateTime.now() + " - " + msg);
        }

        default void logError(String msg) {
            // Same formatting logic duplicated!
            System.out.println("[ERROR] " + java.time.LocalDateTime.now() + " - " + msg);
        }
    }

    // ✅ AFTER Java 9: Shared logic hidden in private method
    interface Logger {
        default void logInfo(String msg) {
            log("INFO", msg);
        }

        default void logError(String msg) {
            log("ERROR", msg);
        }

        default void logWarn(String msg) {
            log("WARN", msg);
        }

        // ✅ Private method — NOT visible to implementing classes
        private void log(String level, String msg) {
            System.out.println("[" + level + "] " + java.time.LocalDateTime.now()
                    + " - " + msg);
        }
    }

    static class UserService implements Logger {
        public void createUser(String name) {
            logInfo("Creating user: " + name);
            // log("DEBUG", "test");  // ❌ Compile error — private, not accessible
        }
    }

    public static void main(String[] args) {
        UserService service = new UserService();
        service.createUser("Alice");
        service.logError("Something went wrong");
        service.logWarn("Disk space low");
    }
}
```

### Private Static Methods

```java
package com.interview.java9;

import java.util.List;
import java.util.stream.Collectors;

public class PrivateStaticMethodDemo {

    interface DataProcessor {
        // ✅ Static method — can call private static methods
        static List<String> processNames(List<String> names) {
            return names.stream()
                    .map(DataProcessor::normalize)   // Calls private static
                    .filter(DataProcessor::isValid)   // Calls private static
                    .collect(Collectors.toList());
        }

        static List<String> processEmails(List<String> emails) {
            return emails.stream()
                    .map(DataProcessor::normalize)   // Reuses same private static
                    .filter(e -> e.contains("@"))
                    .collect(Collectors.toList());
        }

        // ✅ Private static — shared by static methods
        private static String normalize(String input) {
            return input == null ? "" : input.trim().toLowerCase();
        }

        // ✅ Private static — helper for validation
        private static boolean isValid(String input) {
            return input != null && !input.isEmpty() && input.length() <= 100;
        }

        // Default methods can ALSO call private static methods
        default void process(String data) {
            String normalized = normalize(data);  // ✅ Accessible
            System.out.println("Processed: " + normalized);
        }
    }

    public static void main(String[] args) {
        List<String> names = List.of("  Alice  ", "BOB", "", "  Charlie  ");
        List<String> processed = DataProcessor.processNames(names);
        System.out.println(processed);  // [alice, bob, charlie]
    }
}
```

### Rules for Private Interface Methods

| Rule | Private Instance | Private Static |
|------|-----------------|----------------|
| **Can call default methods** | ✅ Yes | ❌ No |
| **Can call abstract methods** | ✅ Yes | ❌ No |
| **Can call static methods** | ✅ Yes | ✅ Yes |
| **Can call other private methods** | ✅ Yes | Only private static |
| **Called by default methods** | ✅ Yes | ✅ Yes |
| **Called by static methods** | ❌ No | ✅ Yes |
| **Inherited by subinterfaces** | ❌ No | ❌ No |
| **Visible to implementing classes** | ❌ No | ❌ No |

### Practical Example: Builder-Style Interface

```java
package com.interview.java9;

import java.util.*;

public class BuilderInterfaceDemo {

    interface QueryBuilder {
        String tableName();

        default String selectAll() {
            return buildSelect("*");
        }

        default String selectColumns(String... columns) {
            return buildSelect(String.join(", ", columns));
        }

        default String selectCount() {
            return buildSelect("COUNT(*)");
        }

        default String selectWithWhere(String columns, String condition) {
            return buildSelect(columns) + buildWhere(condition);
        }

        // ✅ Private helper — shared query building logic
        private String buildSelect(String columns) {
            return "SELECT " + columns + " FROM " + tableName();
        }

        // ✅ Another private helper
        private String buildWhere(String condition) {
            return " WHERE " + condition;
        }
    }

    static class UserQuery implements QueryBuilder {
        @Override
        public String tableName() { return "users"; }
    }

    static class OrderQuery implements QueryBuilder {
        @Override
        public String tableName() { return "orders"; }
    }

    public static void main(String[] args) {
        QueryBuilder userQuery = new UserQuery();
        System.out.println(userQuery.selectAll());
        // SELECT * FROM users

        System.out.println(userQuery.selectColumns("name", "email"));
        // SELECT name, email FROM users

        System.out.println(userQuery.selectWithWhere("*", "age > 18"));
        // SELECT * FROM users WHERE age > 18

        QueryBuilder orderQuery = new OrderQuery();
        System.out.println(orderQuery.selectCount());
        // SELECT COUNT(*) FROM orders
    }
}
```

### Evolution of Interface Methods Across Java Versions

```java
package com.interview.java9;

public class InterfaceEvolutionTimeline {

    // Java 7 and earlier: only public abstract methods + constants
    interface Java7Interface {
        int CONSTANT = 42;                    // public static final (implicit)
        void abstractMethod();                // public abstract (implicit)
    }

    // Java 8: added default and static methods
    interface Java8Interface {
        void abstractMethod();
        default void defaultMethod() { }      // Implementation in interface
        static void staticMethod() { }        // Not inherited
    }

    // Java 9: added private and private static methods
    interface Java9Interface {
        void abstractMethod();
        default void defaultMethod() {
            privateHelper();                   // ✅ Calls private method
        }
        static void staticMethod() {
            privateStaticHelper();             // ✅ Calls private static
        }
        private void privateHelper() { }      // ✅ Hidden from implementors
        private static void privateStaticHelper() { } // ✅ Hidden from all
    }
}
```

### ⚠️ Common Mistake
**Over-engineering interfaces with too many default and private methods**: If your interface has multiple default methods, several private helpers, and complex internal logic, it's likely doing too much. An interface with extensive implementation logic is essentially becoming an abstract class — consider whether an abstract class would be more appropriate. Private interface methods are best used for **small, focused helper functions** that eliminate duplication between 2-3 default methods, not for complex business logic.

### ⚡ Performance Tip
Private interface methods compile to the same bytecode as private methods in classes — they're dispatched via `invokespecial` (direct call, no virtual lookup). Since they cannot be overridden, the JIT compiler can always **inline** them, eliminating call overhead entirely. The encapsulation they provide is a zero-cost abstraction at runtime — the same performance as duplicated code, but with better maintainability.

### 🔧 JVM Internals
At the bytecode level, private interface methods are stored in the interface's class file with the `ACC_PRIVATE` flag. Default methods calling private methods use `invokespecial` (Java 9+ adjusted this to allow `invokespecial` on interface private methods, which was previously restricted). The private method's bytecode is identical to a private method in a regular class. Since interfaces are loaded by the same classloader mechanism as classes, private methods have the same visibility enforcement at the JVM level.

---

## 🎯 Interviewer Insight
This question tests your knowledge of Java's version history and the progressive enhancement of interfaces. Interviewers expect you to explain the **problem** (code duplication in default methods) and the **solution** (private methods). Knowing the full timeline (Java 8: default + static, Java 9: + private) demonstrates awareness of language evolution. The key insight: private interface methods make interfaces more maintainable without expanding their public API surface.

## ⚡ Follow-up Questions
1. Can a private interface method call an abstract method of the same interface?
2. Why can't a static method call a private (non-static) method?
3. What was the workaround before Java 9 for sharing logic between default methods?

## 📚 Related Topics
- [Default Methods](./q08-default-methods.md)
- [Static Interface Methods](./q09-static-interface-methods.md)
- [Functional Interfaces](./q03-functional-interfaces-detail.md)

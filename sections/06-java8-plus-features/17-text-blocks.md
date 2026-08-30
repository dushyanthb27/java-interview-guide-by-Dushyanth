# Q17: Text Blocks (Java 15)

## 📋 Question
What are text blocks in Java? Explain the triple-quote syntax, indentation management, escape sequences, related String methods, and practical use cases.

## Difficulty: 🟢 Easy

---

## 💡 Answer

**Text blocks** (Java 13 preview, Java 15 final) provide a multi-line string literal using **triple double-quote** (`"""`) delimiters. They eliminate the need for string concatenation, escape sequences for newlines, and manual formatting — making embedded JSON, SQL, HTML, and other structured text dramatically more readable.

### Basic Syntax

```java
package com.interview.java15;

public class TextBlockBasicsDemo {

    public static void main(String[] args) {

        // ═══════════════════════════════════════════════════
        // BEFORE Java 15: Painful multi-line strings
        // ═══════════════════════════════════════════════════
        String oldJson = "{\n" +
                "    \"name\": \"Alice\",\n" +
                "    \"age\": 30,\n" +
                "    \"email\": \"alice@example.com\"\n" +
                "}";

        // ═══════════════════════════════════════════════════
        // AFTER Java 15: Clean text block
        // ═══════════════════════════════════════════════════
        String newJson = """
                {
                    "name": "Alice",
                    "age": 30,
                    "email": "alice@example.com"
                }
                """;

        System.out.println(newJson);
        // {
        //     "name": "Alice",
        //     "age": 30,
        //     "email": "alice@example.com"
        // }

        // ✅ Text blocks are regular String objects
        System.out.println(newJson.getClass());  // class java.lang.String
        System.out.println(newJson.contains("Alice"));  // true

        // ✅ Opening """ must be followed by a line terminator
        // String bad = """hello""";  // ❌ Compile error
        String ok = """
                hello""";  // ✅ Valid (no trailing newline)
    }
}
```

### Indentation Management

```java
package com.interview.java15;

public class IndentationDemo {

    public static void main(String[] args) {

        // ═══════════════════════════════════════════════════
        // Rule: The LEFTMOST closing """ determines the indentation baseline
        // All content is stripped of common leading whitespace
        // ═══════════════════════════════════════════════════

        // ✅ Closing """ at column 8 → strips 8 spaces from each line
        String noIndent = """
                Hello
                World
                """;
        System.out.println("|" + noIndent + "|");
        // |Hello
        // World
        // |

        // ✅ Moving closing """ LEFT adds relative indentation
        String indented = """
                    Hello
                    World
                """;
        // Closing """ is at column 16, content at column 20 → 4 spaces indent
        System.out.println("|" + indented + "|");
        // |    Hello
        //     World
        // |

        // ✅ Closing """ on same line as last content → no trailing newline
        String noTrailingNewline = """
                Hello World""";
        System.out.println("|" + noTrailingNewline + "|");
        // |Hello World|

        // ✅ Closing """ on next line → trailing newline included
        String withTrailingNewline = """
                Hello World
                """;
        System.out.println("|" + withTrailingNewline + "|");
        // |Hello World
        // |
    }
}
```

### Escape Sequences

```java
package com.interview.java15;

public class EscapeSequencesDemo {

    public static void main(String[] args) {

        // ═══════════════════════════════════════════════════
        // Standard escapes still work inside text blocks
        // ═══════════════════════════════════════════════════
        String withEscapes = """
                Tab:\there
                Quote: "hello"
                Backslash: \\path\\to\\file
                Unicode: \u0041 (A)
                """;
        System.out.println(withEscapes);

        // ═══════════════════════════════════════════════════
        // NEW escape: \s — explicit space (prevents trailing space stripping)
        // ═══════════════════════════════════════════════════
        String withExplicitSpace = """
                Name:  Alice\s\s
                Age:   30\s\s\s\s
                """;
        // Trailing \s prevents automatic trailing whitespace removal

        // ═══════════════════════════════════════════════════
        // NEW escape: \ at end of line — suppresses the newline
        // ═══════════════════════════════════════════════════
        String singleLine = """
                This is a very long string \
                that we want to keep on one line \
                in the output.""";
        System.out.println(singleLine);
        // This is a very long string that we want to keep on one line in the output.

        // ═══════════════════════════════════════════════════
        // Triple quotes INSIDE text blocks — escape one quote
        // ═══════════════════════════════════════════════════
        String containsTripleQuote = """
                She said: \"""hello\"""
                """;
        System.out.println(containsTripleQuote);
        // She said: """hello"""
    }
}
```

### String Methods for Text Blocks

```java
package com.interview.java15;

public class StringMethodsDemo {

    public static void main(String[] args) {

        // ═══════════════════════════════════════════════════
        // stripIndent() — removes incidental whitespace (auto-applied to text blocks)
        // ═══════════════════════════════════════════════════
        String indented = "    Hello\n    World\n";
        String stripped = indented.stripIndent();
        System.out.println(stripped);  // "Hello\nWorld\n"

        // ═══════════════════════════════════════════════════
        // translateEscapes() — processes escape sequences in runtime strings
        // ═══════════════════════════════════════════════════
        String raw = "Hello\\nWorld\\t!";
        String translated = raw.translateEscapes();
        System.out.println(translated);
        // Hello
        // World	!

        // ═══════════════════════════════════════════════════
        // formatted() — instance version of String.format()
        // ═══════════════════════════════════════════════════
        String template = """
                Dear %s,
                
                Your order #%d has been %s.
                Total: $%.2f
                
                Thank you!
                """;
        String letter = template.formatted("Alice", 12345, "shipped", 99.99);
        System.out.println(letter);

        // ═══════════════════════════════════════════════════
        // indent(n) — adds or removes indentation
        // ═══════════════════════════════════════════════════
        String base = "Line 1\nLine 2\nLine 3";
        String indented2 = base.indent(4);   // Adds 4 spaces to each line
        String outdented = base.indent(-2);  // Removes up to 2 spaces from each line
        System.out.println("Indented:\n" + indented2);
    }
}
```

### Practical Use Cases

```java
package com.interview.java15;

public class TextBlockUseCasesDemo {

    // ═══════════════════════════════════════════════════
    // Use Case 1: JSON
    // ═══════════════════════════════════════════════════
    static String createUserJson(String name, int age, String email) {
        return """
                {
                    "name": "%s",
                    "age": %d,
                    "email": "%s",
                    "active": true
                }
                """.formatted(name, age, email);
    }

    // ═══════════════════════════════════════════════════
    // Use Case 2: SQL queries
    // ═══════════════════════════════════════════════════
    static final String FIND_ACTIVE_USERS = """
            SELECT u.id, u.name, u.email, u.created_at
            FROM users u
            JOIN roles r ON u.role_id = r.id
            WHERE u.active = true
              AND r.name IN ('ADMIN', 'MANAGER')
            ORDER BY u.created_at DESC
            LIMIT 100
            """;

    // ═══════════════════════════════════════════════════
    // Use Case 3: HTML templates
    // ═══════════════════════════════════════════════════
    static String createHtmlPage(String title, String content) {
        return """
                <!DOCTYPE html>
                <html lang="en">
                <head>
                    <meta charset="UTF-8">
                    <title>%s</title>
                </head>
                <body>
                    <h1>%s</h1>
                    <div class="content">
                        %s
                    </div>
                </body>
                </html>
                """.formatted(title, title, content);
    }

    // ═══════════════════════════════════════════════════
    // Use Case 4: Configuration / YAML
    // ═══════════════════════════════════════════════════
    static final String DEFAULT_CONFIG = """
            server:
              port: 8080
              host: localhost
            database:
              url: jdbc:postgresql://localhost:5432/mydb
              pool-size: 10
            logging:
              level: INFO
            """;

    // ═══════════════════════════════════════════════════
    // Use Case 5: Regular expressions with comments
    // ═══════════════════════════════════════════════════
    static final String EMAIL_REGEX = """
            [a-zA-Z0-9._%+-]+\
            @\
            [a-zA-Z0-9.-]+\
            \\.\
            [a-zA-Z]{2,}""";

    // ═══════════════════════════════════════════════════
    // Use Case 6: Error messages
    // ═══════════════════════════════════════════════════
    static String validationError(String field, Object value, String rule) {
        return """
                Validation Error:
                  Field:    %s
                  Value:    %s
                  Rule:     %s
                  Hint:     Please check your input and try again.
                """.formatted(field, value, rule);
    }

    public static void main(String[] args) {
        System.out.println(createUserJson("Alice", 30, "alice@example.com"));
        System.out.println("---");
        System.out.println(FIND_ACTIVE_USERS);
        System.out.println("---");
        System.out.println(validationError("email", "bad-email", "must contain @"));
    }
}
```

### ⚠️ Common Mistake
**Misunderstanding indentation stripping**: The position of the **closing `"""`** determines how much indentation is removed. If the closing `"""` is at column 0, all content whitespace is preserved. If it's at column 16, 16 characters of leading whitespace are stripped from each line. Placing the closing `"""` at an unexpected position is the most common source of text block formatting bugs. Always align the closing `"""` with the indentation level you want as the baseline.

### ⚡ Performance Tip
Text blocks are compiled to the **exact same bytecode** as equivalent regular strings. The `stripIndent()` processing happens at compile time, not runtime — the class file contains the final stripped string. There is **zero runtime overhead** compared to using regular string concatenation or `StringBuilder`. Text blocks are also subject to **string interning** — identical text block constants in the same class share the same `String` object in the constant pool.

### 🔧 JVM Internals
The Java compiler processes text blocks in three steps: (1) **line terminator normalization** — all line endings are converted to `\n` (LF); (2) **incidental whitespace stripping** — common leading whitespace (determined by the closing `"""` position) is removed from each line, and trailing whitespace on each line is stripped; (3) **escape sequence interpretation** — standard escapes (`\n`, `\t`, etc.) and new escapes (`\s`, `\` line continuation) are processed. The resulting string constant is placed in the class file's constant pool, identical to how regular string literals are stored.

---

## 🎯 Interviewer Insight
Text blocks are typically a straightforward question — interviewers primarily check that you know the syntax and indentation rules. Bonus points for mentioning the `\` line continuation escape (prevents newline), the `\s` explicit space escape, and the `formatted()` method for parameterization. Understanding that text blocks compile to regular strings (zero runtime cost) demonstrates depth.

## ⚡ Follow-up Questions
1. How does the position of the closing `"""` affect indentation?
2. What's the difference between `\s` and a regular space in a text block?
3. How do text blocks handle different line endings (Windows vs Unix)?

## 📚 Related Topics
- [Records](./q14-records-detail.md)
- [var Keyword](./q13-var-keyword.md)
- [Lambda Expressions](./q01-lambda-expressions.md)

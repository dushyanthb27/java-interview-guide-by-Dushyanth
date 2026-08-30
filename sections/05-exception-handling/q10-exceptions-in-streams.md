# Q10: How to Handle Exceptions in Streams?

[Home](../../README.md) > [Exception Handling](./README.md) > Exceptions in Streams

---

## Difficulty: 🟠 Hard

## Question
> How do you handle checked exceptions inside Java Stream operations? What patterns and libraries can help?

---

## Answer

### The Problem

Java's built-in functional interfaces (`Function`, `Consumer`, `Supplier`, etc.) do **not** declare checked exceptions. This creates a fundamental incompatibility with methods that throw checked exceptions.

```java
// ❌ Won't compile - Function.apply() doesn't declare throws
List<String> paths = List.of("file1.txt", "file2.txt", "file3.txt");

List<String> contents = paths.stream()
    .map(path -> Files.readString(Path.of(path)))  // Compile error!
    //           ^^^ IOException is checked
    .collect(Collectors.toList());
```

---

### Solution 1: Wrapping in try-catch (Most Common)

```java
// ✅ Wrap checked exception in unchecked inside the lambda
List<String> contents = paths.stream()
    .map(path -> {
        try {
            return Files.readString(Path.of(path));
        } catch (IOException e) {
            throw new UncheckedIOException("Failed to read: " + path, e);
        }
    })
    .collect(Collectors.toList());
```

---

### Solution 2: Utility Wrapper Function

```java
// ✅ Generic wrapper that converts checked to unchecked

@FunctionalInterface
public interface CheckedFunction<T, R> {
    R apply(T t) throws Exception;
}

public class Unchecked {

    public static <T, R> Function<T, R> function(CheckedFunction<T, R> fn) {
        return t -> {
            try {
                return fn.apply(t);
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        };
    }
}

// Usage - clean and readable:
List<String> contents = paths.stream()
    .map(Unchecked.function(path -> Files.readString(Path.of(path))))
    .collect(Collectors.toList());

// ✅ Complete set of wrappers
public class Unchecked {

    @FunctionalInterface
    public interface CheckedFunction<T, R> {
        R apply(T t) throws Exception;
    }

    @FunctionalInterface
    public interface CheckedConsumer<T> {
        void accept(T t) throws Exception;
    }

    @FunctionalInterface
    public interface CheckedSupplier<R> {
        R get() throws Exception;
    }

    @FunctionalInterface
    public interface CheckedRunnable {
        void run() throws Exception;
    }

    public static <T, R> Function<T, R> function(CheckedFunction<T, R> fn) {
        return t -> {
            try { return fn.apply(t); }
            catch (Exception e) { throw new RuntimeException(e); }
        };
    }

    public static <T> Consumer<T> consumer(CheckedConsumer<T> consumer) {
        return t -> {
            try { consumer.accept(t); }
            catch (Exception e) { throw new RuntimeException(e); }
        };
    }

    public static <R> Supplier<R> supplier(CheckedSupplier<R> supplier) {
        return () -> {
            try { return supplier.get(); }
            catch (Exception e) { throw new RuntimeException(e); }
        };
    }
}

// Usage:
paths.forEach(Unchecked.consumer(path -> Files.delete(Path.of(path))));

String content = Unchecked.supplier(() -> Files.readString(Path.of("config.yml"))).get();
```

---

### Solution 3: Sneaky Throws Pattern

```java
// The sneaky throws trick - throws checked exception without declaring it
// Uses Java generics type erasure to bypass the compiler check

public class SneakyThrows {

    @SuppressWarnings("unchecked")
    public static <E extends Throwable> void sneakyThrow(Throwable e) throws E {
        throw (E) e;  // Type erasure makes this cast a no-op
    }

    public static <T, R> Function<T, R> sneaky(CheckedFunction<T, R> fn) {
        return t -> {
            try {
                return fn.apply(t);
            } catch (Exception e) {
                sneakyThrow(e);
                return null; // Never reached
            }
        };
    }
}

// Usage:
List<String> contents = paths.stream()
    .map(SneakyThrows.sneaky(path -> Files.readString(Path.of(path))))
    .collect(Collectors.toList());

// ⚠️ Warning: The IOException is thrown but NOT declared in the method signature
// This can confuse callers who don't expect a checked exception
// Lombok's @SneakyThrows does the same thing
```

---

### Solution 4: Either/Result Pattern

```java
// ✅ Instead of throwing, capture success OR failure

public sealed interface Result<T> {

    record Success<T>(T value) implements Result<T> {}
    record Failure<T>(Exception error) implements Result<T> {}

    static <T> Result<T> of(CheckedSupplier<T> supplier) {
        try {
            return new Success<>(supplier.get());
        } catch (Exception e) {
            return new Failure<>(e);
        }
    }

    default boolean isSuccess() { return this instanceof Success; }
    default boolean isFailure() { return this instanceof Failure; }
}

// Usage:
List<Result<String>> results = paths.stream()
    .map(path -> Result.of(() -> Files.readString(Path.of(path))))
    .collect(Collectors.toList());

// ✅ Separate successes from failures
List<String> successes = results.stream()
    .filter(Result::isSuccess)
    .map(r -> ((Result.Success<String>) r).value())
    .toList();

List<Exception> failures = results.stream()
    .filter(Result::isFailure)
    .map(r -> ((Result.Failure<String>) r).error())
    .toList();

System.out.println("Loaded " + successes.size() + " files");
System.out.println("Failed " + failures.size() + " files");
failures.forEach(e -> log.warn("Failed: {}", e.getMessage()));
```

---

### Solution 5: Vavr Library (Production-Ready Either/Try)

```java
// Vavr provides a mature Try monad for exception handling in streams
// Dependency: io.vavr:vavr:0.10.4

import io.vavr.control.Try;

// ✅ Using Vavr's Try
List<Try<String>> results = paths.stream()
    .map(path -> Try.of(() -> Files.readString(Path.of(path))))
    .collect(Collectors.toList());

// Get only successful results
List<String> successes = results.stream()
    .filter(Try::isSuccess)
    .map(Try::get)
    .toList();

// Get failures
results.stream()
    .filter(Try::isFailure)
    .forEach(t -> log.error("Failed: {}", t.getCause().getMessage()));

// ✅ Using Vavr's Try with recovery
List<String> contentsWithFallback = paths.stream()
    .map(path -> Try.of(() -> Files.readString(Path.of(path)))
        .recover(FileNotFoundException.class, e -> "FILE NOT FOUND")
        .recover(IOException.class, e -> "READ ERROR")
        .get())
    .toList();

// ✅ Using Vavr's Either
import io.vavr.control.Either;

List<Either<Exception, String>> parsed = inputs.stream()
    .map(input -> {
        try {
            return Either.<Exception, String>right(parseData(input));
        } catch (Exception e) {
            return Either.<Exception, String>left(e);
        }
    })
    .toList();

// Partition into successes and failures
var partitioned = parsed.stream()
    .collect(Collectors.partitioningBy(Either::isRight));
```

---

### Solution 6: Collecting Exceptions

```java
// ✅ Process everything, collect all exceptions, report at the end

public class BatchProcessor<T, R> {

    private final List<R> results = new ArrayList<>();
    private final List<ProcessingError<T>> errors = new ArrayList<>();

    public record ProcessingError<T>(T input, Exception error) {}

    public BatchProcessor<T, R> process(List<T> items, CheckedFunction<T, R> processor) {
        for (T item : items) {
            try {
                results.add(processor.apply(item));
            } catch (Exception e) {
                errors.add(new ProcessingError<>(item, e));
            }
        }
        return this;
    }

    public List<R> getResults() { return Collections.unmodifiableList(results); }
    public List<ProcessingError<T>> getErrors() { return Collections.unmodifiableList(errors); }
    public boolean hasErrors() { return !errors.isEmpty(); }
}

// Usage:
BatchProcessor<String, String> batch = new BatchProcessor<String, String>()
    .process(paths, path -> Files.readString(Path.of(path)));

System.out.println("Loaded: " + batch.getResults().size());
if (batch.hasErrors()) {
    batch.getErrors().forEach(err ->
        log.warn("Failed to load {}: {}", err.input(), err.error().getMessage()));
}
```

---

### Handling Exceptions in Parallel Streams

```java
// ⚠️ Parallel streams need thread-safe exception handling

// ❌ Non-thread-safe collection of errors
List<Exception> errors = new ArrayList<>(); // Not thread-safe!
paths.parallelStream()
    .forEach(path -> {
        try { processFile(path); }
        catch (Exception e) { errors.add(e); } // Race condition!
    });

// ✅ Use thread-safe collection
List<Exception> errors = Collections.synchronizedList(new ArrayList<>());
paths.parallelStream()
    .forEach(path -> {
        try { processFile(path); }
        catch (Exception e) { errors.add(e); }
    });

// ✅ Or use Collectors (thread-safe in parallel streams)
Map<Boolean, List<Result<String>>> partitioned = paths.parallelStream()
    .map(path -> Result.of(() -> Files.readString(Path.of(path))))
    .collect(Collectors.partitioningBy(Result::isSuccess));
```

---

### Anti-Patterns

```java
// ❌ Silently swallowing exceptions in streams
paths.stream()
    .map(path -> {
        try { return Files.readString(Path.of(path)); }
        catch (IOException e) { return null; }  // Nulls everywhere!
    })
    .filter(Objects::nonNull)  // Silently drops failures
    .toList();

// ❌ Using Optional.empty() to hide failures
paths.stream()
    .map(path -> {
        try { return Optional.of(Files.readString(Path.of(path))); }
        catch (IOException e) { return Optional.<String>empty(); }
    })
    .flatMap(Optional::stream) // Silently drops failures
    .toList();

// ✅ At minimum, log the failure
paths.stream()
    .map(path -> {
        try { return Files.readString(Path.of(path)); }
        catch (IOException e) {
            log.warn("Skipping file {}: {}", path, e.getMessage());
            return null;
        }
    })
    .filter(Objects::nonNull)
    .toList();
```

---

## Interview Tips
- Start by explaining WHY checked exceptions don't work with lambdas (functional interfaces)
- Show the wrapping approach first (most practical)
- Discuss the Either/Try pattern as an advanced solution
- Mention Vavr library for production code
- Warn about sneaky throws - it works but can confuse callers
- Know the thread safety implications in parallel streams

---

[<< Previous: Q9 - Suppressed Exceptions](./q09-suppressed-exceptions.md) | [Next: Q11 - Exception Performance >>](./q11-exception-performance.md)

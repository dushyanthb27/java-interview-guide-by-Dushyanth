# Q12: Composition vs Inheritance

## Question
What is object composition? Explain the "favor composition over inheritance" principle, Has-A vs Is-A relationships, and demonstrate refactoring from inheritance to composition.

## Difficulty: 🟠 Advanced

---

## Answer

**Composition** is a design technique where a class contains references to other objects to reuse their behavior, establishing a **HAS-A** relationship. The principle "favor composition over inheritance" from *Effective Java* (Item 18) advises using composition for code reuse unless a genuine IS-A relationship exists.

### Inheritance (IS-A) vs Composition (HAS-A)

```java
// ❌ Inheritance: Stack IS-A ArrayList? No! Stack should only allow push/pop.
public class Stack<E> extends ArrayList<E> {
    public void push(E item) { add(item); }
    public E pop() { return remove(size() - 1); }
    // Problem: clients can call add(), remove(index), get(index), etc.
    // This VIOLATES the stack contract!
}

// ✅ Composition: Stack HAS-A List (internal implementation detail)
public class Stack<E> {
    private final List<E> elements = new ArrayList<>();

    public void push(E item) {
        elements.add(item);
    }

    public E pop() {
        if (elements.isEmpty()) throw new EmptyStackException();
        return elements.remove(elements.size() - 1);
    }

    public E peek() {
        if (elements.isEmpty()) throw new EmptyStackException();
        return elements.get(elements.size() - 1);
    }

    public boolean isEmpty() { return elements.isEmpty(); }
    public int size() { return elements.size(); }
    // No unwanted methods exposed!
}
```

### Why Composition Over Inheritance

| Aspect | Inheritance | Composition |
|--------|-----------|-------------|
| Coupling | Tight (child depends on parent internals) | Loose (depends on interface) |
| Flexibility | Fixed at compile time | Can change at runtime |
| Encapsulation | Breaks parent encapsulation | Preserves encapsulation |
| Reuse | Single class only | Multiple objects |
| Testing | Hard to mock parent | Easy to mock components |

### The Fragile Base Class Problem

```java
// ❌ Inheritance creates fragile coupling
public class InstrumentedHashSet<E> extends HashSet<E> {
    private int addCount = 0;

    @Override
    public boolean add(E e) {
        addCount++;
        return super.add(e);
    }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c); // BUG: HashSet.addAll() calls add() internally!
        // addCount is incremented TWICE for each element
    }

    public int getAddCount() { return addCount; }
}

// Test:
var set = new InstrumentedHashSet<String>();
set.addAll(List.of("a", "b", "c"));
System.out.println(set.getAddCount()); // 6! (expected 3)
// addAll adds 3, then internally calls add() 3 more times
```

```java
// ✅ Composition: Wrapper/Decorator pattern
public class InstrumentedSet<E> implements Set<E> {
    private final Set<E> delegate;  // HAS-A relationship
    private int addCount = 0;

    public InstrumentedSet(Set<E> delegate) {
        this.delegate = delegate;
    }

    @Override
    public boolean add(E e) {
        addCount++;
        return delegate.add(e); // No internal coupling issue
    }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return delegate.addAll(c); // delegate.addAll() calls delegate.add(), NOT our add()
    }

    public int getAddCount() { return addCount; }

    // Delegate all other Set methods
    @Override public int size() { return delegate.size(); }
    @Override public boolean isEmpty() { return delegate.isEmpty(); }
    @Override public boolean contains(Object o) { return delegate.contains(o); }
    @Override public Iterator<E> iterator() { return delegate.iterator(); }
    @Override public Object[] toArray() { return delegate.toArray(); }
    @Override public <T> T[] toArray(T[] a) { return delegate.toArray(a); }
    @Override public boolean remove(Object o) { return delegate.remove(o); }
    @Override public boolean containsAll(Collection<?> c) { return delegate.containsAll(c); }
    @Override public boolean retainAll(Collection<?> c) { return delegate.retainAll(c); }
    @Override public boolean removeAll(Collection<?> c) { return delegate.removeAll(c); }
    @Override public void clear() { delegate.clear(); }
}

// Now it works correctly:
var set = new InstrumentedSet<>(new HashSet<>());
set.addAll(List.of("a", "b", "c"));
System.out.println(set.getAddCount()); // 3 ✅
```

### Strategy Pattern (Composition in Action)

```java
// ✅ Composition allows runtime behavior change
public interface SortStrategy {
    <T extends Comparable<T>> void sort(List<T> list);
}

public class QuickSort implements SortStrategy {
    @Override
    public <T extends Comparable<T>> void sort(List<T> list) {
        System.out.println("Quick sorting...");
        Collections.sort(list); // Simplified
    }
}

public class MergeSort implements SortStrategy {
    @Override
    public <T extends Comparable<T>> void sort(List<T> list) {
        System.out.println("Merge sorting...");
        Collections.sort(list);
    }
}

public class DataProcessor {
    private SortStrategy strategy; // Composed, not inherited

    public DataProcessor(SortStrategy strategy) {
        this.strategy = strategy;
    }

    // ✅ Can change strategy at runtime!
    public void setStrategy(SortStrategy strategy) {
        this.strategy = strategy;
    }

    public <T extends Comparable<T>> void process(List<T> data) {
        strategy.sort(data); // Delegates to composed object
        System.out.println("Processed: " + data);
    }
}

// Usage
var processor = new DataProcessor(new QuickSort());
processor.process(new ArrayList<>(List.of(3, 1, 4, 1, 5)));

processor.setStrategy(new MergeSort()); // Swap at runtime!
processor.process(new ArrayList<>(List.of(9, 2, 6, 5, 3)));
```

### Dependency Injection (Composition Pattern)

```java
// ✅ Spring-style composition via dependency injection
public interface NotificationSender {
    void send(String to, String message);
}

public class EmailSender implements NotificationSender {
    @Override
    public void send(String to, String message) {
        System.out.println("Email to " + to + ": " + message);
    }
}

public class SmsSender implements NotificationSender {
    @Override
    public void send(String to, String message) {
        System.out.println("SMS to " + to + ": " + message);
    }
}

// ✅ OrderService composes NotificationSender (doesn't inherit from it)
public class OrderService {
    private final NotificationSender sender;
    private final OrderRepository repository;

    // Constructor injection - composition
    public OrderService(NotificationSender sender, OrderRepository repository) {
        this.sender = sender;
        this.repository = repository;
    }

    public void placeOrder(Order order) {
        repository.save(order);
        sender.send(order.getCustomerEmail(), "Order placed: " + order.getId());
    }
}
```

### Refactoring: Inheritance to Composition

```java
// ❌ BEFORE: Tight coupling via inheritance
public class Robot extends ArrayList<String> {
    public void addCommand(String cmd) { add(cmd); }
    public void execute() { forEach(System.out::println); }
}

// ✅ AFTER: Loose coupling via composition
public class Robot {
    private final List<String> commands = new ArrayList<>();
    private final Executor executor;

    public Robot(Executor executor) {
        this.executor = executor;
    }

    public void addCommand(String cmd) {
        commands.add(cmd);
    }

    public void execute() {
        commands.forEach(executor::execute);
    }
}

public interface Executor {
    void execute(String command);
}
```

### Common Mistake

```java
// ❌ Using inheritance purely for code reuse (no IS-A relationship)
public class StringUtils {
    public String capitalize(String s) { return s.substring(0, 1).toUpperCase() + s.substring(1); }
}
public class UserFormatter extends StringUtils { // UserFormatter IS-A StringUtils? No!
    public String format(User u) { return capitalize(u.getName()); }
}

// ✅ Use composition or static utility
public class UserFormatter {
    public String format(User u) {
        return StringUtils.capitalize(u.getName()); // Static call or inject
    }
}
```

### Performance Tip
Composition adds a layer of indirection (delegation call), but the JIT compiler inlines these calls aggressively. The design flexibility and maintainability benefits far outweigh any negligible performance cost. Profile before optimizing.

---

## Interviewer Insight
This is a design-focused question. Interviewers want to see you explain the **fragile base class problem**, demonstrate the **Decorator/Strategy pattern**, and articulate when inheritance IS appropriate (genuine IS-A + Liskov Substitution). Real refactoring examples are highly valued.

## Follow-up Questions
1. What is the Liskov Substitution Principle and how does it relate to inheritance?
2. How does the Decorator pattern use composition?
3. When IS inheritance the right choice over composition?

## Related Topics
- [Inheritance](./q03-inheritance.md)
- [Interfaces vs Abstract Classes](./q07-interfaces-vs-abstract-classes.md)
- [Polymorphism](./q04-polymorphism.md)

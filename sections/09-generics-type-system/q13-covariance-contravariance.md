# Q13: Explain Covariance and Contravariance

[Home](../../README.md) > [Generics & Type System](./README.md) > Covariance & Contravariance

---

## 🟠 Difficulty: Advanced

## Question
> What are covariance, contravariance, and invariance? How do they apply to Java's arrays, generics, and wildcards?

---

## Quick Answer
**Covariance** preserves the type hierarchy (`Dog → Animal` means `List<Dog> → List<? extends Animal>`). **Contravariance** reverses it (`Dog → Animal` means `Consumer<Animal> → Consumer<? super Dog>`). **Invariance** means no subtype relationship exists (`List<Dog>` has NO relation to `List<Animal>`). Java arrays are covariant by design, generics are invariant by default, and wildcards provide **use-site variance** (`? extends` for covariance, `? super` for contravariance).

---

## Detailed Explanation

### The Three Variance Types

```
Given:  Dog extends Animal

COVARIANT (preserves direction):
  Dog → Animal   ⟹   Box<Dog> → Box<? extends Animal>
  "If Dog is a subtype of Animal, then Box<Dog> is a subtype of Box<Animal>"

CONTRAVARIANT (reverses direction):
  Dog → Animal   ⟹   Consumer<Animal> → Consumer<? super Dog>
  "If Dog is a subtype of Animal, then Consumer<Animal> is a subtype of Consumer<Dog>"

INVARIANT (no relationship):
  Dog → Animal   ⟹   List<Dog> has NO relation to List<Animal>
  "Even though Dog is a subtype of Animal, List<Dog> is NOT related to List<Animal>"
```

### Array Covariance (Built-in)

```java
// Java arrays are covariant — this is a LANGUAGE DESIGN DECISION
String[] strings = {"hello", "world"};
Object[] objects = strings;            // ✅ String[] IS-A Object[]

// Covariance makes arrays flexible for reading:
Object first = objects[0];             // ✅ Read any element as Object

// But writing is checked at RUNTIME:
objects[0] = "safe";                   // ✅ String into String[]
objects[1] = 42;                       // 💥 ArrayStoreException!
// Runtime check catches the mismatch

// Why arrays were made covariant:
// Pre-generics, this was the only way to write generic-like methods:
public static void sort(Object[] array) {  // Works for any array type
    // ...
}
sort(strings);  // ✅ Works because String[] is Object[]
```

### Generic Invariance (Default)

```java
// Generics are INVARIANT — no subtype relationship between different type args
List<Dog> dogs = new ArrayList<>();
// List<Animal> animals = dogs;        // ❌ COMPILE ERROR!

// Even though Dog extends Animal,
// List<Dog> does NOT extend List<Animal>

// Why? Because it would be unsafe:
// If List<Dog> were List<Animal>, you could do:
// animals.add(new Cat());  // Adding Cat to a list of Dogs — bad!

// Invariance catches this at COMPILE time (unlike array's runtime check)
```

### Covariance via ? extends (Use-Site)

```java
// Wildcards add covariance WHERE NEEDED
List<Dog> dogs = List.of(new Dog("Rex"), new Dog("Max"));

// ✅ Covariant: List<Dog> IS-A List<? extends Animal>
List<? extends Animal> animals = dogs;

// Covariance enables READING (producing values)
Animal a = animals.get(0);    // ✅ Read as Animal
// animals.add(new Dog());    // ❌ Cannot write — read-only

// Real-world covariance:
public static void printAnimals(List<? extends Animal> animals) {
    for (Animal animal : animals) {
        System.out.println(animal.getName());
    }
}

printAnimals(dogs);              // ✅ List<Dog> → List<? extends Animal>
printAnimals(List.of(new Cat())); // ✅ List<Cat> → List<? extends Animal>
```

### Contravariance via ? super (Use-Site)

```java
// Contravariance: the type relationship is REVERSED
// Animal is a supertype of Dog
// But Consumer<Animal> is a SUBTYPE of Consumer<? super Dog>

Comparator<Animal> animalComparator = (a, b) ->
    a.getName().compareTo(b.getName());

// ✅ Contravariant: Comparator<Animal> works where Comparator<? super Dog> expected
List<Dog> dogs = new ArrayList<>(List.of(new Dog("Rex"), new Dog("Max")));
dogs.sort(animalComparator);  // ✅ sort expects Comparator<? super Dog>

// Contravariance enables WRITING (consuming values)
List<? super Dog> destination = new ArrayList<Animal>();
destination.add(new Dog("Rex"));  // ✅ Can add Dog
destination.add(new Puppy());     // ✅ Can add Puppy (subtype of Dog)
// Object obj = destination.get(0); // Only get Object back
```

### Comparison Table

```
┌──────────────────┬──────────────┬──────────────────┬───────────────┐
│                  │ Covariant    │ Contravariant    │ Invariant     │
├──────────────────┼──────────────┼──────────────────┼───────────────┤
│ Java syntax      │ ? extends T  │ ? super T        │ T (exact)     │
│ Arrays           │ Built-in     │ N/A              │ N/A           │
│ Read (produce)   │ ✅ Safe      │ ❌ Only Object   │ ✅ Safe       │
│ Write (consume)  │ ❌ Blocked   │ ✅ Safe          │ ✅ Safe       │
│ Type direction   │ Preserved    │ Reversed         │ None          │
│ PECS role        │ Producer     │ Consumer         │ Both          │
│ Kotlin keyword   │ out          │ in               │ (default)     │
│ Scala symbol     │ +T           │ -T               │ T             │
│ C# keyword       │ out          │ in               │ (default)     │
└──────────────────┴──────────────┴──────────────────┴───────────────┘
```

### Use-Site vs Declaration-Site Variance

```java
// JAVA: Use-site variance (applied at each usage point)
// You choose variance EACH TIME you use the type

// At usage site 1: covariant (read-only)
void readAnimals(List<? extends Animal> animals) { }

// At usage site 2: contravariant (write-only)
void addDogs(List<? super Dog> destination) { }

// At usage site 3: invariant (read + write)
void processExactly(List<Dog> dogs) { }


// KOTLIN: Declaration-site variance (applied once at class definition)
// interface Source<out T> {   // Always covariant — "out" = produces T
//     fun nextT(): T
// }
// interface Sink<in T> {     // Always contravariant — "in" = consumes T
//     fun accept(t: T)
// }

// SCALA: Declaration-site variance with symbols
// class Box[+T]              // Covariant (+ prefix)
// class Consumer[-T]         // Contravariant (- prefix)
// class Container[T]         // Invariant (default)
```

### Kotlin Comparison

```kotlin
// Kotlin declaration-site variance:
interface Producer<out T> {    // Covariant — T only in "out" positions
    fun produce(): T           // ✅ T as return type
    // fun consume(t: T)       // ❌ T NOT allowed as parameter
}

interface Consumer<in T> {     // Contravariant — T only in "in" positions
    fun consume(t: T)          // ✅ T as parameter
    // fun produce(): T        // ❌ T NOT allowed as return type
}

// Usage — no wildcards needed!
val producer: Producer<Animal> = dogProducer  // ✅ Covariant
val consumer: Consumer<Dog> = animalConsumer  // ✅ Contravariant

// Kotlin also supports use-site variance (like Java wildcards):
fun copy(from: MutableList<out Animal>, to: MutableList<in Animal>) {
    // out ≈ ? extends (covariant)
    // in  ≈ ? super   (contravariant)
}
```

### Functional Interface Variance

```java
// Java's functional interfaces demonstrate variance patterns:

// Function<T, R>: T is contravariant (consumed), R is covariant (produced)
Function<Animal, String> animalToString = Animal::getName;
Function<? super Dog, ? extends CharSequence> flexible = animalToString;
// ? super Dog  → contravariant input  (Animal is super of Dog) ✅
// ? extends CharSequence → covariant output (String extends CharSequence) ✅

// Predicate<T>: T is contravariant (consumed)
Predicate<Animal> isLarge = a -> a.getWeight() > 50;
Predicate<? super Dog> dogTest = isLarge;  // ✅ Contravariant

// Supplier<T>: T is covariant (produced)
Supplier<Dog> dogSupplier = Dog::new;
Supplier<? extends Animal> animalSupplier = dogSupplier;  // ✅ Covariant

// Consumer<T>: T is contravariant (consumed)
Consumer<Animal> printer = a -> System.out.println(a.getName());
Consumer<? super Dog> dogPrinter = printer;  // ✅ Contravariant
```

### Variance in Method Overriding

```java
// Java supports COVARIANT return types in overriding
class AnimalFactory {
    Animal create() { return new Animal(); }
}

class DogFactory extends AnimalFactory {
    @Override
    Dog create() { return new Dog(); }  // ✅ Dog is more specific (covariant)
}

// Java does NOT support contravariant parameters in overriding
// (Parameters must match exactly for overriding)
class Handler {
    void handle(Dog dog) { }
}

class SubHandler extends Handler {
    // This is OVERLOADING, not overriding:
    void handle(Animal animal) { }  // Different method — takes Animal
}
```

### Real-World Variance Example

```java
public class EventSystem {

    // Covariant: reading events
    public <E extends Event> List<? extends E> getEvents(Class<E> type) {
        // Returns events of type E or more specific
        return eventStore.query(type);
    }

    // Contravariant: event handlers
    public <E extends Event> void registerHandler(
            Class<E> eventType,
            Consumer<? super E> handler  // Handler can accept E or supertypes
    ) {
        handlers.put(eventType, handler);
    }

    // Usage:
    // A handler for all events works for specific events too
    Consumer<Event> generalHandler = e -> log(e);
    registerHandler(ClickEvent.class, generalHandler);  // ✅ Contravariant
}
```

---

## Anti-Patterns

```java
// ❌ Relying on array covariance for type safety
Object[] arr = new String[3];
arr[0] = 42;  // Compiles but 💥 ArrayStoreException at runtime

// ❌ Expecting generics to be covariant like arrays
List<Dog> dogs = new ArrayList<>();
// List<Animal> animals = dogs;  // ❌ COMPILE ERROR — invariant

// ❌ Over-using wildcards when exact type is needed
void process(List<? extends Animal> animals) {
    // animals.add(new Dog());  // ❌ Cannot add — should use List<Animal>
}

// ❌ Confusing use-site and declaration-site variance
// Java doesn't have declaration-site variance — must use wildcards every time
```

## Best Practices

```java
// ✅ Use covariance (? extends) for read-only/producer parameters
double totalWeight(List<? extends Animal> animals) {
    return animals.stream().mapToDouble(Animal::getWeight).sum();
}

// ✅ Use contravariance (? super) for write-only/consumer parameters
void addDogs(Collection<? super Dog> dest) {
    dest.add(new Dog("Rex"));
}

// ✅ Use invariance (exact type) when both reading and writing
void swapFirst(List<Dog> dogs) {
    Dog first = dogs.get(0);    // Read
    dogs.set(0, dogs.get(1));   // Write
    dogs.set(1, first);
}

// ✅ Apply PECS: Producer Extends, Consumer Super
<T> void transfer(List<? extends T> src, List<? super T> dest) {
    for (T item : src) dest.add(item);
}
```

---

## Interview Tips
- Arrays: **covariant** with runtime checks (`ArrayStoreException`)
- Generics: **invariant** by default, variance via wildcards at use-site
- `? extends` = covariance (read/produce), `? super` = contravariance (write/consume)
- Java has **use-site** variance; Kotlin/Scala have **declaration-site** variance
- Covariant return types are supported in method overriding
- Know the comparison: Java `? extends`/`? super` vs Kotlin `out`/`in` vs Scala `+`/`-`

---

[← Q12: Generics & Legacy Code](./q12-generics-legacy-code.md) | [Next: Q14 - Custom Generic Types →](./q14-custom-generic-types.md)

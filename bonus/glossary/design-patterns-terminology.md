# Design Patterns Terminology

Comprehensive reference for Gang of Four and modern design patterns with Java implementations. This glossary covers 50+ essential design pattern terms organized by category.

---

## Creational Patterns

### Singleton Pattern
**Category**: Creational

**Definition**: A pattern that ensures a class has only one instance and provides a global point of access to it. Singleton is useful for managing shared resources like database connections or configuration objects.

**Practical Example**:
```java
// Eager initialization
public class Singleton {
    private static final Singleton INSTANCE = new Singleton();
    
    private Singleton() {
    }
    
    public static Singleton getInstance() {
        return INSTANCE;
    }
}

// Lazy initialization with double-checked locking
public class LazySingleton {
    private static volatile LazySingleton instance;
    
    private LazySingleton() {
    }
    
    public static LazySingleton getInstance() {
        if (instance == null) {
            synchronized (LazySingleton.class) {
                if (instance == null) {
                    instance = new LazySingleton();
                }
            }
        }
        return instance;
    }
}

// Bill Pugh Singleton (thread-safe, lazy)
public class BillPughSingleton {
    private BillPughSingleton() {
    }
    
    private static class SingletonHelper {
        private static final BillPughSingleton INSTANCE = new BillPughSingleton();
    }
    
    public static BillPughSingleton getInstance() {
        return SingletonHelper.INSTANCE;
    }
}

// Enum Singleton (best practice)
public enum EnumSingleton {
    INSTANCE;
    
    public void doSomething() {
        System.out.println("Doing something");
    }
}
```

**Related Terms**: [Factory Pattern](#factory-pattern), [Object Creation](#object-creation), [Thread Safety](#thread-safety)

**Interview Tip**: Know different singleton implementations and their trade-offs. Understand thread safety and serialization issues. Prefer enum singleton.

---

### Factory Pattern
**Category**: Creational

**Definition**: A pattern that provides an interface for creating objects without specifying their exact classes. Factory pattern decouples object creation from usage, making code more flexible and maintainable.

**Practical Example**:
```java
// Simple Factory
class ShapeFactory {
    public static Shape createShape(String type) {
        switch (type) {
            case "CIRCLE":
                return new Circle();
            case "RECTANGLE":
                return new Rectangle();
            case "TRIANGLE":
                return new Triangle();
            default:
                throw new IllegalArgumentException("Unknown shape");
        }
    }
}

// Factory Method Pattern
abstract class ShapeCreator {
    abstract Shape createShape();
    
    void draw() {
        Shape shape = createShape();
        shape.draw();
    }
}

class CircleCreator extends ShapeCreator {
    @Override
    Shape createShape() {
        return new Circle();
    }
}

// Abstract Factory Pattern
interface UIFactory {
    Button createButton();
    TextField createTextField();
}

class WindowsUIFactory implements UIFactory {
    @Override
    public Button createButton() {
        return new WindowsButton();
    }
    
    @Override
    public TextField createTextField() {
        return new WindowsTextField();
    }
}

class MacUIFactory implements UIFactory {
    @Override
    public Button createButton() {
        return new MacButton();
    }
    
    @Override
    public TextField createTextField() {
        return new MacTextField();
    }
}
```

**Related Terms**: [Singleton Pattern](#singleton-pattern), [Builder Pattern](#builder-pattern), [Object Creation](#object-creation)

**Interview Tip**: Know the difference between Simple Factory, Factory Method, and Abstract Factory. Understand when to use each.

---

### Builder Pattern
**Category**: Creational

**Definition**: A pattern that separates the construction of a complex object from its representation. Builder pattern allows step-by-step construction and makes code more readable when dealing with many optional parameters.

**Practical Example**:
```java
// Builder Pattern
public class Person {
    private final String name;
    private final int age;
    private final String email;
    private final String phone;
    
    private Person(Builder builder) {
        this.name = builder.name;
        this.age = builder.age;
        this.email = builder.email;
        this.phone = builder.phone;
    }
    
    public static class Builder {
        private final String name;
        private int age;
        private String email;
        private String phone;
        
        public Builder(String name) {
            this.name = name;
        }
        
        public Builder age(int age) {
            this.age = age;
            return this;
        }
        
        public Builder email(String email) {
            this.email = email;
            return this;
        }
        
        public Builder phone(String phone) {
            this.phone = phone;
            return this;
        }
        
        public Person build() {
            return new Person(this);
        }
    }
}

// Usage
Person person = new Person.Builder("Alice")
    .age(30)
    .email("alice@example.com")
    .phone("123-456-7890")
    .build();

// Lombok @Builder annotation
@Builder
class PersonLombok {
    private String name;
    private int age;
    private String email;
    private String phone;
}

PersonLombok person2 = PersonLombok.builder()
    .name("Bob")
    .age(25)
    .build();
```

**Related Terms**: [Factory Pattern](#factory-pattern), [Fluent Interface](#fluent-interface), [Immutable Objects](#immutable-objects)

**Interview Tip**: Know when to use Builder pattern. Understand the benefits for objects with many optional parameters. Know about Lombok's @Builder.

---

### Prototype Pattern
**Category**: Creational

**Definition**: A pattern that creates new objects by copying an existing object (prototype) rather than creating from scratch. Prototype pattern is useful for expensive object creation.

**Practical Example**:
```java
// Prototype Pattern
public interface Cloneable {
    Object clone();
}

public class Shape implements Cloneable {
    private String color;
    private int x;
    private int y;
    
    public Shape(String color, int x, int y) {
        this.color = color;
        this.x = x;
        this.y = y;
    }
    
    @Override
    public Shape clone() {
        return new Shape(this.color, this.x, this.y);
    }
}

// Using Java's Cloneable interface
public class Person implements Cloneable {
    private String name;
    private int age;
    
    @Override
    public Person clone() throws CloneNotSupportedException {
        return (Person) super.clone();
    }
}

// Shallow vs. Deep copy
public class PersonWithAddress implements Cloneable {
    private String name;
    private Address address;
    
    @Override
    public PersonWithAddress clone() throws CloneNotSupportedException {
        PersonWithAddress cloned = (PersonWithAddress) super.clone();
        // Shallow copy - address is shared
        // For deep copy:
        cloned.address = new Address(address.getCity(), address.getZip());
        return cloned;
    }
}
```

**Related Terms**: [Copy Constructor](#copy-constructor), [Deep Copy](#deep-copy), [Cloning](#cloning), [Object Creation](#object-creation)

**Interview Tip**: Know the difference between shallow and deep copy. Understand when to use Prototype pattern. Know about Cloneable interface.

---

## Structural Patterns

### Adapter Pattern
**Category**: Structural

**Definition**: A pattern that converts the interface of a class into another interface clients expect. Adapter pattern allows incompatible interfaces to work together.

**Practical Example**:
```java
// Adapter Pattern
interface Target {
    void request();
}

class Adaptee {
    public void specificRequest() {
        System.out.println("Specific request");
    }
}

class Adapter implements Target {
    private Adaptee adaptee;
    
    public Adapter(Adaptee adaptee) {
        this.adaptee = adaptee;
    }
    
    @Override
    public void request() {
        adaptee.specificRequest();
    }
}

// Real-world example: Collections adapter
List<String> list = new ArrayList<>();
list.add("item1");
list.add("item2");

// Adapt List to Set
Set<String> set = new HashSet<>(list);

// Adapt array to List
String[] array = {"a", "b", "c"};
List<String> listFromArray = Arrays.asList(array);
```

**Related Terms**: [Bridge Pattern](#bridge-pattern), [Decorator Pattern](#decorator-pattern), [Facade Pattern](#facade-pattern)

**Interview Tip**: Understand the difference between Adapter and Decorator. Know when to use Adapter for interface compatibility.

---

### Decorator Pattern
**Category**: Structural

**Definition**: A pattern that attaches additional responsibilities to an object dynamically. Decorator pattern provides a flexible alternative to subclassing for extending functionality.

**Practical Example**:
```java
// Decorator Pattern
interface Component {
    void operation();
}

class ConcreteComponent implements Component {
    @Override
    public void operation() {
        System.out.println("Basic operation");
    }
}

abstract class Decorator implements Component {
    protected Component component;
    
    public Decorator(Component component) {
        this.component = component;
    }
    
    @Override
    public void operation() {
        component.operation();
    }
}

class ConcreteDecoratorA extends Decorator {
    public ConcreteDecoratorA(Component component) {
        super(component);
    }
    
    @Override
    public void operation() {
        super.operation();
        System.out.println("Added behavior A");
    }
}

class ConcreteDecoratorB extends Decorator {
    public ConcreteDecoratorB(Component component) {
        super(component);
    }
    
    @Override
    public void operation() {
        super.operation();
        System.out.println("Added behavior B");
    }
}

// Usage
Component component = new ConcreteComponent();
component = new ConcreteDecoratorA(component);
component = new ConcreteDecoratorB(component);
component.operation();
// Output:
// Basic operation
// Added behavior A
// Added behavior B

// Real-world example: Java I/O
InputStream input = new FileInputStream("file.txt");
input = new BufferedInputStream(input);
input = new DataInputStream(input);
```

**Related Terms**: [Adapter Pattern](#adapter-pattern), [Composition](#composition), [Inheritance](#inheritance)

**Interview Tip**: Know the difference between Decorator and Inheritance. Understand how Decorator provides flexible alternative to subclassing.

---

### Facade Pattern
**Category**: Structural

**Definition**: A pattern that provides a unified, simplified interface to a set of interfaces in a subsystem. Facade pattern hides the complexity of a subsystem behind a simple interface.

**Practical Example**:
```java
// Facade Pattern
class SubsystemA {
    public void operationA() {
        System.out.println("Subsystem A operation");
    }
}

class SubsystemB {
    public void operationB() {
        System.out.println("Subsystem B operation");
    }
}

class SubsystemC {
    public void operationC() {
        System.out.println("Subsystem C operation");
    }
}

// Facade
class Facade {
    private SubsystemA subsystemA;
    private SubsystemB subsystemB;
    private SubsystemC subsystemC;
    
    public Facade() {
        this.subsystemA = new SubsystemA();
        this.subsystemB = new SubsystemB();
        this.subsystemC = new SubsystemC();
    }
    
    public void complexOperation() {
        subsystemA.operationA();
        subsystemB.operationB();
        subsystemC.operationC();
    }
}

// Usage
Facade facade = new Facade();
facade.complexOperation();

// Real-world example: Spring Framework
// ApplicationContext is a facade for the Spring container
ApplicationContext context = new ClassPathXmlApplicationContext("config.xml");
MyService service = context.getBean(MyService.class);
```

**Related Terms**: [Adapter Pattern](#adapter-pattern), [Decorator Pattern](#decorator-pattern), [Abstraction](#abstraction)

**Interview Tip**: Know when to use Facade to simplify complex subsystems. Understand the difference between Facade and Adapter.

---

### Proxy Pattern
**Category**: Structural

**Definition**: A pattern that provides a surrogate or placeholder for another object to control access to it. Proxy pattern is useful for lazy initialization, access control, and logging.

**Practical Example**:
```java
// Proxy Pattern
interface Subject {
    void request();
}

class RealSubject implements Subject {
    @Override
    public void request() {
        System.out.println("Real subject request");
    }
}

class Proxy implements Subject {
    private RealSubject realSubject;
    
    @Override
    public void request() {
        if (realSubject == null) {
            realSubject = new RealSubject();  // Lazy initialization
        }
        System.out.println("Proxy: before request");
        realSubject.request();
        System.out.println("Proxy: after request");
    }
}

// Real-world example: Dynamic Proxy
interface Service {
    void execute();
}

class ServiceImpl implements Service {
    @Override
    public void execute() {
        System.out.println("Service executing");
    }
}

// Dynamic proxy
Service service = (Service) Proxy.newProxyInstance(
    Service.class.getClassLoader(),
    new Class[]{Service.class},
    (proxy, method, args) -> {
        System.out.println("Before: " + method.getName());
        method.invoke(new ServiceImpl(), args);
        System.out.println("After: " + method.getName());
        return null;
    }
);

service.execute();
```

**Related Terms**: [Decorator Pattern](#decorator-pattern), [Lazy Initialization](#lazy-initialization), [Access Control](#access-control)

**Interview Tip**: Know the difference between Proxy and Decorator. Understand use cases for Proxy: lazy initialization, access control, logging.

---

## Behavioral Patterns

### Observer Pattern
**Category**: Behavioral

**Definition**: A pattern that defines a one-to-many dependency between objects so that when one object changes state, all its dependents are notified automatically. Observer pattern is fundamental for event-driven programming.

**Practical Example**:
```java
// Observer Pattern
interface Observer {
    void update(String message);
}

class Subject {
    private List<Observer> observers = new ArrayList<>();
    
    public void attach(Observer observer) {
        observers.add(observer);
    }
    
    public void detach(Observer observer) {
        observers.remove(observer);
    }
    
    public void notifyObservers(String message) {
        for (Observer observer : observers) {
            observer.update(message);
        }
    }
}

class ConcreteObserver implements Observer {
    private String name;
    
    public ConcreteObserver(String name) {
        this.name = name;
    }
    
    @Override
    public void update(String message) {
        System.out.println(name + " received: " + message);
    }
}

// Usage
Subject subject = new Subject();
Observer observer1 = new ConcreteObserver("Observer 1");
Observer observer2 = new ConcreteObserver("Observer 2");

subject.attach(observer1);
subject.attach(observer2);
subject.notifyObservers("Event occurred");

// Real-world example: Java event system
button.addActionListener(event -> {
    System.out.println("Button clicked");
});

// Real-world example: PropertyChangeListener
PropertyChangeSupport support = new PropertyChangeSupport(this);
support.addPropertyChangeListener(event -> {
    System.out.println("Property changed: " + event.getPropertyName());
});
```

**Related Terms**: [Event-Driven Programming](#event-driven-programming), [Publish-Subscribe](#publish-subscribe), [Reactive Programming](#reactive-programming)

**Interview Tip**: Know the Observer pattern and its applications. Understand the difference between Observer and Pub-Sub. Know about Java's built-in observer mechanisms.

---

### Strategy Pattern
**Category**: Behavioral

**Definition**: A pattern that defines a family of algorithms, encapsulates each one, and makes them interchangeable. Strategy pattern lets the algorithm vary independently from clients that use it.

**Practical Example**:
```java
// Strategy Pattern
interface PaymentStrategy {
    void pay(double amount);
}

class CreditCardPayment implements PaymentStrategy {
    @Override
    public void pay(double amount) {
        System.out.println("Paying " + amount + " with credit card");
    }
}

class PayPalPayment implements PaymentStrategy {
    @Override
    public void pay(double amount) {
        System.out.println("Paying " + amount + " with PayPal");
    }
}

class BitcoinPayment implements PaymentStrategy {
    @Override
    public void pay(double amount) {
        System.out.println("Paying " + amount + " with Bitcoin");
    }
}

class ShoppingCart {
    private PaymentStrategy paymentStrategy;
    
    public void setPaymentStrategy(PaymentStrategy strategy) {
        this.paymentStrategy = strategy;
    }
    
    public void checkout(double amount) {
        paymentStrategy.pay(amount);
    }
}

// Usage
ShoppingCart cart = new ShoppingCart();
cart.setPaymentStrategy(new CreditCardPayment());
cart.checkout(100);

cart.setPaymentStrategy(new PayPalPayment());
cart.checkout(50);

// Using lambda (functional approach)
cart.setPaymentStrategy(amount -> System.out.println("Paying " + amount + " with gift card"));
cart.checkout(25);
```

**Related Terms**: [Composition](#composition), [Polymorphism](#polymorphism), [Functional Programming](#functional-programming)

**Interview Tip**: Know when to use Strategy pattern. Understand how it enables runtime algorithm selection. Know about lambda expressions as strategies.

---

### Template Method Pattern
**Category**: Behavioral

**Definition**: A pattern that defines the skeleton of an algorithm in a base class and lets subclasses override specific steps. Template Method pattern promotes code reuse and defines the structure of algorithms.

**Practical Example**:
```java
// Template Method Pattern
abstract class DataProcessor {
    public final void process() {
        readData();
        validateData();
        transformData();
        saveData();
    }
    
    protected abstract void readData();
    protected abstract void validateData();
    protected abstract void transformData();
    protected abstract void saveData();
}

class CSVProcessor extends DataProcessor {
    @Override
    protected void readData() {
        System.out.println("Reading CSV file");
    }
    
    @Override
    protected void validateData() {
        System.out.println("Validating CSV data");
    }
    
    @Override
    protected void transformData() {
        System.out.println("Transforming CSV data");
    }
    
    @Override
    protected void saveData() {
        System.out.println("Saving CSV data");
    }
}

class JSONProcessor extends DataProcessor {
    @Override
    protected void readData() {
        System.out.println("Reading JSON file");
    }
    
    @Override
    protected void validateData() {
        System.out.println("Validating JSON data");
    }
    
    @Override
    protected void transformData() {
        System.out.println("Transforming JSON data");
    }
    
    @Override
    protected void saveData() {
        System.out.println("Saving JSON data");
    }
}

// Usage
DataProcessor processor = new CSVProcessor();
processor.process();
```

**Related Terms**: [Inheritance](#inheritance), [Abstract Class](#abstract-class), [Hook Method](#hook-method)

**Interview Tip**: Know the Template Method pattern and its benefits. Understand the difference between Template Method and Strategy.

---

### State Pattern
**Category**: Behavioral

**Definition**: A pattern that allows an object to alter its behavior when its internal state changes. State pattern encapsulates state-specific behavior in separate classes.

**Practical Example**:
```java
// State Pattern
interface State {
    void handle(Context context);
}

class ConcreteStateA implements State {
    @Override
    public void handle(Context context) {
        System.out.println("State A handling");
        context.setState(new ConcreteStateB());
    }
}

class ConcreteStateB implements State {
    @Override
    public void handle(Context context) {
        System.out.println("State B handling");
        context.setState(new ConcreteStateA());
    }
}

class Context {
    private State state;
    
    public Context(State state) {
        this.state = state;
    }
    
    public void setState(State state) {
        this.state = state;
    }
    
    public void request() {
        state.handle(this);
    }
}

// Real-world example: Order states
enum OrderState {
    PENDING, PROCESSING, SHIPPED, DELIVERED
}

class Order {
    private OrderState state = OrderState.PENDING;
    
    public void process() {
        switch (state) {
            case PENDING:
                state = OrderState.PROCESSING;
                break;
            case PROCESSING:
                state = OrderState.SHIPPED;
                break;
            case SHIPPED:
                state = OrderState.DELIVERED;
                break;
        }
    }
}
```

**Related Terms**: [Strategy Pattern](#strategy-pattern), [Enum](#enum), [State Machine](#state-machine)

**Interview Tip**: Know the difference between State and Strategy patterns. Understand when to use State for state-dependent behavior.

---

### Command Pattern
**Category**: Behavioral

**Definition**: A pattern that encapsulates a request as an object, allowing parameterization of clients with different requests, queuing of requests, and logging of requests. Command pattern decouples sender from receiver.

**Practical Example**:
```java
// Command Pattern
interface Command {
    void execute();
    void undo();
}

class Light {
    public void turnOn() {
        System.out.println("Light is on");
    }
    
    public void turnOff() {
        System.out.println("Light is off");
    }
}

class LightOnCommand implements Command {
    private Light light;
    
    public LightOnCommand(Light light) {
        this.light = light;
    }
    
    @Override
    public void execute() {
        light.turnOn();
    }
    
    @Override
    public void undo() {
        light.turnOff();
    }
}

class RemoteControl {
    private Command command;
    private Stack<Command> history = new Stack<>();
    
    public void setCommand(Command command) {
        this.command = command;
    }
    
    public void pressButton() {
        command.execute();
        history.push(command);
    }
    
    public void pressUndo() {
        if (!history.isEmpty()) {
            history.pop().undo();
        }
    }
}

// Usage
Light light = new Light();
Command command = new LightOnCommand(light);
RemoteControl remote = new RemoteControl();
remote.setCommand(command);
remote.pressButton();
remote.pressUndo();
```

**Related Terms**: [Encapsulation](#encapsulation), [Undo/Redo](#undoredo), [Macro Commands](#macro-commands)

**Interview Tip**: Know the Command pattern and its applications. Understand how it enables undo/redo and command queuing.

---

### Iterator Pattern
**Category**: Behavioral

**Definition**: A pattern that provides a way to access elements of a collection sequentially without exposing its underlying representation. Iterator pattern decouples collection from traversal algorithms.

**Practical Example**:
```java
// Iterator Pattern
interface Iterator<T> {
    boolean hasNext();
    T next();
}

interface Iterable<T> {
    Iterator<T> iterator();
}

class Collection<T> implements Iterable<T> {
    private T[] items;
    
    public Collection(T[] items) {
        this.items = items;
    }
    
    @Override
    public Iterator<T> iterator() {
        return new CollectionIterator();
    }
    
    private class CollectionIterator implements Iterator<T> {
        private int index = 0;
        
        @Override
        public boolean hasNext() {
            return index < items.length;
        }
        
        @Override
        public T next() {
            return items[index++];
        }
    }
}

// Usage
String[] array = {"a", "b", "c"};
Collection<String> collection = new Collection<>(array);

for (String item : collection) {
    System.out.println(item);
}

// Java's built-in iterators
List<String> list = new ArrayList<>();
Iterator<String> iterator = list.iterator();
while (iterator.hasNext()) {
    System.out.println(iterator.next());
}
```

**Related Terms**: [Collection](#collection), [Enumeration](#enumeration), [Traversal](#traversal)

**Interview Tip**: Know the Iterator pattern. Understand Java's Iterator interface and for-each loop.

---

This glossary provides comprehensive coverage of design patterns essential for senior-level interviews.

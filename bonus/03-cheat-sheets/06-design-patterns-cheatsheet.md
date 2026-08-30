# Design Patterns - Cheat Sheet

Gang of Four design patterns with Java implementations and real-world examples.

## 🏗️ Creational Patterns

### Singleton Pattern
```java
// Eager initialization
public class Database {
    private static final Database instance = new Database();
    
    private Database() { }
    
    public static Database getInstance() {
        return instance;
    }
}

// Lazy initialization (thread-safe)
public class Database {
    private static Database instance;
    
    private Database() { }
    
    public static synchronized Database getInstance() {
        if (instance == null) {
            instance = new Database();
        }
        return instance;
    }
}

// Double-checked locking (optimized)
public class Database {
    private static volatile Database instance;
    
    private Database() { }
    
    public static Database getInstance() {
        if (instance == null) {
            synchronized (Database.class) {
                if (instance == null) {
                    instance = new Database();
                }
            }
        }
        return instance;
    }
}

// Bill Pugh Singleton (best practice)
public class Database {
    private Database() { }
    
    private static class SingletonHolder {
        static final Database INSTANCE = new Database();
    }
    
    public static Database getInstance() {
        return SingletonHolder.INSTANCE;
    }
}

// Enum Singleton (most robust)
public enum Database {
    INSTANCE;
    
    public void connect() {
        System.out.println("Connected");
    }
}

// Usage
Database db = Database.INSTANCE;
db.connect();
```

### Factory Pattern
```java
// Simple Factory
public class ShapeFactory {
    public static Shape createShape(String type) {
        switch (type) {
            case "circle":
                return new Circle();
            case "rectangle":
                return new Rectangle();
            case "triangle":
                return new Triangle();
            default:
                throw new IllegalArgumentException("Unknown shape: " + type);
        }
    }
}

// Usage
Shape shape = ShapeFactory.createShape("circle");

// Abstract Factory
public interface UIFactory {
    Button createButton();
    Checkbox createCheckbox();
}

public class WindowsUIFactory implements UIFactory {
    @Override
    public Button createButton() {
        return new WindowsButton();
    }
    
    @Override
    public Checkbox createCheckbox() {
        return new WindowsCheckbox();
    }
}

public class MacUIFactory implements UIFactory {
    @Override
    public Button createButton() {
        return new MacButton();
    }
    
    @Override
    public Checkbox createCheckbox() {
        return new MacCheckbox();
    }
}

// Usage
UIFactory factory = new WindowsUIFactory();
Button button = factory.createButton();
Checkbox checkbox = factory.createCheckbox();
```

### Builder Pattern
```java
// Traditional approach
public class User {
    private final String name;
    private final String email;
    private final int age;
    private final String phone;
    
    public User(String name, String email, int age, String phone) {
        this.name = name;
        this.email = email;
        this.age = age;
        this.phone = phone;
    }
}

// Builder pattern
public class User {
    private final String name;
    private final String email;
    private final int age;
    private final String phone;
    
    private User(Builder builder) {
        this.name = builder.name;
        this.email = builder.email;
        this.age = builder.age;
        this.phone = builder.phone;
    }
    
    public static class Builder {
        private String name;
        private String email;
        private int age;
        private String phone;
        
        public Builder name(String name) {
            this.name = name;
            return this;
        }
        
        public Builder email(String email) {
            this.email = email;
            return this;
        }
        
        public Builder age(int age) {
            this.age = age;
            return this;
        }
        
        public Builder phone(String phone) {
            this.phone = phone;
            return this;
        }
        
        public User build() {
            return new User(this);
        }
    }
}

// Usage
User user = new User.Builder()
    .name("John")
    .email("john@example.com")
    .age(30)
    .phone("123-456-7890")
    .build();

// With Lombok
@Builder
public class User {
    private String name;
    private String email;
    private int age;
    private String phone;
}

User user = User.builder()
    .name("John")
    .email("john@example.com")
    .build();
```

### Prototype Pattern
```java
public abstract class Shape implements Cloneable {
    protected String color;
    
    public abstract void draw();
    
    @Override
    public Shape clone() {
        try {
            return (Shape) super.clone();
        } catch (CloneNotSupportedException e) {
            return null;
        }
    }
}

public class Circle extends Shape {
    @Override
    public void draw() {
        System.out.println("Drawing circle");
    }
}

// Usage
Circle original = new Circle();
original.color = "red";

Circle copy = (Circle) original.clone();
copy.color = "blue";

// original and copy are different objects
```

## 🔗 Structural Patterns

### Adapter Pattern
```java
// Incompatible interface
public interface LegacyPaymentProcessor {
    void processPayment(double amount);
}

// New interface
public interface PaymentGateway {
    void pay(BigDecimal amount);
}

// Adapter
public class PaymentAdapter implements PaymentGateway {
    private LegacyPaymentProcessor processor;
    
    public PaymentAdapter(LegacyPaymentProcessor processor) {
        this.processor = processor;
    }
    
    @Override
    public void pay(BigDecimal amount) {
        processor.processPayment(amount.doubleValue());
    }
}

// Usage
LegacyPaymentProcessor legacy = new LegacyPaymentProcessor() {
    @Override
    public void processPayment(double amount) {
        System.out.println("Processing: " + amount);
    }
};

PaymentGateway gateway = new PaymentAdapter(legacy);
gateway.pay(new BigDecimal("100.00"));
```

### Decorator Pattern
```java
// Component interface
public interface Coffee {
    double cost();
    String description();
}

// Concrete component
public class SimpleCoffee implements Coffee {
    @Override
    public double cost() {
        return 1.0;
    }
    
    @Override
    public String description() {
        return "Simple coffee";
    }
}

// Decorator
public abstract class CoffeeDecorator implements Coffee {
    protected Coffee coffee;
    
    public CoffeeDecorator(Coffee coffee) {
        this.coffee = coffee;
    }
}

// Concrete decorators
public class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) {
        super(coffee);
    }
    
    @Override
    public double cost() {
        return coffee.cost() + 0.5;
    }
    
    @Override
    public String description() {
        return coffee.description() + ", milk";
    }
}

public class SugarDecorator extends CoffeeDecorator {
    public SugarDecorator(Coffee coffee) {
        super(coffee);
    }
    
    @Override
    public double cost() {
        return coffee.cost() + 0.2;
    }
    
    @Override
    public String description() {
        return coffee.description() + ", sugar";
    }
}

// Usage
Coffee coffee = new SimpleCoffee();
coffee = new MilkDecorator(coffee);
coffee = new SugarDecorator(coffee);

System.out.println(coffee.description());  // Simple coffee, milk, sugar
System.out.println(coffee.cost());         // 1.7
```

### Facade Pattern
```java
// Complex subsystem
public class DatabaseConnection {
    public void connect() { System.out.println("Connecting to DB"); }
    public void disconnect() { System.out.println("Disconnecting from DB"); }
}

public class CacheManager {
    public void initialize() { System.out.println("Initializing cache"); }
    public void clear() { System.out.println("Clearing cache"); }
}

public class Logger {
    public void log(String message) { System.out.println("Log: " + message); }
}

// Facade
public class ApplicationFacade {
    private DatabaseConnection db;
    private CacheManager cache;
    private Logger logger;
    
    public ApplicationFacade() {
        this.db = new DatabaseConnection();
        this.cache = new CacheManager();
        this.logger = new Logger();
    }
    
    public void startup() {
        logger.log("Starting application");
        db.connect();
        cache.initialize();
    }
    
    public void shutdown() {
        logger.log("Shutting down application");
        cache.clear();
        db.disconnect();
    }
}

// Usage
ApplicationFacade app = new ApplicationFacade();
app.startup();
app.shutdown();
```

### Proxy Pattern
```java
// Subject interface
public interface Image {
    void display();
}

// Real subject
public class RealImage implements Image {
    private String filename;
    
    public RealImage(String filename) {
        this.filename = filename;
        loadImage();
    }
    
    private void loadImage() {
        System.out.println("Loading image: " + filename);
    }
    
    @Override
    public void display() {
        System.out.println("Displaying image: " + filename);
    }
}

// Proxy
public class ImageProxy implements Image {
    private String filename;
    private RealImage realImage;
    
    public ImageProxy(String filename) {
        this.filename = filename;
    }
    
    @Override
    public void display() {
        if (realImage == null) {
            realImage = new RealImage(filename);
        }
        realImage.display();
    }
}

// Usage
Image image = new ImageProxy("photo.jpg");
image.display();  // Loads and displays
image.display();  // Just displays (already loaded)
```

## 🎭 Behavioral Patterns

### Observer Pattern
```java
// Observer interface
public interface Observer {
    void update(String message);
}

// Subject
public class EventPublisher {
    private List<Observer> observers = new ArrayList<>();
    
    public void subscribe(Observer observer) {
        observers.add(observer);
    }
    
    public void unsubscribe(Observer observer) {
        observers.remove(observer);
    }
    
    public void publish(String message) {
        for (Observer observer : observers) {
            observer.update(message);
        }
    }
}

// Concrete observer
public class EmailNotifier implements Observer {
    @Override
    public void update(String message) {
        System.out.println("Email sent: " + message);
    }
}

public class SMSNotifier implements Observer {
    @Override
    public void update(String message) {
        System.out.println("SMS sent: " + message);
    }
}

// Usage
EventPublisher publisher = new EventPublisher();
publisher.subscribe(new EmailNotifier());
publisher.subscribe(new SMSNotifier());
publisher.publish("Important event");
```

### Strategy Pattern
```java
// Strategy interface
public interface PaymentStrategy {
    void pay(BigDecimal amount);
}

// Concrete strategies
public class CreditCardPayment implements PaymentStrategy {
    @Override
    public void pay(BigDecimal amount) {
        System.out.println("Paying " + amount + " via credit card");
    }
}

public class PayPalPayment implements PaymentStrategy {
    @Override
    public void pay(BigDecimal amount) {
        System.out.println("Paying " + amount + " via PayPal");
    }
}

public class BitcoinPayment implements PaymentStrategy {
    @Override
    public void pay(BigDecimal amount) {
        System.out.println("Paying " + amount + " via Bitcoin");
    }
}

// Context
public class PaymentProcessor {
    private PaymentStrategy strategy;
    
    public PaymentProcessor(PaymentStrategy strategy) {
        this.strategy = strategy;
    }
    
    public void processPayment(BigDecimal amount) {
        strategy.pay(amount);
    }
}

// Usage
PaymentProcessor processor = new PaymentProcessor(new CreditCardPayment());
processor.processPayment(new BigDecimal("100.00"));

processor = new PaymentProcessor(new PayPalPayment());
processor.processPayment(new BigDecimal("50.00"));
```

### State Pattern
```java
// State interface
public interface OrderState {
    void process(Order order);
}

// Concrete states
public class PendingState implements OrderState {
    @Override
    public void process(Order order) {
        System.out.println("Processing pending order");
        order.setState(new ShippedState());
    }
}

public class ShippedState implements OrderState {
    @Override
    public void process(Order order) {
        System.out.println("Order shipped");
        order.setState(new DeliveredState());
    }
}

public class DeliveredState implements OrderState {
    @Override
    public void process(Order order) {
        System.out.println("Order delivered");
    }
}

// Context
public class Order {
    private OrderState state;
    
    public Order() {
        this.state = new PendingState();
    }
    
    public void setState(OrderState state) {
        this.state = state;
    }
    
    public void process() {
        state.process(this);
    }
}

// Usage
Order order = new Order();
order.process();  // Pending -> Shipped
order.process();  // Shipped -> Delivered
order.process();  // Delivered (no change)
```

### Template Method Pattern
```java
// Abstract class with template method
public abstract class DataProcessor {
    
    public final void process(String data) {
        String validated = validate(data);
        String transformed = transform(validated);
        save(transformed);
    }
    
    protected abstract String validate(String data);
    protected abstract String transform(String data);
    protected abstract void save(String data);
}

// Concrete implementations
public class CSVProcessor extends DataProcessor {
    @Override
    protected String validate(String data) {
        System.out.println("Validating CSV");
        return data;
    }
    
    @Override
    protected String transform(String data) {
        System.out.println("Transforming CSV");
        return data.toUpperCase();
    }
    
    @Override
    protected void save(String data) {
        System.out.println("Saving CSV: " + data);
    }
}

public class JSONProcessor extends DataProcessor {
    @Override
    protected String validate(String data) {
        System.out.println("Validating JSON");
        return data;
    }
    
    @Override
    protected String transform(String data) {
        System.out.println("Transforming JSON");
        return data.toLowerCase();
    }
    
    @Override
    protected void save(String data) {
        System.out.println("Saving JSON: " + data);
    }
}

// Usage
DataProcessor processor = new CSVProcessor();
processor.process("data");
```

### Command Pattern
```java
// Command interface
public interface Command {
    void execute();
    void undo();
}

// Concrete commands
public class OpenFileCommand implements Command {
    private String filename;
    
    public OpenFileCommand(String filename) {
        this.filename = filename;
    }
    
    @Override
    public void execute() {
        System.out.println("Opening file: " + filename);
    }
    
    @Override
    public void undo() {
        System.out.println("Closing file: " + filename);
    }
}

// Invoker
public class Editor {
    private Stack<Command> history = new Stack<>();
    
    public void executeCommand(Command command) {
        command.execute();
        history.push(command);
    }
    
    public void undo() {
        if (!history.isEmpty()) {
            history.pop().undo();
        }
    }
}

// Usage
Editor editor = new Editor();
editor.executeCommand(new OpenFileCommand("file.txt"));
editor.undo();
```

## 📊 Pattern Selection Guide

| Pattern | Problem | Solution |
|---------|---------|----------|
| Singleton | Single instance needed | Private constructor, static instance |
| Factory | Object creation | Encapsulate creation logic |
| Builder | Complex object construction | Step-by-step construction |
| Adapter | Incompatible interfaces | Wrapper class |
| Decorator | Add responsibilities | Wrapper with same interface |
| Facade | Complex subsystem | Simplified interface |
| Observer | Loose coupling | Publish-subscribe mechanism |
| Strategy | Multiple algorithms | Encapsulate algorithms |
| State | Object behavior changes | State-specific classes |
| Template Method | Algorithm skeleton | Abstract methods |

## 🎓 Interview Tips

- **Know the intent**: Why use this pattern?
- **Understand trade-offs**: Complexity vs flexibility
- **Real-world examples**: Spring uses many patterns (Singleton, Factory, Proxy)
- **When NOT to use**: Over-engineering with patterns
- **Recognize patterns**: Identify patterns in existing code

---

**Related**: See Java 8+ Features cheat sheet for functional programming alternatives to some patterns

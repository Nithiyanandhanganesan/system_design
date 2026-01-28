# SOLID Principles

## Overview
SOLID is an acronym for five fundamental object-oriented design principles that make software more maintainable, flexible, and scalable. These principles were promoted by Robert C. Martin (Uncle Bob).

## The Five Principles

### 1. Single Responsibility Principle (SRP)
**"A class should have only one reason to change."**

#### Problem
Classes that handle multiple responsibilities become harder to maintain and test.

#### Solution
Each class should have only one job or responsibility.

**Bad Example:**
```java
// Violates SRP - handles both user data and email sending
public class User {
    private String name;
    private String email;
    
    public User(String name, String email) {
        this.name = name;
        this.email = email;
    }
    
    public void save() {
        // Database logic
        Database.save(this);
    }
    
    public void sendEmail(String message) {
        // Email sending logic
        EmailService.send(this.email, message);
    }
    
    // Getters and setters
}
```

**Good Example:**
```java
// User class only handles user data
public class User {
    private String name;
    private String email;
    
    public User(String name, String email) {
        this.name = name;
        this.email = email;
    }
    
    // Getters and setters only
}

// Separate class for user persistence
public class UserRepository {
    public void save(User user) {
        Database.save(user);
    }
    
    public User findById(Long id) {
        return Database.findById(id);
    }
}

// Separate class for email functionality
public class EmailService {
    public void sendEmail(User user, String message) {
        // Email sending logic
        send(user.getEmail(), message);
    }
}
```

### 2. Open/Closed Principle (OCP)
**"Software entities should be open for extension but closed for modification."**

#### Problem
Modifying existing code can introduce bugs and break existing functionality.

#### Solution
Design classes that can be extended without modifying existing code.

**Example using Strategy Pattern:**
```java
// Bad - Need to modify existing code for new discount types
public class DiscountCalculator {
    public double calculateDiscount(String customerType, double amount) {
        switch (customerType) {
            case "REGULAR":
                return amount * 0.05;
            case "PREMIUM":
                return amount * 0.10;
            case "VIP":
                return amount * 0.20;
            default:
                return 0;
        }
    }
}

// Good - Open for extension, closed for modification
public interface DiscountStrategy {
    double calculateDiscount(double amount);
}

public class RegularCustomerDiscount implements DiscountStrategy {
    public double calculateDiscount(double amount) {
        return amount * 0.05;
    }
}

public class PremiumCustomerDiscount implements DiscountStrategy {
    public double calculateDiscount(double amount) {
        return amount * 0.10;
    }
}

public class VipCustomerDiscount implements DiscountStrategy {
    public double calculateDiscount(double amount) {
        return amount * 0.20;
    }
}

// Can add new discount types without modifying existing code
public class StudentDiscount implements DiscountStrategy {
    public double calculateDiscount(double amount) {
        return amount * 0.15;
    }
}

public class DiscountCalculator {
    public double calculateDiscount(DiscountStrategy strategy, double amount) {
        return strategy.calculateDiscount(amount);
    }
}
```

### 3. Liskov Substitution Principle (LSP)
**"Objects of a superclass should be replaceable with objects of a subclass without breaking the application."**

#### What LSP Really Means
LSP is about **behavioral subtyping**. It's not enough for a subclass to just compile - it must behave correctly when used in place of its parent. The subclass should:
- **Strengthen postconditions** (can do more than parent promises)
- **Weaken preconditions** (accept more inputs than parent requires)
- **Maintain invariants** (preserve the core behavior contracts)

#### Problem
Subclasses that don't properly implement their parent's contract can break code that relies on the parent type. This leads to runtime errors, unexpected behavior, and violated assumptions.

#### LSP Violation Signs
1. **instanceof checks** before calling methods
2. **Overridden methods that do nothing** or throw exceptions
3. **Behavioral changes** that break client expectations
4. **Subclasses that strengthen preconditions** (more restrictive inputs)
5. **Subclasses that weaken postconditions** (less guarantees)

#### Detailed Java Examples

**Example 1: Classic Rectangle-Square Problem**
```java
// Bad - Square violates LSP when extending Rectangle
class Rectangle {
    protected int width;
    protected int height;
    
    public void setWidth(int width) {
        this.width = width;
    }
    
    public void setHeight(int height) {
        this.height = height;
    }
    
    public int getWidth() { return width; }
    public int getHeight() { return height; }
    public int getArea() { return width * height; }
}

class Square extends Rectangle {
    @Override
    public void setWidth(int width) {
        this.width = width;
        this.height = width;  // Violates LSP - changes both dimensions
    }
    
    @Override
    public void setHeight(int height) {
        this.width = height;   // Violates LSP - changes both dimensions
        this.height = height;
    }
}

// This client code breaks with Square!
public class AreaCalculator {
    public static void demonstrateProblem(Rectangle rect) {
        rect.setWidth(5);
        rect.setHeight(4);
        
        // Client expects: width=5, height=4, area=20
        // With Rectangle: works correctly (area = 20)
        // With Square: BREAKS! width=4, height=4, area=16
        
        System.out.println("Width: " + rect.getWidth());    // Expected: 5, Square gives: 4
        System.out.println("Height: " + rect.getHeight());  // Expected: 4, Square gives: 4
        System.out.println("Area: " + rect.getArea());      // Expected: 20, Square gives: 16
        
        // Assertion fails with Square!
        assert rect.getArea() == 20 : "Area calculation is wrong!";
    }
}
```

**Why This Violates LSP:**
- **Square changes the behavior contract** - setting width shouldn't affect height
- **Client code assumptions are broken** - width and height should be independent
- **Square strengthens postconditions** - both dimensions must be equal (parent doesn't require this)

**Example 2: Better Solution Using Proper Abstraction**
```java
// Good - Proper abstraction that doesn't violate LSP
abstract class Shape {
    public abstract int getArea();
    public abstract int getPerimeter();
}

class Rectangle extends Shape {
    private final int width;
    private final int height;
    
    public Rectangle(int width, int height) {
        this.width = width;
        this.height = height;
    }
    
    public int getWidth() { return width; }
    public int getHeight() { return height; }
    
    @Override
    public int getArea() {
        return width * height;
    }
    
    @Override
    public int getPerimeter() {
        return 2 * (width + height);
    }
}

class Square extends Shape {
    private final int side;
    
    public Square(int side) {
        this.side = side;
    }
    
    public int getSide() { return side; }
    
    @Override
    public int getArea() {
        return side * side;
    }
    
    @Override
    public int getPerimeter() {
        return 4 * side;
    }
}

// Client code works with any Shape
public class ShapeProcessor {
    public static void processShapes(List<Shape> shapes) {
        for (Shape shape : shapes) {
            // Works correctly with both Rectangle and Square
            System.out.println("Area: " + shape.getArea());
            System.out.println("Perimeter: " + shape.getPerimeter());
        }
    }
}
```

**Example 3: Bird Flying Example**
```java
// Bad - Not all birds can fly, violates LSP
abstract class Bird {
    public abstract void fly();
    public abstract void eat();
}

class Sparrow extends Bird {
    @Override
    public void fly() {
        System.out.println("Sparrow flying high!");
    }
    
    @Override
    public void eat() {
        System.out.println("Sparrow eating seeds");
    }
}

class Penguin extends Bird {
    @Override
    public void fly() {
        // Violates LSP - throws exception or does nothing
        throw new UnsupportedOperationException("Penguins can't fly!");
    }
    
    @Override
    public void eat() {
        System.out.println("Penguin eating fish");
    }
}

// Client code breaks with Penguin
public class BirdTrainer {
    public static void trainBirds(List<Bird> birds) {
        for (Bird bird : birds) {
            bird.eat();     // Works for all
            bird.fly();     // CRASHES with Penguin!
        }
    }
}
```

**Good Solution - Interface Segregation + Proper Hierarchy:**
```java
// Good - Proper abstraction respects LSP
abstract class Bird {
    public abstract void eat();
    public abstract void sleep();
}

interface Flyable {
    void fly();
    int getFlightSpeed();
}

interface Swimmable {
    void swim();
    int getSwimSpeed();
}

class Sparrow extends Bird implements Flyable {
    @Override
    public void eat() {
        System.out.println("Sparrow eating seeds");
    }
    
    @Override
    public void sleep() {
        System.out.println("Sparrow sleeping in nest");
    }
    
    @Override
    public void fly() {
        System.out.println("Sparrow flying high!");
    }
    
    @Override
    public int getFlightSpeed() {
        return 24; // mph
    }
}

class Penguin extends Bird implements Swimmable {
    @Override
    public void eat() {
        System.out.println("Penguin eating fish");
    }
    
    @Override
    public void sleep() {
        System.out.println("Penguin sleeping on ice");
    }
    
    @Override
    public void swim() {
        System.out.println("Penguin swimming gracefully!");
    }
    
    @Override
    public int getSwimSpeed() {
        return 15; // mph
    }
}

// Duck can both fly and swim
class Duck extends Bird implements Flyable, Swimmable {
    @Override
    public void eat() {
        System.out.println("Duck eating aquatic plants");
    }
    
    @Override
    public void sleep() {
        System.out.println("Duck sleeping on water");
    }
    
    @Override
    public void fly() {
        System.out.println("Duck flying to migrate");
    }
    
    @Override
    public int getFlightSpeed() {
        return 40; // mph
    }
    
    @Override
    public void swim() {
        System.out.println("Duck swimming and diving");
    }
    
    @Override
    public int getSwimSpeed() {
        return 8; // mph
    }
}

// Client code that respects capabilities
public class WildlifeSimulator {
    public static void simulateBirds(List<Bird> birds) {
        for (Bird bird : birds) {
            bird.eat();  // All birds can eat
            bird.sleep(); // All birds can sleep
        }
    }
    
    public static void simulateFlight(List<Flyable> flyingCreatures) {
        for (Flyable creature : flyingCreatures) {
            creature.fly(); // Only flying creatures
            System.out.println("Speed: " + creature.getFlightSpeed() + " mph");
        }
    }
    
    public static void simulateSwimming(List<Swimmable> swimmingCreatures) {
        for (Swimmable creature : swimmingCreatures) {
            creature.swim(); // Only swimming creatures
            System.out.println("Speed: " + creature.getSwimSpeed() + " mph");
        }
    }
}
```

#### LSP Rules to Follow

1. **Preconditions cannot be strengthened** by subclasses
   ```java
   // Parent accepts any positive number
   void processPayment(double amount) {
       if (amount <= 0) throw new IllegalArgumentException();
       // process payment
   }
   
   // BAD: Subclass strengthens precondition
   @Override
   void processPayment(double amount) {
       if (amount <= 0 || amount > 1000) throw new IllegalArgumentException();
       // process payment
   }
   ```

2. **Postconditions cannot be weakened** by subclasses
   ```java
   // Parent guarantees non-null return
   public List<String> getItems() {
       List<String> items = fetchItems();
       return items != null ? items : new ArrayList<>();
   }
   
   // BAD: Subclass weakens postcondition by returning null
   @Override
   public List<String> getItems() {
       return fetchItems(); // Might return null!
   }
   ```

3. **Invariants must be preserved** by subclasses
   ```java
   // Parent maintains: balance >= 0
   class BankAccount {
       protected double balance;
       
       public void withdraw(double amount) {
           if (balance - amount < 0) {
               throw new InsufficientFundsException();
           }
           balance -= amount;
       }
   }
   
   // BAD: Subclass breaks invariant
   class OverdraftAccount extends BankAccount {
       @Override
       public void withdraw(double amount) {
           balance -= amount; // Allows negative balance!
       }
   }
   ```

#### Key Takeaways
- **LSP is about behavior, not just structure**
- **Subclasses must honor the contracts of their parents**
- **Design your inheritance hierarchies carefully**
- **Use composition or interfaces when inheritance doesn't fit**
- **Test your substitutions to ensure they work correctly**

### 4. Interface Segregation Principle (ISP)
**"Clients should not be forced to depend on interfaces they don't use."**

#### Problem
Large interfaces force classes to implement methods they don't need.

#### Solution
Create specific, focused interfaces rather than large, general-purpose ones.

**Bad Example:**
```typescript
// Bad - Large interface forces unnecessary implementations
interface Worker {
    work(): void;
    eat(): void;
    sleep(): void;
}

class HumanWorker implements Worker {
    work(): void {
        console.log("Human working");
    }
    
    eat(): void {
        console.log("Human eating");
    }
    
    sleep(): void {
        console.log("Human sleeping");
    }
}

class RobotWorker implements Worker {
    work(): void {
        console.log("Robot working");
    }
    
    eat(): void {
        // Robots don't eat - unnecessary implementation
        throw new Error("Robots don't eat");
    }
    
    sleep(): void {
        // Robots don't sleep - unnecessary implementation
        throw new Error("Robots don't sleep");
    }
}
```

**Good Example:**
```typescript
// Good - Segregated interfaces
interface Workable {
    work(): void;
}

interface Eatable {
    eat(): void;
}

interface Sleepable {
    sleep(): void;
}

class HumanWorker implements Workable, Eatable, Sleepable {
    work(): void {
        console.log("Human working");
    }
    
    eat(): void {
        console.log("Human eating");
    }
    
    sleep(): void {
        console.log("Human sleeping");
    }
}

class RobotWorker implements Workable {
    work(): void {
        console.log("Robot working");
    }
    // Only implements what it needs
}
```

**Java Example:**
```java
// Bad - Fat interface
interface Document {
    void open();
    void save();
    void print();
    void fax();
    void scan();
}

// Good - Segregated interfaces
interface Readable {
    void open();
}

interface Writable {
    void save();
}

interface Printable {
    void print();
}

interface Faxable {
    void fax();
}

interface Scannable {
    void scan();
}

// Classes implement only what they need
class PDFDocument implements Readable, Writable, Printable {
    public void open() { /* PDF opening logic */ }
    public void save() { /* PDF saving logic */ }
    public void print() { /* PDF printing logic */ }
}

class MultiFunctionPrinter implements Printable, Faxable, Scannable {
    public void print() { /* Printing logic */ }
    public void fax() { /* Faxing logic */ }
    public void scan() { /* Scanning logic */ }
}
```

### 5. Dependency Inversion Principle (DIP)
**"High-level modules should not depend on low-level modules. Both should depend on abstractions."**

#### Problem
Direct dependencies on concrete classes make code rigid and hard to test.

#### Solution
Depend on abstractions (interfaces/abstract classes) rather than concrete implementations.

**Bad Example:**
```python
# Bad - High-level class depends on low-level implementation
class MySQLDatabase:
    def save(self, data):
        print(f"Saving {data} to MySQL database")

class UserService:
    def __init__(self):
        self.database = MySQLDatabase()  # Direct dependency
    
    def create_user(self, user_data):
        # Business logic
        processed_data = self.process_user_data(user_data)
        self.database.save(processed_data)
```

**Good Example:**
```python
# Good - Depends on abstraction
from abc import ABC, abstractmethod

class DatabaseInterface(ABC):
    @abstractmethod
    def save(self, data):
        pass

class MySQLDatabase(DatabaseInterface):
    def save(self, data):
        print(f"Saving {data} to MySQL database")

class PostgreSQLDatabase(DatabaseInterface):
    def save(self, data):
        print(f"Saving {data} to PostgreSQL database")

class UserService:
    def __init__(self, database: DatabaseInterface):
        self.database = database  # Depends on abstraction
    
    def create_user(self, user_data):
        processed_data = self.process_user_data(user_data)
        self.database.save(processed_data)

# Usage with Dependency Injection
mysql_db = MySQLDatabase()
user_service = UserService(mysql_db)

# Easy to switch implementations
postgres_db = PostgreSQLDatabase()
user_service_postgres = UserService(postgres_db)
```

**Spring Boot Example:**
```java
// Interface
public interface PaymentProcessor {
    PaymentResult process(PaymentRequest request);
}

// Implementations
@Component
public class StripePaymentProcessor implements PaymentProcessor {
    @Override
    public PaymentResult process(PaymentRequest request) {
        // Stripe-specific logic
        return new PaymentResult("stripe_transaction_id");
    }
}

@Component
public class PayPalPaymentProcessor implements PaymentProcessor {
    @Override
    public PaymentResult process(PaymentRequest request) {
        // PayPal-specific logic
        return new PaymentResult("paypal_transaction_id");
    }
}

// High-level service depends on abstraction
@Service
public class OrderService {
    private final PaymentProcessor paymentProcessor;
    
    // Constructor injection
    public OrderService(@Qualifier("stripePaymentProcessor") PaymentProcessor paymentProcessor) {
        this.paymentProcessor = paymentProcessor;
    }
    
    public void processOrder(Order order) {
        // Business logic
        PaymentRequest request = createPaymentRequest(order);
        PaymentResult result = paymentProcessor.process(request);
        
        if (result.isSuccessful()) {
            completeOrder(order);
        }
    }
}
```

## Benefits of Following SOLID Principles

1. **Maintainability**: Code is easier to modify and extend
2. **Testability**: Individual components can be easily unit tested
3. **Flexibility**: Easy to add new features without breaking existing code
4. **Reusability**: Components can be reused in different contexts
5. **Reduced Coupling**: Components are loosely coupled and highly cohesive

## Common Anti-patterns

### God Object (Violates SRP)
```java
// Anti-pattern: One class doing everything
public class OrderManager {
    public void processOrder() { }
    public void calculateTax() { }
    public void sendEmail() { }
    public void updateInventory() { }
    public void generateInvoice() { }
    public void processPayment() { }
    // ... many more responsibilities
}
```

### Shotgun Surgery (Result of not following OCP)
```java
// Anti-pattern: Changes require modifications in multiple places
public class PriceCalculator {
    public double calculatePrice(String customerType, double basePrice) {
        // Scattered logic that needs to be changed everywhere
        // when adding new customer types
    }
}
```

## Best Practices

1. **Start with Interfaces**: Design interfaces before implementations
2. **Favor Composition over Inheritance**: Use composition to achieve flexibility
3. **Use Dependency Injection**: Let frameworks handle object creation and wiring
4. **Keep Interfaces Small**: Follow ISP by creating focused interfaces
5. **Write Tests First**: TDD naturally leads to SOLID design

## Related Patterns
- **Strategy Pattern**: Implements OCP
- **Decorator Pattern**: Implements OCP
- **Factory Pattern**: Implements DIP
- **Observer Pattern**: Implements OCP and DIP
- **Adapter Pattern**: Helps with LSP compliance
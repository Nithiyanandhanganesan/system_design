# Builder Pattern - Complete Guide

## Introduction

The **Builder Pattern** is a creational design pattern that provides a flexible solution for constructing complex objects step by step. It separates the construction of a complex object from its representation, allowing the same construction process to create different representations.

## Definition

> **Builder Pattern**: Separates the construction of a complex object from its representation so that the same construction process can create different representations.

## Problem Solved

- **Complex Object Construction**: Objects with many parameters or complex initialization
- **Telescoping Constructor Problem**: Avoiding constructors with many parameters
- **Immutable Objects**: Create immutable objects step by step
- **Optional Parameters**: Handle objects with many optional parameters
- **Readable Code**: Make object creation more readable and maintainable

## Structure

```
┌─────────────────┐    ┌─────────────────┐
│    Director     │───▶│     Builder     │
├─────────────────┤    ├─────────────────┤
│ +construct()    │    │ +buildPartA()   │
└─────────────────┘    │ +buildPartB()   │
                       │ +getResult()    │
                       └─────────────────┘
                                △
                                │
                    ┌─────────────────┐
                    │ ConcreteBuilder │
                    ├─────────────────┤
                    │ +buildPartA()   │
                    │ +buildPartB()   │
                    │ +getResult()    │
                    └─────────────────┘
```

## Basic Implementation

### Computer Builder Example

```java
/**
 * Product class - Complex object to be built
 */
public class Computer {
    
    // Required parameters
    private final String cpu;
    private final String memory;
    
    // Optional parameters
    private final String storage;
    private final String graphics;
    private final String powerSupply;
    private final String motherboard;
    private final String coolingSystem;
    private final boolean wifi;
    private final boolean bluetooth;
    private final int usbPorts;
    private final String operatingSystem;
    
    // Private constructor - only builder can create instances
    private Computer(ComputerBuilder builder) {
        this.cpu = builder.cpu;
        this.memory = builder.memory;
        this.storage = builder.storage;
        this.graphics = builder.graphics;
        this.powerSupply = builder.powerSupply;
        this.motherboard = builder.motherboard;
        this.coolingSystem = builder.coolingSystem;
        this.wifi = builder.wifi;
        this.bluetooth = builder.bluetooth;
        this.usbPorts = builder.usbPorts;
        this.operatingSystem = builder.operatingSystem;
    }
    
    // Getters
    public String getCpu() { return cpu; }
    public String getMemory() { return memory; }
    public String getStorage() { return storage; }
    public String getGraphics() { return graphics; }
    public String getPowerSupply() { return powerSupply; }
    public String getMotherboard() { return motherboard; }
    public String getCoolingSystem() { return coolingSystem; }
    public boolean hasWifi() { return wifi; }
    public boolean hasBluetooth() { return bluetooth; }
    public int getUsbPorts() { return usbPorts; }
    public String getOperatingSystem() { return operatingSystem; }
    
    @Override
    public String toString() {
        return String.format(
            "Computer{cpu='%s', memory='%s', storage='%s', graphics='%s', " +
            "powerSupply='%s', motherboard='%s', coolingSystem='%s', " +
            "wifi=%s, bluetooth=%s, usbPorts=%d, os='%s'}",
            cpu, memory, storage, graphics, powerSupply, motherboard, 
            coolingSystem, wifi, bluetooth, usbPorts, operatingSystem
        );
    }
    
    /**
     * Static nested builder class
     */
    public static class ComputerBuilder {
        
        // Required parameters
        private final String cpu;
        private final String memory;
        
        // Optional parameters with default values
        private String storage = "256GB SSD";
        private String graphics = "Integrated Graphics";
        private String powerSupply = "500W";
        private String motherboard = "Standard ATX";
        private String coolingSystem = "Stock Cooler";
        private boolean wifi = false;
        private boolean bluetooth = false;
        private int usbPorts = 4;
        private String operatingSystem = "No OS";
        
        // Constructor with required parameters
        public ComputerBuilder(String cpu, String memory) {
            this.cpu = requireNonNull(cpu, "CPU cannot be null");
            this.memory = requireNonNull(memory, "Memory cannot be null");
        }
        
        public ComputerBuilder storage(String storage) {
            this.storage = requireNonNull(storage, "Storage cannot be null");
            return this;
        }
        
        public ComputerBuilder graphics(String graphics) {
            this.graphics = requireNonNull(graphics, "Graphics cannot be null");
            return this;
        }
        
        public ComputerBuilder powerSupply(String powerSupply) {
            this.powerSupply = requireNonNull(powerSupply, "Power supply cannot be null");
            return this;
        }
        
        public ComputerBuilder motherboard(String motherboard) {
            this.motherboard = requireNonNull(motherboard, "Motherboard cannot be null");
            return this;
        }
        
        public ComputerBuilder coolingSystem(String coolingSystem) {
            this.coolingSystem = requireNonNull(coolingSystem, "Cooling system cannot be null");
            return this;
        }
        
        public ComputerBuilder enableWifi() {
            this.wifi = true;
            return this;
        }
        
        public ComputerBuilder enableBluetooth() {
            this.bluetooth = true;
            return this;
        }
        
        public ComputerBuilder usbPorts(int usbPorts) {
            if (usbPorts < 0) {
                throw new IllegalArgumentException("USB ports cannot be negative");
            }
            this.usbPorts = usbPorts;
            return this;
        }
        
        public ComputerBuilder operatingSystem(String operatingSystem) {
            this.operatingSystem = requireNonNull(operatingSystem, "Operating system cannot be null");
            return this;
        }
        
        // Build method
        public Computer build() {
            validate();
            return new Computer(this);
        }
        
        private void validate() {
            // Business logic validation
            if (graphics.contains("RTX") && !powerSupply.contains("750W") && !powerSupply.contains("850W")) {
                throw new IllegalStateException("High-end graphics cards require at least 750W power supply");
            }
            
            if (cpu.contains("i9") && coolingSystem.equals("Stock Cooler")) {
                System.out.println("Warning: High-end CPU with stock cooler may cause thermal throttling");
            }
        }
        
        private static <T> T requireNonNull(T obj, String message) {
            if (obj == null) {
                throw new IllegalArgumentException(message);
            }
            return obj;
        }
    }
}
```

## Advanced Builder with Method Chaining

### Restaurant Order Builder

```java
/**
 * Order class representing a complex restaurant order
 */
public class RestaurantOrder {
    
    private final String customerName;
    private final String customerPhone;
    private final List<OrderItem> items;
    private final OrderType orderType;
    private final String deliveryAddress;
    private final LocalDateTime orderTime;
    private final PaymentMethod paymentMethod;
    private final String specialInstructions;
    private final boolean expedited;
    private final double discount;
    private final double tip;
    private final double totalAmount;
    
    private RestaurantOrder(RestaurantOrderBuilder builder) {
        this.customerName = builder.customerName;
        this.customerPhone = builder.customerPhone;
        this.items = Collections.unmodifiableList(new ArrayList<>(builder.items));
        this.orderType = builder.orderType;
        this.deliveryAddress = builder.deliveryAddress;
        this.orderTime = builder.orderTime;
        this.paymentMethod = builder.paymentMethod;
        this.specialInstructions = builder.specialInstructions;
        this.expedited = builder.expedited;
        this.discount = builder.discount;
        this.tip = builder.tip;
        this.totalAmount = calculateTotal(builder);
    }
    
    private double calculateTotal(RestaurantOrderBuilder builder) {
        double itemsTotal = builder.items.stream()
            .mapToDouble(item -> item.getPrice() * item.getQuantity())
            .sum();
        
        double afterDiscount = itemsTotal - builder.discount;
        double withTip = afterDiscount + builder.tip;
        
        if (builder.expedited) {
            withTip += itemsTotal * 0.15; // 15% expedite fee
        }
        
        if (builder.orderType == OrderType.DELIVERY) {
            withTip += 5.0; // Delivery fee
        }
        
        return withTip;
    }
    
    // Getters
    public String getCustomerName() { return customerName; }
    public String getCustomerPhone() { return customerPhone; }
    public List<OrderItem> getItems() { return items; }
    public OrderType getOrderType() { return orderType; }
    public String getDeliveryAddress() { return deliveryAddress; }
    public LocalDateTime getOrderTime() { return orderTime; }
    public PaymentMethod getPaymentMethod() { return paymentMethod; }
    public String getSpecialInstructions() { return specialInstructions; }
    public boolean isExpedited() { return expedited; }
    public double getDiscount() { return discount; }
    public double getTip() { return tip; }
    public double getTotalAmount() { return totalAmount; }
    
    @Override
    public String toString() {
        return String.format(
            "RestaurantOrder{customer='%s', phone='%s', items=%d, type=%s, " +
            "total=$%.2f, expedited=%s, time=%s}",
            customerName, customerPhone, items.size(), orderType, 
            totalAmount, expedited, orderTime
        );
    }
    
    /**
     * Fluent builder for restaurant orders
     */
    public static class RestaurantOrderBuilder {
        
        // Required fields
        private final String customerName;
        private final String customerPhone;
        private final List<OrderItem> items = new ArrayList<>();
        
        // Optional fields with defaults
        private OrderType orderType = OrderType.PICKUP;
        private String deliveryAddress;
        private LocalDateTime orderTime = LocalDateTime.now();
        private PaymentMethod paymentMethod = PaymentMethod.CASH;
        private String specialInstructions = "";
        private boolean expedited = false;
        private double discount = 0.0;
        private double tip = 0.0;
        
        public RestaurantOrderBuilder(String customerName, String customerPhone) {
            this.customerName = requireNonEmpty(customerName, "Customer name is required");
            this.customerPhone = requireNonEmpty(customerPhone, "Customer phone is required");
        }
        
        public RestaurantOrderBuilder addItem(String name, double price, int quantity) {
            if (quantity <= 0) {
                throw new IllegalArgumentException("Quantity must be positive");
            }
            this.items.add(new OrderItem(name, price, quantity));
            return this;
        }
        
        public RestaurantOrderBuilder addItem(String name, double price) {
            return addItem(name, price, 1);
        }
        
        public RestaurantOrderBuilder removeItem(String name) {
            items.removeIf(item -> item.getName().equals(name));
            return this;
        }
        
        public RestaurantOrderBuilder forDelivery(String address) {
            this.orderType = OrderType.DELIVERY;
            this.deliveryAddress = requireNonEmpty(address, "Delivery address is required");
            return this;
        }
        
        public RestaurantOrderBuilder forPickup() {
            this.orderType = OrderType.PICKUP;
            this.deliveryAddress = null;
            return this;
        }
        
        public RestaurantOrderBuilder forDineIn(String tableNumber) {
            this.orderType = OrderType.DINE_IN;
            this.specialInstructions = "Table: " + tableNumber;
            return this;
        }
        
        public RestaurantOrderBuilder scheduledFor(LocalDateTime time) {
            if (time.isBefore(LocalDateTime.now())) {
                throw new IllegalArgumentException("Order time cannot be in the past");
            }
            this.orderTime = time;
            return this;
        }
        
        public RestaurantOrderBuilder payWith(PaymentMethod method) {
            this.paymentMethod = requireNonNull(method, "Payment method is required");
            return this;
        }
        
        public RestaurantOrderBuilder withSpecialInstructions(String instructions) {
            this.specialInstructions = instructions != null ? instructions : "";
            return this;
        }
        
        public RestaurantOrderBuilder expedited() {
            this.expedited = true;
            return this;
        }
        
        public RestaurantOrderBuilder withDiscount(double discount) {
            if (discount < 0) {
                throw new IllegalArgumentException("Discount cannot be negative");
            }
            this.discount = discount;
            return this;
        }
        
        public RestaurantOrderBuilder withTip(double tip) {
            if (tip < 0) {
                throw new IllegalArgumentException("Tip cannot be negative");
            }
            this.tip = tip;
            return this;
        }
        
        public RestaurantOrderBuilder withTipPercent(double percentage) {
            if (percentage < 0 || percentage > 100) {
                throw new IllegalArgumentException("Tip percentage must be between 0 and 100");
            }
            double itemsTotal = items.stream()
                .mapToDouble(item -> item.getPrice() * item.getQuantity())
                .sum();
            this.tip = itemsTotal * (percentage / 100.0);
            return this;
        }
        
        public RestaurantOrder build() {
            validate();
            return new RestaurantOrder(this);
        }
        
        private void validate() {
            if (items.isEmpty()) {
                throw new IllegalStateException("Order must contain at least one item");
            }
            
            if (orderType == OrderType.DELIVERY && (deliveryAddress == null || deliveryAddress.trim().isEmpty())) {
                throw new IllegalStateException("Delivery address is required for delivery orders");
            }
            
            double itemsTotal = items.stream()
                .mapToDouble(item -> item.getPrice() * item.getQuantity())
                .sum();
            
            if (discount > itemsTotal) {
                throw new IllegalStateException("Discount cannot be greater than items total");
            }
        }
        
        private static String requireNonEmpty(String value, String message) {
            if (value == null || value.trim().isEmpty()) {
                throw new IllegalArgumentException(message);
            }
            return value;
        }
        
        private static <T> T requireNonNull(T obj, String message) {
            if (obj == null) {
                throw new IllegalArgumentException(message);
            }
            return obj;
        }
    }
}

// Supporting classes
public class OrderItem {
    private final String name;
    private final double price;
    private final int quantity;
    
    public OrderItem(String name, double price, int quantity) {
        this.name = name;
        this.price = price;
        this.quantity = quantity;
    }
    
    public String getName() { return name; }
    public double getPrice() { return price; }
    public int getQuantity() { return quantity; }
    
    @Override
    public String toString() {
        return String.format("%s x%d ($%.2f each)", name, quantity, price);
    }
}

public enum OrderType {
    PICKUP, DELIVERY, DINE_IN
}

public enum PaymentMethod {
    CASH, CREDIT_CARD, DEBIT_CARD, MOBILE_PAY, GIFT_CARD
}
```

## Director Pattern with Builder

```java
/**
 * Computer configurations using Director pattern
 */
public class ComputerDirector {
    
    public Computer buildGamingComputer() {
        return new Computer.ComputerBuilder("Intel i7-12700K", "32GB DDR4")
            .graphics("NVIDIA RTX 4070")
            .storage("1TB NVMe SSD")
            .powerSupply("750W 80+ Gold")
            .motherboard("ASUS ROG Strix B660")
            .coolingSystem("Liquid Cooling")
            .enableWifi()
            .enableBluetooth()
            .usbPorts(8)
            .operatingSystem("Windows 11 Pro")
            .build();
    }
    
    public Computer buildOfficeComputer() {
        return new Computer.ComputerBuilder("Intel i5-12400", "16GB DDR4")
            .storage("512GB SSD")
            .powerSupply("450W 80+ Bronze")
            .motherboard("MSI B660M Pro")
            .enableWifi()
            .usbPorts(6)
            .operatingSystem("Windows 11 Home")
            .build();
    }
    
    public Computer buildBudgetComputer() {
        return new Computer.ComputerBuilder("AMD Ryzen 5 5600G", "16GB DDR4")
            .storage("256GB SSD + 1TB HDD")
            .powerSupply("400W")
            .enableWifi()
            .usbPorts(4)
            .operatingSystem("Linux Ubuntu")
            .build();
    }
    
    public Computer buildWorkstationComputer() {
        return new Computer.ComputerBuilder("Intel i9-12900K", "64GB DDR4")
            .graphics("NVIDIA RTX 4080")
            .storage("2TB NVMe SSD")
            .powerSupply("850W 80+ Platinum")
            .motherboard("ASUS Pro WS W680")
            .coolingSystem("Custom Liquid Cooling")
            .enableWifi()
            .enableBluetooth()
            .usbPorts(10)
            .operatingSystem("Windows 11 Pro for Workstations")
            .build();
    }
}
```

## Spring Boot Integration

```java
/**
 * Configuration builder for Spring Boot applications
 */
@Component
public class ApplicationConfigurationBuilder {
    
    @Value("${app.name:DefaultApp}")
    private String appName;
    
    @Value("${app.version:1.0.0}")
    private String version;
    
    public ApplicationConfiguration buildDevelopmentConfig() {
        return new ApplicationConfiguration.Builder(appName, version)
            .environment("development")
            .enableDebugMode()
            .databaseUrl("jdbc:h2:mem:devdb")
            .databaseUsername("sa")
            .databasePassword("")
            .enableCaching(false)
            .logLevel("DEBUG")
            .enableMetrics()
            .build();
    }
    
    public ApplicationConfiguration buildProductionConfig() {
        return new ApplicationConfiguration.Builder(appName, version)
            .environment("production")
            .databaseUrl("${DB_URL}")
            .databaseUsername("${DB_USERNAME}")
            .databasePassword("${DB_PASSWORD}")
            .enableCaching(true)
            .cacheProvider("Redis")
            .logLevel("INFO")
            .enableMetrics()
            .enableSecurity()
            .maxConnections(100)
            .connectionTimeout(30000)
            .build();
    }
    
    public ApplicationConfiguration buildTestConfig() {
        return new ApplicationConfiguration.Builder(appName, version)
            .environment("test")
            .databaseUrl("jdbc:h2:mem:testdb")
            .databaseUsername("sa")
            .databasePassword("")
            .enableCaching(false)
            .logLevel("WARN")
            .enableTestMode()
            .build();
    }
}

/**
 * Application Configuration using Builder pattern
 */
public class ApplicationConfiguration {
    
    private final String appName;
    private final String version;
    private final String environment;
    private final boolean debugMode;
    private final String databaseUrl;
    private final String databaseUsername;
    private final String databasePassword;
    private final boolean cachingEnabled;
    private final String cacheProvider;
    private final String logLevel;
    private final boolean metricsEnabled;
    private final boolean securityEnabled;
    private final boolean testMode;
    private final int maxConnections;
    private final int connectionTimeout;
    
    private ApplicationConfiguration(Builder builder) {
        this.appName = builder.appName;
        this.version = builder.version;
        this.environment = builder.environment;
        this.debugMode = builder.debugMode;
        this.databaseUrl = builder.databaseUrl;
        this.databaseUsername = builder.databaseUsername;
        this.databasePassword = builder.databasePassword;
        this.cachingEnabled = builder.cachingEnabled;
        this.cacheProvider = builder.cacheProvider;
        this.logLevel = builder.logLevel;
        this.metricsEnabled = builder.metricsEnabled;
        this.securityEnabled = builder.securityEnabled;
        this.testMode = builder.testMode;
        this.maxConnections = builder.maxConnections;
        this.connectionTimeout = builder.connectionTimeout;
    }
    
    // Getters
    public String getAppName() { return appName; }
    public String getVersion() { return version; }
    public String getEnvironment() { return environment; }
    public boolean isDebugMode() { return debugMode; }
    public String getDatabaseUrl() { return databaseUrl; }
    public String getDatabaseUsername() { return databaseUsername; }
    public String getDatabasePassword() { return databasePassword; }
    public boolean isCachingEnabled() { return cachingEnabled; }
    public String getCacheProvider() { return cacheProvider; }
    public String getLogLevel() { return logLevel; }
    public boolean isMetricsEnabled() { return metricsEnabled; }
    public boolean isSecurityEnabled() { return securityEnabled; }
    public boolean isTestMode() { return testMode; }
    public int getMaxConnections() { return maxConnections; }
    public int getConnectionTimeout() { return connectionTimeout; }
    
    @Override
    public String toString() {
        return String.format(
            "ApplicationConfiguration{name='%s', version='%s', env='%s', " +
            "debug=%s, caching=%s, security=%s}",
            appName, version, environment, debugMode, cachingEnabled, securityEnabled
        );
    }
    
    public static class Builder {
        
        // Required fields
        private final String appName;
        private final String version;
        
        // Optional fields with defaults
        private String environment = "development";
        private boolean debugMode = false;
        private String databaseUrl = "jdbc:h2:mem:defaultdb";
        private String databaseUsername = "sa";
        private String databasePassword = "";
        private boolean cachingEnabled = false;
        private String cacheProvider = "Caffeine";
        private String logLevel = "INFO";
        private boolean metricsEnabled = false;
        private boolean securityEnabled = false;
        private boolean testMode = false;
        private int maxConnections = 10;
        private int connectionTimeout = 5000;
        
        public Builder(String appName, String version) {
            this.appName = appName;
            this.version = version;
        }
        
        public Builder environment(String environment) {
            this.environment = environment;
            return this;
        }
        
        public Builder enableDebugMode() {
            this.debugMode = true;
            return this;
        }
        
        public Builder databaseUrl(String url) {
            this.databaseUrl = url;
            return this;
        }
        
        public Builder databaseUsername(String username) {
            this.databaseUsername = username;
            return this;
        }
        
        public Builder databasePassword(String password) {
            this.databasePassword = password;
            return this;
        }
        
        public Builder enableCaching(boolean enabled) {
            this.cachingEnabled = enabled;
            return this;
        }
        
        public Builder cacheProvider(String provider) {
            this.cacheProvider = provider;
            return this;
        }
        
        public Builder logLevel(String level) {
            this.logLevel = level;
            return this;
        }
        
        public Builder enableMetrics() {
            this.metricsEnabled = true;
            return this;
        }
        
        public Builder enableSecurity() {
            this.securityEnabled = true;
            return this;
        }
        
        public Builder enableTestMode() {
            this.testMode = true;
            return this;
        }
        
        public Builder maxConnections(int maxConnections) {
            this.maxConnections = maxConnections;
            return this;
        }
        
        public Builder connectionTimeout(int timeout) {
            this.connectionTimeout = timeout;
            return this;
        }
        
        public ApplicationConfiguration build() {
            return new ApplicationConfiguration(this);
        }
    }
}

/**
 * Service that uses the builder
 */
@Service
public class ConfigurationService {
    
    private final ApplicationConfigurationBuilder configBuilder;
    
    public ConfigurationService(ApplicationConfigurationBuilder configBuilder) {
        this.configBuilder = configBuilder;
    }
    
    @PostConstruct
    public void initializeConfiguration() {
        ApplicationConfiguration config = configBuilder.buildProductionConfig();
        System.out.println("Application initialized with configuration: " + config);
    }
}
```

## Testing Builder Pattern

```java
@ExtendWith(MockitoExtension.class)
class BuilderPatternTest {
    
    @Test
    void testComputerBuilder() {
        Computer computer = new Computer.ComputerBuilder("Intel i7", "16GB")
            .storage("1TB SSD")
            .graphics("NVIDIA RTX 4060")
            .enableWifi()
            .enableBluetooth()
            .usbPorts(6)
            .operatingSystem("Windows 11")
            .build();
        
        assertThat(computer.getCpu()).isEqualTo("Intel i7");
        assertThat(computer.getMemory()).isEqualTo("16GB");
        assertThat(computer.getStorage()).isEqualTo("1TB SSD");
        assertThat(computer.getGraphics()).isEqualTo("NVIDIA RTX 4060");
        assertThat(computer.hasWifi()).isTrue();
        assertThat(computer.hasBluetooth()).isTrue();
        assertThat(computer.getUsbPorts()).isEqualTo(6);
        assertThat(computer.getOperatingSystem()).isEqualTo("Windows 11");
    }
    
    @Test
    void testComputerBuilderValidation() {
        assertThrows(IllegalArgumentException.class, () -> {
            new Computer.ComputerBuilder(null, "16GB").build();
        });
        
        assertThrows(IllegalArgumentException.class, () -> {
            new Computer.ComputerBuilder("Intel i7", "16GB")
                .usbPorts(-1)
                .build();
        });
        
        assertThrows(IllegalStateException.class, () -> {
            new Computer.ComputerBuilder("Intel i9", "32GB")
                .graphics("NVIDIA RTX 4090")
                .powerSupply("400W") // Insufficient for high-end graphics
                .build();
        });
    }
    
    @Test
    void testRestaurantOrderBuilder() {
        RestaurantOrder order = new RestaurantOrder.RestaurantOrderBuilder("John Doe", "123-456-7890")
            .addItem("Pizza Margherita", 15.99, 2)
            .addItem("Caesar Salad", 8.99)
            .addItem("Coke", 2.99, 2)
            .forDelivery("123 Main St")
            .payWith(PaymentMethod.CREDIT_CARD)
            .withTipPercent(18)
            .withSpecialInstructions("Ring the doorbell twice")
            .expedited()
            .build();
        
        assertThat(order.getCustomerName()).isEqualTo("John Doe");
        assertThat(order.getCustomerPhone()).isEqualTo("123-456-7890");
        assertThat(order.getItems()).hasSize(3);
        assertThat(order.getOrderType()).isEqualTo(OrderType.DELIVERY);
        assertThat(order.getDeliveryAddress()).isEqualTo("123 Main St");
        assertThat(order.getPaymentMethod()).isEqualTo(PaymentMethod.CREDIT_CARD);
        assertThat(order.isExpedited()).isTrue();
        assertThat(order.getTotalAmount()).isGreaterThan(0);
    }
    
    @Test
    void testOrderBuilderValidation() {
        assertThrows(IllegalArgumentException.class, () -> {
            new RestaurantOrder.RestaurantOrderBuilder("", "123-456-7890");
        });
        
        assertThrows(IllegalStateException.class, () -> {
            new RestaurantOrder.RestaurantOrderBuilder("John Doe", "123-456-7890")
                .build(); // No items added
        });
        
        assertThrows(IllegalStateException.class, () -> {
            new RestaurantOrder.RestaurantOrderBuilder("John Doe", "123-456-7890")
                .addItem("Pizza", 10.99)
                .forDelivery("") // Empty delivery address
                .build();
        });
    }
    
    @Test
    void testDirectorPattern() {
        ComputerDirector director = new ComputerDirector();
        
        Computer gaming = director.buildGamingComputer();
        Computer office = director.buildOfficeComputer();
        Computer budget = director.buildBudgetComputer();
        Computer workstation = director.buildWorkstationComputer();
        
        assertThat(gaming.getGraphics()).contains("RTX");
        assertThat(office.getOperatingSystem()).contains("Windows");
        assertThat(budget.getOperatingSystem()).contains("Linux");
        assertThat(workstation.getMemory()).contains("64GB");
    }
    
    @Test
    void testApplicationConfigurationBuilder() {
        ApplicationConfiguration config = new ApplicationConfiguration.Builder("TestApp", "1.0.0")
            .environment("test")
            .enableDebugMode()
            .databaseUrl("jdbc:h2:mem:testdb")
            .enableCaching(true)
            .cacheProvider("Redis")
            .logLevel("DEBUG")
            .enableMetrics()
            .maxConnections(50)
            .build();
        
        assertThat(config.getAppName()).isEqualTo("TestApp");
        assertThat(config.getVersion()).isEqualTo("1.0.0");
        assertThat(config.getEnvironment()).isEqualTo("test");
        assertThat(config.isDebugMode()).isTrue();
        assertThat(config.isCachingEnabled()).isTrue();
        assertThat(config.getCacheProvider()).isEqualTo("Redis");
        assertThat(config.getMaxConnections()).isEqualTo(50);
    }
}
```

## Best Practices

### ✅ **Do's:**

1. **Use for complex objects** with many optional parameters
2. **Make the built object immutable** when possible
3. **Validate parameters** in the builder before creating the object
4. **Use method chaining** for fluent interface
5. **Provide meaningful defaults** for optional parameters
6. **Use static nested builder class** within the product class
7. **Implement proper validation** in both builder and build() method

### ❌ **Don'ts:**

1. **Don't use for simple objects** - constructor is sufficient
2. **Don't forget validation** - ensure object is in valid state
3. **Don't make builder mutable** after build() is called
4. **Don't expose the builder** outside the product class unnecessarily
5. **Don't create builders** for objects with few parameters

## When to Use Builder Pattern

### ✅ **Good Use Cases:**
- **Complex object creation**: Many parameters (especially optional ones)
- **Immutable objects**: Step-by-step construction of immutable instances
- **Configuration objects**: Application settings, connection parameters
- **Test data creation**: Building test objects with various configurations
- **API request/response objects**: HTTP requests, database queries

### ❌ **Avoid Builder For:**
- **Simple objects**: 3 or fewer parameters
- **Mutable objects**: If the object will change after creation
- **Performance-critical code**: Builder adds slight overhead

## Summary

The Builder pattern is excellent for:

**Key Benefits:**
- **Readability**: Clear, self-documenting object creation
- **Flexibility**: Easy to add new optional parameters
- **Immutability**: Create immutable objects step by step
- **Validation**: Centralized validation logic
- **Default Values**: Sensible defaults for optional parameters

**Common Use Cases:**
- **Configuration Objects**: Database configs, application settings
- **Complex Domain Objects**: Orders, users, products with many fields
- **Test Data Builders**: Creating test objects with various configurations
- **HTTP Requests**: Building complex API requests

Use Builder when object creation is complex and you want to make it more readable and maintainable!

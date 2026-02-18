# Strategy Pattern - Complete Guide

## Introduction

The **Strategy Pattern** is a behavioral design pattern that defines a family of algorithms, encapsulates each one, and makes them interchangeable. It lets the algorithm vary independently from clients that use it.

## Definition

> **Strategy Pattern**: Defines a family of algorithms, encapsulates each one, and makes them interchangeable. Strategy lets the algorithm vary independently from clients that use it.

## Problem Solved

- **Algorithm Selection**: Choose different algorithms at runtime
- **Code Duplication**: Avoid if-else or switch statements for algorithm selection
- **Open/Closed Principle**: Add new algorithms without modifying existing code
- **Single Responsibility**: Each algorithm is in its own class
- **Testability**: Easy to test algorithms independently

## Structure

```
┌─────────────────┐    ┌─────────────────┐
│    Context      │───▶│    Strategy     │
├─────────────────┤    ├─────────────────┤
│ -strategy       │    │ +algorithm()    │
│ +setStrategy()  │    └─────────────────┘
│ +executeAlg()   │              △
└─────────────────┘              │
                      ┌──────────┼──────────┐
                      │          │          │
        ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
        │ ConcreteStratA  │ │ ConcreteStratB  │ │ ConcreteStratC  │
        ├─────────────────┤ ├─────────────────┤ ├─────────────────┤
        │ +algorithm()    │ │ +algorithm()    │ │ +algorithm()    │
        └─────────────────┘ └─────────────────┘ └─────────────────┘
```

## Basic Implementation

### Payment Processing Example

```java
/**
 * Strategy interface for payment processing
 */
public interface PaymentStrategy {
    PaymentResult processPayment(double amount, PaymentDetails details);
    boolean isPaymentSupported(String paymentType);
    String getPaymentMethodName();
}

/**
 * Credit Card Payment Strategy
 */
public class CreditCardPaymentStrategy implements PaymentStrategy {
    
    @Override
    public PaymentResult processPayment(double amount, PaymentDetails details) {
        CreditCardDetails cardDetails = (CreditCardDetails) details;
        
        System.out.println("Processing credit card payment of $" + String.format("%.2f", amount));
        System.out.println("Card Number: **** **** **** " + cardDetails.getCardNumber().substring(12));
        
        // Simulate payment processing
        if (validateCreditCard(cardDetails)) {
            if (amount <= cardDetails.getCreditLimit()) {
                return new PaymentResult(true, "Payment successful", generateTransactionId());
            } else {
                return new PaymentResult(false, "Credit limit exceeded", null);
            }
        } else {
            return new PaymentResult(false, "Invalid credit card details", null);
        }
    }
    
    @Override
    public boolean isPaymentSupported(String paymentType) {
        return "CREDIT_CARD".equalsIgnoreCase(paymentType);
    }
    
    @Override
    public String getPaymentMethodName() {
        return "Credit Card";
    }
    
    private boolean validateCreditCard(CreditCardDetails details) {
        // Simple validation - in reality would use Luhn algorithm
        return details.getCardNumber().length() == 16 && 
               details.getCvv().length() == 3 &&
               details.getExpiryDate().isAfter(LocalDate.now());
    }
    
    private String generateTransactionId() {
        return "CC_" + System.currentTimeMillis() + "_" + (int)(Math.random() * 1000);
    }
}

/**
 * PayPal Payment Strategy
 */
public class PayPalPaymentStrategy implements PaymentStrategy {
    
    @Override
    public PaymentResult processPayment(double amount, PaymentDetails details) {
        PayPalDetails paypalDetails = (PayPalDetails) details;
        
        System.out.println("Processing PayPal payment of $" + String.format("%.2f", amount));
        System.out.println("PayPal Email: " + paypalDetails.getEmail());
        
        // Simulate PayPal API call
        if (authenticateWithPayPal(paypalDetails)) {
            if (checkPayPalBalance(paypalDetails, amount)) {
                return new PaymentResult(true, "PayPal payment successful", generateTransactionId());
            } else {
                return new PaymentResult(false, "Insufficient PayPal balance", null);
            }
        } else {
            return new PaymentResult(false, "PayPal authentication failed", null);
        }
    }
    
    @Override
    public boolean isPaymentSupported(String paymentType) {
        return "PAYPAL".equalsIgnoreCase(paymentType);
    }
    
    @Override
    public String getPaymentMethodName() {
        return "PayPal";
    }
    
    private boolean authenticateWithPayPal(PayPalDetails details) {
        // Simulate PayPal authentication
        return details.getEmail().contains("@") && details.getPassword().length() >= 6;
    }
    
    private boolean checkPayPalBalance(PayPalDetails details, double amount) {
        // Simulate balance check
        return amount <= 1000.0; // Mock balance limit
    }
    
    private String generateTransactionId() {
        return "PP_" + System.currentTimeMillis() + "_" + (int)(Math.random() * 1000);
    }
}

/**
 * Cryptocurrency Payment Strategy
 */
public class CryptocurrencyPaymentStrategy implements PaymentStrategy {
    
    @Override
    public PaymentResult processPayment(double amount, PaymentDetails details) {
        CryptocurrencyDetails cryptoDetails = (CryptocurrencyDetails) details;
        
        System.out.println("Processing cryptocurrency payment of $" + String.format("%.2f", amount));
        System.out.println("Wallet Address: " + cryptoDetails.getWalletAddress());
        System.out.println("Currency: " + cryptoDetails.getCurrencyType());
        
        // Simulate blockchain transaction
        if (validateWalletAddress(cryptoDetails)) {
            if (checkWalletBalance(cryptoDetails, amount)) {
                return new PaymentResult(true, "Cryptocurrency payment successful", generateTransactionId());
            } else {
                return new PaymentResult(false, "Insufficient cryptocurrency balance", null);
            }
        } else {
            return new PaymentResult(false, "Invalid wallet address", null);
        }
    }
    
    @Override
    public boolean isPaymentSupported(String paymentType) {
        return "CRYPTOCURRENCY".equalsIgnoreCase(paymentType) || 
               "BITCOIN".equalsIgnoreCase(paymentType) || 
               "ETHEREUM".equalsIgnoreCase(paymentType);
    }
    
    @Override
    public String getPaymentMethodName() {
        return "Cryptocurrency";
    }
    
    private boolean validateWalletAddress(CryptocurrencyDetails details) {
        // Simple wallet address validation
        String address = details.getWalletAddress();
        return address.length() >= 26 && address.length() <= 35;
    }
    
    private boolean checkWalletBalance(CryptocurrencyDetails details, double amount) {
        // Simulate blockchain balance check
        return amount <= 5000.0; // Mock balance limit
    }
    
    private String generateTransactionId() {
        return "CRYPTO_" + System.currentTimeMillis() + "_" + (int)(Math.random() * 1000);
    }
}
```

### Payment Details Classes

```java
/**
 * Abstract payment details
 */
public abstract class PaymentDetails {
    protected final String paymentType;
    
    protected PaymentDetails(String paymentType) {
        this.paymentType = paymentType;
    }
    
    public String getPaymentType() {
        return paymentType;
    }
}

/**
 * Credit Card Details
 */
public class CreditCardDetails extends PaymentDetails {
    private final String cardNumber;
    private final String cvv;
    private final LocalDate expiryDate;
    private final String cardHolderName;
    private final double creditLimit;
    
    public CreditCardDetails(String cardNumber, String cvv, LocalDate expiryDate, 
                           String cardHolderName, double creditLimit) {
        super("CREDIT_CARD");
        this.cardNumber = cardNumber;
        this.cvv = cvv;
        this.expiryDate = expiryDate;
        this.cardHolderName = cardHolderName;
        this.creditLimit = creditLimit;
    }
    
    // Getters
    public String getCardNumber() { return cardNumber; }
    public String getCvv() { return cvv; }
    public LocalDate getExpiryDate() { return expiryDate; }
    public String getCardHolderName() { return cardHolderName; }
    public double getCreditLimit() { return creditLimit; }
}

/**
 * PayPal Details
 */
public class PayPalDetails extends PaymentDetails {
    private final String email;
    private final String password;
    
    public PayPalDetails(String email, String password) {
        super("PAYPAL");
        this.email = email;
        this.password = password;
    }
    
    public String getEmail() { return email; }
    public String getPassword() { return password; }
}

/**
 * Cryptocurrency Details
 */
public class CryptocurrencyDetails extends PaymentDetails {
    private final String walletAddress;
    private final String currencyType;
    private final String privateKey;
    
    public CryptocurrencyDetails(String walletAddress, String currencyType, String privateKey) {
        super("CRYPTOCURRENCY");
        this.walletAddress = walletAddress;
        this.currencyType = currencyType;
        this.privateKey = privateKey;
    }
    
    public String getWalletAddress() { return walletAddress; }
    public String getCurrencyType() { return currencyType; }
    public String getPrivateKey() { return privateKey; }
}

/**
 * Payment Result
 */
public class PaymentResult {
    private final boolean success;
    private final String message;
    private final String transactionId;
    
    public PaymentResult(boolean success, String message, String transactionId) {
        this.success = success;
        this.message = message;
        this.transactionId = transactionId;
    }
    
    public boolean isSuccess() { return success; }
    public String getMessage() { return message; }
    public String getTransactionId() { return transactionId; }
    
    @Override
    public String toString() {
        return String.format("PaymentResult{success=%s, message='%s', transactionId='%s'}", 
            success, message, transactionId);
    }
}
```

### Payment Context (Strategy User)

```java
/**
 * Payment processor that uses different payment strategies
 */
public class PaymentProcessor {
    
    private final Map<String, PaymentStrategy> strategies = new HashMap<>();
    private PaymentStrategy defaultStrategy;
    
    public PaymentProcessor() {
        initializeStrategies();
    }
    
    private void initializeStrategies() {
        registerStrategy(new CreditCardPaymentStrategy());
        registerStrategy(new PayPalPaymentStrategy());
        registerStrategy(new CryptocurrencyPaymentStrategy());
        
        // Set default strategy
        defaultStrategy = strategies.get("CREDIT_CARD");
    }
    
    public void registerStrategy(PaymentStrategy strategy) {
        String key = strategy.getPaymentMethodName().toUpperCase().replace(" ", "_");
        strategies.put(key, strategy);
        
        System.out.println("Registered payment strategy: " + strategy.getPaymentMethodName());
    }
    
    public PaymentResult processPayment(String paymentType, double amount, PaymentDetails details) {
        PaymentStrategy strategy = selectStrategy(paymentType);
        
        if (strategy == null) {
            return new PaymentResult(false, "Unsupported payment method: " + paymentType, null);
        }
        
        // Log payment attempt
        System.out.println(String.format(
            "Processing payment: Type=%s, Amount=$%.2f, Method=%s", 
            paymentType, amount, strategy.getPaymentMethodName()
        ));
        
        try {
            PaymentResult result = strategy.processPayment(amount, details);
            
            // Log result
            if (result.isSuccess()) {
                System.out.println("Payment successful: " + result.getTransactionId());
            } else {
                System.out.println("Payment failed: " + result.getMessage());
            }
            
            return result;
            
        } catch (Exception e) {
            System.err.println("Payment processing error: " + e.getMessage());
            return new PaymentResult(false, "Payment processing error: " + e.getMessage(), null);
        }
    }
    
    private PaymentStrategy selectStrategy(String paymentType) {
        // Try exact match first
        PaymentStrategy strategy = strategies.get(paymentType.toUpperCase());
        
        if (strategy != null) {
            return strategy;
        }
        
        // Try finding strategy that supports this payment type
        for (PaymentStrategy candidateStrategy : strategies.values()) {
            if (candidateStrategy.isPaymentSupported(paymentType)) {
                return candidateStrategy;
            }
        }
        
        // Return default strategy if no match found
        return defaultStrategy;
    }
    
    public List<String> getSupportedPaymentMethods() {
        return strategies.values().stream()
            .map(PaymentStrategy::getPaymentMethodName)
            .collect(Collectors.toList());
    }
    
    public void setDefaultStrategy(String paymentType) {
        PaymentStrategy strategy = strategies.get(paymentType.toUpperCase());
        if (strategy != null) {
            this.defaultStrategy = strategy;
            System.out.println("Default payment method set to: " + strategy.getPaymentMethodName());
        }
    }
}
```

## Real-World Example: Sorting Strategies

```java
/**
 * Sorting strategy interface
 */
public interface SortingStrategy<T> {
    void sort(List<T> data, Comparator<T> comparator);
    String getAlgorithmName();
    String getTimeComplexity();
    String getSpaceComplexity();
    boolean isStable();
}

/**
 * Quick Sort Strategy
 */
public class QuickSortStrategy<T> implements SortingStrategy<T> {
    
    @Override
    public void sort(List<T> data, Comparator<T> comparator) {
        if (data.size() <= 1) return;
        
        System.out.println("Executing QuickSort on " + data.size() + " elements");
        quickSort(data, 0, data.size() - 1, comparator);
    }
    
    private void quickSort(List<T> data, int low, int high, Comparator<T> comparator) {
        if (low < high) {
            int partitionIndex = partition(data, low, high, comparator);
            quickSort(data, low, partitionIndex - 1, comparator);
            quickSort(data, partitionIndex + 1, high, comparator);
        }
    }
    
    private int partition(List<T> data, int low, int high, Comparator<T> comparator) {
        T pivot = data.get(high);
        int i = low - 1;
        
        for (int j = low; j < high; j++) {
            if (comparator.compare(data.get(j), pivot) <= 0) {
                i++;
                Collections.swap(data, i, j);
            }
        }
        
        Collections.swap(data, i + 1, high);
        return i + 1;
    }
    
    @Override
    public String getAlgorithmName() { return "Quick Sort"; }
    
    @Override
    public String getTimeComplexity() { return "O(n log n) average, O(n²) worst"; }
    
    @Override
    public String getSpaceComplexity() { return "O(log n)"; }
    
    @Override
    public boolean isStable() { return false; }
}

/**
 * Merge Sort Strategy
 */
public class MergeSortStrategy<T> implements SortingStrategy<T> {
    
    @Override
    public void sort(List<T> data, Comparator<T> comparator) {
        if (data.size() <= 1) return;
        
        System.out.println("Executing MergeSort on " + data.size() + " elements");
        List<T> temp = new ArrayList<>(data);
        mergeSort(data, temp, 0, data.size() - 1, comparator);
    }
    
    private void mergeSort(List<T> data, List<T> temp, int left, int right, Comparator<T> comparator) {
        if (left < right) {
            int middle = (left + right) / 2;
            mergeSort(data, temp, left, middle, comparator);
            mergeSort(data, temp, middle + 1, right, comparator);
            merge(data, temp, left, middle, right, comparator);
        }
    }
    
    private void merge(List<T> data, List<T> temp, int left, int middle, int right, Comparator<T> comparator) {
        // Copy data to temp array
        for (int i = left; i <= right; i++) {
            temp.set(i, data.get(i));
        }
        
        int i = left, j = middle + 1, k = left;
        
        while (i <= middle && j <= right) {
            if (comparator.compare(temp.get(i), temp.get(j)) <= 0) {
                data.set(k++, temp.get(i++));
            } else {
                data.set(k++, temp.get(j++));
            }
        }
        
        while (i <= middle) {
            data.set(k++, temp.get(i++));
        }
        
        while (j <= right) {
            data.set(k++, temp.get(j++));
        }
    }
    
    @Override
    public String getAlgorithmName() { return "Merge Sort"; }
    
    @Override
    public String getTimeComplexity() { return "O(n log n)"; }
    
    @Override
    public String getSpaceComplexity() { return "O(n)"; }
    
    @Override
    public boolean isStable() { return true; }
}

/**
 * Heap Sort Strategy
 */
public class HeapSortStrategy<T> implements SortingStrategy<T> {
    
    @Override
    public void sort(List<T> data, Comparator<T> comparator) {
        if (data.size() <= 1) return;
        
        System.out.println("Executing HeapSort on " + data.size() + " elements");
        
        int n = data.size();
        
        // Build heap
        for (int i = n / 2 - 1; i >= 0; i--) {
            heapify(data, n, i, comparator);
        }
        
        // Extract elements from heap
        for (int i = n - 1; i > 0; i--) {
            Collections.swap(data, 0, i);
            heapify(data, i, 0, comparator);
        }
    }
    
    private void heapify(List<T> data, int n, int i, Comparator<T> comparator) {
        int largest = i;
        int left = 2 * i + 1;
        int right = 2 * i + 2;
        
        if (left < n && comparator.compare(data.get(left), data.get(largest)) > 0) {
            largest = left;
        }
        
        if (right < n && comparator.compare(data.get(right), data.get(largest)) > 0) {
            largest = right;
        }
        
        if (largest != i) {
            Collections.swap(data, i, largest);
            heapify(data, n, largest, comparator);
        }
    }
    
    @Override
    public String getAlgorithmName() { return "Heap Sort"; }
    
    @Override
    public String getTimeComplexity() { return "O(n log n)"; }
    
    @Override
    public String getSpaceComplexity() { return "O(1)"; }
    
    @Override
    public boolean isStable() { return false; }
}

/**
 * Smart Sorting Context
 */
public class SmartSorter<T> {
    
    private final Map<String, SortingStrategy<T>> strategies = new HashMap<>();
    
    public SmartSorter() {
        registerStrategy("quicksort", new QuickSortStrategy<>());
        registerStrategy("mergesort", new MergeSortStrategy<>());
        registerStrategy("heapsort", new HeapSortStrategy<>());
    }
    
    public void registerStrategy(String name, SortingStrategy<T> strategy) {
        strategies.put(name.toLowerCase(), strategy);
        System.out.println("Registered sorting strategy: " + strategy.getAlgorithmName());
    }
    
    public void sort(List<T> data, Comparator<T> comparator) {
        SortingStrategy<T> strategy = selectOptimalStrategy(data.size());
        sort(data, comparator, strategy);
    }
    
    public void sort(List<T> data, Comparator<T> comparator, String strategyName) {
        SortingStrategy<T> strategy = strategies.get(strategyName.toLowerCase());
        if (strategy == null) {
            throw new IllegalArgumentException("Unknown sorting strategy: " + strategyName);
        }
        sort(data, comparator, strategy);
    }
    
    private void sort(List<T> data, Comparator<T> comparator, SortingStrategy<T> strategy) {
        long startTime = System.nanoTime();
        
        System.out.println("\n=== Sorting Analysis ===");
        System.out.println("Algorithm: " + strategy.getAlgorithmName());
        System.out.println("Data size: " + data.size());
        System.out.println("Time complexity: " + strategy.getTimeComplexity());
        System.out.println("Space complexity: " + strategy.getSpaceComplexity());
        System.out.println("Stable: " + strategy.isStable());
        
        strategy.sort(data, comparator);
        
        long endTime = System.nanoTime();
        double executionTime = (endTime - startTime) / 1_000_000.0; // Convert to milliseconds
        
        System.out.println("Execution time: " + String.format("%.2f ms", executionTime));
        System.out.println("=========================\n");
    }
    
    private SortingStrategy<T> selectOptimalStrategy(int dataSize) {
        if (dataSize < 10) {
            // For very small arrays, simple algorithms might be faster
            return strategies.get("quicksort");
        } else if (dataSize < 1000) {
            // Quick sort is generally good for medium-sized arrays
            return strategies.get("quicksort");
        } else {
            // Merge sort for larger arrays (guaranteed O(n log n))
            return strategies.get("mergesort");
        }
    }
    
    public void printAvailableStrategies() {
        System.out.println("Available sorting strategies:");
        strategies.forEach((name, strategy) -> {
            System.out.println(String.format("  %s: %s (%s, %s, Stable: %s)",
                name, strategy.getAlgorithmName(), strategy.getTimeComplexity(),
                strategy.getSpaceComplexity(), strategy.isStable()));
        });
    }
}
```

## Spring Boot Integration

```java
/**
 * Discount calculation strategies
 */
public interface DiscountStrategy {
    double calculateDiscount(double originalPrice, Customer customer);
    String getStrategyName();
    boolean isApplicable(Customer customer);
}

/**
 * SPRING STRATEGY DISCOVERY EXPLANATION:
 * =====================================
 * @Component annotation tells Spring to:
 * 1. Automatically discover this class during component scanning
 * 2. Create a singleton bean (instance) of this class
 * 3. Include it in dependency injection when List<DiscountStrategy> is requested
 * 
 * Spring sees: RegularCustomerDiscountStrategy implements DiscountStrategy
 * So it includes this bean when injecting List<DiscountStrategy>
 */
@Component  // <- This is the KEY! Spring discovers this class because of @Component
public class RegularCustomerDiscountStrategy implements DiscountStrategy {
    
    @Override
    public double calculateDiscount(double originalPrice, Customer customer) {
        return originalPrice * 0.05; // 5% discount for regular customers
    }
    
    @Override
    public String getStrategyName() {
        return "Regular Customer Discount";
    }
    
    @Override
    public boolean isApplicable(Customer customer) {
        // This method determines WHEN this strategy should be used
        return customer.getCustomerType() == CustomerType.REGULAR;
    }
}

/**
 * Another strategy that Spring will automatically discover
 * Same process: @Component -> Bean Creation -> Injection into List<DiscountStrategy>
 */
@Component  // <- Spring finds this too and adds to the strategies list
public class PremiumCustomerDiscountStrategy implements DiscountStrategy {
    
    @Override
    public double calculateDiscount(double originalPrice, Customer customer) {
        return originalPrice * 0.15; // 15% discount for premium customers
    }
    
    @Override
    public String getStrategyName() {
        return "Premium Customer Discount";
    }
    
    @Override
    public boolean isApplicable(Customer customer) {
        // Different condition - this strategy applies to premium customers
        return customer.getCustomerType() == CustomerType.PREMIUM;
    }
}

/**
 * VIP strategy - also automatically discovered by Spring
 */
@Component  // <- All @Component classes implementing DiscountStrategy go into the List
public class VIPCustomerDiscountStrategy implements DiscountStrategy {
    
    @Override
    public double calculateDiscount(double originalPrice, Customer customer) {
        return originalPrice * 0.25; // 25% discount for VIP customers
    }
    
    @Override
    public String getStrategyName() {
        return "VIP Customer Discount";
    }
    
    @Override
    public boolean isApplicable(Customer customer) {
        return customer.getCustomerType() == CustomerType.VIP;
    }
}

/**
 * Seasonal strategy - applies to ALL customers but only during certain months
 * This shows how strategies can have complex applicability logic
 */
@Component  // <- Spring includes this in the List<DiscountStrategy> too
public class SeasonalDiscountStrategy implements DiscountStrategy {
    
    @Override
    public double calculateDiscount(double originalPrice, Customer customer) {
        LocalDate now = LocalDate.now();
        Month currentMonth = now.getMonth();
        
        // Holiday season discount (November, December)
        if (currentMonth == Month.NOVEMBER || currentMonth == Month.DECEMBER) {
            return originalPrice * 0.20; // 20% holiday discount
        }
        
        // Summer sale (June, July, August)
        if (currentMonth == Month.JUNE || currentMonth == Month.JULY || currentMonth == Month.AUGUST) {
            return originalPrice * 0.10; // 10% summer discount
        }
        
        return 0; // No seasonal discount for other months
    }
    
    @Override
    public String getStrategyName() {
        return "Seasonal Discount";
    }
    
    @Override
    public boolean isApplicable(Customer customer) {
        // This strategy applies to ALL customers regardless of type
        // The actual discount amount depends on the current date/season
        return true; // Always applicable, but discount amount varies by season
    }
}

/*
 * SPRING BOOT COLLECTION INJECTION SUMMARY:
 * =========================================
 * 
 * What happens at startup:
 * 
 * 1. COMPONENT SCANNING:
 *    Spring scans your package for classes with @Component, @Service, @Repository
 *    
 * 2. BEAN REGISTRATION:
 *    Spring registers these classes as bean definitions:
 *    - RegularCustomerDiscountStrategy -> Bean 
 *    - PremiumCustomerDiscountStrategy -> Bean
 *    - VIPCustomerDiscountStrategy -> Bean  
 *    - SeasonalDiscountStrategy -> Bean
 *    
 * 3. DEPENDENCY RESOLUTION:
 *    When creating DiscountService, Spring sees: List<DiscountStrategy>
 *    Spring finds ALL beans that implement DiscountStrategy interface
 *    
 * 4. COLLECTION INJECTION:
 *    Spring creates a List containing all DiscountStrategy beans:
 *    [RegularCustomerDiscountStrategy, PremiumCustomerDiscountStrategy, 
 *     VIPCustomerDiscountStrategy, SeasonalDiscountStrategy]
 *     
 * 5. INJECTION:
 *    Spring injects this complete list into DiscountService constructor
 * 
 * MAGIC: You never explicitly tell Spring which strategies to inject!
 *        Spring automatically finds them based on:
 *        - @Component annotation (makes them discoverable)  
 *        - implements DiscountStrategy (makes them type-compatible)
 *        - List<DiscountStrategy> parameter (tells Spring to collect all matching beans)
 * 
 * RESULT: Pure Strategy Pattern with zero configuration!
 *         Add new strategy? Just create class with @Component + implements DiscountStrategy
 *         Spring automatically includes it in the list - no other changes needed!
 */
}

/**
 * Discount service that uses strategies
 * 
 * HOW SPRING FINDS THE CORRECT STRATEGIES:
 * =======================================
 * 1. Spring scans for all classes annotated with @Component (or @Service, @Repository)
 * 2. It finds all classes that implement DiscountStrategy interface:
 *    - RegularCustomerDiscountStrategy (annotated with @Component)
 *    - PremiumCustomerDiscountStrategy (annotated with @Component) 
 *    - VIPCustomerDiscountStrategy (annotated with @Component)
 *    - SeasonalDiscountStrategy (annotated with @Component)
 * 3. Spring creates instances (beans) of all these strategy classes
 * 4. When creating DiscountService, Spring sees List<DiscountStrategy> parameter
 * 5. Spring automatically collects ALL beans that implement DiscountStrategy 
 * 6. Spring injects the complete list into the constructor
 * 
 * This is called "Collection Injection" - Spring automatically finds and injects
 * all beans of the same type into a collection (List, Set, Array)
 */
@Service
public class DiscountService {
    
    // ❌ THIS IS JUST A FIELD DECLARATION - NO INJECTION HAPPENS HERE!
    // This is just declaring a field to hold the list of strategies
    // At this point, the field is null/uninitialized
    // Spring does NOT inject anything here - it's just a variable declaration
    private final List<DiscountStrategy> discountStrategies;
    
    /**
     * ✅ THIS IS WHERE SPRING ACTUALLY DOES THE INJECTION!
     * Constructor Injection - Spring automatically injects ALL DiscountStrategy beans
     * 
     * WHEN DOES INJECTION HAPPEN:
     * ===========================
     * 1. Spring creates an instance of DiscountService
     * 2. Spring sees the constructor parameter: List<DiscountStrategy>
     * 3. Spring finds ALL beans implementing DiscountStrategy interface
     * 4. Spring creates a List containing all those strategy beans
     * 5. Spring calls this constructor with the populated List
     * 6. The constructor assigns the injected List to the field
     * 
     * @param discountStrategies - Spring finds all @Component classes implementing DiscountStrategy
     *                            and creates a List containing all of them, THEN passes it here
     */
    public DiscountService(List<DiscountStrategy> discountStrategies) {
        // ✅ THIS IS WHERE THE FIELD GETS POPULATED!
        // Spring has already created the List with all strategies
        // Now we're just assigning it to our instance field
        this.discountStrategies = discountStrategies;
        
        // Optional: Log what strategies were found (useful for debugging)
        System.out.println("DiscountService initialized with " + discountStrategies.size() + " strategies:");
        discountStrategies.forEach(strategy -> 
            System.out.println("  - " + strategy.getClass().getSimpleName() + ": " + strategy.getStrategyName())
        );
    }
    
    /*
     * DETAILED SPRING INJECTION SEQUENCE:
     * ===================================
     * 
     * STEP 1: APPLICATION STARTUP
     * ---------------------------
     * - Spring Boot starts up
     * - Component scanning begins (@ComponentScan)
     * 
     * STEP 2: BEAN DISCOVERY
     * ----------------------
     * Spring finds these classes:
     * - RegularCustomerDiscountStrategy (has @Component, implements DiscountStrategy)
     * - PremiumCustomerDiscountStrategy (has @Component, implements DiscountStrategy)  
     * - VIPCustomerDiscountStrategy (has @Component, implements DiscountStrategy)
     * - SeasonalDiscountStrategy (has @Component, implements DiscountStrategy)
     * - DiscountService (has @Service, needs List<DiscountStrategy>)
     * 
     * STEP 3: BEAN CREATION ORDER
     * ---------------------------
     * Spring creates beans in dependency order:
     * 
     * 3a) First, Spring creates all DiscountStrategy beans:
     *     Bean1: RegularCustomerDiscountStrategy instance
     *     Bean2: PremiumCustomerDiscountStrategy instance  
     *     Bean3: VIPCustomerDiscountStrategy instance
     *     Bean4: SeasonalDiscountStrategy instance
     * 
     * 3b) Then Spring needs to create DiscountService:
     *     - Spring sees: DiscountService needs List<DiscountStrategy>
     *     - Spring collects all DiscountStrategy beans: [Bean1, Bean2, Bean3, Bean4]
     *     - Spring creates List<DiscountStrategy> = [Bean1, Bean2, Bean3, Bean4]
     * 
     * STEP 4: CONSTRUCTOR INJECTION
     * -----------------------------
     * - Spring calls: new DiscountService(listOfAllStrategies)
     * - Inside constructor: this.discountStrategies = discountStrategies
     * - Field is now populated with all strategy instances
     * 
     * STEP 5: BEAN READY
     * ------------------
     * - DiscountService bean is now ready for use
     * - Field contains all strategy implementations
     * - Service can be injected into controllers/other services
     * 
     * KEY POINT: Field declaration ≠ Injection
     * ==========================================
     * private final List<DiscountStrategy> discountStrategies;  // ← DECLARATION ONLY
     * 
     * public DiscountService(List<DiscountStrategy> discountStrategies) {  // ← INJECTION HAPPENS HERE
     *     this.discountStrategies = discountStrategies;  // ← ASSIGNMENT HAPPENS HERE
     * }
     */
    
    /**
     * Calculate discount by trying ALL available strategies
     * This is the core of Strategy Pattern - iterate through all strategies
     * and apply those that are applicable to the customer
     */
    public DiscountCalculationResult calculateDiscount(double originalPrice, Customer customer) {
        List<DiscountApplication> appliedDiscounts = new ArrayList<>();
        double totalDiscount = 0;
        
        // STRATEGY PATTERN IN ACTION:
        // Iterate through ALL strategy implementations that Spring injected
        for (DiscountStrategy strategy : discountStrategies) {
            
            // Each strategy decides if it applies to this customer
            // This is polymorphism - same method call, different implementations
            if (strategy.isApplicable(customer)) {
                
                // If applicable, calculate the discount using this strategy
                // Again polymorphism - each strategy has its own calculation logic
                double discount = strategy.calculateDiscount(originalPrice, customer);
                
                if (discount > 0) {
                    // Track which discounts were applied
                    appliedDiscounts.add(new DiscountApplication(
                        strategy.getStrategyName(), discount
                    ));
                    totalDiscount += discount;
                    
                    // Debug log to see which strategy was applied
                    System.out.println("Applied " + strategy.getStrategyName() + 
                                     ": $" + String.format("%.2f", discount));
                }
            }
        }
        
        // Calculate final price (ensure it's not negative)
        double finalPrice = Math.max(0, originalPrice - totalDiscount);
        
        return new DiscountCalculationResult(originalPrice, totalDiscount, finalPrice, appliedDiscounts);
    }
    
    /**
     * Get list of available discounts for a customer
     * Useful for UI to show what discounts the customer qualifies for
     */
    public List<String> getAvailableDiscounts(Customer customer) {
        return discountStrategies.stream()
            .filter(strategy -> strategy.isApplicable(customer))  // Filter applicable strategies
            .map(DiscountStrategy::getStrategyName)              // Get strategy names
            .collect(Collectors.toList());                       // Collect to list
    }
}

/**
 * REST Controller
 */
@RestController
@RequestMapping("/api/pricing")
public class PricingController {
    
    private final DiscountService discountService;
    
    public PricingController(DiscountService discountService) {
        this.discountService = discountService;
    }
    
    @PostMapping("/calculate-discount")
    public ResponseEntity<DiscountCalculationResult> calculateDiscount(
            @RequestBody PricingRequest request) {
        
        DiscountCalculationResult result = discountService.calculateDiscount(
            request.getOriginalPrice(), request.getCustomer());
        
        return ResponseEntity.ok(result);
    }
    
    @GetMapping("/available-discounts/{customerId}")
    public ResponseEntity<List<String>> getAvailableDiscounts(
            @PathVariable String customerId) {
        
        // In real application, fetch customer from database
        Customer customer = new Customer(customerId, CustomerType.PREMIUM);
        
        List<String> discounts = discountService.getAvailableDiscounts(customer);
        return ResponseEntity.ok(discounts);
    }
}

// Supporting classes
public class Customer {
    private final String customerId;
    private final CustomerType customerType;
    
    public Customer(String customerId, CustomerType customerType) {
        this.customerId = customerId;
        this.customerType = customerType;
    }
    
    public String getCustomerId() { return customerId; }
    public CustomerType getCustomerType() { return customerType; }
}

public enum CustomerType {
    REGULAR, PREMIUM, VIP
}

public class DiscountCalculationResult {
    private final double originalPrice;
    private final double totalDiscount;
    private final double finalPrice;
    private final List<DiscountApplication> appliedDiscounts;
    
    public DiscountCalculationResult(double originalPrice, double totalDiscount, 
                                   double finalPrice, List<DiscountApplication> appliedDiscounts) {
        this.originalPrice = originalPrice;
        this.totalDiscount = totalDiscount;
        this.finalPrice = finalPrice;
        this.appliedDiscounts = appliedDiscounts;
    }
    
    // Getters
    public double getOriginalPrice() { return originalPrice; }
    public double getTotalDiscount() { return totalDiscount; }
    public double getFinalPrice() { return finalPrice; }
    public List<DiscountApplication> getAppliedDiscounts() { return appliedDiscounts; }
}

public class DiscountApplication {
    private final String discountName;
    private final double discountAmount;
    
    public DiscountApplication(String discountName, double discountAmount) {
        this.discountName = discountName;
        this.discountAmount = discountAmount;
    }
    
    public String getDiscountName() { return discountName; }
    public double getDiscountAmount() { return discountAmount; }
}

public class PricingRequest {
    private double originalPrice;
    private Customer customer;
    
    // Getters and setters
    public double getOriginalPrice() { return originalPrice; }
    public void setOriginalPrice(double originalPrice) { this.originalPrice = originalPrice; }
    public Customer getCustomer() { return customer; }
    public void setCustomer(Customer customer) { this.customer = customer; }
}
```

## Testing Strategy Pattern

```java
@ExtendWith(MockitoExtension.class)
class StrategyPatternTest {
    
    @Test
    void testPaymentStrategies() {
        PaymentProcessor processor = new PaymentProcessor();
        
        // Test Credit Card Payment
        CreditCardDetails cardDetails = new CreditCardDetails(
            "1234567890123456", "123", LocalDate.now().plusYears(2), "John Doe", 5000.0);
        
        PaymentResult result = processor.processPayment("CREDIT_CARD", 100.0, cardDetails);
        
        assertThat(result.isSuccess()).isTrue();
        assertThat(result.getTransactionId()).isNotNull();
        
        // Test PayPal Payment
        PayPalDetails paypalDetails = new PayPalDetails("john@example.com", "password123");
        
        result = processor.processPayment("PAYPAL", 50.0, paypalDetails);
        
        assertThat(result.isSuccess()).isTrue();
        
        // Test unsupported payment method
        result = processor.processPayment("UNSUPPORTED", 100.0, cardDetails);
        
        assertThat(result.isSuccess()).isFalse();
        assertThat(result.getMessage()).contains("Unsupported payment method");
    }
    
    @Test
    void testSortingStrategies() {
        SmartSorter<Integer> sorter = new SmartSorter<>();
        List<Integer> data = Arrays.asList(64, 34, 25, 12, 22, 11, 90);
        List<Integer> expected = Arrays.asList(11, 12, 22, 25, 34, 64, 90);
        
        // Test QuickSort
        List<Integer> quickSortData = new ArrayList<>(data);
        sorter.sort(quickSortData, Integer::compareTo, "quicksort");
        assertThat(quickSortData).isEqualTo(expected);
        
        // Test MergeSort
        List<Integer> mergeSortData = new ArrayList<>(data);
        sorter.sort(mergeSortData, Integer::compareTo, "mergesort");
        assertThat(mergeSortData).isEqualTo(expected);
        
        // Test HeapSort
        List<Integer> heapSortData = new ArrayList<>(data);
        sorter.sort(heapSortData, Integer::compareTo, "heapsort");
        assertThat(heapSortData).isEqualTo(expected);
    }
    
    @Test
    void testDiscountStrategies() {
        List<DiscountStrategy> strategies = Arrays.asList(
            new RegularCustomerDiscountStrategy(),
            new PremiumCustomerDiscountStrategy(),
            new VIPCustomerDiscountStrategy(),
            new SeasonalDiscountStrategy()
        );
        
        DiscountService discountService = new DiscountService(strategies);
        
        // Test VIP customer
        Customer vipCustomer = new Customer("VIP123", CustomerType.VIP);
        DiscountCalculationResult result = discountService.calculateDiscount(100.0, vipCustomer);
        
        assertThat(result.getOriginalPrice()).isEqualTo(100.0);
        assertThat(result.getTotalDiscount()).isGreaterThan(0);
        assertThat(result.getFinalPrice()).isLessThan(100.0);
        assertThat(result.getAppliedDiscounts()).isNotEmpty();
    }
}
```

## When to Use Strategy Pattern

### ✅ **Good Use Cases:**
- **Multiple algorithms**: Different ways to solve the same problem
- **Runtime algorithm selection**: Choose algorithm based on conditions
- **Avoiding conditionals**: Replace if-else chains with strategy objects
- **Plugin architectures**: Allow different implementations
- **Business rules**: Different pricing, discount, or validation rules

### ❌ **Avoid Strategy For:**
- **Simple conditions**: Basic if-else is sufficient
- **Stable algorithms**: No need for runtime changing
- **Single implementation**: No variation in behavior needed

## Summary

The Strategy pattern is excellent for:

**Key Benefits:**
- **Flexibility**: Easy to add new algorithms
- **Testability**: Each strategy can be tested independently  
- **Clean Code**: Eliminates complex conditional logic
- **Open/Closed Principle**: Add strategies without modifying existing code

**Common Use Cases:**
- **Payment Processing**: Different payment methods
- **Sorting Algorithms**: Various sorting strategies
- **Discount Calculation**: Multiple discount rules
- **Compression**: Different compression algorithms
- **Validation**: Various validation rules

Use Strategy when you have multiple ways to perform a task and want to make them interchangeable!

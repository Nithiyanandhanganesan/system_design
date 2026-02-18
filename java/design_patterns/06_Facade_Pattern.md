# Facade Pattern - Complete Guide

## Introduction

The **Facade Pattern** is a structural design pattern that provides a simplified interface to a complex subsystem of classes, libraries, or frameworks. It hides the complexity of the subsystem and provides a single entry point for clients to interact with it.

## Definition

> **Facade Pattern**: Provides a unified interface to a set of interfaces in a subsystem. Facade defines a higher-level interface that makes the subsystem easier to use.

## Problem Solved

- **Complex Subsystems**: Simplify interaction with complex libraries or frameworks
- **Tight Coupling**: Reduce dependencies between client code and subsystem classes
- **Learning Curve**: Provide easy-to-use interface for complex functionality
- **Code Organization**: Centralize subsystem interactions
- **API Abstraction**: Hide implementation details from clients
- **Legacy Integration**: Provide modern interface to legacy systems

## Structure

```
┌─────────────────┐    ┌─────────────────┐
│     Client      │───▶│     Facade      │
└─────────────────┘    ├─────────────────┤
                       │ +operation()    │
                       └─────────────────┘
                                │
                    ┌───────────┼───────────┐
                    │           │           │
        ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
        │  SubsystemA     │ │  SubsystemB     │ │  SubsystemC     │
        ├─────────────────┤ ├─────────────────┤ ├─────────────────┤
        │ +operationA()   │ │ +operationB()   │ │ +operationC()   │
        │ +methodA()      │ │ +methodB()      │ │ +methodC()      │
        └─────────────────┘ └─────────────────┘ └─────────────────┘
```

## Basic Implementation

### Complex Subsystem Classes

```java
/**
 * Complex subsystem for computer hardware components
 */

// CPU subsystem
public class CPU {
    public void freeze() {
        System.out.println("CPU: Freezing processor");
    }
    
    public void jump(long position) {
        System.out.println("CPU: Jumping to position " + position);
    }
    
    public void execute() {
        System.out.println("CPU: Executing instructions");
    }
    
    public void halt() {
        System.out.println("CPU: Halting processor");
    }
}

// Memory subsystem
public class Memory {
    public void load(long position, String data) {
        System.out.println("Memory: Loading data '" + data + "' at position " + position);
    }
    
    public String read(long position) {
        System.out.println("Memory: Reading from position " + position);
        return "data_" + position;
    }
    
    public void clear() {
        System.out.println("Memory: Clearing memory");
    }
}

// Hard Drive subsystem
public class HardDrive {
    public String read(long lba, int size) {
        System.out.println("HardDrive: Reading " + size + " bytes from sector " + lba);
        return "boot_data_" + lba;
    }
    
    public void write(long lba, String data) {
        System.out.println("HardDrive: Writing data to sector " + lba);
    }
    
    public void spinUp() {
        System.out.println("HardDrive: Spinning up");
    }
    
    public void spinDown() {
        System.out.println("HardDrive: Spinning down");
    }
}

// Graphics subsystem
public class Graphics {
    public void initializeDisplay() {
        System.out.println("Graphics: Initializing display");
    }
    
    public void loadDriver() {
        System.out.println("Graphics: Loading graphics driver");
    }
    
    public void setResolution(int width, int height) {
        System.out.println("Graphics: Setting resolution to " + width + "x" + height);
    }
    
    public void enableAcceleration() {
        System.out.println("Graphics: Enabling hardware acceleration");
    }
}

// Network subsystem
public class NetworkAdapter {
    public void initialize() {
        System.out.println("Network: Initializing network adapter");
    }
    
    public void connect(String ssid) {
        System.out.println("Network: Connecting to " + ssid);
    }
    
    public void disconnect() {
        System.out.println("Network: Disconnecting from network");
    }
    
    public boolean isConnected() {
        System.out.println("Network: Checking connection status");
        return true;
    }
}
```

### Facade Implementation

```java
/**
 * Computer Facade - Simplifies complex computer startup/shutdown process
 */
public class ComputerFacade {
    
    private final CPU cpu;
    private final Memory memory;
    private final HardDrive hardDrive;
    private final Graphics graphics;
    private final NetworkAdapter network;
    
    // Boot sequence constants
    private static final long BOOT_ADDRESS = 0x0000;
    private static final long BOOT_SECTOR = 0x01;
    private static final int BOOT_SIZE = 1024;
    
    public ComputerFacade() {
        this.cpu = new CPU();
        this.memory = new Memory();
        this.hardDrive = new HardDrive();
        this.graphics = new Graphics();
        this.network = new NetworkAdapter();
    }
    
    // Constructor injection for testing
    public ComputerFacade(CPU cpu, Memory memory, HardDrive hardDrive, 
                         Graphics graphics, NetworkAdapter network) {
        this.cpu = cpu;
        this.memory = memory;
        this.hardDrive = hardDrive;
        this.graphics = graphics;
        this.network = network;
    }
    
    /**
     * Simplified start operation - hides complex boot sequence
     */
    public void startComputer() {
        System.out.println("=== Starting Computer ===");
        
        try {
            // Step 1: Freeze CPU
            cpu.freeze();
            
            // Step 2: Initialize hardware
            hardDrive.spinUp();
            graphics.initializeDisplay();
            
            // Step 3: Load boot data
            String bootData = hardDrive.read(BOOT_SECTOR, BOOT_SIZE);
            memory.load(BOOT_ADDRESS, bootData);
            
            // Step 4: Start execution
            cpu.jump(BOOT_ADDRESS);
            cpu.execute();
            
            // Step 5: Initialize graphics
            graphics.loadDriver();
            graphics.setResolution(1920, 1080);
            graphics.enableAcceleration();
            
            // Step 6: Connect to network
            network.initialize();
            network.connect("Home_WiFi");
            
            System.out.println("Computer started successfully!\n");
            
        } catch (Exception e) {
            System.err.println("Failed to start computer: " + e.getMessage());
            shutdownComputer(); // Cleanup on failure
        }
    }
    
    /**
     * Simplified shutdown operation
     */
    public void shutdownComputer() {
        System.out.println("=== Shutting Down Computer ===");
        
        try {
            // Step 1: Disconnect network
            if (network.isConnected()) {
                network.disconnect();
            }
            
            // Step 2: Clear memory
            memory.clear();
            
            // Step 3: Stop CPU
            cpu.halt();
            
            // Step 4: Spin down hard drive
            hardDrive.spinDown();
            
            System.out.println("Computer shut down successfully!\n");
            
        } catch (Exception e) {
            System.err.println("Error during shutdown: " + e.getMessage());
        }
    }
    
    /**
     * Restart computer
     */
    public void restartComputer() {
        System.out.println("=== Restarting Computer ===");
        shutdownComputer();
        
        // Wait a moment between shutdown and startup
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        startComputer();
    }
    
    /**
     * Sleep mode - partial shutdown
     */
    public void sleepMode() {
        System.out.println("=== Entering Sleep Mode ===");
        network.disconnect();
        hardDrive.spinDown();
        cpu.freeze();
        System.out.println("Computer is now in sleep mode\n");
    }
    
    /**
     * Wake up from sleep
     */
    public void wakeUp() {
        System.out.println("=== Waking Up Computer ===");
        hardDrive.spinUp();
        cpu.execute();
        network.connect("Home_WiFi");
        System.out.println("Computer is now awake\n");
    }
    
    /**
     * Get system status
     */
    public SystemStatus getSystemStatus() {
        boolean networkConnected = network.isConnected();
        String memoryData = memory.read(BOOT_ADDRESS);
        
        return new SystemStatus(networkConnected, memoryData != null);
    }
    
    /**
     * System status information
     */
    public static class SystemStatus {
        private final boolean networkConnected;
        private final boolean memoryLoaded;
        
        public SystemStatus(boolean networkConnected, boolean memoryLoaded) {
            this.networkConnected = networkConnected;
            this.memoryLoaded = memoryLoaded;
        }
        
        public boolean isNetworkConnected() { return networkConnected; }
        public boolean isMemoryLoaded() { return memoryLoaded; }
        
        @Override
        public String toString() {
            return String.format("SystemStatus{network=%s, memory=%s}", 
                networkConnected ? "Connected" : "Disconnected",
                memoryLoaded ? "Loaded" : "Empty");
        }
    }
}
```

## Real-World Example: Banking System Facade

### Complex Banking Subsystems

```java
/**
 * Account Management Subsystem
 */
public class AccountManager {
    
    private final Map<String, Double> accounts = new HashMap<>();
    
    public AccountManager() {
        // Initialize with some demo accounts
        accounts.put("123456789", 5000.0);
        accounts.put("987654321", 2500.0);
    }
    
    public boolean validateAccount(String accountNumber) {
        boolean valid = accounts.containsKey(accountNumber);
        System.out.println("AccountManager: Account " + accountNumber + 
                         (valid ? " is valid" : " is invalid"));
        return valid;
    }
    
    public double getBalance(String accountNumber) {
        double balance = accounts.getOrDefault(accountNumber, 0.0);
        System.out.println("AccountManager: Account " + accountNumber + 
                         " balance is $" + String.format("%.2f", balance));
        return balance;
    }
    
    public void updateBalance(String accountNumber, double newBalance) {
        accounts.put(accountNumber, newBalance);
        System.out.println("AccountManager: Updated account " + accountNumber + 
                         " balance to $" + String.format("%.2f", newBalance));
    }
    
    public boolean freezeAccount(String accountNumber) {
        System.out.println("AccountManager: Freezing account " + accountNumber);
        return true;
    }
}

/**
 * Security Manager Subsystem
 */
public class SecurityManager {
    
    private final Map<String, String> pins = new HashMap<>();
    private final Set<String> authenticatedSessions = new HashSet<>();
    
    public SecurityManager() {
        pins.put("123456789", "1234");
        pins.put("987654321", "5678");
    }
    
    public boolean authenticateUser(String accountNumber, String pin) {
        boolean authenticated = pins.get(accountNumber) != null && 
                              pins.get(accountNumber).equals(pin);
        
        if (authenticated) {
            String sessionId = generateSessionId(accountNumber);
            authenticatedSessions.add(sessionId);
            System.out.println("SecurityManager: User authenticated for account " + accountNumber);
        } else {
            System.out.println("SecurityManager: Authentication failed for account " + accountNumber);
        }
        
        return authenticated;
    }
    
    public boolean isSessionValid(String accountNumber) {
        String sessionId = generateSessionId(accountNumber);
        boolean valid = authenticatedSessions.contains(sessionId);
        System.out.println("SecurityManager: Session for account " + accountNumber + 
                         (valid ? " is valid" : " is invalid"));
        return valid;
    }
    
    public void logSecurityEvent(String event, String accountNumber) {
        System.out.println("SecurityManager: SECURITY LOG - " + event + 
                         " for account " + accountNumber);
    }
    
    public void invalidateSession(String accountNumber) {
        String sessionId = generateSessionId(accountNumber);
        authenticatedSessions.remove(sessionId);
        System.out.println("SecurityManager: Session invalidated for account " + accountNumber);
    }
    
    private String generateSessionId(String accountNumber) {
        return "SESSION_" + accountNumber;
    }
}

/**
 * Transaction Processor Subsystem
 */
public class TransactionProcessor {
    
    private final List<Transaction> transactionHistory = new ArrayList<>();
    
    public boolean processDebit(String accountNumber, double amount, String description) {
        System.out.println("TransactionProcessor: Processing debit of $" + 
                         String.format("%.2f", amount) + " from account " + accountNumber);
        
        Transaction transaction = new Transaction(
            generateTransactionId(),
            accountNumber,
            TransactionType.DEBIT,
            amount,
            description,
            LocalDateTime.now()
        );
        
        transactionHistory.add(transaction);
        
        // Simulate processing time
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        System.out.println("TransactionProcessor: Debit transaction completed - ID: " + 
                         transaction.getTransactionId());
        return true;
    }
    
    public boolean processCredit(String accountNumber, double amount, String description) {
        System.out.println("TransactionProcessor: Processing credit of $" + 
                         String.format("%.2f", amount) + " to account " + accountNumber);
        
        Transaction transaction = new Transaction(
            generateTransactionId(),
            accountNumber,
            TransactionType.CREDIT,
            amount,
            description,
            LocalDateTime.now()
        );
        
        transactionHistory.add(transaction);
        
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        System.out.println("TransactionProcessor: Credit transaction completed - ID: " + 
                         transaction.getTransactionId());
        return true;
    }
    
    public List<Transaction> getTransactionHistory(String accountNumber) {
        List<Transaction> accountTransactions = transactionHistory.stream()
            .filter(t -> t.getAccountNumber().equals(accountNumber))
            .collect(Collectors.toList());
        
        System.out.println("TransactionProcessor: Retrieved " + accountTransactions.size() + 
                         " transactions for account " + accountNumber);
        return accountTransactions;
    }
    
    private String generateTransactionId() {
        return "TXN_" + System.currentTimeMillis() + "_" + (int)(Math.random() * 1000);
    }
}

/**
 * Notification Service Subsystem
 */
public class NotificationService {
    
    public void sendSMS(String phoneNumber, String message) {
        System.out.println("NotificationService: SMS sent to " + phoneNumber + ": " + message);
    }
    
    public void sendEmail(String email, String subject, String message) {
        System.out.println("NotificationService: Email sent to " + email + 
                         " - Subject: " + subject);
    }
    
    public void sendPushNotification(String deviceId, String message) {
        System.out.println("NotificationService: Push notification sent to device " + 
                         deviceId + ": " + message);
    }
    
    public void logNotification(String type, String recipient, String message) {
        System.out.println("NotificationService: LOG - " + type + " notification to " + 
                         recipient + ": " + message);
    }
}

/**
 * Transaction data class
 */
public class Transaction {
    private final String transactionId;
    private final String accountNumber;
    private final TransactionType type;
    private final double amount;
    private final String description;
    private final LocalDateTime timestamp;
    
    public Transaction(String transactionId, String accountNumber, TransactionType type,
                      double amount, String description, LocalDateTime timestamp) {
        this.transactionId = transactionId;
        this.accountNumber = accountNumber;
        this.type = type;
        this.amount = amount;
        this.description = description;
        this.timestamp = timestamp;
    }
    
    // Getters
    public String getTransactionId() { return transactionId; }
    public String getAccountNumber() { return accountNumber; }
    public TransactionType getType() { return type; }
    public double getAmount() { return amount; }
    public String getDescription() { return description; }
    public LocalDateTime getTimestamp() { return timestamp; }
    
    @Override
    public String toString() {
        return String.format("Transaction{id='%s', type=%s, amount=%.2f, desc='%s', time=%s}",
            transactionId, type, amount, description, timestamp);
    }
}

public enum TransactionType {
    DEBIT, CREDIT
}
```

### Banking Facade Implementation

```java
/**
 * Banking Facade - Simplifies complex banking operations
 */
public class BankingFacade {
    
    private final AccountManager accountManager;
    private final SecurityManager securityManager;
    private final TransactionProcessor transactionProcessor;
    private final NotificationService notificationService;
    
    // Customer contact information (in real system, would be in database)
    private final Map<String, CustomerContact> customerContacts = new HashMap<>();
    
    public BankingFacade() {
        this.accountManager = new AccountManager();
        this.securityManager = new SecurityManager();
        this.transactionProcessor = new TransactionProcessor();
        this.notificationService = new NotificationService();
        
        initializeCustomerContacts();
    }
    
    // Constructor injection for testing
    public BankingFacade(AccountManager accountManager, SecurityManager securityManager,
                        TransactionProcessor transactionProcessor, NotificationService notificationService) {
        this.accountManager = accountManager;
        this.securityManager = securityManager;
        this.transactionProcessor = transactionProcessor;
        this.notificationService = notificationService;
        
        initializeCustomerContacts();
    }
    
    private void initializeCustomerContacts() {
        customerContacts.put("123456789", 
            new CustomerContact("john.doe@email.com", "+1234567890", "DEVICE_123"));
        customerContacts.put("987654321", 
            new CustomerContact("jane.smith@email.com", "+0987654321", "DEVICE_456"));
    }
    
    /**
     * Simplified withdrawal operation
     */
    public TransactionResult withdraw(String accountNumber, String pin, double amount, String description) {
        System.out.println("\n=== Processing Withdrawal ===");
        
        try {
            // Step 1: Validate account
            if (!accountManager.validateAccount(accountNumber)) {
                return new TransactionResult(false, "Invalid account number", null);
            }
            
            // Step 2: Authenticate user
            if (!securityManager.authenticateUser(accountNumber, pin)) {
                securityManager.logSecurityEvent("FAILED_AUTHENTICATION", accountNumber);
                return new TransactionResult(false, "Authentication failed", null);
            }
            
            // Step 3: Check balance
            double currentBalance = accountManager.getBalance(accountNumber);
            if (currentBalance < amount) {
                securityManager.logSecurityEvent("INSUFFICIENT_FUNDS_ATTEMPT", accountNumber);
                return new TransactionResult(false, "Insufficient funds", null);
            }
            
            // Step 4: Process transaction
            boolean transactionSuccess = transactionProcessor.processDebit(
                accountNumber, amount, description);
            
            if (!transactionSuccess) {
                return new TransactionResult(false, "Transaction processing failed", null);
            }
            
            // Step 5: Update account balance
            double newBalance = currentBalance - amount;
            accountManager.updateBalance(accountNumber, newBalance);
            
            // Step 6: Send notifications
            sendWithdrawalNotifications(accountNumber, amount, newBalance);
            
            // Step 7: Log security event
            securityManager.logSecurityEvent("SUCCESSFUL_WITHDRAWAL", accountNumber);
            
            String transactionId = "TXN_" + System.currentTimeMillis();
            System.out.println("Withdrawal completed successfully!\n");
            
            return new TransactionResult(true, "Withdrawal successful", transactionId);
            
        } catch (Exception e) {
            securityManager.logSecurityEvent("WITHDRAWAL_ERROR: " + e.getMessage(), accountNumber);
            return new TransactionResult(false, "System error: " + e.getMessage(), null);
        }
    }
    
    /**
     * Simplified deposit operation
     */
    public TransactionResult deposit(String accountNumber, String pin, double amount, String description) {
        System.out.println("\n=== Processing Deposit ===");
        
        try {
            // Step 1: Validate account
            if (!accountManager.validateAccount(accountNumber)) {
                return new TransactionResult(false, "Invalid account number", null);
            }
            
            // Step 2: Authenticate user
            if (!securityManager.authenticateUser(accountNumber, pin)) {
                securityManager.logSecurityEvent("FAILED_AUTHENTICATION", accountNumber);
                return new TransactionResult(false, "Authentication failed", null);
            }
            
            // Step 3: Process transaction
            boolean transactionSuccess = transactionProcessor.processCredit(
                accountNumber, amount, description);
            
            if (!transactionSuccess) {
                return new TransactionResult(false, "Transaction processing failed", null);
            }
            
            // Step 4: Update account balance
            double currentBalance = accountManager.getBalance(accountNumber);
            double newBalance = currentBalance + amount;
            accountManager.updateBalance(accountNumber, newBalance);
            
            // Step 5: Send notifications
            sendDepositNotifications(accountNumber, amount, newBalance);
            
            // Step 6: Log security event
            securityManager.logSecurityEvent("SUCCESSFUL_DEPOSIT", accountNumber);
            
            String transactionId = "TXN_" + System.currentTimeMillis();
            System.out.println("Deposit completed successfully!\n");
            
            return new TransactionResult(true, "Deposit successful", transactionId);
            
        } catch (Exception e) {
            securityManager.logSecurityEvent("DEPOSIT_ERROR: " + e.getMessage(), accountNumber);
            return new TransactionResult(false, "System error: " + e.getMessage(), null);
        }
    }
    
    /**
     * Simplified balance inquiry
     */
    public BalanceResult getBalance(String accountNumber, String pin) {
        System.out.println("\n=== Balance Inquiry ===");
        
        try {
            // Step 1: Validate account
            if (!accountManager.validateAccount(accountNumber)) {
                return new BalanceResult(false, "Invalid account number", 0.0);
            }
            
            // Step 2: Authenticate user
            if (!securityManager.authenticateUser(accountNumber, pin)) {
                securityManager.logSecurityEvent("FAILED_AUTHENTICATION", accountNumber);
                return new BalanceResult(false, "Authentication failed", 0.0);
            }
            
            // Step 3: Get balance
            double balance = accountManager.getBalance(accountNumber);
            
            // Step 4: Log security event
            securityManager.logSecurityEvent("BALANCE_INQUIRY", accountNumber);
            
            System.out.println("Balance inquiry completed successfully!\n");
            
            return new BalanceResult(true, "Balance retrieved successfully", balance);
            
        } catch (Exception e) {
            securityManager.logSecurityEvent("BALANCE_INQUIRY_ERROR: " + e.getMessage(), accountNumber);
            return new BalanceResult(false, "System error: " + e.getMessage(), 0.0);
        }
    }
    
    /**
     * Get transaction history
     */
    public List<Transaction> getTransactionHistory(String accountNumber, String pin) {
        System.out.println("\n=== Transaction History Request ===");
        
        try {
            // Validate and authenticate
            if (!accountManager.validateAccount(accountNumber)) {
                return new ArrayList<>();
            }
            
            if (!securityManager.authenticateUser(accountNumber, pin)) {
                securityManager.logSecurityEvent("FAILED_AUTHENTICATION", accountNumber);
                return new ArrayList<>();
            }
            
            // Get transaction history
            List<Transaction> transactions = transactionProcessor.getTransactionHistory(accountNumber);
            
            securityManager.logSecurityEvent("TRANSACTION_HISTORY_REQUEST", accountNumber);
            
            return transactions;
            
        } catch (Exception e) {
            securityManager.logSecurityEvent("HISTORY_REQUEST_ERROR: " + e.getMessage(), accountNumber);
            return new ArrayList<>();
        }
    }
    
    /**
     * Logout user
     */
    public void logout(String accountNumber) {
        securityManager.invalidateSession(accountNumber);
        securityManager.logSecurityEvent("USER_LOGOUT", accountNumber);
        System.out.println("User logged out successfully\n");
    }
    
    private void sendWithdrawalNotifications(String accountNumber, double amount, double newBalance) {
        CustomerContact contact = customerContacts.get(accountNumber);
        if (contact != null) {
            String message = String.format("Withdrawal of $%.2f completed. New balance: $%.2f", 
                amount, newBalance);
            
            notificationService.sendSMS(contact.getPhoneNumber(), message);
            notificationService.sendPushNotification(contact.getDeviceId(), message);
            notificationService.sendEmail(contact.getEmail(), "Withdrawal Confirmation", message);
        }
    }
    
    private void sendDepositNotifications(String accountNumber, double amount, double newBalance) {
        CustomerContact contact = customerContacts.get(accountNumber);
        if (contact != null) {
            String message = String.format("Deposit of $%.2f completed. New balance: $%.2f", 
                amount, newBalance);
            
            notificationService.sendSMS(contact.getPhoneNumber(), message);
            notificationService.sendPushNotification(contact.getDeviceId(), message);
            notificationService.sendEmail(contact.getEmail(), "Deposit Confirmation", message);
        }
    }
    
    // Result classes
    public static class TransactionResult {
        private final boolean success;
        private final String message;
        private final String transactionId;
        
        public TransactionResult(boolean success, String message, String transactionId) {
            this.success = success;
            this.message = message;
            this.transactionId = transactionId;
        }
        
        public boolean isSuccess() { return success; }
        public String getMessage() { return message; }
        public String getTransactionId() { return transactionId; }
    }
    
    public static class BalanceResult {
        private final boolean success;
        private final String message;
        private final double balance;
        
        public BalanceResult(boolean success, String message, double balance) {
            this.success = success;
            this.message = message;
            this.balance = balance;
        }
        
        public boolean isSuccess() { return success; }
        public String getMessage() { return message; }
        public double getBalance() { return balance; }
    }
    
    private static class CustomerContact {
        private final String email;
        private final String phoneNumber;
        private final String deviceId;
        
        public CustomerContact(String email, String phoneNumber, String deviceId) {
            this.email = email;
            this.phoneNumber = phoneNumber;
            this.deviceId = deviceId;
        }
        
        public String getEmail() { return email; }
        public String getPhoneNumber() { return phoneNumber; }
        public String getDeviceId() { return deviceId; }
    }
}
```

## Spring Boot Integration

### Service Facade in Spring Boot

```java
/**
 * E-commerce Order Processing Facade
 */
@Service
@Transactional
public class OrderProcessingFacade {
    
    private final InventoryService inventoryService;
    private final PaymentService paymentService;
    private final ShippingService shippingService;
    private final NotificationService notificationService;
    private final OrderRepository orderRepository;
    private final CustomerService customerService;
    
    public OrderProcessingFacade(InventoryService inventoryService,
                               PaymentService paymentService,
                               ShippingService shippingService,
                               NotificationService notificationService,
                               OrderRepository orderRepository,
                               CustomerService customerService) {
        this.inventoryService = inventoryService;
        this.paymentService = paymentService;
        this.shippingService = shippingService;
        this.notificationService = notificationService;
        this.orderRepository = orderRepository;
        this.customerService = customerService;
    }
    
    /**
     * Simplified order placement - coordinates multiple services
     */
    public OrderResult placeOrder(OrderRequest orderRequest) {
        String orderId = generateOrderId();
        
        try {
            // Step 1: Validate customer
            Customer customer = customerService.validateCustomer(orderRequest.getCustomerId());
            if (customer == null) {
                return OrderResult.failure("Invalid customer");
            }
            
            // Step 2: Check inventory
            InventoryCheckResult inventoryCheck = inventoryService.checkAvailability(
                orderRequest.getItems());
            
            if (!inventoryCheck.isAvailable()) {
                return OrderResult.failure("Items not available: " + 
                    inventoryCheck.getUnavailableItems());
            }
            
            // Step 3: Calculate total
            BigDecimal totalAmount = calculateTotal(orderRequest.getItems());
            
            // Step 4: Process payment
            PaymentResult paymentResult = paymentService.processPayment(
                customer.getPaymentMethod(),
                totalAmount,
                orderId
            );
            
            if (!paymentResult.isSuccess()) {
                return OrderResult.failure("Payment failed: " + paymentResult.getErrorMessage());
            }
            
            // Step 5: Reserve inventory
            inventoryService.reserveItems(orderRequest.getItems(), orderId);
            
            // Step 6: Create order record
            Order order = new Order(orderId, orderRequest.getCustomerId(), 
                orderRequest.getItems(), totalAmount, OrderStatus.CONFIRMED);
            orderRepository.save(order);
            
            // Step 7: Schedule shipping
            ShippingResult shippingResult = shippingService.scheduleShipping(
                order, customer.getShippingAddress());
            
            // Step 8: Send notifications
            sendOrderConfirmationNotifications(customer, order, shippingResult);
            
            return OrderResult.success(orderId, "Order placed successfully", 
                shippingResult.getTrackingNumber());
            
        } catch (Exception e) {
            // Rollback operations
            rollbackOrder(orderId);
            return OrderResult.failure("Order processing failed: " + e.getMessage());
        }
    }
    
    /**
     * Simplified order cancellation
     */
    public CancellationResult cancelOrder(String orderId, String customerId) {
        try {
            // Step 1: Validate order
            Order order = orderRepository.findById(orderId)
                .orElse(null);
            
            if (order == null) {
                return CancellationResult.failure("Order not found");
            }
            
            if (!order.getCustomerId().equals(customerId)) {
                return CancellationResult.failure("Unauthorized");
            }
            
            if (order.getStatus() == OrderStatus.SHIPPED) {
                return CancellationResult.failure("Cannot cancel shipped order");
            }
            
            // Step 2: Refund payment
            PaymentResult refundResult = paymentService.refundPayment(
                order.getPaymentId(), order.getTotalAmount());
            
            if (!refundResult.isSuccess()) {
                return CancellationResult.failure("Refund failed: " + 
                    refundResult.getErrorMessage());
            }
            
            // Step 3: Release inventory
            inventoryService.releaseReservation(order.getItems(), orderId);
            
            // Step 4: Cancel shipping if scheduled
            if (order.getStatus() == OrderStatus.PROCESSING) {
                shippingService.cancelShipping(order.getShippingId());
            }
            
            // Step 5: Update order status
            order.setStatus(OrderStatus.CANCELLED);
            orderRepository.save(order);
            
            // Step 6: Notify customer
            Customer customer = customerService.findById(customerId);
            notificationService.sendCancellationNotification(customer, order);
            
            return CancellationResult.success("Order cancelled successfully");
            
        } catch (Exception e) {
            return CancellationResult.failure("Cancellation failed: " + e.getMessage());
        }
    }
    
    /**
     * Get order status with all related information
     */
    public OrderStatusInfo getOrderStatus(String orderId, String customerId) {
        try {
            Order order = orderRepository.findById(orderId)
                .orElse(null);
            
            if (order == null || !order.getCustomerId().equals(customerId)) {
                return null;
            }
            
            // Gather information from various services
            PaymentInfo paymentInfo = paymentService.getPaymentInfo(order.getPaymentId());
            ShippingInfo shippingInfo = shippingService.getShippingInfo(order.getShippingId());
            InventoryStatus inventoryStatus = inventoryService.getReservationStatus(orderId);
            
            return new OrderStatusInfo(order, paymentInfo, shippingInfo, inventoryStatus);
            
        } catch (Exception e) {
            return null;
        }
    }
    
    private BigDecimal calculateTotal(List<OrderItem> items) {
        return items.stream()
            .map(item -> item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
    
    private void sendOrderConfirmationNotifications(Customer customer, Order order, 
                                                   ShippingResult shippingResult) {
        notificationService.sendOrderConfirmationEmail(customer, order);
        notificationService.sendOrderConfirmationSMS(customer, order);
        
        if (shippingResult.getTrackingNumber() != null) {
            notificationService.sendShippingNotification(customer, 
                shippingResult.getTrackingNumber());
        }
    }
    
    private void rollbackOrder(String orderId) {
        try {
            inventoryService.releaseReservation(orderId);
            paymentService.voidPayment(orderId);
            orderRepository.deleteById(orderId);
        } catch (Exception e) {
            // Log rollback errors
        }
    }
    
    private String generateOrderId() {
        return "ORD_" + System.currentTimeMillis() + "_" + (int)(Math.random() * 1000);
    }
}

/**
 * REST Controller using the facade
 */
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    private final OrderProcessingFacade orderFacade;
    
    public OrderController(OrderProcessingFacade orderFacade) {
        this.orderFacade = orderFacade;
    }
    
    @PostMapping
    public ResponseEntity<OrderResult> placeOrder(@RequestBody @Valid OrderRequest request) {
        OrderResult result = orderFacade.placeOrder(request);
        
        if (result.isSuccess()) {
            return ResponseEntity.status(HttpStatus.CREATED).body(result);
        } else {
            return ResponseEntity.badRequest().body(result);
        }
    }
    
    @DeleteMapping("/{orderId}")
    public ResponseEntity<CancellationResult> cancelOrder(
            @PathVariable String orderId,
            @RequestHeader("Customer-Id") String customerId) {
        
        CancellationResult result = orderFacade.cancelOrder(orderId, customerId);
        
        if (result.isSuccess()) {
            return ResponseEntity.ok(result);
        } else {
            return ResponseEntity.badRequest().body(result);
        }
    }
    
    @GetMapping("/{orderId}/status")
    public ResponseEntity<OrderStatusInfo> getOrderStatus(
            @PathVariable String orderId,
            @RequestHeader("Customer-Id") String customerId) {
        
        OrderStatusInfo statusInfo = orderFacade.getOrderStatus(orderId, customerId);
        
        if (statusInfo != null) {
            return ResponseEntity.ok(statusInfo);
        } else {
            return ResponseEntity.notFound().build();
        }
    }
}
```

## Configuration and Properties

### Configuration for Facade Pattern

```java
/**
 * Configuration for facade-related services
 */
@Configuration
@EnableConfigurationProperties({OrderProcessingProperties.class, BankingProperties.class})
public class FacadeConfig {
    
    @Bean
    @ConditionalOnProperty(value = "banking.facade.enabled", havingValue = "true")
    public BankingFacade bankingFacade() {
        return new BankingFacade();
    }
    
    @Bean
    @ConditionalOnProperty(value = "computer.facade.enabled", havingValue = "true")
    public ComputerFacade computerFacade() {
        return new ComputerFacade();
    }
    
    @Bean
    @ConditionalOnMissingBean
    public OrderProcessingFacade orderProcessingFacade(
            InventoryService inventoryService,
            PaymentService paymentService,
            ShippingService shippingService,
            NotificationService notificationService,
            OrderRepository orderRepository,
            CustomerService customerService) {
        
        return new OrderProcessingFacade(inventoryService, paymentService, 
            shippingService, notificationService, orderRepository, customerService);
    }
}

@ConfigurationProperties(prefix = "order.processing")
@Data
public class OrderProcessingProperties {
    private boolean enableNotifications = true;
    private boolean enableInventoryReservation = true;
    private int orderTimeoutMinutes = 30;
    private String defaultShippingMethod = "STANDARD";
}

@ConfigurationProperties(prefix = "banking")
@Data
public class BankingProperties {
    private boolean enableSmsNotifications = true;
    private boolean enableEmailNotifications = true;
    private double maxDailyWithdrawal = 5000.0;
    private int sessionTimeoutMinutes = 15;
}
```

## Testing Facade Pattern

### Unit Tests

```java
/**
 * Tests for Facade Pattern implementations
 */
@ExtendWith(MockitoExtension.class)
class FacadePatternTest {
    
    @Mock private CPU mockCpu;
    @Mock private Memory mockMemory;
    @Mock private HardDrive mockHardDrive;
    @Mock private Graphics mockGraphics;
    @Mock private NetworkAdapter mockNetwork;
    
    private ComputerFacade computerFacade;
    
    @BeforeEach
    void setUp() {
        computerFacade = new ComputerFacade(mockCpu, mockMemory, 
            mockHardDrive, mockGraphics, mockNetwork);
    }
    
    @Test
    void testComputerStartup() {
        // Given
        when(mockHardDrive.read(anyLong(), anyInt())).thenReturn("boot_data");
        when(mockNetwork.isConnected()).thenReturn(false, true);
        
        // When
        computerFacade.startComputer();
        
        // Then
        verify(mockCpu).freeze();
        verify(mockCpu).jump(anyLong());
        verify(mockCpu).execute();
        verify(mockHardDrive).spinUp();
        verify(mockMemory).load(anyLong(), anyString());
        verify(mockGraphics).initializeDisplay();
        verify(mockNetwork).connect("Home_WiFi");
    }
    
    @Test
    void testComputerShutdown() {
        // Given
        when(mockNetwork.isConnected()).thenReturn(true);
        
        // When
        computerFacade.shutdownComputer();
        
        // Then
        verify(mockNetwork).disconnect();
        verify(mockMemory).clear();
        verify(mockCpu).halt();
        verify(mockHardDrive).spinDown();
    }
    
    @Test
    void testSystemStatus() {
        // Given
        when(mockNetwork.isConnected()).thenReturn(true);
        when(mockMemory.read(anyLong())).thenReturn("boot_data");
        
        // When
        ComputerFacade.SystemStatus status = computerFacade.getSystemStatus();
        
        // Then
        assertThat(status.isNetworkConnected()).isTrue();
        assertThat(status.isMemoryLoaded()).isTrue();
    }
}

/**
 * Banking facade tests
 */
@ExtendWith(MockitoExtension.class)
class BankingFacadeTest {
    
    @Mock private AccountManager mockAccountManager;
    @Mock private SecurityManager mockSecurityManager;
    @Mock private TransactionProcessor mockTransactionProcessor;
    @Mock private NotificationService mockNotificationService;
    
    private BankingFacade bankingFacade;
    
    @BeforeEach
    void setUp() {
        bankingFacade = new BankingFacade(mockAccountManager, mockSecurityManager,
            mockTransactionProcessor, mockNotificationService);
    }
    
    @Test
    void testSuccessfulWithdrawal() {
        // Given
        String accountNumber = "123456789";
        String pin = "1234";
        double amount = 100.0;
        double currentBalance = 500.0;
        
        when(mockAccountManager.validateAccount(accountNumber)).thenReturn(true);
        when(mockSecurityManager.authenticateUser(accountNumber, pin)).thenReturn(true);
        when(mockAccountManager.getBalance(accountNumber)).thenReturn(currentBalance);
        when(mockTransactionProcessor.processDebit(eq(accountNumber), eq(amount), anyString()))
            .thenReturn(true);
        
        // When
        BankingFacade.TransactionResult result = bankingFacade.withdraw(
            accountNumber, pin, amount, "ATM Withdrawal");
        
        // Then
        assertThat(result.isSuccess()).isTrue();
        assertThat(result.getMessage()).isEqualTo("Withdrawal successful");
        assertThat(result.getTransactionId()).isNotNull();
        
        verify(mockAccountManager).updateBalance(accountNumber, currentBalance - amount);
        verify(mockTransactionProcessor).processDebit(accountNumber, amount, "ATM Withdrawal");
    }
    
    @Test
    void testWithdrawalWithInsufficientFunds() {
        // Given
        String accountNumber = "123456789";
        String pin = "1234";
        double amount = 600.0;
        double currentBalance = 500.0;
        
        when(mockAccountManager.validateAccount(accountNumber)).thenReturn(true);
        when(mockSecurityManager.authenticateUser(accountNumber, pin)).thenReturn(true);
        when(mockAccountManager.getBalance(accountNumber)).thenReturn(currentBalance);
        
        // When
        BankingFacade.TransactionResult result = bankingFacade.withdraw(
            accountNumber, pin, amount, "ATM Withdrawal");
        
        // Then
        assertThat(result.isSuccess()).isFalse();
        assertThat(result.getMessage()).isEqualTo("Insufficient funds");
        
        verify(mockTransactionProcessor, never()).processDebit(anyString(), anyDouble(), anyString());
        verify(mockAccountManager, never()).updateBalance(anyString(), anyDouble());
    }
    
    @Test
    void testWithdrawalWithInvalidPin() {
        // Given
        String accountNumber = "123456789";
        String invalidPin = "0000";
        
        when(mockAccountManager.validateAccount(accountNumber)).thenReturn(true);
        when(mockSecurityManager.authenticateUser(accountNumber, invalidPin)).thenReturn(false);
        
        // When
        BankingFacade.TransactionResult result = bankingFacade.withdraw(
            accountNumber, invalidPin, 100.0, "ATM Withdrawal");
        
        // Then
        assertThat(result.isSuccess()).isFalse();
        assertThat(result.getMessage()).isEqualTo("Authentication failed");
        
        verify(mockSecurityManager).logSecurityEvent("FAILED_AUTHENTICATION", accountNumber);
        verify(mockAccountManager, never()).getBalance(anyString());
    }
}
```

### Integration Tests

```java
/**
 * Integration tests for Spring Boot facade
 */
@SpringBootTest
@AutoConfigureTestDatabase
@TestPropertySource(properties = {
    "order.processing.enable-notifications=true",
    "order.processing.enable-inventory-reservation=true"
})
class OrderProcessingFacadeIntegrationTest {
    
    @Autowired
    private OrderProcessingFacade orderFacade;
    
    @Autowired
    private OrderRepository orderRepository;
    
    @MockBean
    private PaymentService paymentService;
    
    @MockBean
    private InventoryService inventoryService;
    
    @MockBean
    private ShippingService shippingService;
    
    @Test
    @Transactional
    void testCompleteOrderProcess() {
        // Given
        OrderRequest orderRequest = createTestOrderRequest();
        
        when(inventoryService.checkAvailability(anyList()))
            .thenReturn(new InventoryCheckResult(true, List.of()));
        when(paymentService.processPayment(any(), any(), anyString()))
            .thenReturn(PaymentResult.success("PAY_123"));
        when(shippingService.scheduleShipping(any(), any()))
            .thenReturn(new ShippingResult(true, "TRACK_123"));
        
        // When
        OrderResult result = orderFacade.placeOrder(orderRequest);
        
        // Then
        assertThat(result.isSuccess()).isTrue();
        assertThat(result.getOrderId()).isNotNull();
        
        // Verify order was saved
        Optional<Order> savedOrder = orderRepository.findById(result.getOrderId());
        assertThat(savedOrder).isPresent();
        assertThat(savedOrder.get().getStatus()).isEqualTo(OrderStatus.CONFIRMED);
    }
    
    private OrderRequest createTestOrderRequest() {
        OrderItem item = new OrderItem("ITEM_001", 2, BigDecimal.valueOf(25.00));
        return new OrderRequest("CUST_123", List.of(item));
    }
}
```

## Best Practices

### ✅ **Do's:**

1. **Keep facade interface simple** - hide complex subsystem interactions
2. **Don't expose subsystem classes** through the facade
3. **Handle errors gracefully** - provide meaningful error messages
4. **Make facades stateless** when possible
5. **Use dependency injection** for subsystem components
6. **Provide multiple facade methods** for different use cases
7. **Document the simplified operations** clearly

### ❌ **Don'ts:**

1. **Don't create god objects** - keep facades focused
2. **Don't bypass the facade** to access subsystems directly
3. **Don't make facades too thin** - provide real value
4. **Don't ignore error handling** from subsystems
5. **Don't create circular dependencies** between facade and subsystems
6. **Don't make facades overly complex** - defeats the purpose

## When to Use Facade Pattern

### ✅ **Good Use Cases:**
- **Complex libraries/frameworks**: Simplify API usage
- **Legacy system integration**: Modern interface to old systems
- **Microservices coordination**: Orchestrate multiple service calls
- **Third-party API integration**: Simplified wrapper for external APIs
- **Subsystem abstraction**: Hide implementation details
- **Workflow orchestration**: Multi-step processes with error handling

### ❌ **Avoid Facade For:**
- **Simple operations**: Direct method calls are sufficient
- **Single responsibility tasks**: No need for coordination
- **Performance-critical paths**: Avoid additional layers
- **Already simple interfaces**: Don't add unnecessary abstraction

## Related Patterns

- **Adapter**: Changes interface vs. simplifies interface
- **Mediator**: Centralizes complex communications vs. simplifies access
- **Proxy**: Controls access vs. simplifies access
- **Abstract Factory**: Creates objects vs. coordinates operations

## Summary

The Facade pattern is excellent for:

**Key Benefits:**
- **Simplification**: Easy-to-use interface for complex subsystems
- **Decoupling**: Reduce client dependencies on subsystem classes
- **Flexibility**: Change subsystem implementations without affecting clients
- **Error Handling**: Centralized error management and recovery

**Common Use Cases:**
- **Banking Systems**: Account operations, transaction processing
- **E-commerce**: Order processing, payment coordination
- **Computer Systems**: Startup/shutdown procedures
- **API Gateways**: Simplify microservices interactions

Use Facade when you need to provide a simple interface to a complex subsystem or coordinate multiple related operations!

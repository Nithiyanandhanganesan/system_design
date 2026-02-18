# Adapter Pattern - Complete Guide

## Introduction

The **Adapter Pattern** is a structural design pattern that allows incompatible interfaces to work together. It acts as a bridge between two incompatible interfaces, enabling classes with different interfaces to collaborate without modifying their source code.

## Definition

> **Adapter Pattern**: Allows incompatible interfaces to work together by wrapping an existing class with a new interface.

## Problem Solved

- **Interface Incompatibility**: Integrate classes with different interfaces
- **Legacy Code Integration**: Use old code with new systems
- **Third-Party Library Integration**: Adapt external libraries to your interface
- **Code Reusability**: Reuse existing functionality with different interfaces
- **API Consistency**: Provide consistent interface across different implementations

## Structure

```
┌─────────────────┐    ┌─────────────────┐
│     Client      │───▶│     Target      │
└─────────────────┘    ├─────────────────┤
                       │ +request()      │
                       └─────────────────┘
                                △
                                │
                    ┌─────────────────┐    ┌─────────────────┐
                    │     Adapter     │───▶│    Adaptee      │
                    ├─────────────────┤    ├─────────────────┤
                    │ +request()      │    │ +specificReq()  │
                    └─────────────────┘    └─────────────────┘
```

## Basic Implementation

### Media Player Example

```java
/**
 * Target interface that client expects
 */
public interface MediaPlayer {
    void play(String audioType, String fileName);
}

/**
 * Adaptee classes - existing incompatible interfaces
 */
public interface AdvancedMediaPlayer {
    void playVlc(String fileName);
    void playMp4(String fileName);
}

/**
 * Concrete Adaptee implementations
 */
public class VlcPlayer implements AdvancedMediaPlayer {
    
    @Override
    public void playVlc(String fileName) {
        System.out.println("Playing VLC file: " + fileName);
    }
    
    @Override
    public void playMp4(String fileName) {
        // VLC player doesn't support MP4 in this example
        System.out.println("VLC player: MP4 format not supported");
    }
}

public class Mp4Player implements AdvancedMediaPlayer {
    
    @Override
    public void playVlc(String fileName) {
        // MP4 player doesn't support VLC in this example
        System.out.println("MP4 player: VLC format not supported");
    }
    
    @Override
    public void playMp4(String fileName) {
        System.out.println("Playing MP4 file: " + fileName);
    }
}

/**
 * Adapter class that makes AdvancedMediaPlayer compatible with MediaPlayer
 */
public class MediaAdapter implements MediaPlayer {
    
    private final AdvancedMediaPlayer advancedPlayer;
    
    public MediaAdapter(String audioType) {
        if ("vlc".equalsIgnoreCase(audioType)) {
            advancedPlayer = new VlcPlayer();
        } else if ("mp4".equalsIgnoreCase(audioType)) {
            advancedPlayer = new Mp4Player();
        } else {
            throw new IllegalArgumentException("Unsupported audio type: " + audioType);
        }
    }
    
    @Override
    public void play(String audioType, String fileName) {
        if ("vlc".equalsIgnoreCase(audioType)) {
            advancedPlayer.playVlc(fileName);
        } else if ("mp4".equalsIgnoreCase(audioType)) {
            advancedPlayer.playMp4(fileName);
        } else {
            throw new IllegalArgumentException("Unsupported audio type: " + audioType);
        }
    }
}

/**
 * Client code using the adapter
 */
public class AudioPlayer implements MediaPlayer {
    
    private MediaAdapter adapter;
    
    @Override
    public void play(String audioType, String fileName) {
        
        // Built-in support for MP3
        if ("mp3".equalsIgnoreCase(audioType)) {
            System.out.println("Playing MP3 file: " + fileName);
        }
        // Use adapter for other formats
        else if ("vlc".equalsIgnoreCase(audioType) || "mp4".equalsIgnoreCase(audioType)) {
            adapter = new MediaAdapter(audioType);
            adapter.play(audioType, fileName);
        } 
        else {
            System.out.println("Invalid media format: " + audioType + " format not supported");
        }
    }
}
```

## Real-World Example: Payment Gateway Integration

```java
/**
 * Target interface - what our application expects
 */
public interface PaymentProcessor {
    PaymentResult processPayment(PaymentRequest request);
    PaymentStatus checkPaymentStatus(String transactionId);
    RefundResult refundPayment(String transactionId, double amount);
}

/**
 * Our standard payment request/response objects
 */
public class PaymentRequest {
    private final String orderId;
    private final double amount;
    private final String currency;
    private final String cardNumber;
    private final String cvv;
    private final String expiryDate;
    private final String customerEmail;
    
    public PaymentRequest(String orderId, double amount, String currency, 
                         String cardNumber, String cvv, String expiryDate, String customerEmail) {
        this.orderId = orderId;
        this.amount = amount;
        this.currency = currency;
        this.cardNumber = cardNumber;
        this.cvv = cvv;
        this.expiryDate = expiryDate;
        this.customerEmail = customerEmail;
    }
    
    // Getters
    public String getOrderId() { return orderId; }
    public double getAmount() { return amount; }
    public String getCurrency() { return currency; }
    public String getCardNumber() { return cardNumber; }
    public String getCvv() { return cvv; }
    public String getExpiryDate() { return expiryDate; }
    public String getCustomerEmail() { return customerEmail; }
}

public class PaymentResult {
    private final boolean success;
    private final String transactionId;
    private final String message;
    private final PaymentStatus status;
    
    public PaymentResult(boolean success, String transactionId, String message, PaymentStatus status) {
        this.success = success;
        this.transactionId = transactionId;
        this.message = message;
        this.status = status;
    }
    
    public boolean isSuccess() { return success; }
    public String getTransactionId() { return transactionId; }
    public String getMessage() { return message; }
    public PaymentStatus getStatus() { return status; }
}

public enum PaymentStatus {
    PENDING, COMPLETED, FAILED, REFUNDED, CANCELLED
}

public class RefundResult {
    private final boolean success;
    private final String refundId;
    private final String message;
    
    public RefundResult(boolean success, String refundId, String message) {
        this.success = success;
        this.refundId = refundId;
        this.message = message;
    }
    
    public boolean isSuccess() { return success; }
    public String getRefundId() { return refundId; }
    public String getMessage() { return message; }
}
```

### Legacy Payment Systems (Adaptees)

```java
/**
 * Legacy Stripe API (Adaptee 1)
 */
public class LegacyStripeAPI {
    
    public StripeTransactionResponse chargeCard(StripeCardDetails card, int amountInCents, String currency) {
        System.out.println("Processing Stripe payment: " + amountInCents + " cents in " + currency);
        
        // Simulate Stripe API call
        try {
            Thread.sleep(1000); // Simulate network delay
            
            if (amountInCents > 100000) { // Amounts over $1000 fail for demo
                return new StripeTransactionResponse(false, null, "Amount too high", "failed");
            }
            
            String txnId = "stripe_" + System.currentTimeMillis();
            return new StripeTransactionResponse(true, txnId, "Payment successful", "succeeded");
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return new StripeTransactionResponse(false, null, "Payment interrupted", "failed");
        }
    }
    
    public StripeTransactionResponse getTransactionStatus(String transactionId) {
        System.out.println("Checking Stripe transaction status: " + transactionId);
        // Simulate status check
        return new StripeTransactionResponse(true, transactionId, "Transaction found", "succeeded");
    }
    
    public StripeRefundResponse refund(String transactionId, int amountInCents) {
        System.out.println("Refunding Stripe transaction: " + transactionId + " for " + amountInCents + " cents");
        String refundId = "stripe_refund_" + System.currentTimeMillis();
        return new StripeRefundResponse(true, refundId, "Refund successful");
    }
}

/**
 * Legacy PayPal API (Adaptee 2)
 */
public class LegacyPayPalAPI {
    
    public PayPalPaymentResponse executePayment(PayPalPaymentInfo paymentInfo) {
        System.out.println("Processing PayPal payment: $" + paymentInfo.getAmountDollars());
        
        try {
            Thread.sleep(1500); // Simulate network delay
            
            if (paymentInfo.getAmountDollars() > 500) { // Amounts over $500 fail for demo
                return new PayPalPaymentResponse("FAILED", null, "Payment declined");
            }
            
            String paymentId = "paypal_" + System.currentTimeMillis();
            return new PayPalPaymentResponse("COMPLETED", paymentId, "Payment completed successfully");
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return new PayPalPaymentResponse("ERROR", null, "Payment processing error");
        }
    }
    
    public PayPalStatusResponse getPaymentDetails(String paymentId) {
        System.out.println("Checking PayPal payment details: " + paymentId);
        return new PayPalStatusResponse("COMPLETED", "Payment verified");
    }
    
    public PayPalRefundResponse issueRefund(String paymentId, double refundAmount) {
        System.out.println("Issuing PayPal refund: " + paymentId + " for $" + refundAmount);
        String refundTxnId = "paypal_refund_" + System.currentTimeMillis();
        return new PayPalRefundResponse("SUCCESS", refundTxnId, "Refund issued");
    }
}

// Legacy API response classes
public class StripeCardDetails {
    private final String number;
    private final String cvc;
    private final String expMonth;
    private final String expYear;
    private final String email;
    
    public StripeCardDetails(String number, String cvc, String expMonth, String expYear, String email) {
        this.number = number;
        this.cvc = cvc;
        this.expMonth = expMonth;
        this.expYear = expYear;
        this.email = email;
    }
    
    // Getters
    public String getNumber() { return number; }
    public String getCvc() { return cvc; }
    public String getExpMonth() { return expMonth; }
    public String getExpYear() { return expYear; }
    public String getEmail() { return email; }
}

public class StripeTransactionResponse {
    private final boolean success;
    private final String id;
    private final String message;
    private final String status;
    
    public StripeTransactionResponse(boolean success, String id, String message, String status) {
        this.success = success;
        this.id = id;
        this.message = message;
        this.status = status;
    }
    
    public boolean isSuccess() { return success; }
    public String getId() { return id; }
    public String getMessage() { return message; }
    public String getStatus() { return status; }
}

public class StripeRefundResponse {
    private final boolean success;
    private final String refundId;
    private final String message;
    
    public StripeRefundResponse(boolean success, String refundId, String message) {
        this.success = success;
        this.refundId = refundId;
        this.message = message;
    }
    
    public boolean isSuccess() { return success; }
    public String getRefundId() { return refundId; }
    public String getMessage() { return message; }
}

// PayPal classes
public class PayPalPaymentInfo {
    private final double amountDollars;
    private final String currency;
    private final String orderId;
    private final String cardNumber;
    private final String securityCode;
    private final String expirationDate;
    
    public PayPalPaymentInfo(double amountDollars, String currency, String orderId, 
                           String cardNumber, String securityCode, String expirationDate) {
        this.amountDollars = amountDollars;
        this.currency = currency;
        this.orderId = orderId;
        this.cardNumber = cardNumber;
        this.securityCode = securityCode;
        this.expirationDate = expirationDate;
    }
    
    public double getAmountDollars() { return amountDollars; }
    public String getCurrency() { return currency; }
    public String getOrderId() { return orderId; }
    public String getCardNumber() { return cardNumber; }
    public String getSecurityCode() { return securityCode; }
    public String getExpirationDate() { return expirationDate; }
}

public class PayPalPaymentResponse {
    private final String status;
    private final String paymentId;
    private final String message;
    
    public PayPalPaymentResponse(String status, String paymentId, String message) {
        this.status = status;
        this.paymentId = paymentId;
        this.message = message;
    }
    
    public String getStatus() { return status; }
    public String getPaymentId() { return paymentId; }
    public String getMessage() { return message; }
}

public class PayPalStatusResponse {
    private final String status;
    private final String message;
    
    public PayPalStatusResponse(String status, String message) {
        this.status = status;
        this.message = message;
    }
    
    public String getStatus() { return status; }
    public String getMessage() { return message; }
}

public class PayPalRefundResponse {
    private final String status;
    private final String refundId;
    private final String message;
    
    public PayPalRefundResponse(String status, String refundId, String message) {
        this.status = status;
        this.refundId = refundId;
        this.message = message;
    }
    
    public String getStatus() { return status; }
    public String getRefundId() { return refundId; }
    public String getMessage() { return message; }
}
```

### Payment Adapters

```java
/**
 * Stripe Payment Adapter
 */
public class StripePaymentAdapter implements PaymentProcessor {
    
    private final LegacyStripeAPI stripeAPI;
    
    public StripePaymentAdapter(LegacyStripeAPI stripeAPI) {
        this.stripeAPI = stripeAPI;
    }
    
    @Override
    public PaymentResult processPayment(PaymentRequest request) {
        try {
            // Convert our PaymentRequest to Stripe's format
            StripeCardDetails cardDetails = new StripeCardDetails(
                request.getCardNumber(),
                request.getCvv(),
                extractMonth(request.getExpiryDate()),
                extractYear(request.getExpiryDate()),
                request.getCustomerEmail()
            );
            
            // Convert amount to cents (Stripe uses cents)
            int amountInCents = (int) (request.getAmount() * 100);
            
            // Call Stripe API
            StripeTransactionResponse response = stripeAPI.chargeCard(
                cardDetails, amountInCents, request.getCurrency());
            
            // Convert Stripe response to our format
            PaymentStatus status = convertStripeStatus(response.getStatus());
            
            return new PaymentResult(
                response.isSuccess(),
                response.getId(),
                response.getMessage(),
                status
            );
            
        } catch (Exception e) {
            return new PaymentResult(false, null, "Stripe payment failed: " + e.getMessage(), PaymentStatus.FAILED);
        }
    }
    
    @Override
    public PaymentStatus checkPaymentStatus(String transactionId) {
        try {
            StripeTransactionResponse response = stripeAPI.getTransactionStatus(transactionId);
            return convertStripeStatus(response.getStatus());
        } catch (Exception e) {
            return PaymentStatus.FAILED;
        }
    }
    
    @Override
    public RefundResult refundPayment(String transactionId, double amount) {
        try {
            int amountInCents = (int) (amount * 100);
            StripeRefundResponse response = stripeAPI.refund(transactionId, amountInCents);
            
            return new RefundResult(
                response.isSuccess(),
                response.getRefundId(),
                response.getMessage()
            );
        } catch (Exception e) {
            return new RefundResult(false, null, "Stripe refund failed: " + e.getMessage());
        }
    }
    
    private String extractMonth(String expiryDate) {
        // Assuming format "MM/YY"
        return expiryDate.substring(0, 2);
    }
    
    private String extractYear(String expiryDate) {
        // Assuming format "MM/YY"
        return "20" + expiryDate.substring(3, 5);
    }
    
    private PaymentStatus convertStripeStatus(String stripeStatus) {
        switch (stripeStatus.toLowerCase()) {
            case "succeeded":
                return PaymentStatus.COMPLETED;
            case "pending":
                return PaymentStatus.PENDING;
            case "failed":
                return PaymentStatus.FAILED;
            case "canceled":
                return PaymentStatus.CANCELLED;
            default:
                return PaymentStatus.FAILED;
        }
    }
}

/**
 * PayPal Payment Adapter
 */
public class PayPalPaymentAdapter implements PaymentProcessor {
    
    private final LegacyPayPalAPI paypalAPI;
    
    public PayPalPaymentAdapter(LegacyPayPalAPI paypalAPI) {
        this.paypalAPI = paypalAPI;
    }
    
    @Override
    public PaymentResult processPayment(PaymentRequest request) {
        try {
            // Convert our PaymentRequest to PayPal's format
            PayPalPaymentInfo paymentInfo = new PayPalPaymentInfo(
                request.getAmount(),
                request.getCurrency(),
                request.getOrderId(),
                request.getCardNumber(),
                request.getCvv(),
                request.getExpiryDate()
            );
            
            // Call PayPal API
            PayPalPaymentResponse response = paypalAPI.executePayment(paymentInfo);
            
            // Convert PayPal response to our format
            PaymentStatus status = convertPayPalStatus(response.getStatus());
            boolean success = "COMPLETED".equals(response.getStatus());
            
            return new PaymentResult(
                success,
                response.getPaymentId(),
                response.getMessage(),
                status
            );
            
        } catch (Exception e) {
            return new PaymentResult(false, null, "PayPal payment failed: " + e.getMessage(), PaymentStatus.FAILED);
        }
    }
    
    @Override
    public PaymentStatus checkPaymentStatus(String transactionId) {
        try {
            PayPalStatusResponse response = paypalAPI.getPaymentDetails(transactionId);
            return convertPayPalStatus(response.getStatus());
        } catch (Exception e) {
            return PaymentStatus.FAILED;
        }
    }
    
    @Override
    public RefundResult refundPayment(String transactionId, double amount) {
        try {
            PayPalRefundResponse response = paypalAPI.issueRefund(transactionId, amount);
            
            boolean success = "SUCCESS".equals(response.getStatus());
            return new RefundResult(
                success,
                response.getRefundId(),
                response.getMessage()
            );
        } catch (Exception e) {
            return new RefundResult(false, null, "PayPal refund failed: " + e.getMessage());
        }
    }
    
    private PaymentStatus convertPayPalStatus(String paypalStatus) {
        switch (paypalStatus.toUpperCase()) {
            case "COMPLETED":
                return PaymentStatus.COMPLETED;
            case "PENDING":
                return PaymentStatus.PENDING;
            case "FAILED":
                return PaymentStatus.FAILED;
            case "CANCELLED":
                return PaymentStatus.CANCELLED;
            default:
                return PaymentStatus.FAILED;
        }
    }
}
```

## Spring Boot Integration

```java
/**
 * Payment service factory using adapters
 */
@Component
public class PaymentServiceFactory {
    
    public PaymentProcessor createPaymentProcessor(String provider) {
        switch (provider.toLowerCase()) {
            case "stripe":
                return new StripePaymentAdapter(new LegacyStripeAPI());
            case "paypal":
                return new PayPalPaymentAdapter(new LegacyPayPalAPI());
            default:
                throw new IllegalArgumentException("Unsupported payment provider: " + provider);
        }
    }
}

/**
 * Service that uses different payment processors through adapters
 */
@Service
public class PaymentService {
    
    private final PaymentServiceFactory paymentFactory;
    private final Map<String, PaymentProcessor> processorCache = new ConcurrentHashMap<>();
    
    public PaymentService(PaymentServiceFactory paymentFactory) {
        this.paymentFactory = paymentFactory;
    }
    
    public PaymentResult processPayment(String provider, PaymentRequest request) {
        PaymentProcessor processor = getOrCreateProcessor(provider);
        
        System.out.println("Processing payment with " + provider + " for amount: $" + request.getAmount());
        
        PaymentResult result = processor.processPayment(request);
        
        if (result.isSuccess()) {
            System.out.println("Payment successful: " + result.getTransactionId());
        } else {
            System.out.println("Payment failed: " + result.getMessage());
        }
        
        return result;
    }
    
    public PaymentStatus checkPaymentStatus(String provider, String transactionId) {
        PaymentProcessor processor = getOrCreateProcessor(provider);
        return processor.checkPaymentStatus(transactionId);
    }
    
    public RefundResult refundPayment(String provider, String transactionId, double amount) {
        PaymentProcessor processor = getOrCreateProcessor(provider);
        
        System.out.println("Processing refund with " + provider + " for transaction: " + transactionId);
        
        RefundResult result = processor.refundPayment(transactionId, amount);
        
        if (result.isSuccess()) {
            System.out.println("Refund successful: " + result.getRefundId());
        } else {
            System.out.println("Refund failed: " + result.getMessage());
        }
        
        return result;
    }
    
    private PaymentProcessor getOrCreateProcessor(String provider) {
        return processorCache.computeIfAbsent(provider, paymentFactory::createPaymentProcessor);
    }
    
    public List<String> getSupportedProviders() {
        return Arrays.asList("stripe", "paypal");
    }
}

/**
 * REST Controller using payment adapters
 */
@RestController
@RequestMapping("/api/payments")
public class PaymentController {
    
    private final PaymentService paymentService;
    
    public PaymentController(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
    
    @PostMapping("/process/{provider}")
    public ResponseEntity<PaymentResult> processPayment(
            @PathVariable String provider,
            @RequestBody @Valid PaymentRequestDTO requestDTO) {
        
        try {
            PaymentRequest request = convertToPaymentRequest(requestDTO);
            PaymentResult result = paymentService.processPayment(provider, request);
            
            if (result.isSuccess()) {
                return ResponseEntity.ok(result);
            } else {
                return ResponseEntity.badRequest().body(result);
            }
            
        } catch (IllegalArgumentException e) {
            PaymentResult errorResult = new PaymentResult(false, null, e.getMessage(), PaymentStatus.FAILED);
            return ResponseEntity.badRequest().body(errorResult);
        }
    }
    
    @GetMapping("/status/{provider}/{transactionId}")
    public ResponseEntity<PaymentStatus> getPaymentStatus(
            @PathVariable String provider,
            @PathVariable String transactionId) {
        
        try {
            PaymentStatus status = paymentService.checkPaymentStatus(provider, transactionId);
            return ResponseEntity.ok(status);
        } catch (IllegalArgumentException e) {
            return ResponseEntity.badRequest().build();
        }
    }
    
    @PostMapping("/refund/{provider}/{transactionId}")
    public ResponseEntity<RefundResult> refundPayment(
            @PathVariable String provider,
            @PathVariable String transactionId,
            @RequestParam double amount) {
        
        try {
            RefundResult result = paymentService.refundPayment(provider, transactionId, amount);
            
            if (result.isSuccess()) {
                return ResponseEntity.ok(result);
            } else {
                return ResponseEntity.badRequest().body(result);
            }
            
        } catch (IllegalArgumentException e) {
            RefundResult errorResult = new RefundResult(false, null, e.getMessage());
            return ResponseEntity.badRequest().body(errorResult);
        }
    }
    
    @GetMapping("/providers")
    public ResponseEntity<List<String>> getSupportedProviders() {
        return ResponseEntity.ok(paymentService.getSupportedProviders());
    }
    
    private PaymentRequest convertToPaymentRequest(PaymentRequestDTO dto) {
        return new PaymentRequest(
            dto.getOrderId(),
            dto.getAmount(),
            dto.getCurrency(),
            dto.getCardNumber(),
            dto.getCvv(),
            dto.getExpiryDate(),
            dto.getCustomerEmail()
        );
    }
}

// DTO for REST API
@Data
@NoArgsConstructor
@AllArgsConstructor
public class PaymentRequestDTO {
    
    @NotBlank(message = "Order ID is required")
    private String orderId;
    
    @DecimalMin(value = "0.01", message = "Amount must be greater than 0")
    private double amount;
    
    @NotBlank(message = "Currency is required")
    private String currency;
    
    @NotBlank(message = "Card number is required")
    @Pattern(regexp = "\\d{16}", message = "Card number must be 16 digits")
    private String cardNumber;
    
    @NotBlank(message = "CVV is required")
    @Pattern(regexp = "\\d{3,4}", message = "CVV must be 3 or 4 digits")
    private String cvv;
    
    @NotBlank(message = "Expiry date is required")
    @Pattern(regexp = "\\d{2}/\\d{2}", message = "Expiry date must be in MM/YY format")
    private String expiryDate;
    
    @Email(message = "Valid email is required")
    private String customerEmail;
}
```

## Database Adapter Example

```java
/**
 * Target interface for our application
 */
public interface DataRepository {
    void save(String id, String data);
    String findById(String id);
    List<String> findAll();
    boolean delete(String id);
}

/**
 * Legacy file-based storage system (Adaptee)
 */
public class LegacyFileStorage {
    
    private final String baseDirectory;
    
    public LegacyFileStorage(String baseDirectory) {
        this.baseDirectory = baseDirectory;
        createDirectoryIfNotExists();
    }
    
    public boolean writeToFile(String filename, String content) {
        try {
            Files.write(Paths.get(baseDirectory, filename), content.getBytes());
            System.out.println("Written to file: " + filename);
            return true;
        } catch (IOException e) {
            System.err.println("Error writing file: " + e.getMessage());
            return false;
        }
    }
    
    public String readFromFile(String filename) {
        try {
            return Files.readString(Paths.get(baseDirectory, filename));
        } catch (IOException e) {
            System.err.println("Error reading file: " + e.getMessage());
            return null;
        }
    }
    
    public String[] listFiles() {
        try {
            return Files.list(Paths.get(baseDirectory))
                .filter(Files::isRegularFile)
                .map(path -> path.getFileName().toString())
                .toArray(String[]::new);
        } catch (IOException e) {
            System.err.println("Error listing files: " + e.getMessage());
            return new String[0];
        }
    }
    
    public boolean removeFile(String filename) {
        try {
            Files.delete(Paths.get(baseDirectory, filename));
            System.out.println("Deleted file: " + filename);
            return true;
        } catch (IOException e) {
            System.err.println("Error deleting file: " + e.getMessage());
            return false;
        }
    }
    
    private void createDirectoryIfNotExists() {
        try {
            Files.createDirectories(Paths.get(baseDirectory));
        } catch (IOException e) {
            throw new RuntimeException("Failed to create directory: " + baseDirectory, e);
        }
    }
}

/**
 * Adapter that makes LegacyFileStorage compatible with DataRepository
 */
public class FileStorageAdapter implements DataRepository {
    
    private final LegacyFileStorage fileStorage;
    
    public FileStorageAdapter(LegacyFileStorage fileStorage) {
        this.fileStorage = fileStorage;
    }
    
    @Override
    public void save(String id, String data) {
        String filename = id + ".txt";
        boolean success = fileStorage.writeToFile(filename, data);
        if (!success) {
            throw new RuntimeException("Failed to save data with ID: " + id);
        }
    }
    
    @Override
    public String findById(String id) {
        String filename = id + ".txt";
        return fileStorage.readFromFile(filename);
    }
    
    @Override
    public List<String> findAll() {
        String[] filenames = fileStorage.listFiles();
        return Arrays.stream(filenames)
            .map(filename -> filename.replace(".txt", ""))
            .collect(Collectors.toList());
    }
    
    @Override
    public boolean delete(String id) {
        String filename = id + ".txt";
        return fileStorage.removeFile(filename);
    }
}

/**
 * Modern database implementation
 */
@Repository
public class DatabaseRepository implements DataRepository {
    
    private final Map<String, String> database = new ConcurrentHashMap<>();
    
    @Override
    public void save(String id, String data) {
        database.put(id, data);
        System.out.println("Saved to database: " + id);
    }
    
    @Override
    public String findById(String id) {
        return database.get(id);
    }
    
    @Override
    public List<String> findAll() {
        return new ArrayList<>(database.keySet());
    }
    
    @Override
    public boolean delete(String id) {
        boolean existed = database.remove(id) != null;
        if (existed) {
            System.out.println("Deleted from database: " + id);
        }
        return existed;
    }
}
```

## Testing Adapter Pattern

```java
@ExtendWith(MockitoExtension.class)
class AdapterPatternTest {
    
    @Test
    void testMediaAdapter() {
        AudioPlayer player = new AudioPlayer();
        
        // Test MP3 (built-in support)
        player.play("mp3", "song.mp3");
        
        // Test VLC (through adapter)
        player.play("vlc", "movie.vlc");
        
        // Test MP4 (through adapter)
        player.play("mp4", "video.mp4");
        
        // Test unsupported format
        player.play("avi", "file.avi");
    }
    
    @Test
    void testStripePaymentAdapter() {
        LegacyStripeAPI mockStripeAPI = mock(LegacyStripeAPI.class);
        StripePaymentAdapter adapter = new StripePaymentAdapter(mockStripeAPI);
        
        PaymentRequest request = new PaymentRequest(
            "ORDER123", 100.0, "USD", "1234567890123456", "123", "12/25", "test@example.com"
        );
        
        StripeTransactionResponse mockResponse = new StripeTransactionResponse(
            true, "stripe_123", "Payment successful", "succeeded"
        );
        
        when(mockStripeAPI.chargeCard(any(StripeCardDetails.class), eq(10000), eq("USD")))
            .thenReturn(mockResponse);
        
        PaymentResult result = adapter.processPayment(request);
        
        assertThat(result.isSuccess()).isTrue();
        assertThat(result.getTransactionId()).isEqualTo("stripe_123");
        assertThat(result.getStatus()).isEqualTo(PaymentStatus.COMPLETED);
        
        verify(mockStripeAPI).chargeCard(any(StripeCardDetails.class), eq(10000), eq("USD"));
    }
    
    @Test
    void testPayPalPaymentAdapter() {
        LegacyPayPalAPI mockPayPalAPI = mock(LegacyPayPalAPI.class);
        PayPalPaymentAdapter adapter = new PayPalPaymentAdapter(mockPayPalAPI);
        
        PaymentRequest request = new PaymentRequest(
            "ORDER456", 50.0, "USD", "1234567890123456", "123", "12/25", "test@example.com"
        );
        
        PayPalPaymentResponse mockResponse = new PayPalPaymentResponse(
            "COMPLETED", "paypal_456", "Payment completed successfully"
        );
        
        when(mockPayPalAPI.executePayment(any(PayPalPaymentInfo.class)))
            .thenReturn(mockResponse);
        
        PaymentResult result = adapter.processPayment(request);
        
        assertThat(result.isSuccess()).isTrue();
        assertThat(result.getTransactionId()).isEqualTo("paypal_456");
        assertThat(result.getStatus()).isEqualTo(PaymentStatus.COMPLETED);
        
        verify(mockPayPalAPI).executePayment(any(PayPalPaymentInfo.class));
    }
    
    @Test
    void testFileStorageAdapter() throws IOException {
        // Create temporary directory for testing
        Path tempDir = Files.createTempDirectory("test_storage");
        LegacyFileStorage fileStorage = new LegacyFileStorage(tempDir.toString());
        FileStorageAdapter adapter = new FileStorageAdapter(fileStorage);
        
        try {
            // Test save
            adapter.save("test1", "Hello World");
            
            // Test findById
            String data = adapter.findById("test1");
            assertThat(data).isEqualTo("Hello World");
            
            // Test findAll
            adapter.save("test2", "Another test");
            List<String> allIds = adapter.findAll();
            assertThat(allIds).containsExactlyInAnyOrder("test1", "test2");
            
            // Test delete
            boolean deleted = adapter.delete("test1");
            assertThat(deleted).isTrue();
            
            allIds = adapter.findAll();
            assertThat(allIds).containsExactly("test2");
            
        } finally {
            // Clean up
            Files.walk(tempDir)
                .sorted(Comparator.reverseOrder())
                .map(Path::toFile)
                .forEach(File::delete);
        }
    }
    
    @Test
    void testPaymentServiceWithMultipleProviders() {
        PaymentServiceFactory factory = new PaymentServiceFactory();
        PaymentService paymentService = new PaymentService(factory);
        
        PaymentRequest request = new PaymentRequest(
            "ORDER789", 75.0, "USD", "1234567890123456", "123", "12/25", "test@example.com"
        );
        
        // Test with different providers
        PaymentResult stripeResult = paymentService.processPayment("stripe", request);
        PaymentResult paypalResult = paymentService.processPayment("paypal", request);
        
        assertThat(stripeResult.getTransactionId()).startsWith("stripe_");
        assertThat(paypalResult.getTransactionId()).startsWith("paypal_");
        
        // Test supported providers
        List<String> providers = paymentService.getSupportedProviders();
        assertThat(providers).containsExactlyInAnyOrder("stripe", "paypal");
    }
}
```

## Best Practices

### ✅ **Do's:**

1. **Keep adapters focused** - one adapter per incompatible interface
2. **Handle data conversion** properly between different formats
3. **Add error handling** for conversion failures
4. **Document the adaptation** - what's being converted and why
5. **Use composition over inheritance** when possible
6. **Cache adapter instances** if they're expensive to create
7. **Validate inputs** before passing to adaptee

### ❌ **Don'ts:**

1. **Don't add business logic** to adapters - keep them focused on adaptation
2. **Don't create deep adapter chains** - can become confusing
3. **Don't ignore error handling** from adaptee classes
4. **Don't modify the adaptee** if you control it - use direct integration
5. **Don't create adapters for simple type conversions**

## When to Use Adapter Pattern

### ✅ **Good Use Cases:**
- **Legacy system integration**: Make old systems work with new interfaces
- **Third-party library integration**: Adapt external APIs to your interface
- **Interface standardization**: Provide consistent interface across different implementations
- **API versioning**: Support multiple API versions
- **Cross-platform development**: Adapt platform-specific APIs

### ❌ **Avoid Adapter For:**
- **Simple type conversion**: Use utility methods instead
- **Classes you control**: Modify the interface directly
- **One-time use**: Direct conversion might be simpler

## Summary

The Adapter pattern is excellent for:

**Key Benefits:**
- **Integration**: Connect incompatible interfaces seamlessly
- **Reusability**: Use existing code without modification
- **Consistency**: Provide uniform interface across different implementations
- **Legacy Support**: Keep old systems working with new code

**Common Use Cases:**
- **Payment Gateway Integration**: Different payment providers
- **Database Adapters**: Various database systems
- **File Format Converters**: Different file formats
- **API Integration**: Third-party services with different interfaces

Use Adapter when you need to make incompatible interfaces work together without modifying the existing code!

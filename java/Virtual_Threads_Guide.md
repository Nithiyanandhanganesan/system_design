# Virtual Threads in Java - Complete Guide

## Introduction

**Virtual threads** are a groundbreaking feature introduced in Java 19 (preview) and became stable in **Java 21** as part of Project Loom. They represent a paradigm shift in Java concurrency, offering lightweight threads that can dramatically improve application scalability and throughput.

### What are Virtual Threads?

Virtual threads are **lightweight threads** managed by the JVM rather than the operating system. Unlike traditional platform threads (OS threads), virtual threads have minimal memory overhead and can be created in massive numbers without significant performance degradation.

## Traditional Threads vs Virtual Threads

### Platform Threads (Traditional)
```java
// Traditional thread creation
Thread platformThread = new Thread(() -> {
    System.out.println("Running on platform thread: " + Thread.currentThread());
});
platformThread.start();

// Platform thread characteristics:
// - Mapped 1:1 to OS threads
// - ~2MB stack size per thread
// - Limited by OS thread limits (typically 1000-5000)
// - Expensive context switching
```

### Virtual Threads
```java
// Virtual thread creation
Thread virtualThread = Thread.startVirtualThread(() -> {
    System.out.println("Running on virtual thread: " + Thread.currentThread());
});

// Virtual thread characteristics:
// - Managed by JVM
// - ~Few KB memory overhead
// - Can create millions of threads
// - Cheap to create and destroy
```

## Creating Virtual Threads

### 1. **Using Thread.startVirtualThread()**
```java
public class VirtualThreadBasics {
    
    public static void main(String[] args) throws InterruptedException {
        // Simple virtual thread
        Thread vt1 = Thread.startVirtualThread(() -> {
            System.out.println("Hello from virtual thread: " + Thread.currentThread().getName());
        });
        
        // Wait for completion
        vt1.join();
        
        // Multiple virtual threads
        List<Thread> virtualThreads = new ArrayList<>();
        
        for (int i = 0; i < 10; i++) {
            final int threadId = i;
            Thread vt = Thread.startVirtualThread(() -> {
                try {
                    Thread.sleep(1000); // Simulate work
                    System.out.println("Virtual thread " + threadId + " completed");
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
            virtualThreads.add(vt);
        }
        
        // Wait for all to complete
        for (Thread vt : virtualThreads) {
            vt.join();
        }
    }
}
```

### 2. **Using Thread.Builder**
```java
public class VirtualThreadBuilder {
    
    public static void main(String[] args) throws InterruptedException {
        // Create virtual thread with custom name
        Thread.Builder virtualBuilder = Thread.ofVirtual().name("MyVirtualThread");
        
        Thread vt1 = virtualBuilder.start(() -> {
            System.out.println("Running on: " + Thread.currentThread().getName());
        });
        
        // Create virtual thread with name pattern
        Thread.Builder namedBuilder = Thread.ofVirtual().name("worker-", 0);
        
        for (int i = 0; i < 5; i++) {
            Thread vt = namedBuilder.start(() -> {
                System.out.println("Worker thread: " + Thread.currentThread().getName());
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }
        
        Thread.sleep(2000); // Wait for completion
    }
}
```

### 3. **Using ExecutorService with Virtual Threads**
```java
public class VirtualThreadExecutor {
    
    public static void main(String[] args) throws InterruptedException, ExecutionException {
        // Create virtual thread executor
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            
            // Submit multiple tasks
            List<Future<String>> futures = new ArrayList<>();
            
            for (int i = 0; i < 100; i++) {
                final int taskId = i;
                Future<String> future = executor.submit(() -> {
                    try {
                        Thread.sleep(1000); // Simulate I/O operation
                        return "Task " + taskId + " completed on " + 
                               Thread.currentThread().getName();
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        return "Task " + taskId + " interrupted";
                    }
                });
                futures.add(future);
            }
            
            // Collect results
            for (Future<String> future : futures) {
                System.out.println(future.get());
            }
        }
    }
}
```

## Virtual Thread Lifecycle and Management

### Understanding Virtual Thread States
```java
public class VirtualThreadLifecycle {
    
    public static void demonstrateLifecycle() {
        Thread virtualThread = Thread.ofVirtual().start(() -> {
            System.out.println("Virtual thread started: " + Thread.currentThread().getName());
            
            try {
                // Simulate different states
                System.out.println("State: RUNNABLE");
                
                // Blocking I/O - thread becomes parked
                Thread.sleep(1000);
                System.out.println("State: After BLOCKING/WAITING");
                
                // CPU intensive work
                long sum = 0;
                for (int i = 0; i < 1000000; i++) {
                    sum += i;
                }
                System.out.println("CPU work completed, sum: " + sum);
                
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                System.out.println("Thread interrupted");
            }
            
            System.out.println("Virtual thread ending");
        });
        
        // Monitor thread state
        monitorThreadState(virtualThread);
    }
    
    private static void monitorThreadState(Thread thread) {
        Thread monitor = Thread.startVirtualThread(() -> {
            while (thread.isAlive()) {
                System.out.println("Thread " + thread.getName() + 
                                 " state: " + thread.getState());
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        });
        
        try {
            thread.join();
            monitor.interrupt();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

## Virtual Threads in Web Applications

### 1. **Spring Boot with Virtual Threads**
```java
@SpringBootApplication
public class VirtualThreadWebApp {
    
    public static void main(String[] args) {
        // Enable virtual threads in Spring Boot
        System.setProperty("spring.threads.virtual.enabled", "true");
        SpringApplication.run(VirtualThreadWebApp.class, args);
    }
    
    @Bean
    @ConditionalOnProperty(value = "spring.threads.virtual.enabled", havingValue = "true")
    public TomcatProtocolHandlerCustomizer<?> protocolHandlerVirtualThreadExecutorCustomizer() {
        return protocolHandler -> {
            protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
        };
    }
}

@RestController
@RequestMapping("/api")
public class VirtualThreadController {
    
    @Autowired
    private ExternalService externalService;
    
    @GetMapping("/blocking-operation")
    public ResponseEntity<String> blockingOperation() throws InterruptedException {
        // This will now run on a virtual thread
        String threadName = Thread.currentThread().getName();
        
        // Simulate blocking I/O
        Thread.sleep(2000);
        
        // Call external service
        String result = externalService.callExternalApi();
        
        return ResponseEntity.ok(
            "Operation completed on thread: " + threadName + 
            ", Result: " + result
        );
    }
    
    @GetMapping("/concurrent-operations")
    public ResponseEntity<List<String>> concurrentOperations() {
        List<CompletableFuture<String>> futures = new ArrayList<>();
        
        // Create 100 concurrent operations
        for (int i = 0; i < 100; i++) {
            final int operationId = i;
            CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
                try {
                    Thread.sleep(1000); // Simulate I/O
                    return "Operation " + operationId + " on " + 
                           Thread.currentThread().getName();
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return "Operation " + operationId + " interrupted";
                }
            });
            futures.add(future);
        }
        
        // Collect all results
        List<String> results = futures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList());
        
        return ResponseEntity.ok(results);
    }
}

@Service
public class ExternalService {
    
    private final RestTemplate restTemplate = new RestTemplate();
    
    public String callExternalApi() {
        try {
            // Simulate external API call
            Thread.sleep(500);
            return "External API response from " + Thread.currentThread().getName();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return "API call interrupted";
        }
    }
}
```

### 2. **HTTP Client with Virtual Threads**
```java
public class VirtualThreadHttpClient {
    
    public static void main(String[] args) throws InterruptedException {
        demonstrateHttpCalls();
    }
    
    public static void demonstrateHttpCalls() throws InterruptedException {
        HttpClient client = HttpClient.newHttpClient();
        List<String> urls = List.of(
            "https://jsonplaceholder.typicode.com/posts/1",
            "https://jsonplaceholder.typicode.com/posts/2",
            "https://jsonplaceholder.typicode.com/posts/3",
            "https://jsonplaceholder.typicode.com/posts/4",
            "https://jsonplaceholder.typicode.com/posts/5"
        );
        
        // Using virtual threads for concurrent HTTP requests
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            
            List<Future<HttpResponse<String>>> futures = urls.stream()
                .map(url -> executor.submit(() -> makeHttpRequest(client, url)))
                .collect(Collectors.toList());
            
            // Process results
            futures.forEach(future -> {
                try {
                    HttpResponse<String> response = future.get();
                    System.out.println("Response from " + response.uri() + 
                                     " on thread " + Thread.currentThread().getName() +
                                     " - Status: " + response.statusCode());
                } catch (Exception e) {
                    System.err.println("Error processing response: " + e.getMessage());
                }
            });
        }
    }
    
    private static HttpResponse<String> makeHttpRequest(HttpClient client, String url) {
        try {
            HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(url))
                .timeout(Duration.ofSeconds(10))
                .build();
                
            return client.send(request, HttpResponse.BodyHandlers.ofString());
        } catch (Exception e) {
            throw new RuntimeException("HTTP request failed for " + url, e);
        }
    }
}
```

## Database Operations with Virtual Threads

### 1. **JDBC with Virtual Threads**
```java
@Repository
public class VirtualThreadUserRepository {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    public List<User> findAllUsersAsync() throws InterruptedException, ExecutionException {
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            
            // Split query into chunks for parallel processing
            List<Future<List<User>>> futures = new ArrayList<>();
            
            // Get total count
            Integer totalUsers = jdbcTemplate.queryForObject(
                "SELECT COUNT(*) FROM users", Integer.class);
            
            int batchSize = 1000;
            int totalBatches = (totalUsers + batchSize - 1) / batchSize;
            
            for (int i = 0; i < totalBatches; i++) {
                final int offset = i * batchSize;
                Future<List<User>> future = executor.submit(() -> {
                    return jdbcTemplate.query(
                        "SELECT * FROM users LIMIT ? OFFSET ?",
                        new Object[]{batchSize, offset},
                        (rs, rowNum) -> new User(
                            rs.getLong("id"),
                            rs.getString("name"),
                            rs.getString("email")
                        )
                    );
                });
                futures.add(future);
            }
            
            // Combine results
            List<User> allUsers = new ArrayList<>();
            for (Future<List<User>> future : futures) {
                allUsers.addAll(future.get());
            }
            
            return allUsers;
        }
    }
    
    public CompletableFuture<Void> bulkInsertUsers(List<User> users) {
        return CompletableFuture.runAsync(() -> {
            String sql = "INSERT INTO users (name, email) VALUES (?, ?)";
            
            List<Object[]> batchArgs = users.stream()
                .map(user -> new Object[]{user.getName(), user.getEmail()})
                .collect(Collectors.toList());
                
            jdbcTemplate.batchUpdate(sql, batchArgs);
            
            System.out.println("Bulk insert completed on " + 
                             Thread.currentThread().getName());
        });
    }
}

@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private String email;
    
    // Constructors, getters, setters
    public User() {}
    
    public User(Long id, String name, String email) {
        this.id = id;
        this.name = name;
        this.email = email;
    }
    
    // ...existing code...
}
```

### 2. **Connection Pool Configuration**
```java
@Configuration
public class DatabaseConfig {
    
    @Bean
    @ConfigurationProperties("app.datasource")
    public HikariConfig hikariConfig() {
        HikariConfig config = new HikariConfig();
        
        // Increase pool size for virtual threads
        // Virtual threads can handle more concurrent connections
        config.setMaximumPoolSize(200); // Increased from typical 10-20
        config.setMinimumIdle(50);
        config.setConnectionTimeout(30000);
        config.setIdleTimeout(600000);
        config.setMaxLifetime(1800000);
        
        // Optimize for virtual threads
        config.setLeakDetectionThreshold(60000);
        
        return config;
    }
    
    @Bean
    public DataSource dataSource() {
        return new HikariDataSource(hikariConfig());
    }
}
```

## File I/O Operations with Virtual Threads

### Parallel File Processing
```java
public class VirtualThreadFileProcessor {
    
    public static void processFilesInParallel(List<Path> filePaths) {
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            
            List<CompletableFuture<FileProcessResult>> futures = filePaths.stream()
                .map(path -> CompletableFuture.supplyAsync(() -> processFile(path), executor))
                .collect(Collectors.toList());
            
            // Wait for all files to be processed
            CompletableFuture<Void> allDone = CompletableFuture.allOf(
                futures.toArray(new CompletableFuture[0]));
            
            allDone.thenRun(() -> {
                System.out.println("All files processed!");
                
                // Collect results
                List<FileProcessResult> results = futures.stream()
                    .map(CompletableFuture::join)
                    .collect(Collectors.toList());
                
                printProcessingStats(results);
            }).join();
        }
    }
    
    private static FileProcessResult processFile(Path filePath) {
        try {
            String threadName = Thread.currentThread().getName();
            long startTime = System.currentTimeMillis();
            
            // Read file content
            List<String> lines = Files.readAllLines(filePath);
            
            // Process content (example: count words)
            long wordCount = lines.stream()
                .flatMap(line -> Arrays.stream(line.split("\\s+")))
                .count();
            
            // Simulate additional processing
            Thread.sleep(100);
            
            long endTime = System.currentTimeMillis();
            
            return new FileProcessResult(
                filePath.getFileName().toString(),
                threadName,
                lines.size(),
                wordCount,
                endTime - startTime
            );
            
        } catch (Exception e) {
            return new FileProcessResult(
                filePath.getFileName().toString(),
                Thread.currentThread().getName(),
                -1, -1, -1
            );
        }
    }
    
    private static void printProcessingStats(List<FileProcessResult> results) {
        System.out.println("\n=== File Processing Results ===");
        results.forEach(result -> {
            System.out.printf("File: %s | Thread: %s | Lines: %d | Words: %d | Time: %dms%n",
                result.fileName(), result.threadName(), result.lineCount(), 
                result.wordCount(), result.processingTime());
        });
        
        long totalTime = results.stream()
            .mapToLong(FileProcessResult::processingTime)
            .max().orElse(0);
        
        System.out.println("Total processing time: " + totalTime + "ms");
    }
    
    public static void main(String[] args) throws IOException {
        // Create sample files for testing
        Path tempDir = Files.createTempDirectory("virtual-thread-test");
        List<Path> testFiles = createSampleFiles(tempDir, 10);
        
        System.out.println("Processing " + testFiles.size() + " files...");
        processFilesInParallel(testFiles);
        
        // Cleanup
        testFiles.forEach(path -> {
            try {
                Files.delete(path);
            } catch (IOException e) {
                System.err.println("Failed to delete " + path);
            }
        });
        
        try {
            Files.delete(tempDir);
        } catch (IOException e) {
            System.err.println("Failed to delete temp directory");
        }
    }
    
    private static List<Path> createSampleFiles(Path dir, int count) throws IOException {
        List<Path> files = new ArrayList<>();
        Random random = new Random();
        
        for (int i = 0; i < count; i++) {
            Path file = dir.resolve("sample_" + i + ".txt");
            
            // Generate random content
            List<String> content = new ArrayList<>();
            for (int j = 0; j < 100 + random.nextInt(900); j++) {
                content.add("This is line " + j + " in file " + i + 
                          " with some random content " + random.nextInt(1000));
            }
            
            Files.write(file, content);
            files.add(file);
        }
        
        return files;
    }
}

record FileProcessResult(String fileName, String threadName, 
                        long lineCount, long wordCount, long processingTime) {}
```

## Performance Comparison: Platform vs Virtual Threads

### Benchmark Example
```java
public class ThreadPerformanceBenchmark {
    
    private static final int THREAD_COUNT = 10000;
    private static final int WORK_DURATION_MS = 100;
    
    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== Thread Performance Benchmark ===\n");
        
        // Test platform threads
        benchmarkPlatformThreads();
        
        // Test virtual threads
        benchmarkVirtualThreads();
        
        // Test virtual thread executor
        benchmarkVirtualThreadExecutor();
    }
    
    private static void benchmarkPlatformThreads() throws InterruptedException {
        System.out.println("Testing Platform Threads...");
        long startTime = System.currentTimeMillis();
        long startMemory = getUsedMemory();
        
        List<Thread> threads = new ArrayList<>();
        
        try {
            for (int i = 0; i < THREAD_COUNT; i++) {
                Thread t = new Thread(() -> {
                    try {
                        Thread.sleep(WORK_DURATION_MS);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                });
                threads.add(t);
                t.start();
            }
            
            // Wait for all threads to complete
            for (Thread t : threads) {
                t.join();
            }
            
        } catch (OutOfMemoryError e) {
            System.out.println("OutOfMemoryError occurred with platform threads");
            // Force cleanup
            threads.forEach(Thread::interrupt);
            return;
        }
        
        long endTime = System.currentTimeMillis();
        long endMemory = getUsedMemory();
        
        System.out.printf("Platform Threads - Time: %d ms, Memory used: %d MB%n%n", 
                         endTime - startTime, (endMemory - startMemory) / (1024 * 1024));
    }
    
    private static void benchmarkVirtualThreads() throws InterruptedException {
        System.out.println("Testing Virtual Threads...");
        long startTime = System.currentTimeMillis();
        long startMemory = getUsedMemory();
        
        List<Thread> threads = new ArrayList<>();
        
        for (int i = 0; i < THREAD_COUNT; i++) {
            Thread vt = Thread.startVirtualThread(() -> {
                try {
                    Thread.sleep(WORK_DURATION_MS);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
            threads.add(vt);
        }
        
        // Wait for all threads to complete
        for (Thread vt : threads) {
            vt.join();
        }
        
        long endTime = System.currentTimeMillis();
        long endMemory = getUsedMemory();
        
        System.out.printf("Virtual Threads - Time: %d ms, Memory used: %d MB%n%n", 
                         endTime - startTime, (endMemory - startMemory) / (1024 * 1024));
    }
    
    private static void benchmarkVirtualThreadExecutor() throws InterruptedException {
        System.out.println("Testing Virtual Thread Executor...");
        long startTime = System.currentTimeMillis();
        long startMemory = getUsedMemory();
        
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            List<Future<?>> futures = new ArrayList<>();
            
            for (int i = 0; i < THREAD_COUNT; i++) {
                Future<?> future = executor.submit(() -> {
                    try {
                        Thread.sleep(WORK_DURATION_MS);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                });
                futures.add(future);
            }
            
            // Wait for all tasks to complete
            for (Future<?> future : futures) {
                try {
                    future.get();
                } catch (Exception e) {
                    System.err.println("Task execution failed: " + e.getMessage());
                }
            }
        }
        
        long endTime = System.currentTimeMillis();
        long endMemory = getUsedMemory();
        
        System.out.printf("Virtual Thread Executor - Time: %d ms, Memory used: %d MB%n%n", 
                         endTime - startTime, (endMemory - startMemory) / (1024 * 1024));
    }
    
    private static long getUsedMemory() {
        Runtime runtime = Runtime.getRuntime();
        return runtime.totalMemory() - runtime.freeMemory();
    }
}
```

## Best Practices and Guidelines

### 1. **When to Use Virtual Threads**
```java
public class VirtualThreadUseCases {
    
    // ✅ GOOD: I/O-bound operations
    public void ioIntensiveTask() {
        Thread.startVirtualThread(() -> {
            try {
                // Database calls
                String userData = fetchUserFromDatabase();
                
                // HTTP API calls
                String externalData = callExternalApi();
                
                // File I/O
                Files.writeString(Path.of("result.txt"), userData + externalData);
                
            } catch (Exception e) {
                e.printStackTrace();
            }
        });
    }
    
    // ✅ GOOD: High-concurrency scenarios
    public void handleManyRequests() {
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < 100000; i++) {
                executor.submit(() -> {
                    // Handle web request, database query, etc.
                    processRequest();
                });
            }
        }
    }
    
    // ❌ AVOID: CPU-intensive tasks
    public void cpuIntensiveTask() {
        // Don't use virtual threads for pure CPU work
        Thread.startVirtualThread(() -> {
            // This will block the carrier thread
            for (long i = 0; i < 1_000_000_000L; i++) {
                Math.sqrt(i); // CPU-intensive work
            }
        });
        
        // Better: Use platform threads or ForkJoinPool for CPU work
        ForkJoinPool.commonPool().submit(() -> {
            for (long i = 0; i < 1_000_000_000L; i++) {
                Math.sqrt(i);
            }
        });
    }
    
    // ❌ AVOID: Synchronization-heavy code
    public void heavySynchronization() {
        Object lock = new Object();
        
        // Avoid this pattern with virtual threads
        Thread.startVirtualThread(() -> {
            synchronized (lock) {
                // Long-running synchronized block
                try {
                    Thread.sleep(5000); // This pins the carrier thread
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        });
        
        // Better: Use concurrent data structures or locks
        ReentrantLock reentrantLock = new ReentrantLock();
        Thread.startVirtualThread(() -> {
            reentrantLock.lock();
            try {
                // Work here
                Thread.sleep(100);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                reentrantLock.unlock();
            }
        });
    }
    
    private String fetchUserFromDatabase() {
        // Simulate database call
        try {
            Thread.sleep(200);
            return "User data from DB";
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return "Error";
        }
    }
    
    private String callExternalApi() {
        // Simulate API call
        try {
            Thread.sleep(300);
            return "External API data";
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return "Error";
        }
    }
    
    private void processRequest() {
        try {
            Thread.sleep(50); // Simulate request processing
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

### 2. **Migration Strategies**
```java
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private EmailService emailService;
    
    // Before: Traditional async approach
    @Async
    public CompletableFuture<User> createUserAsync(CreateUserRequest request) {
        User user = new User();
        user.setName(request.getName());
        user.setEmail(request.getEmail());
        
        User savedUser = userRepository.save(user);
        
        // Send welcome email asynchronously
        emailService.sendWelcomeEmailAsync(savedUser.getEmail());
        
        return CompletableFuture.completedFuture(savedUser);
    }
    
    // After: Virtual threads approach
    public User createUserWithVirtualThread(CreateUserRequest request) {
        User user = new User();
        user.setName(request.getName());
        user.setEmail(request.getEmail());
        
        User savedUser = userRepository.save(user);
        
        // Send welcome email on virtual thread
        Thread.startVirtualThread(() -> {
            emailService.sendWelcomeEmail(savedUser.getEmail());
        });
        
        return savedUser;
    }
    
    // Bulk operations with virtual threads
    public List<User> createUsersInBatch(List<CreateUserRequest> requests) {
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            
            List<Future<User>> futures = requests.stream()
                .map(request -> executor.submit(() -> {
                    User user = new User();
                    user.setName(request.getName());
                    user.setEmail(request.getEmail());
                    
                    User savedUser = userRepository.save(user);
                    
                    // Send welcome email
                    emailService.sendWelcomeEmail(savedUser.getEmail());
                    
                    return savedUser;
                }))
                .collect(Collectors.toList());
            
            return futures.stream()
                .map(future -> {
                    try {
                        return future.get();
                    } catch (Exception e) {
                        throw new RuntimeException("Failed to create user", e);
                    }
                })
                .collect(Collectors.toList());
        }
    }
}
```

### 3. **Error Handling and Monitoring**
```java
public class VirtualThreadMonitoring {
    
    private static final Logger logger = LoggerFactory.getLogger(VirtualThreadMonitoring.class);
    
    public static void demonstrateErrorHandling() {
        Thread.Builder virtualBuilder = Thread.ofVirtual()
            .name("monitored-virtual-thread")
            .uncaughtExceptionHandler((thread, exception) -> {
                logger.error("Uncaught exception in virtual thread {}: {}", 
                           thread.getName(), exception.getMessage(), exception);
                
                // Send alert or metrics
                sendErrorMetrics(thread.getName(), exception);
            });
        
        virtualBuilder.start(() -> {
            try {
                // Simulate work that might fail
                if (Math.random() > 0.5) {
                    throw new RuntimeException("Random failure");
                }
                
                Thread.sleep(1000);
                logger.info("Virtual thread completed successfully");
                
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                logger.warn("Virtual thread interrupted");
            }
        });
    }
    
    public static void monitorVirtualThreadPool() {
        // Custom virtual thread factory with monitoring
        ThreadFactory virtualThreadFactory = Thread.ofVirtual()
            .name("monitored-vt-", 0)
            .factory();
        
        AtomicLong activeThreads = new AtomicLong(0);
        AtomicLong completedThreads = new AtomicLong(0);
        
        try (ExecutorService executor = Executors.newThreadPerTaskExecutor(virtualThreadFactory)) {
            
            // Submit multiple tasks
            List<Future<String>> futures = new ArrayList<>();
            
            for (int i = 0; i < 100; i++) {
                final int taskId = i;
                Future<String> future = executor.submit(() -> {
                    long active = activeThreads.incrementAndGet();
                    logger.debug("Active virtual threads: {}", active);
                    
                    try {
                        // Simulate work
                        Thread.sleep(1000 + (long)(Math.random() * 1000));
                        return "Task " + taskId + " completed";
                        
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        return "Task " + taskId + " interrupted";
                        
                    } finally {
                        long remaining = activeThreads.decrementAndGet();
                        long completed = completedThreads.incrementAndGet();
                        logger.debug("Virtual threads - Active: {}, Completed: {}", 
                                   remaining, completed);
                    }
                });
                futures.add(future);
            }
            
            // Monitor completion
            futures.forEach(future -> {
                try {
                    String result = future.get();
                    logger.info("Result: {}", result);
                } catch (Exception e) {
                    logger.error("Task execution failed", e);
                }
            });
        }
        
        logger.info("All virtual threads completed. Total: {}", completedThreads.get());
    }
    
    private static void sendErrorMetrics(String threadName, Throwable exception) {
        // Send to monitoring system (e.g., Micrometer, CloudWatch)
        // Example: Micrometer counter
        // Metrics.counter("virtual.thread.errors", 
        //     "thread", threadName, 
        //     "exception", exception.getClass().getSimpleName())
        //     .increment();
        
        logger.info("Error metrics sent for thread: {}", threadName);
    }
}
```

## Limitations and Considerations

### 1. **Pinning Issues**
```java
public class VirtualThreadPinning {
    
    // ❌ PROBLEMATIC: synchronized blocks pin virtual threads to carrier threads
    private static final Object lock = new Object();
    
    public static void demonstratePinning() {
        // This will pin the virtual thread to its carrier thread
        Thread.startVirtualThread(() -> {
            synchronized (lock) {
                try {
                    Thread.sleep(5000); // Long-running synchronized block
                    System.out.println("Work done in synchronized block");
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        });
    }
    
    // ✅ BETTER: Use ReentrantLock instead of synchronized
    private static final ReentrantLock reentrantLock = new ReentrantLock();
    
    public static void avoidPinning() {
        Thread.startVirtualThread(() -> {
            reentrantLock.lock();
            try {
                Thread.sleep(5000); // This won't pin the virtual thread
                System.out.println("Work done with ReentrantLock");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                reentrantLock.unlock();
            }
        });
    }
    
    // ❌ PROBLEMATIC: Native method calls can also cause pinning
    public static void nativeMethodPinning() {
        Thread.startVirtualThread(() -> {
            // Some native method calls may pin virtual threads
            // Example: certain file I/O operations
            try (FileInputStream fis = new FileInputStream("large-file.dat")) {
                byte[] buffer = new byte[8192];
                while (fis.read(buffer) != -1) {
                    // Process buffer
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        });
    }
    
    // ✅ BETTER: Use NIO for non-blocking I/O
    public static void nonBlockingIo() {
        Thread.startVirtualThread(() -> {
            try {
                Path path = Path.of("large-file.dat");
                byte[] content = Files.readAllBytes(path);
                // Process content
                System.out.println("File read: " + content.length + " bytes");
            } catch (IOException e) {
                e.printStackTrace();
            }
        });
    }
}
```

### 2. **Memory Considerations**
```java
public class VirtualThreadMemory {
    
    public static void demonstrateMemoryUsage() {
        Runtime runtime = Runtime.getRuntime();
        
        System.out.println("Initial memory usage:");
        printMemoryStats(runtime);
        
        // Create many virtual threads
        List<Thread> virtualThreads = new ArrayList<>();
        
        for (int i = 0; i < 100000; i++) {
            Thread vt = Thread.startVirtualThread(() -> {
                try {
                    Thread.sleep(60000); // Keep thread alive for measurement
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
            virtualThreads.add(vt);
        }
        
        System.out.println("\nAfter creating 100,000 virtual threads:");
        printMemoryStats(runtime);
        
        // Interrupt all threads to clean up
        virtualThreads.forEach(Thread::interrupt);
        
        // Force garbage collection
        System.gc();
        
        try {
            Thread.sleep(2000); // Wait for cleanup
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        System.out.println("\nAfter cleanup:");
        printMemoryStats(runtime);
    }
    
    private static void printMemoryStats(Runtime runtime) {
        long totalMemory = runtime.totalMemory();
        long freeMemory = runtime.freeMemory();
        long usedMemory = totalMemory - freeMemory;
        
        System.out.printf("Total: %d MB, Used: %d MB, Free: %d MB%n",
                         totalMemory / (1024 * 1024),
                         usedMemory / (1024 * 1024),
                         freeMemory / (1024 * 1024));
    }
}
```

## Integration with Frameworks

### 1. **Spring Boot Configuration**
```java
@Configuration
@EnableAsync
public class VirtualThreadConfig {
    
    @Bean(name = "virtualThreadExecutor")
    public Executor virtualThreadExecutor() {
        return Executors.newVirtualThreadPerTaskExecutor();
    }
    
    @Bean
    @Primary
    public AsyncTaskExecutor asyncTaskExecutor() {
        return new VirtualThreadTaskExecutor();
    }
    
    // Custom implementation
    public static class VirtualThreadTaskExecutor implements AsyncTaskExecutor {
        private final ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
        
        @Override
        public void execute(Runnable task) {
            executor.submit(task);
        }
        
        @Override
        public void execute(Runnable task, long startTimeout) {
            execute(task); // Virtual threads start immediately
        }
        
        @Override
        public Future<?> submit(Runnable task) {
            return executor.submit(task);
        }
        
        @Override
        public <T> Future<T> submit(Callable<T> task) {
            return executor.submit(task);
        }
    }
}
```

### 2. **Reactive Streams Integration**
```java
@Service
public class ReactiveVirtualThreadService {
    
    public Flux<String> processDataStream(Flux<String> inputStream) {
        return inputStream
            .publishOn(Schedulers.fromExecutor(Executors.newVirtualThreadPerTaskExecutor()))
            .map(this::processData)
            .subscribeOn(Schedulers.fromExecutor(Executors.newVirtualThreadPerTaskExecutor()));
    }
    
    public Mono<String> processDataAsync(String input) {
        return Mono.fromCallable(() -> processData(input))
            .subscribeOn(Schedulers.fromExecutor(Executors.newVirtualThreadPerTaskExecutor()));
    }
    
    private String processData(String input) {
        try {
            // Simulate processing
            Thread.sleep(100);
            return "Processed: " + input + " on " + Thread.currentThread().getName();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return "Error processing: " + input;
        }
    }
}
```

## Testing Virtual Threads

### 1. **Unit Testing**
```java
@ExtendWith(MockitoExtension.class)
class VirtualThreadServiceTest {
    
    @Mock
    private ExternalService externalService;
    
    @InjectMocks
    private VirtualThreadService virtualThreadService;
    
    @Test
    void testConcurrentProcessing() throws InterruptedException {
        // Arrange
        when(externalService.fetchData(anyString())).thenReturn("mocked data");
        
        int numberOfTasks = 100;
        List<String> inputs = IntStream.range(0, numberOfTasks)
            .mapToObj(i -> "input-" + i)
            .collect(Collectors.toList());
        
        // Act
        long startTime = System.currentTimeMillis();
        List<CompletableFuture<String>> futures = inputs.stream()
            .map(input -> CompletableFuture.supplyAsync(() -> 
                virtualThreadService.processInput(input)))
            .collect(Collectors.toList());
        
        List<String> results = futures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList());
        
        long endTime = System.currentTimeMillis();
        
        // Assert
        assertThat(results).hasSize(numberOfTasks);
        assertThat(results).allMatch(result -> result.startsWith("processed:"));
        
        // Virtual threads should complete much faster than sequential processing
        assertThat(endTime - startTime).isLessThan(5000); // Should be much less than 100 * 100ms
        
        // Verify all external service calls were made
        verify(externalService, times(numberOfTasks)).fetchData(anyString());
    }
    
    @Test
    void testVirtualThreadProperties() {
        AtomicReference<Thread> capturedThread = new AtomicReference<>();
        
        Thread virtualThread = Thread.startVirtualThread(() -> {
            capturedThread.set(Thread.currentThread());
        });
        
        try {
            virtualThread.join();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        Thread thread = capturedThread.get();
        assertThat(thread).isNotNull();
        assertThat(thread.isVirtual()).isTrue();
        assertThat(thread.getName()).contains("VirtualThread");
    }
}
```

### 2. **Performance Testing**
```java
@SpringBootTest
@ActiveProfiles("test")
class VirtualThreadPerformanceTest {
    
    @Autowired
    private VirtualThreadService virtualThreadService;
    
    @Test
    @Timeout(30) // Should complete within 30 seconds
    void testHighConcurrencyPerformance() {
        int numberOfRequests = 10000;
        List<CompletableFuture<String>> futures = new ArrayList<>();
        
        long startTime = System.currentTimeMillis();
        
        for (int i = 0; i < numberOfRequests; i++) {
            final int requestId = i;
            CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
                return virtualThreadService.handleRequest("request-" + requestId);
            });
            futures.add(future);
        }
        
        // Wait for all requests to complete
        CompletableFuture<Void> allDone = CompletableFuture.allOf(
            futures.toArray(new CompletableFuture[0]));
        
        allDone.join();
        
        long endTime = System.currentTimeMillis();
        long totalTime = endTime - startTime;
        
        System.out.printf("Processed %d requests in %d ms%n", numberOfRequests, totalTime);
        System.out.printf("Average: %.2f requests/second%n", 
                         (numberOfRequests * 1000.0) / totalTime);
        
        // Assertions
        assertThat(totalTime).isLessThan(10000); // Should complete in less than 10 seconds
        
        List<String> results = futures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList());
        
        assertThat(results).hasSize(numberOfRequests);
        assertThat(results).allMatch(result -> result.startsWith("handled:"));
    }
}
```

## Migration Guide

### From Platform Threads to Virtual Threads
```java
public class MigrationExamples {
    
    // BEFORE: Platform thread pool
    private final ExecutorService platformExecutor = Executors.newFixedThreadPool(100);
    
    public void processWithPlatformThreads(List<Task> tasks) {
        List<Future<Result>> futures = tasks.stream()
            .map(task -> platformExecutor.submit(() -> processTask(task)))
            .collect(Collectors.toList());
        
        futures.forEach(future -> {
            try {
                Result result = future.get();
                handleResult(result);
            } catch (Exception e) {
                handleError(e);
            }
        });
    }
    
    // AFTER: Virtual threads
    public void processWithVirtualThreads(List<Task> tasks) {
        try (ExecutorService virtualExecutor = Executors.newVirtualThreadPerTaskExecutor()) {
            List<Future<Result>> futures = tasks.stream()
                .map(task -> virtualExecutor.submit(() -> processTask(task)))
                .collect(Collectors.toList());
            
            futures.forEach(future -> {
                try {
                    Result result = future.get();
                    handleResult(result);
                } catch (Exception e) {
                    handleError(e);
                }
            });
        }
    }
    
    // BEFORE: CompletableFuture with custom executor
    public CompletableFuture<String> asyncOperationBefore(String input) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(1000); // Simulate I/O
                return "Processed: " + input;
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new RuntimeException(e);
            }
        }, platformExecutor);
    }
    
    // AFTER: Simplified with virtual threads
    public CompletableFuture<String> asyncOperationAfter(String input) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(1000); // Simulate I/O
                return "Processed: " + input;
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new RuntimeException(e);
            }
        }); // Uses common ForkJoinPool, but can also use virtual thread executor
    }
    
    // Helper methods
    private Result processTask(Task task) {
        try {
            Thread.sleep(100); // Simulate processing
            return new Result("Processed: " + task.getData());
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return new Result("Error processing: " + task.getData());
        }
    }
    
    private void handleResult(Result result) {
        System.out.println("Result: " + result.getValue());
    }
    
    private void handleError(Exception e) {
        System.err.println("Error: " + e.getMessage());
    }
    
    // Helper classes
    static class Task {
        private final String data;
        
        public Task(String data) {
            this.data = data;
        }
        
        public String getData() {
            return data;
        }
    }
    
    static class Result {
        private final String value;
        
        public Result(String value) {
            this.value = value;
        }
        
        public String getValue() {
            return value;
        }
    }
}
```

## Conclusion

Virtual threads represent a significant advancement in Java concurrency, offering:

### **Key Benefits:**
- **Massive Scalability**: Handle millions of concurrent operations
- **Simplified Programming Model**: Write blocking code that scales
- **Reduced Resource Usage**: Lower memory footprint per thread
- **Better Throughput**: Especially for I/O-bound applications

### **Best Use Cases:**
- Web servers and HTTP clients
- Database access layers
- File I/O operations
- Microservices communication
- High-concurrency applications

### **Migration Strategy:**
1. **Start with I/O-bound operations**
2. **Replace fixed thread pools with virtual thread executors**
3. **Monitor for pinning issues**
4. **Measure performance improvements**
5. **Gradually migrate existing async patterns**

Virtual threads are not a silver bullet, but they significantly simplify concurrent programming while providing excellent performance for I/O-intensive applications. They represent the future of Java concurrency and are essential knowledge for modern Java developers.

Remember: Virtual threads excel at **blocking I/O operations** but aren't necessarily better for **CPU-intensive tasks**. Choose the right tool for the right job, and enjoy the simplified concurrency model that virtual threads provide!

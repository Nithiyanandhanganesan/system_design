# Proxy Pattern - Complete Guide

## Introduction

The **Proxy Pattern** is a structural design pattern that provides a placeholder or surrogate for another object to control access to it. The proxy acts as an intermediary between the client and the real object, providing additional functionality like access control, caching, lazy loading, or logging.

## Definition

> **Proxy Pattern**: Provides a placeholder for another object to control access to it.

## Problem Solved

- **Access Control**: Control who can access the real object
- **Expensive Object Creation**: Delay creation until actually needed (lazy loading)
- **Remote Object Access**: Access objects in different address spaces
- **Caching**: Cache expensive operations or results
- **Logging/Monitoring**: Add logging without changing the real object
- **Security**: Add security checks before accessing the real object

## Types of Proxies

1. **Virtual Proxy**: Controls access to expensive resources
2. **Remote Proxy**: Represents objects in different address spaces
3. **Protection Proxy**: Controls access based on permissions
4. **Caching Proxy**: Caches results of expensive operations
5. **Smart Proxy**: Adds additional functionality (reference counting, logging)

## Structure

```
┌─────────────────┐    ┌─────────────────┐
│     Client      │───▶│    Subject      │
└─────────────────┘    ├─────────────────┤
                       │ +operation()    │
                       └─────────────────┘
                                △
                                │
                    ┌───────────┴───────────┐
                    │                       │
        ┌─────────────────┐    ┌─────────────────┐
        │      Proxy      │───▶│   RealSubject   │
        ├─────────────────┤    ├─────────────────┤
        │ +operation()    │    │ +operation()    │
        └─────────────────┘    └─────────────────┘
```

## Basic Implementation

### Subject Interface and Real Subject

```java
/**
 * Subject interface
 */
public interface FileDownloader {
    void downloadFile(String url, String destination);
    long getFileSize(String url);
    boolean isFileAvailable(String url);
}

/**
 * Real Subject - Expensive file download operations
 */
public class RealFileDownloader implements FileDownloader {
    
    @Override
    public void downloadFile(String url, String destination) {
        System.out.println("Connecting to: " + url);
        simulateNetworkDelay();
        
        System.out.println("Downloading file from: " + url);
        simulateDownload();
        
        System.out.println("File saved to: " + destination);
    }
    
    @Override
    public long getFileSize(String url) {
        System.out.println("Fetching file size for: " + url);
        simulateNetworkDelay();
        
        // Simulate file size calculation
        return url.hashCode() % 100000;
    }
    
    @Override
    public boolean isFileAvailable(String url) {
        System.out.println("Checking availability for: " + url);
        simulateNetworkDelay();
        
        return !url.contains("unavailable");
    }
    
    private void simulateNetworkDelay() {
        try {
            Thread.sleep(1000); // Simulate network delay
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
    
    private void simulateDownload() {
        try {
            Thread.sleep(3000); // Simulate download time
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

## 1. Virtual Proxy (Lazy Loading)

```java
/**
 * Virtual Proxy - Delays creation and caches results
 */
public class VirtualFileDownloaderProxy implements FileDownloader {
    
    private RealFileDownloader realDownloader;
    private final Map<String, Long> sizeCache = new ConcurrentHashMap<>();
    private final Map<String, Boolean> availabilityCache = new ConcurrentHashMap<>();
    
    private RealFileDownloader getRealDownloader() {
        if (realDownloader == null) {
            System.out.println("Creating RealFileDownloader instance...");
            realDownloader = new RealFileDownloader();
        }
        return realDownloader;
    }
    
    @Override
    public void downloadFile(String url, String destination) {
        System.out.println("Proxy: Preparing to download file");
        
        // Pre-download validation
        if (!isFileAvailable(url)) {
            throw new RuntimeException("File not available: " + url);
        }
        
        // Delegate to real object
        getRealDownloader().downloadFile(url, destination);
    }
    
    @Override
    public long getFileSize(String url) {
        // Check cache first
        if (sizeCache.containsKey(url)) {
            System.out.println("Proxy: Returning cached file size for: " + url);
            return sizeCache.get(url);
        }
        
        // Get from real object and cache
        long size = getRealDownloader().getFileSize(url);
        sizeCache.put(url, size);
        return size;
    }
    
    @Override
    public boolean isFileAvailable(String url) {
        // Check cache first
        if (availabilityCache.containsKey(url)) {
            System.out.println("Proxy: Returning cached availability for: " + url);
            return availabilityCache.get(url);
        }
        
        // Get from real object and cache
        boolean available = getRealDownloader().isFileAvailable(url);
        availabilityCache.put(url, available);
        return available;
    }
    
    public void clearCache() {
        sizeCache.clear();
        availabilityCache.clear();
        System.out.println("Proxy: Cache cleared");
    }
}
```

## 2. Protection Proxy (Access Control)

```java
/**
 * User class for access control
 */
public class User {
    private final String username;
    private final String role;
    
    public User(String username, String role) {
        this.username = username;
        this.role = role;
    }
    
    public String getUsername() { return username; }
    public String getRole() { return role; }
    
    @Override
    public String toString() {
        return "User{" + "username='" + username + "', role='" + role + "'}";
    }
}

/**
 * Protection Proxy - Controls access based on user permissions
 */
public class ProtectionFileDownloaderProxy implements FileDownloader {
    
    private final RealFileDownloader realDownloader;
    private final User currentUser;
    private final Set<String> restrictedUrls;
    
    public ProtectionFileDownloaderProxy(User currentUser) {
        this.realDownloader = new RealFileDownloader();
        this.currentUser = currentUser;
        this.restrictedUrls = Set.of(
            "confidential.pdf",
            "admin-only.zip",
            "secret-document.docx"
        );
    }
    
    @Override
    public void downloadFile(String url, String destination) {
        if (!hasDownloadPermission(url)) {
            throw new SecurityException(
                "Access denied: User " + currentUser.getUsername() + 
                " does not have permission to download: " + url
            );
        }
        
        logAccess("DOWNLOAD", url);
        realDownloader.downloadFile(url, destination);
    }
    
    @Override
    public long getFileSize(String url) {
        if (!hasReadPermission(url)) {
            throw new SecurityException(
                "Access denied: User " + currentUser.getUsername() + 
                " does not have permission to read file info: " + url
            );
        }
        
        logAccess("FILE_SIZE", url);
        return realDownloader.getFileSize(url);
    }
    
    @Override
    public boolean isFileAvailable(String url) {
        // Basic availability check allowed for all users
        logAccess("AVAILABILITY_CHECK", url);
        return realDownloader.isFileAvailable(url);
    }
    
    private boolean hasDownloadPermission(String url) {
        if (isRestrictedFile(url)) {
            return "ADMIN".equals(currentUser.getRole()) || 
                   "MANAGER".equals(currentUser.getRole());
        }
        return true; // Public files
    }
    
    private boolean hasReadPermission(String url) {
        if (isRestrictedFile(url)) {
            return "ADMIN".equals(currentUser.getRole()) || 
                   "MANAGER".equals(currentUser.getRole()) ||
                   "USER".equals(currentUser.getRole());
        }
        return true; // Public files
    }
    
    private boolean isRestrictedFile(String url) {
        return restrictedUrls.stream().anyMatch(url::contains);
    }
    
    private void logAccess(String operation, String url) {
        System.out.println(String.format(
            "Access Log: User=%s, Role=%s, Operation=%s, Resource=%s, Time=%s",
            currentUser.getUsername(),
            currentUser.getRole(),
            operation,
            url,
            LocalDateTime.now()
        ));
    }
}
```

## 3. Caching Proxy

```java
/**
 * Caching Proxy with TTL (Time To Live)
 */
public class CachingFileDownloaderProxy implements FileDownloader {
    
    private final RealFileDownloader realDownloader;
    private final Map<String, CacheEntry<Long>> sizeCache = new ConcurrentHashMap<>();
    private final Map<String, CacheEntry<Boolean>> availabilityCache = new ConcurrentHashMap<>();
    private final long cacheTTL = 5000; // 5 seconds TTL
    
    public CachingFileDownloaderProxy() {
        this.realDownloader = new RealFileDownloader();
        
        // Start cache cleanup thread
        startCacheCleanupThread();
    }
    
    @Override
    public void downloadFile(String url, String destination) {
        // Downloads are not cached, always go to real object
        System.out.println("Caching Proxy: Delegating download to real downloader");
        realDownloader.downloadFile(url, destination);
        
        // Invalidate related cache entries after successful download
        invalidateCache(url);
    }
    
    @Override
    public long getFileSize(String url) {
        CacheEntry<Long> cached = sizeCache.get(url);
        
        if (cached != null && !cached.isExpired()) {
            System.out.println("Caching Proxy: Returning cached file size for: " + url);
            return cached.getValue();
        }
        
        System.out.println("Caching Proxy: Cache miss, fetching from real downloader");
        long size = realDownloader.getFileSize(url);
        
        sizeCache.put(url, new CacheEntry<>(size, System.currentTimeMillis() + cacheTTL));
        return size;
    }
    
    @Override
    public boolean isFileAvailable(String url) {
        CacheEntry<Boolean> cached = availabilityCache.get(url);
        
        if (cached != null && !cached.isExpired()) {
            System.out.println("Caching Proxy: Returning cached availability for: " + url);
            return cached.getValue();
        }
        
        System.out.println("Caching Proxy: Cache miss, checking with real downloader");
        boolean available = realDownloader.isFileAvailable(url);
        
        availabilityCache.put(url, new CacheEntry<>(available, System.currentTimeMillis() + cacheTTL));
        return available;
    }
    
    private void invalidateCache(String url) {
        sizeCache.remove(url);
        availabilityCache.remove(url);
        System.out.println("Caching Proxy: Cache invalidated for: " + url);
    }
    
    public void clearCache() {
        sizeCache.clear();
        availabilityCache.clear();
        System.out.println("Caching Proxy: All cache cleared");
    }
    
    public CacheStats getCacheStats() {
        return new CacheStats(sizeCache.size(), availabilityCache.size());
    }
    
    private void startCacheCleanupThread() {
        Thread cleanupThread = new Thread(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                try {
                    Thread.sleep(10000); // Cleanup every 10 seconds
                    cleanupExpiredEntries();
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        });
        cleanupThread.setDaemon(true);
        cleanupThread.start();
    }
    
    private void cleanupExpiredEntries() {
        long now = System.currentTimeMillis();
        
        sizeCache.entrySet().removeIf(entry -> entry.getValue().isExpired());
        availabilityCache.entrySet().removeIf(entry -> entry.getValue().isExpired());
        
        System.out.println("Caching Proxy: Expired cache entries cleaned up");
    }
    
    /**
     * Cache entry with TTL
     */
    private static class CacheEntry<T> {
        private final T value;
        private final long expirationTime;
        
        public CacheEntry(T value, long expirationTime) {
            this.value = value;
            this.expirationTime = expirationTime;
        }
        
        public T getValue() {
            return value;
        }
        
        public boolean isExpired() {
            return System.currentTimeMillis() > expirationTime;
        }
    }
    
    /**
     * Cache statistics
     */
    public static class CacheStats {
        private final int sizeCacheEntries;
        private final int availabilityCacheEntries;
        
        public CacheStats(int sizeCacheEntries, int availabilityCacheEntries) {
            this.sizeCacheEntries = sizeCacheEntries;
            this.availabilityCacheEntries = availabilityCacheEntries;
        }
        
        @Override
        public String toString() {
            return String.format("CacheStats{sizeCache=%d, availabilityCache=%d}",
                sizeCacheEntries, availabilityCacheEntries);
        }
    }
}
```

## 4. Smart Proxy (Logging and Monitoring)

```java
/**
 * Smart Proxy with comprehensive monitoring and logging
 */
public class SmartFileDownloaderProxy implements FileDownloader {
    
    private final RealFileDownloader realDownloader;
    private final Map<String, Integer> accessCounts = new ConcurrentHashMap<>();
    private final List<AccessLog> accessHistory = Collections.synchronizedList(new ArrayList<>());
    private final long startTime = System.currentTimeMillis();
    
    public SmartFileDownloaderProxy() {
        this.realDownloader = new RealFileDownloader();
    }
    
    @Override
    public void downloadFile(String url, String destination) {
        long operationStart = System.currentTimeMillis();
        String operation = "DOWNLOAD";
        
        try {
            logMethodCall(operation, url);
            incrementAccessCount(url);
            
            realDownloader.downloadFile(url, destination);
            
            long duration = System.currentTimeMillis() - operationStart;
            logMethodComplete(operation, url, duration, true);
            
        } catch (Exception e) {
            long duration = System.currentTimeMillis() - operationStart;
            logMethodComplete(operation, url, duration, false);
            logError(operation, url, e);
            throw e;
        }
    }
    
    @Override
    public long getFileSize(String url) {
        long operationStart = System.currentTimeMillis();
        String operation = "GET_SIZE";
        
        try {
            logMethodCall(operation, url);
            incrementAccessCount(url);
            
            long size = realDownloader.getFileSize(url);
            
            long duration = System.currentTimeMillis() - operationStart;
            logMethodComplete(operation, url, duration, true);
            
            return size;
            
        } catch (Exception e) {
            long duration = System.currentTimeMillis() - operationStart;
            logMethodComplete(operation, url, duration, false);
            logError(operation, url, e);
            throw e;
        }
    }
    
    @Override
    public boolean isFileAvailable(String url) {
        long operationStart = System.currentTimeMillis();
        String operation = "CHECK_AVAILABILITY";
        
        try {
            logMethodCall(operation, url);
            incrementAccessCount(url);
            
            boolean available = realDownloader.isFileAvailable(url);
            
            long duration = System.currentTimeMillis() - operationStart;
            logMethodComplete(operation, url, duration, true);
            
            return available;
            
        } catch (Exception e) {
            long duration = System.currentTimeMillis() - operationStart;
            logMethodComplete(operation, url, duration, false);
            logError(operation, url, e);
            throw e;
        }
    }
    
    private void logMethodCall(String operation, String url) {
        System.out.println(String.format(
            "[Smart Proxy] Starting %s operation for: %s",
            operation, url
        ));
    }
    
    private void logMethodComplete(String operation, String url, long duration, boolean success) {
        AccessLog log = new AccessLog(operation, url, duration, success, System.currentTimeMillis());
        accessHistory.add(log);
        
        System.out.println(String.format(
            "[Smart Proxy] %s operation %s for: %s (took %dms)",
            operation, success ? "completed" : "failed", url, duration
        ));
    }
    
    private void logError(String operation, String url, Exception e) {
        System.err.println(String.format(
            "[Smart Proxy] Error in %s operation for %s: %s",
            operation, url, e.getMessage()
        ));
    }
    
    private void incrementAccessCount(String url) {
        accessCounts.merge(url, 1, Integer::sum);
    }
    
    // Monitoring methods
    public Map<String, Integer> getAccessCounts() {
        return new HashMap<>(accessCounts);
    }
    
    public List<AccessLog> getAccessHistory() {
        return new ArrayList<>(accessHistory);
    }
    
    public ProxyStatistics getStatistics() {
        long uptime = System.currentTimeMillis() - startTime;
        int totalAccesses = accessCounts.values().stream().mapToInt(Integer::intValue).sum();
        
        long successfulOperations = accessHistory.stream()
            .mapToLong(log -> log.success ? 1 : 0)
            .sum();
        
        double averageResponseTime = accessHistory.stream()
            .mapToLong(log -> log.duration)
            .average()
            .orElse(0.0);
        
        return new ProxyStatistics(uptime, totalAccesses, successfulOperations, averageResponseTime);
    }
    
    public void printStatistics() {
        ProxyStatistics stats = getStatistics();
        System.out.println("\n=== Smart Proxy Statistics ===");
        System.out.println("Uptime: " + stats.uptime + "ms");
        System.out.println("Total Accesses: " + stats.totalAccesses);
        System.out.println("Successful Operations: " + stats.successfulOperations);
        System.out.println("Average Response Time: " + String.format("%.2f", stats.averageResponseTime) + "ms");
        
        System.out.println("\nMost Accessed URLs:");
        accessCounts.entrySet().stream()
            .sorted(Map.Entry.<String, Integer>comparingByValue().reversed())
            .limit(5)
            .forEach(entry -> System.out.println("  " + entry.getKey() + ": " + entry.getValue() + " accesses"));
    }
    
    /**
     * Access log entry
     */
    public static class AccessLog {
        public final String operation;
        public final String url;
        public final long duration;
        public final boolean success;
        public final long timestamp;
        
        public AccessLog(String operation, String url, long duration, boolean success, long timestamp) {
            this.operation = operation;
            this.url = url;
            this.duration = duration;
            this.success = success;
            this.timestamp = timestamp;
        }
        
        @Override
        public String toString() {
            return String.format("AccessLog{operation='%s', url='%s', duration=%d, success=%s, timestamp=%d}",
                operation, url, duration, success, timestamp);
        }
    }
    
    /**
     * Proxy statistics
     */
    public static class ProxyStatistics {
        public final long uptime;
        public final int totalAccesses;
        public final long successfulOperations;
        public final double averageResponseTime;
        
        public ProxyStatistics(long uptime, int totalAccesses, long successfulOperations, double averageResponseTime) {
            this.uptime = uptime;
            this.totalAccesses = totalAccesses;
            this.successfulOperations = successfulOperations;
            this.averageResponseTime = averageResponseTime;
        }
    }
}
```

## 5. Remote Proxy (Simulated)

```java
/**
 * Remote Proxy - simulates access to remote objects
 */
public class RemoteFileDownloaderProxy implements FileDownloader {
    
    private final String serverAddress;
    private final int serverPort;
    private boolean connected = false;
    
    public RemoteFileDownloaderProxy(String serverAddress, int serverPort) {
        this.serverAddress = serverAddress;
        this.serverPort = serverPort;
    }
    
    @Override
    public void downloadFile(String url, String destination) {
        ensureConnection();
        
        String command = String.format("DOWNLOAD|%s|%s", url, destination);
        String response = sendCommand(command);
        
        if (!response.startsWith("SUCCESS")) {
            throw new RuntimeException("Remote download failed: " + response);
        }
        
        System.out.println("Remote download completed: " + response);
    }
    
    @Override
    public long getFileSize(String url) {
        ensureConnection();
        
        String command = String.format("GET_SIZE|%s", url);
        String response = sendCommand(command);
        
        if (response.startsWith("ERROR")) {
            throw new RuntimeException("Remote size check failed: " + response);
        }
        
        return Long.parseLong(response.split("\\|")[1]);
    }
    
    @Override
    public boolean isFileAvailable(String url) {
        ensureConnection();
        
        String command = String.format("CHECK_AVAILABLE|%s", url);
        String response = sendCommand(command);
        
        return response.startsWith("AVAILABLE");
    }
    
    private void ensureConnection() {
        if (!connected) {
            connect();
        }
    }
    
    private void connect() {
        System.out.println(String.format(
            "Connecting to remote server: %s:%d", serverAddress, serverPort
        ));
        
        // Simulate connection establishment
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        connected = true;
        System.out.println("Connected to remote server");
    }
    
    private String sendCommand(String command) {
        System.out.println("Sending remote command: " + command);
        
        // Simulate network communication
        try {
            Thread.sleep(200);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return "ERROR|Connection interrupted";
        }
        
        // Simulate server response
        if (command.contains("unavailable")) {
            return "ERROR|File not found";
        } else if (command.startsWith("GET_SIZE")) {
            String url = command.split("\\|")[1];
            return "SIZE|" + (url.hashCode() % 100000);
        } else if (command.startsWith("CHECK_AVAILABLE")) {
            return "AVAILABLE|true";
        } else if (command.startsWith("DOWNLOAD")) {
            return "SUCCESS|File downloaded";
        }
        
        return "ERROR|Unknown command";
    }
    
    public void disconnect() {
        if (connected) {
            System.out.println("Disconnecting from remote server");
            connected = false;
        }
    }
    
    public boolean isConnected() {
        return connected;
    }
}
```

## Real-World Example: Database Connection Proxy

### Database Interface and Implementation

```java
/**
 * Database connection interface
 */
public interface DatabaseConnection {
    ResultSet executeQuery(String sql);
    int executeUpdate(String sql);
    void beginTransaction();
    void commit();
    void rollback();
    void close();
    boolean isClosed();
}

/**
 * Real database connection
 */
public class RealDatabaseConnection implements DatabaseConnection {
    
    private final String connectionString;
    private boolean closed = false;
    private boolean inTransaction = false;
    
    public RealDatabaseConnection(String connectionString) {
        this.connectionString = connectionString;
        System.out.println("Established real database connection to: " + connectionString);
    }
    
    @Override
    public ResultSet executeQuery(String sql) {
        checkConnection();
        System.out.println("Executing query: " + sql);
        
        // Simulate query execution
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        return new MockResultSet(sql);
    }
    
    @Override
    public int executeUpdate(String sql) {
        checkConnection();
        System.out.println("Executing update: " + sql);
        
        // Simulate update execution
        try {
            Thread.sleep(50);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        return 1; // Simulated affected rows
    }
    
    @Override
    public void beginTransaction() {
        checkConnection();
        inTransaction = true;
        System.out.println("Transaction started");
    }
    
    @Override
    public void commit() {
        checkConnection();
        if (!inTransaction) {
            throw new RuntimeException("No active transaction to commit");
        }
        inTransaction = false;
        System.out.println("Transaction committed");
    }
    
    @Override
    public void rollback() {
        checkConnection();
        if (!inTransaction) {
            throw new RuntimeException("No active transaction to rollback");
        }
        inTransaction = false;
        System.out.println("Transaction rolled back");
    }
    
    @Override
    public void close() {
        if (!closed) {
            if (inTransaction) {
                rollback();
            }
            closed = true;
            System.out.println("Database connection closed");
        }
    }
    
    @Override
    public boolean isClosed() {
        return closed;
    }
    
    private void checkConnection() {
        if (closed) {
            throw new RuntimeException("Connection is closed");
        }
    }
    
    // Mock ResultSet for demonstration
    private static class MockResultSet implements ResultSet {
        private final String query;
        
        public MockResultSet(String query) {
            this.query = query;
        }
        
        // Implement only necessary methods for demo
        @Override
        public String toString() {
            return "MockResultSet{query='" + query + "'}";
        }
        
        // ... other ResultSet methods would be implemented
        @Override public boolean next() throws SQLException { return false; }
        @Override public void close() throws SQLException {}
        @Override public boolean wasNull() throws SQLException { return false; }
        // ... (remaining methods omitted for brevity)
    }
}
```

### Database Connection Proxy

```java
/**
 * Database Connection Proxy with connection pooling, caching, and monitoring
 */
public class DatabaseConnectionProxy implements DatabaseConnection {
    
    private RealDatabaseConnection realConnection;
    private final String connectionString;
    private final Map<String, CachedResult> queryCache = new ConcurrentHashMap<>();
    private final List<String> queryHistory = Collections.synchronizedList(new ArrayList<>());
    private final long cacheTTL = 30000; // 30 seconds
    private boolean autoCommit = true;
    
    public DatabaseConnectionProxy(String connectionString) {
        this.connectionString = connectionString;
    }
    
    @Override
    public ResultSet executeQuery(String sql) {
        logQuery(sql);
        
        // Check cache for SELECT queries
        if (sql.trim().toUpperCase().startsWith("SELECT")) {
            CachedResult cached = queryCache.get(sql);
            if (cached != null && !cached.isExpired()) {
                System.out.println("Returning cached result for query: " + sql);
                return cached.getResultSet();
            }
        }
        
        // Execute query through real connection
        getRealConnection();
        ResultSet result = realConnection.executeQuery(sql);
        
        // Cache SELECT query results
        if (sql.trim().toUpperCase().startsWith("SELECT")) {
            queryCache.put(sql, new CachedResult(result, System.currentTimeMillis() + cacheTTL));
        }
        
        return result;
    }
    
    @Override
    public int executeUpdate(String sql) {
        logQuery(sql);
        
        // Updates invalidate cache
        invalidateCache();
        
        getRealConnection();
        return realConnection.executeUpdate(sql);
    }
    
    @Override
    public void beginTransaction() {
        getRealConnection();
        autoCommit = false;
        realConnection.beginTransaction();
    }
    
    @Override
    public void commit() {
        if (realConnection != null) {
            realConnection.commit();
            autoCommit = true;
        }
    }
    
    @Override
    public void rollback() {
        if (realConnection != null) {
            realConnection.rollback();
            autoCommit = true;
        }
    }
    
    @Override
    public void close() {
        if (realConnection != null) {
            realConnection.close();
            realConnection = null;
        }
        queryCache.clear();
    }
    
    @Override
    public boolean isClosed() {
        return realConnection == null || realConnection.isClosed();
    }
    
    private RealDatabaseConnection getRealConnection() {
        if (realConnection == null || realConnection.isClosed()) {
            System.out.println("Proxy: Creating new database connection");
            realConnection = new RealDatabaseConnection(connectionString);
        }
        return realConnection;
    }
    
    private void logQuery(String sql) {
        queryHistory.add(sql);
        System.out.println("Proxy: Query logged - " + sql);
    }
    
    private void invalidateCache() {
        if (!queryCache.isEmpty()) {
            queryCache.clear();
            System.out.println("Proxy: Query cache invalidated");
        }
    }
    
    // Monitoring methods
    public List<String> getQueryHistory() {
        return new ArrayList<>(queryHistory);
    }
    
    public int getCacheSize() {
        return queryCache.size();
    }
    
    public void printStatistics() {
        System.out.println("\n=== Database Proxy Statistics ===");
        System.out.println("Total queries executed: " + queryHistory.size());
        System.out.println("Cache entries: " + queryCache.size());
        System.out.println("Connection status: " + (isClosed() ? "Closed" : "Open"));
        
        // Count query types
        Map<String, Long> queryTypes = queryHistory.stream()
            .collect(Collectors.groupingBy(
                sql -> sql.trim().split("\\s+")[0].toUpperCase(),
                Collectors.counting()
            ));
        
        System.out.println("Query types:");
        queryTypes.forEach((type, count) -> 
            System.out.println("  " + type + ": " + count));
    }
    
    /**
     * Cached result with TTL
     */
    private static class CachedResult {
        private final ResultSet resultSet;
        private final long expirationTime;
        
        public CachedResult(ResultSet resultSet, long expirationTime) {
            this.resultSet = resultSet;
            this.expirationTime = expirationTime;
        }
        
        public ResultSet getResultSet() {
            return resultSet;
        }
        
        public boolean isExpired() {
            return System.currentTimeMillis() > expirationTime;
        }
    }
}
```

## Spring Boot Integration

### Proxy Configuration

```java
/**
 * Proxy configuration in Spring Boot
 */
@Configuration
public class ProxyConfig {
    
    @Bean
    @Primary
    public FileDownloader fileDownloader(@Value("${app.proxy.type:smart}") String proxyType,
                                        @Value("${app.current.user:}") String currentUser) {
        
        switch (proxyType.toLowerCase()) {
            case "virtual":
                return new VirtualFileDownloaderProxy();
            case "caching":
                return new CachingFileDownloaderProxy();
            case "protection":
                User user = parseUser(currentUser);
                return new ProtectionFileDownloaderProxy(user);
            case "smart":
            default:
                return new SmartFileDownloaderProxy();
        }
    }
    
    @Bean
    @ConditionalOnProperty(value = "app.database.proxy.enabled", havingValue = "true")
    public DatabaseConnection databaseConnection(@Value("${app.database.url}") String connectionString) {
        return new DatabaseConnectionProxy(connectionString);
    }
    
    private User parseUser(String userString) {
        if (userString.isEmpty()) {
            return new User("anonymous", "GUEST");
        }
        
        String[] parts = userString.split(":");
        return new User(parts[0], parts.length > 1 ? parts[1] : "USER");
    }
}

/**
 * Service using proxied objects
 */
@Service
public class FileService {
    
    private final FileDownloader fileDownloader;
    
    public FileService(FileDownloader fileDownloader) {
        this.fileDownloader = fileDownloader;
    }
    
    public void downloadFile(String url, String destination) {
        try {
            if (fileDownloader.isFileAvailable(url)) {
                long size = fileDownloader.getFileSize(url);
                System.out.println("Downloading file of size: " + size + " bytes");
                
                fileDownloader.downloadFile(url, destination);
                System.out.println("Download completed successfully");
            } else {
                System.out.println("File not available: " + url);
            }
        } catch (Exception e) {
            System.err.println("Download failed: " + e.getMessage());
        }
    }
}
```

## Testing Proxy Pattern

### Unit Tests

```java
/**
 * Tests for different proxy implementations
 */
@ExtendWith(MockitoExtension.class)
class ProxyPatternTest {
    
    @Test
    void testVirtualProxy() {
        VirtualFileDownloaderProxy proxy = new VirtualFileDownloaderProxy();
        
        // First call should create real object and cache result
        long size1 = proxy.getFileSize("test.pdf");
        
        // Second call should use cached result
        long size2 = proxy.getFileSize("test.pdf");
        
        assertThat(size1).isEqualTo(size2);
    }
    
    @Test
    void testProtectionProxy() {
        User adminUser = new User("admin", "ADMIN");
        User regularUser = new User("user", "USER");
        
        ProtectionFileDownloaderProxy adminProxy = new ProtectionFileDownloaderProxy(adminUser);
        ProtectionFileDownloaderProxy userProxy = new ProtectionFileDownloaderProxy(regularUser);
        
        // Admin should be able to download restricted files
        assertDoesNotThrow(() -> adminProxy.downloadFile("confidential.pdf", "downloads/"));
        
        // Regular user should be denied access to restricted files
        assertThrows(SecurityException.class, () -> 
            userProxy.downloadFile("confidential.pdf", "downloads/"));
    }
    
    @Test
    void testCachingProxy() throws InterruptedException {
        CachingFileDownloaderProxy proxy = new CachingFileDownloaderProxy();
        
        // First call - should hit real object
        boolean available1 = proxy.isFileAvailable("test.pdf");
        
        // Second call - should use cache
        boolean available2 = proxy.isFileAvailable("test.pdf");
        
        assertThat(available1).isEqualTo(available2);
        
        // Wait for cache to expire and test again
        Thread.sleep(6000); // TTL is 5 seconds
        boolean available3 = proxy.isFileAvailable("test.pdf");
        
        assertThat(available3).isEqualTo(available1);
    }
    
    @Test
    void testSmartProxy() {
        SmartFileDownloaderProxy proxy = new SmartFileDownloaderProxy();
        
        // Perform some operations
        proxy.isFileAvailable("file1.pdf");
        proxy.isFileAvailable("file2.pdf");
        proxy.isFileAvailable("file1.pdf"); // Repeat access
        
        // Check statistics
        Map<String, Integer> accessCounts = proxy.getAccessCounts();
        assertThat(accessCounts.get("file1.pdf")).isEqualTo(2);
        assertThat(accessCounts.get("file2.pdf")).isEqualTo(1);
        
        assertThat(proxy.getAccessHistory()).hasSize(3);
    }
    
    @Test
    void testDatabaseProxy() {
        DatabaseConnectionProxy proxy = new DatabaseConnectionProxy("test-db");
        
        // Execute some queries
        proxy.executeQuery("SELECT * FROM users");
        proxy.executeQuery("SELECT * FROM orders");
        proxy.executeQuery("SELECT * FROM users"); // Cached
        
        assertThat(proxy.getQueryHistory()).hasSize(3);
        assertThat(proxy.getCacheSize()).isEqualTo(2); // Two unique SELECT queries
    }
    
    @Test
    void testRemoteProxy() {
        RemoteFileDownloaderProxy proxy = new RemoteFileDownloaderProxy("localhost", 8080);
        
        // Should establish connection on first call
        assertThat(proxy.isConnected()).isFalse();
        
        boolean available = proxy.isFileAvailable("remote-file.pdf");
        
        assertThat(proxy.isConnected()).isTrue();
        assertThat(available).isTrue();
        
        proxy.disconnect();
        assertThat(proxy.isConnected()).isFalse();
    }
}
```

### Integration Tests

```java
/**
 * Integration tests for Spring Boot proxy configuration
 */
@SpringBootTest
@TestPropertySource(properties = {
    "app.proxy.type=smart",
    "app.current.user=testuser:USER"
})
class ProxyIntegrationTest {
    
    @Autowired
    private FileDownloader fileDownloader;
    
    @Autowired
    private FileService fileService;
    
    @Test
    void testProxyIntegration() {
        // Test that the correct proxy is injected
        assertThat(fileDownloader).isInstanceOf(SmartFileDownloaderProxy.class);
        
        // Test service functionality
        fileService.downloadFile("test-file.pdf", "downloads/");
        
        // Cast to smart proxy to check statistics
        SmartFileDownloaderProxy smartProxy = (SmartFileDownloaderProxy) fileDownloader;
        assertThat(smartProxy.getAccessCounts()).isNotEmpty();
    }
}
```

## Best Practices

### ✅ **Do's:**

1. **Implement same interface** as the real object
2. **Cache expensive operations** when appropriate
3. **Add proper error handling** in proxy logic
4. **Log proxy activities** for debugging
5. **Make proxies transparent** to clients
6. **Use dependency injection** for proxy configuration
7. **Consider thread safety** for shared proxies

### ❌ **Don'ts:**

1. **Don't add unnecessary complexity** if simple delegation suffices
2. **Don't leak proxy-specific details** to clients
3. **Don't ignore performance overhead** of proxy operations
4. **Don't forget to clean up resources** (caches, connections)
5. **Don't create deep proxy chains** - maintain simplicity
6. **Don't break the Liskov Substitution Principle**

## When to Use Proxy Pattern

### ✅ **Good Use Cases:**
- **Expensive object creation**: Virtual proxies for lazy loading
- **Access control**: Protection proxies for security
- **Remote objects**: Representing objects in different locations
- **Caching**: Storing results of expensive operations
- **Monitoring**: Adding logging without changing original objects
- **Resource management**: Controlling access to limited resources

### ❌ **Avoid Proxy For:**
- **Simple object access**: Direct instantiation is sufficient
- **Stateless operations**: No benefit from proxy overhead
- **Performance-critical paths**: Proxy overhead might be significant
- **Already complex systems**: Avoid adding unnecessary indirection

## Related Patterns

- **Adapter**: Changes interface vs. controls access
- **Decorator**: Adds behavior vs. controls access
- **Facade**: Simplifies interface vs. controls access to single object

## Summary

The Proxy pattern is powerful for:

**Key Benefits:**
- **Access Control**: Security and permission management
- **Performance**: Caching and lazy loading
- **Monitoring**: Logging and statistics without changing original code
- **Transparency**: Clients don't know they're using a proxy

**Common Types:**
- **Virtual Proxy**: Lazy loading and caching
- **Protection Proxy**: Access control and security
- **Remote Proxy**: Representing remote objects
- **Smart Proxy**: Additional functionality like logging

Use proxies when you need to control access to objects or add cross-cutting concerns without modifying the original implementation!

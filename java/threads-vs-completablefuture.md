# Threads vs CompletableFuture

## Threads
Threads are the smallest unit of processing that can be scheduled by an operating system. In Java, threads can be created by extending the `Thread` class or implementing the `Runnable` interface. They allow for concurrent execution, enabling developers to run different parts of a program in parallel.

### Pros of Using Threads:
- Lightweight compared to processes.
- Can achieve efficient resource utilization.
- Suitable for CPU-intensive tasks.

### Cons of Using Threads:
- Complexity in managing thread lifecycle.
- Risk of concurrency issues (like race conditions).
- Threads may consume system resources if not managed properly.

## CompletableFuture
`CompletableFuture` is a part of the Java 8 `java.util.concurrent` package and provides a more flexible approach to asynchronous programming. Unlike traditional threads, `CompletableFuture` helps in writing non-blocking code.

### Advantages of CompletableFuture:
- Easier to read and maintain asynchronous code.
- Support for chaining multiple tasks with functional-style programming.
- Allows combining multiple futures and handling exceptions gracefully.

### Example Usage:
```java
CompletableFuture.supplyAsync(() -> {
    // Long running task
})
.thenApply(result -> {
    // Process result
});
```

## ExecutorService
The `ExecutorService` is a higher-level replacement for managing threads. It provides a thread pool that can manage a pool of threads and delegate tasks to them.

### Benefits of ExecutorService:
- Reuse threads for multiple tasks, reducing overhead.
- Control over the concurrency level with configurable thread pool sizes.
- Simplifies execution of asynchronous tasks.

### Example Usage:
```java
ExecutorService executor = Executors.newFixedThreadPool(10);
executor.submit(() -> {
    // Your concurrent code here
});
executor.shutdown();
```
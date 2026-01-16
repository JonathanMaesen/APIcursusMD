# Concurrency Cheatsheet

This cheatsheet provides a quick reference for handling concurrency in .NET, focusing on asynchronous programming with `async`/`await` and parallel processing.

---

## I. Asynchronous Programming (`async` / `await`)

Asynchronous programming frees up the calling thread (usually the UI or request thread) while waiting for I/O-bound operations (database, file system, network) to complete.

### 1. Basic Syntax

*   **`async`**: Marks a method as asynchronous. It allows the use of `await` inside.
*   **`await`**: Pauses the execution of the method until the awaited task completes, returning control to the caller.
*   **Return Types**:
    *   `Task<T>`: For methods that return a value.
    *   `Task`: For methods that return `void`.
    *   `void`: **Avoid** (except for event handlers).

**Example:** Fetching data from a database asynchronously.

```csharp
public async Task<User> GetUserAsync(int id)
{
    // The thread is freed here while the DB query runs
    var user = await _context.Users.FindAsync(id);
    return user;
}
```

### 2. Best Practices

*   **Async All the Way:** If you call an async method, your caller should also be async. Avoid `.Result` or `.Wait()` as they block threads and can cause deadlocks.
*   **Suffix:** By convention, append `Async` to the name of asynchronous methods.
*   **CancellationTokens:** Pass `CancellationToken` to async methods to allow operations to be cancelled gracefully.

**Example:** Correctly calling an async method.

```csharp
// BAD: Blocks the thread
var user = GetUserAsync(1).Result;

// GOOD: Awaits non-blocking
var user = await GetUserAsync(1);
```

---

## II. Task Parallel Library (TPL)

Used for CPU-bound operations where you want to utilize multiple processor cores.

### 1. `Task.Run`

Offloads work to a background thread from the thread pool.

**Example:** Performing a heavy calculation without freezing the UI.

```csharp
public async Task ProcessDataAsync()
{
    // Offload CPU-intensive work to a background thread
    await Task.Run(() =>
    {
        HeavyCalculation();
    });
}
```

### 2. `Task.WhenAll`

Runs multiple tasks concurrently and waits for all of them to complete.

**Example:** Fetching data from two sources simultaneously.

```csharp
var task1 = GetOrdersAsync();
var task2 = GetCustomersAsync();

// Both tasks run in parallel
await Task.WhenAll(task1, task2);

var orders = await task1;
var customers = await task2;
```

### 3. `Task.WhenAny`

Waits for the first of multiple tasks to complete.

**Example:** implementing a timeout.

```csharp
var downloadTask = DownloadFileAsync();
var timeoutTask = Task.Delay(5000); // 5 seconds

var completedTask = await Task.WhenAny(downloadTask, timeoutTask);

if (completedTask == timeoutTask)
{
    Console.WriteLine("Download timed out.");
}
```

---

## III. Thread Safety

When multiple threads access shared data, you must ensure thread safety to avoid race conditions.

### 1. `lock` Statement

Ensures that only one thread can execute a block of code at a time.

**Example:** Safely incrementing a counter.

```csharp
private readonly object _lockObject = new object();
private int _counter = 0;

public void Increment()
{
    lock (_lockObject)
    {
        _counter++;
    }
}
```

### 2. Concurrent Collections

Use collections from `System.Collections.Concurrent` which are thread-safe by default.

*   `ConcurrentDictionary<TKey, TValue>`
*   `ConcurrentQueue<T>`
*   `ConcurrentBag<T>`

**Example:** Using a thread-safe dictionary.

```csharp
var cache = new ConcurrentDictionary<int, string>();

// Safe to call from multiple threads
cache.TryAdd(1, "Value");
```

### 3. `Interlocked`

Provides atomic operations for variables that are shared by multiple threads. Faster than a full lock for simple operations.

**Example:** Atomic increment.

```csharp
private int _usageCount = 0;

public void AddUsage()
{
    Interlocked.Increment(ref _usageCount);
}
```

---

## IV. Common Pitfalls

### 1. Fire-and-Forget (`void` async)

Avoid `async void` methods (except event handlers). Exceptions thrown in `async void` methods cannot be caught by the caller and will crash the process.

### 2. Capturing Context

In UI apps (WPF, WinForms), `await` tries to return to the original context (UI thread). In libraries, use `ConfigureAwait(false)` to avoid this overhead and potential deadlocks.

```csharp
// In library code
await DoWorkAsync().ConfigureAwait(false);
```

### 3. Over-Parallelization

Creating too many threads can degrade performance due to context switching overhead. Let the ThreadPool manage thread creation.

# Part 7: Concurrency and Event Loops

## 16. Event Loop vs. Threads

### Node.js: Event Loop Model

**Single-threaded JavaScript execution** + **multi-threaded I/O operations**

**Architecture:**
```
┌─────────────────────────┐
│  JavaScript Thread      │  ← Single thread executing JS
│  (V8 Engine)            │
└────────┬────────────────┘
         │
    ┌────▼─────────────────────────┐
    │  libuv Event Loop             │
    │  ┌─────────────────────────┐  │
    │  │  Event Queue            │  │
    │  └─────────────────────────┘  │
    │  ┌─────────────────────────┐  │
    │  │  Thread Pool (4 threads)│  │  ← For blocking ops
    │  │  - File I/O             │  │
    │  │  - DNS lookup           │  │
    │  │  - Crypto               │  │
    │  └─────────────────────────┘  │
    └────────────────────────────────┘
```

**Event loop phases:**
```
   ┌───────────────────────────┐
┌─>│           timers          │  setTimeout, setInterval callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │     pending callbacks     │  I/O callbacks deferred to next loop
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │       idle, prepare       │  Internal use only
│  └─────────────┬─────────────┘      
│  ┌─────────────▼─────────────┐
│  │           poll            │  Retrieve new I/O events, execute callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │           check           │  setImmediate() callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
└──┤      close callbacks      │  socket.on('close', ...)
   └───────────────────────────┘
```

**Example:**
```javascript
const fs = require('fs');

console.log('1: Start');

// Async file read - delegated to thread pool
fs.readFile('file.txt', (err, data) => {
    console.log('4: File read complete');
});

// Async timer
setTimeout(() => {
    console.log('5: Timer callback');
}, 0);

// Synchronous operation - blocks event loop
for (let i = 0; i < 1000000000; i++) {
    // Blocking!
}

console.log('2: After loop');

setImmediate(() => {
    console.log('6: Immediate callback');
});

console.log('3: End');

// Output order:
// 1: Start
// 2: After loop  (after blocking loop completes)
// 3: End
// 5: Timer callback
// 6: Immediate callback
// 4: File read complete
```

**Non-blocking I/O:**
```javascript
const http = require('http');

const server = http.createServer((req, res) => {
    // This doesn't block the event loop
    // While waiting for DB, event loop handles other requests
    db.query('SELECT * FROM users', (err, results) => {
        res.end(JSON.stringify(results));
    });
});

server.listen(3000);

// Can handle thousands of concurrent connections
// with single thread!
```

**Blocking the event loop (bad):**
```javascript
// BAD: Blocks event loop
app.get('/compute', (req, res) => {
    let result = 0;
    for (let i = 0; i < 1e9; i++) {
        result += i;  // CPU-bound work blocks other requests
    }
    res.send({ result });
});

// BETTER: Use worker threads for CPU-bound work
const { Worker } = require('worker_threads');

app.get('/compute', (req, res) => {
    const worker = new Worker('./compute-worker.js');
    worker.on('message', (result) => {
        res.send({ result });
    });
});
```

### Python: asyncio (Event Loop)

**Python added async/await** (Python 3.5+) with the `asyncio` library.

**Single-threaded cooperative multitasking:**

```python
import asyncio

async def fetch_data(id):
    print(f"Fetching {id}")
    await asyncio.sleep(1)  # Simulates async I/O
    print(f"Done {id}")
    return f"Data {id}"

async def main():
    # Run concurrently (not parallel - single thread)
    results = await asyncio.gather(
        fetch_data(1),
        fetch_data(2),
        fetch_data(3)
    )
    print(results)

# Run event loop
asyncio.run(main())

# Output:
# Fetching 1
# Fetching 2
# Fetching 3
# (1 second pause - concurrent waiting)
# Done 1
# Done 2
# Done 3
# ['Data 1', 'Data 2', 'Data 3']
```

**asyncio event loop:**
```python
import asyncio

async def task1():
    print("Task 1 start")
    await asyncio.sleep(0.5)
    print("Task 1 end")

async def task2():
    print("Task 2 start")
    await asyncio.sleep(0.3)
    print("Task 2 end")

async def main():
    # Create tasks - they run concurrently
    await asyncio.gather(task1(), task2())

asyncio.run(main())

# Output:
# Task 1 start
# Task 2 start
# Task 2 end (after 0.3s)
# Task 1 end (after 0.5s)
```

**Blocking vs. non-blocking:**
```python
import asyncio
import time
import aiohttp

# BAD: Synchronous (blocking) code in async function
async def fetch_blocking(url):
    # time.sleep blocks the entire event loop!
    time.sleep(2)  # DON'T DO THIS
    return "data"

# GOOD: Async I/O
async def fetch_async(url):
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.text()

# GOOD: CPU-bound work in executor
async def compute_heavy():
    loop = asyncio.get_event_loop()
    # Run in thread pool
    result = await loop.run_in_executor(None, expensive_computation)
    return result
```

**When to use asyncio:**
- ✓ I/O-bound workloads (network requests, file I/O)
- ✓ High concurrency with many connections
- ✗ CPU-bound workloads (use multiprocessing instead)

### Java: Native Threading Model

**Java uses OS threads** - true parallel execution (unlike Node.js and asyncio).

**Thread example:**
```java
public class ThreadExample {
    public static void main(String[] args) {
        // Create threads
        Thread thread1 = new Thread(() -> {
            for (int i = 0; i < 5; i++) {
                System.out.println("Thread 1: " + i);
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {}
            }
        });
        
        Thread thread2 = new Thread(() -> {
            for (int i = 0; i < 5; i++) {
                System.out.println("Thread 2: " + i);
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {}
            }
        });
        
        thread1.start();
        thread2.start();
        
        // Output interleaved - truly parallel execution
    }
}
```

**Thread pools (better for production):**
```java
import java.util.concurrent.*;

public class ThreadPoolExample {
    public static void main(String[] args) {
        // Create thread pool with 4 threads
        ExecutorService executor = Executors.newFixedThreadPool(4);
        
        // Submit tasks
        for (int i = 0; i < 10; i++) {
            final int taskId = i;
            executor.submit(() -> {
                System.out.println("Task " + taskId + 
                    " on thread " + Thread.currentThread().getName());
                // Do work
            });
        }
        
        executor.shutdown();
    }
}
```

**CompletableFuture (async/await-like):**
```java
import java.util.concurrent.CompletableFuture;

public class AsyncExample {
    public static void main(String[] args) {
        CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
            sleep(1000);
            return "Result 1";
        });
        
        CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
            sleep(500);
            return "Result 2";
        });
        
        // Combine results
        CompletableFuture<String> combined = future1.thenCombine(future2, 
            (result1, result2) -> result1 + " " + result2
        );
        
        combined.thenAccept(System.out::println);
        
        // Wait for completion
        combined.join();
    }
    
    static void sleep(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) {}
    }
}
```

**Synchronization:**
```java
public class Counter {
    private int count = 0;
    
    // Synchronized method - thread-safe
    public synchronized void increment() {
        count++;
    }
    
    public synchronized int getCount() {
        return count;
    }
}

// Or use locks
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class CounterWithLock {
    private int count = 0;
    private Lock lock = new ReentrantLock();
    
    public void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock();
        }
    }
}
```

**Virtual Threads (Java 19+, Project Loom):**
```java
// Lightweight threads - millions possible!
public class VirtualThreadExample {
    public static void main(String[] args) throws Exception {
        // Create virtual thread
        Thread vThread = Thread.ofVirtual().start(() -> {
            System.out.println("Virtual thread: " + Thread.currentThread());
        });
        
        vThread.join();
        
        // Execute many virtual threads
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < 100000; i++) {
                executor.submit(() -> {
                    // Can handle millions of concurrent tasks!
                    Thread.sleep(Duration.ofSeconds(1));
                    return i;
                });
            }
        }
    }
}
```

### I/O-Bound vs. CPU-Bound Workloads

**I/O-Bound: Waiting for external resources**
- Network requests
- Database queries
- File system operations
- User input

**Best approach:**
- **Node.js / asyncio**: Excellent! Event loop handles many concurrent I/O operations efficiently
- **Java threads**: Good, but overhead per thread (pre-virtual threads)

**Example (I/O-bound):**
```javascript
// Node.js - handles 10,000 concurrent requests easily
app.get('/users/:id', async (req, res) => {
    const user = await db.query('SELECT * FROM users WHERE id = ?', req.params.id);
    res.json(user);
});
```

**CPU-Bound: Heavy computation**
- Image processing
- Video encoding
- Cryptography
- Scientific computing
- Complex algorithms

**Best approach:**
- **Node.js**: Bad for single-threaded execution, use worker threads or separate processes
- **Python asyncio**: Bad, use multiprocessing
- **Java threads**: Excellent! True parallelism across CPU cores

**Example (CPU-bound):**
```java
// Java - true parallel execution
ExecutorService executor = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors()
);

List<Future<Result>> futures = new ArrayList<>();
for (Task task : tasks) {
    futures.add(executor.submit(() -> processTask(task)));  // Runs on separate core
}

// Collect results
for (Future<Result> future : futures) {
    results.add(future.get());
}
```

### Comparison Table

| Feature | Node.js | Python asyncio | Java Threads |
|---------|---------|----------------|--------------|
| **Model** | Event loop | Event loop | OS threads |
| **Execution** | Single-threaded JS | Single-threaded | Multi-threaded |
| **True Parallelism** | ✗ (worker threads for workaround) | ✗ (multiprocessing for workaround) | ✓ Yes |
| **Concurrency** | High (I/O) | High (I/O) | Medium (pre-virtual threads) |
| **I/O-Bound** | ✓✓✓ Excellent | ✓✓✓ Excellent | ✓✓ Good |
| **CPU-Bound** | ✗ Poor (unless workers) | ✗ Poor (unless multiprocessing) | ✓✓✓ Excellent |
| **Memory per Task** | Low | Low | High (1MB+ per thread) |
| **Max Concurrent Tasks** | 10,000+ | 10,000+ | 1,000s (pre-virtual threads), millions (virtual threads) |
| **Learning Curve** | Medium | Medium | Medium-High (synchronization) |
| **Use Case** | Web servers, APIs, microservices | Web scraping, network services | Heavy computation, enterprise apps |

### When to Use What

**Use Node.js / asyncio when:**
- Building I/O-heavy applications (web servers, REST APIs, microservices)
- High concurrent connections needed
- Response time is more important than throughput
- Want lightweight concurrency without thread overhead

**Use Java threads when:**
- CPU-intensive workloads that benefit from parallelism
- Need true parallel execution across cores
- Building enterprise applications with complex concurrent logic
- Throughput is critical

**Mixed workloads:**
- Node.js: Use worker threads for CPU-bound tasks
- Python: Use multiprocessing for CPU-bound tasks
- Java: Use CompletableFuture or virtual threads for I/O + thread pools for CPU


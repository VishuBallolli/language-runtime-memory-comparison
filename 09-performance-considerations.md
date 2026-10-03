# Part 9: Performance and Memory Lifetime

## 18. Performance: The Nuanced Reality

### Don't Ask "Which Language is Fastest?"

**The right question:** "Which language is best for *this specific workload*?"

Performance depends on:
1. **Workload type** (I/O-bound, CPU-bound, mixed)
2. **Concurrency requirements** (10 requests/sec vs 10,000)
3. **Data size and structure**
4. **Memory constraints**
5. **Latency vs throughput priorities**
6. **JIT warmup tolerance**

### Workload Types

#### I/O-Bound: Network and Database Operations

**Characteristics:**
- Waiting for external resources (network, disk, database)
- CPU mostly idle
- High concurrency possible

**Performance ranking:**
1. **Node.js / Python asyncio**: Excellent (event loop handles concurrent I/O efficiently)
2. **Java (WebFlux/Reactor)**: Excellent (reactive streams)
3. **Java (traditional servlets)**: Good (limited by thread pool size)
4. **Python (Flask/Django sync)**: Moderate (limited by processes/threads)

**Example benchmark (simplified):**
```
10,000 concurrent HTTP requests to external API:

Node.js:           ~2 seconds  (handles all concurrently)
Python asyncio:    ~2 seconds  (handles all concurrently)
Java WebFlux:      ~2 seconds  (handles all concurrently)
Java servlets:     ~4 seconds  (200 threads, batched)
Python Flask:      ~20 seconds (20 processes, sequential batches)
```

#### CPU-Bound: Heavy Computation

**Characteristics:**
- Intensive calculations
- Little waiting
- Benefits from parallel execution

**Performance ranking:**
1. **Java (multi-threaded)**: Excellent (true parallelism + JIT optimization)
2. **Node.js (worker threads)**: Good (but less mature than Java)
3. **Python (multiprocessing)**: Moderate (GIL prevents thread parallelism)
4. **JavaScript (single-threaded)**: Poor (blocks event loop)
5. **Python (single-threaded)**: Poor (GIL + no JIT in CPython)

**Example benchmark (image processing):**
```
Process 1000 images with CPU-intensive filters:

Java (multi-threaded):     ~5 seconds   (8 cores fully utilized)
Node.js (worker threads):  ~7 seconds   (8 workers)
Python (multiprocessing):  ~15 seconds  (8 processes, IPC overhead)
JavaScript (main thread):  ~60 seconds  (single core)
Python (main thread):      ~120 seconds (single core, no JIT)
```

#### Memory-Intensive: Large Data Structures

**Characteristics:**
- Large collections
- Complex object graphs
- Garbage collection pressure

**Performance considerations:**

**Java:**
- ✓ Efficient memory representation
- ✓ Configurable GC (G1, ZGC)
- ✓ Primitive arrays (no boxing overhead)
- ✗ Higher base memory overhead

**JavaScript:**
- ✓ Efficient for moderate data sizes
- ✓ Fast generational GC
- ✗ Everything is boxed (overhead)
- ✗ GC pauses can be noticeable

**Python:**
- ✓ Simple memory model
- ✗ High per-object overhead (~50 bytes)
- ✗ Reference counting overhead
- ✗ Dictionary overhead for object attributes

**Example memory usage (1 million objects):**
```
class Point { x: number; y: number }  // or equivalent

Java:    ~24 MB  (8 bytes * 2 fields * 1M objects + overhead)
Node.js: ~40 MB  (objects + hidden classes)
Python:  ~120 MB (dict per object + refcount overhead)
```

### JIT Warmup vs Interpreted Performance

**Cold start (first run):**
```
Java:       Slow (JVM startup + class loading + initial interpretation)
Node.js:    Fast (quick startup, immediate interpretation)
Python:     Very fast (quick startup, immediate interpretation)
```

**Warmed up (after many iterations):**
```
Java:       Very fast (C2 JIT optimizations)
Node.js:    Very fast (TurboFan optimizations)
Python:     Slow (no JIT in CPython)
```

**Example (computing Fibonacci 40):**
```
                First run    After 10,000 runs
Java:           50ms         0.5ms  (100x faster!)
Node.js:        20ms         2ms    (10x faster)
Python:         800ms        800ms  (no improvement)
```

**When JIT matters:**
- Long-running services (✓ JIT warmup pays off)
- Short scripts (✗ warmup overhead wasted)
- Microservices with low traffic (✗ may never warm up)

### Architecture Matters More Than Language

**Bad architecture in any language:**
```javascript
// Node.js - BLOCKS EVENT LOOP (BAD)
app.get('/users', async (req, res) => {
    const users = await db.getAllUsers();  // 1M users
    
    // CPU-intensive sorting on main thread
    users.sort((a, b) => complexComparison(a, b));
    
    res.json(users);
});
```

**Good architecture:**
```javascript
// Node.js - proper architecture
app.get('/users', async (req, res) => {
    // 1. Pagination (don't load 1M records)
    const page = parseInt(req.query.page) || 1;
    const limit = 100;
    
    // 2. Database sorting (let DB do it)
    const users = await db.getUsers({
        page,
        limit,
        orderBy: 'name'
    });
    
    res.json(users);
});
```

### Real-World Performance Example

**Scenario:** REST API serving 10,000 requests/second

**Node.js:**
```javascript
// Strengths:
// - High concurrency with low memory
// - Fast response times
// - Good for JSON APIs

// Weaknesses:
// - CPU-bound operations block event loop
// - Memory grows with object creation rate

const express = require('express');
const app = express();

app.get('/api/data', async (req, res) => {
    const data = await cache.get(req.query.key);  // I/O
    res.json(data);
});

// Can handle 10k req/s easily with ~500MB RAM
```

**Python (asyncio):**
```python
# Strengths:
# - Clean async/await syntax
# - Good for I/O-bound workloads
# - Lower memory than sync Python

# Weaknesses:
# - Slower than Node.js for JSON parsing
# - Ecosystem less mature than Node.js

from aiohttp import web

async def get_data(request):
    key = request.query.get('key')
    data = await cache.get(key)
    return web.json_response(data)

app = web.Application()
app.router.add_get('/api/data', get_data)

# Can handle ~8k req/s with ~600MB RAM
```

**Java (Spring WebFlux):**
```java
// Strengths:
// - Very high throughput after warmup
// - Efficient memory usage
// - Excellent for complex business logic

// Weaknesses:
// - Higher latency for first requests (warmup)
// - More complex code

@RestController
public class DataController {
    
    @GetMapping("/api/data")
    public Mono<Data> getData(@RequestParam String key) {
        return cache.get(key);
    }
}

// Can handle 15k req/s after warmup with ~1GB RAM
// But first 100 requests slower (JIT warmup)
```

### Benchmark Considerations

**Microbenchmarks lie:**
```javascript
// Microbenchmark: "How fast is array iteration?"
console.time('iterate');
for (let i = 0; i < 1000000; i++) {
    arr[i];
}
console.timeEnd('iterate');

// This measures:
// - JIT optimization
// - CPU cache
// - Array access
//
// This does NOT measure real application performance!
```

**Realistic benchmarks:**
- Full request/response cycle
- Realistic data sizes
- Concurrent load
- Cold and warm states
- Mixed workloads
- Error handling

**Example tool outputs:**
```bash
# wrk - HTTP benchmarking tool
wrk -t12 -c400 -d30s http://localhost:3000/api/users

# Results show:
# - Requests/sec (throughput)
# - Latency distribution (p50, p99)
# - Error rate

Running 30s test @ http://localhost:3000/api/users
  12 threads and 400 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    12.43ms    8.21ms  89.32ms   71.23%
    Req/Sec     2.79k   312.12     4.12k    68.24%
  1002341 requests in 30.03s, 201.45MB read
Requests/sec:  33384.71
Transfer/sec:      6.71MB
```

---

## 19. Memory Lifetime

### When Does a Variable Exist?

**JavaScript:**
```javascript
function example() {
    // Variable 'x' exists from declaration to end of scope
    let x = 10;
    
    if (true) {
        let y = 20;  // 'y' exists only in this block
        console.log(x, y);  // Both accessible
    }
    
    // console.log(y);  // Error: y not defined
    
    console.log(x);  // x still accessible
}  // x destroyed here (scope ends)

// x and y are both gone
```

**Stack lifetime:**
```
example() called
    ↓
Stack frame created
    ┌──────────┐
    │ x: 10    │  ← x exists here
    └──────────┘
    ↓
Inner block
    ┌──────────┐
    │ x: 10    │
    │ y: 20    │  ← y exists here
    └──────────┘
    ↓
Block ends
    ┌──────────┐
    │ x: 10    │  ← y destroyed
    └──────────┘
    ↓
Function ends
    (empty)      ← x destroyed
```

**Python:**
```python
def example():
    x = 10  # x references int(10)
    
    if True:
        y = 20  # y references int(20)
        print(x, y)
    
    # y still accessible! Python has function scope, not block scope
    print(y)  # 20 - works!
    
# x and y references destroyed here
# int objects may be destroyed (if refcount = 0)
```

**Java:**
```java
void example() {
    int x = 10;  // x on stack
    
    if (true) {
        int y = 20;  // y on stack
        System.out.println(x + y);
    }
    
    // System.out.println(y);  // Error: y out of scope
    
    System.out.println(x);  // x still accessible
}  // x destroyed here
```

### When Does an Object Remain Reachable?

An object is **reachable** if there's a path from GC roots to it.

**Example 1: Immediate unreachability**
```javascript
function create() {
    let obj = { data: "important" };
    // obj goes out of scope
}

create();
// Object immediately unreachable → eligible for GC
```

**Example 2: Closure keeps object alive**
```javascript
function create() {
    let obj = { data: "important" };
    
    return function() {
        console.log(obj.data);  // Closes over obj
    };
}

const fn = create();
// obj still reachable through fn's closure
// Object NOT eligible for GC

fn = null;
// Now obj is unreachable → eligible for GC
```

**Example 3: Global reference**
```javascript
let globalCache = [];

function addToCache(item) {
    globalCache.push(item);
}

addToCache({ data: "x" });
addToCache({ data: "y" });

// Objects in globalCache remain reachable
// Never eligible for GC (memory leak potential)
```

**Example 4: Circular references**
```javascript
function createCircular() {
    let a = {};
    let b = {};
    
    a.ref = b;
    b.ref = a;  // Circular reference
    
    // Both go out of scope
}

createCircular();
// Even with circular references, both unreachable
// Modern GC handles this → eligible for collection
```

### When Does an Object Become Eligible for Collection?

**JavaScript/Java (tracing GC):**
Object eligible when **unreachable from GC roots**

```javascript
let obj = { data: "x" };  // Reachable (stack reference)

obj = null;  // No longer reachable → eligible

// GC will collect it during next GC cycle (timing non-deterministic)
```

**Python (reference counting):**
Object destroyed when **reference count reaches zero**

```python
obj = { 'data': 'x' }  # refcount = 1

obj2 = obj             # refcount = 2

del obj                # refcount = 1
del obj2               # refcount = 0 → IMMEDIATELY destroyed

# With cycles:
a = {}
b = {}
a['ref'] = b
b['ref'] = a

del a  # refcount still > 0 (b references it)
del b  # refcount still > 0 (a references it)

# Cyclic GC will eventually collect both
```

**Visual timeline:**
```
Time →

JavaScript/Java:
Object created ──[reachable]──→ Last ref removed ──[eligible]──→ GC cycle ──→ Collected
                 (immediate)                        (immediate)    (0-∞ ms)      (immediate)

Python (refcounting):
Object created ──[reachable]──→ Refcount = 0 ──→ Collected
                 (immediate)     (immediate)      (immediate)

Python (with cycles):
Object created ──[reachable]──→ Refs removed ──[eligible]──→ Cyclic GC ──→ Collected
                 (immediate)     (immediate)     (immediate)   (periodic)     (immediate)
```

### Lifetime Example: HTTP Request

```javascript
app.get('/process', (req, res) => {
    // Request objects created - reachable from stack
    const data = req.body;
    const result = processData(data);
    
    res.json(result);
    // Handler ends
    // req, res, data, result all unreachable
    // Eligible for GC
});

// Timeline:
// 1. Request arrives → objects created
// 2. Handler executes → objects on stack (reachable)
// 3. Handler returns → stack popped (unreachable)
// 4. Next GC cycle → objects collected
// 5. Memory reclaimed
```

### Memory Leak: Object Stays Reachable Forever

```javascript
const globalListeners = [];

function setupListener() {
    const largeData = new Array(1000000).fill("x");
    
    const listener = () => {
        console.log(largeData.length);  // Closes over largeData
    };
    
    globalListeners.push(listener);
    // largeData stays reachable forever through listener
    // MEMORY LEAK
}

setupListener();
setupListener();
setupListener();

// All largeData arrays still in memory!
```

**Fix:**
```javascript
const globalListeners = new Map();

function setupListener(id) {
    const largeData = new Array(1000000).fill("x");
    
    const listener = () => {
        console.log(largeData.length);
    };
    
    globalListeners.set(id, listener);
}

function removeListener(id) {
    globalListeners.delete(id);
    // Now largeData can be collected
}
```


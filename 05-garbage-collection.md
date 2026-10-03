# Part 5: Garbage Collection

## 11. Garbage Collection Basics

### Why Does Garbage Collection Exist?

**Manual memory management problems (C/C++):**
```c
// C code
int* data = malloc(sizeof(int) * 1000);
// ... use data ...
free(data);  // Must manually free

// Problems:
// 1. Forgetting to free → memory leak
// 2. Freeing too early → dangling pointer, crashes
// 3. Freeing twice → corruption
// 4. Complex ownership → who should free?
```

**Garbage collection benefits:**
- Automatic memory management
- No manual `free()` or `delete`
- Eliminates entire classes of bugs:
  - Memory leaks (mostly)
  - Dangling pointers
  - Double frees
  - Use-after-free

**The trade-off:**
- Performance overhead (GC pauses)
- Less control over when memory is released
- Can still have "memory leaks" (reachable but unused objects)

### When Does an Object Become Eligible for Collection?

An object is eligible for garbage collection when it becomes **unreachable** from the program's **GC roots**.

**GC Roots include:**
- Global variables
- Local variables on the call stack
- Static fields
- Active threads
- JNI references (Java)

**Example:**
```javascript
function example() {
    let obj1 = { data: "A" };
    let obj2 = { data: "B" };
    
    obj1.ref = obj2;  // obj1 → obj2
    obj2.ref = obj1;  // obj2 → obj1 (circular reference)
    
    obj1 = null;
    obj2 = null;
    // Both objects now unreachable from stack
    // Eligible for GC (despite circular reference)
}
```

**Reachability:**
```
GC Roots (Stack, Globals)
    ↓
    [Object A] → [Object B] → [Object C]
                      ↓
                 [Object D]

All four objects are reachable.

If reference from A to B is removed:
GC Roots (Stack, Globals)
    ↓
    [Object A]    [Object B] → [Object C]
                      ↓
                 [Object D]

Objects B, C, D are now unreachable → eligible for GC
```

### Why Doesn't `delete` or `del` Mean Immediate Memory Release?

**JavaScript `delete`:**
```javascript
let obj = { x: 1, y: 2 };
delete obj.x;  // Removes property x, doesn't free obj
console.log(obj);  // { y: 2 }

obj = null;  // Now obj is unreachable
// Object will be GC'd eventually (not immediately)
```

**Python `del`:**
```python
class Heavy:
    def __init__(self):
        self.data = [0] * 1000000
    
    def __del__(self):
        print("Heavy object being destroyed")

obj = Heavy()
del obj  # Removes name binding
# If no other references, object may be destroyed
# But timing is not guaranteed!

# Multiple references:
obj1 = Heavy()
obj2 = obj1  # Second reference
del obj1     # Removes first reference
# obj2 still references it → not destroyed yet
del obj2     # Now unreachable → destroyed
```

**Java (no delete):**
```java
MyObject obj = new MyObject();
obj = null;  // Remove reference
// Object eligible for GC, but not freed yet
// GC runs when JVM decides to
```

**Key points:**
1. **`delete`/`del`** removes references, doesn't free memory
2. **GC runs periodically**, not immediately
3. **Memory released** when GC decides to collect
4. **Timing is non-deterministic**

---

## 12. Garbage Collection Comparison

### JavaScript / Node.js: V8's Garbage Collector

**Algorithm: Generational Tracing GC**

**Two generations:**

1. **Young Generation (New Space / Scavenger):**
   - New objects allocated here
   - Fast, frequent collections
   - Uses Scavenging algorithm (copying collector)
   - Most objects die young → efficient

2. **Old Generation (Old Space):**
   - Objects that survive young generation promoted here
   - Slower, less frequent collections
   - Uses Mark-Sweep and Mark-Compact

**Mark-Sweep-Compact process:**
```
1. Mark phase:
   - Start from GC roots
   - Mark all reachable objects

2. Sweep phase:
   - Reclaim unmarked objects

3. Compact phase (optional):
   - Move objects together to reduce fragmentation
```

**Example timeline:**
```javascript
// Many short-lived objects
for (let i = 0; i < 1000000; i++) {
    let temp = { value: i };  // Allocated in young generation
    // temp immediately unreachable after iteration
}
// Young generation GC runs frequently, cleans up quickly

// Long-lived object
const cache = {};
for (let i = 0; i < 1000; i++) {
    cache[i] = { data: i };  // Survives → promoted to old generation
}
```

**Incremental marking:**
- GC work interleaved with application code
- Reduces pause times
- "Stop-the-world" pauses minimized

### CPython: Reference Counting + Cyclic GC

**Two mechanisms:**

#### 1. Reference Counting (Primary)

Every object has a reference count. When count reaches zero, object is immediately destroyed.

```python
import sys

a = []               # refcount = 1
b = a                # refcount = 2
c = a                # refcount = 3

print(sys.getrefcount(a))  # 4 (includes getrefcount's own reference)

del b                # refcount = 2
del c                # refcount = 1
# del a              # refcount = 0 → object destroyed immediately
```

**Advantages:**
- Immediate collection when refcount = 0
- Deterministic timing
- Good for resource management (files, locks)

**Disadvantages:**
- Overhead: every assignment updates refcount
- Cannot handle circular references

#### 2. Cyclic Garbage Collector (Backup)

Handles circular references that reference counting misses:

```python
class Node:
    def __init__(self, value):
        self.value = value
        self.next = None

# Create cycle
a = Node(1)
b = Node(2)
a.next = b
b.next = a  # Circular reference

del a
del b
# Objects still reference each other (refcount > 0)
# Cyclic GC detects and collects them
```

**How cyclic GC works:**
- Periodically scans for cycles
- Uses generational approach (like V8)
- Three generations: 0 (young), 1, 2 (old)

**Triggering cyclic GC:**
```python
import gc

# Manual control
gc.disable()  # Disable automatic cyclic GC
gc.enable()   # Enable it
gc.collect()  # Force collection

# Get stats
print(gc.get_stats())
print(gc.get_count())  # Objects in each generation
```

### Java: JVM Garbage Collectors

Java has multiple GC implementations:

#### 1. Serial GC (`-XX:+UseSerialGC`)
- Single-threaded
- Stop-the-world for both young and old generation
- Good for small applications

#### 2. Parallel GC (`-XX:+UseParallelGC`) - Default for many JVMs
- Multi-threaded young and old generation collection
- Throughput-oriented
- Longer pause times acceptable

#### 3. CMS (Concurrent Mark Sweep) - Deprecated
- Low-pause collector
- Concurrent marking of old generation
- Still stop-the-world for young generation

#### 4. G1 GC (`-XX:+UseG1GC`) - Default since Java 9
- Region-based heap
- Predictable pause times
- Concurrent and parallel
- Good for large heaps

#### 5. ZGC (`-XX:+UseZGC`) and Shenandoah
- Ultra-low latency
- Pause times < 10ms
- Concurrent compaction

**Example GC behavior:**
```java
public class GCExample {
    public static void main(String[] args) {
        // Create many short-lived objects
        for (int i = 0; i < 1000000; i++) {
            String temp = "string" + i;  // Eligible for GC immediately
        }
        
        // Suggest GC (doesn't guarantee it runs)
        System.gc();
        
        // Long-lived objects
        List<String> cache = new ArrayList<>();
        for (int i = 0; i < 1000; i++) {
            cache.add("data" + i);  // Promoted to old generation
        }
    }
}
```

**Monitoring GC:**
```bash
# Run with GC logging
java -Xlog:gc* -Xms512m -Xmx1g MyApp

# Output shows:
# - GC events
# - Pause times
# - Heap usage before/after
```

### Comparison Table

| Feature | JavaScript/V8 | CPython | Java/JVM |
|---------|--------------|---------|----------|
| **Primary Algorithm** | Generational tracing | Reference counting | Generational tracing |
| **Backup Mechanism** | - | Cyclic GC for cycles | - |
| **Generations** | 2 (young, old) | 3 (0, 1, 2) | 2+ (depends on GC) |
| **Deterministic Timing** | No | Mostly (refcount) | No |
| **Pause Times** | Short (incremental) | Very short (refcount) + periodic | Varies by GC |
| **Handles Cycles** | Yes | Yes (cyclic GC) | Yes |
| **Concurrent Collection** | Yes (incremental) | Limited | Yes (G1, ZGC) |
| **Manual Control** | Limited | Yes (`gc` module) | Yes (flags, `System.gc()`) |

---

## 13. Memory Leaks Despite Garbage Collection

GC doesn't prevent all memory leaks. **Memory leaks** = objects that are **reachable** but **no longer needed**.

### Common Causes of Memory Leaks

#### 1. Global Variables and Long-Lived References

**JavaScript:**
```javascript
// Leak: global accumulates forever
let cache = [];
function addToCache(item) {
    cache.push(item);  // Never removed!
}

// Better: add expiration or limit
let cache = [];
function addToCache(item) {
    cache.push(item);
    if (cache.length > 1000) {
        cache.shift();  // Remove oldest
    }
}
```

**Python:**
```python
# Leak: class variable accumulates
class DataProcessor:
    all_results = []  # Class variable - never cleared
    
    def process(self, data):
        result = expensive_computation(data)
        DataProcessor.all_results.append(result)
```

#### 2. Event Listeners and Callbacks

**JavaScript:**
```javascript
// Leak: listeners not removed
function setupComponent() {
    const data = new Array(1000000).fill("x");  // Large data
    
    document.getElementById("btn").addEventListener("click", function() {
        console.log(data.length);  // Closure keeps data alive
    });
    
    // Component removed from DOM, but listener remains!
}

// Fix: remove listener
function setupComponentFixed() {
    const data = new Array(1000000).fill("x");
    const button = document.getElementById("btn");
    
    const handler = function() {
        console.log(data.length);
    };
    
    button.addEventListener("click", handler);
    
    // Later, when cleaning up:
    // button.removeEventListener("click", handler);
}
```

#### 3. Detached DOM Nodes

**JavaScript:**
```javascript
// Leak: DOM node removed but referenced
let detachedNode;

function createNode() {
    const div = document.createElement("div");
    div.innerHTML = "<div>".repeat(1000);
    document.body.appendChild(div);
    detachedNode = div;  // Keep reference
}

function removeNode() {
    document.body.removeChild(detachedNode);
    // detachedNode still references the removed node!
    // Entire subtree kept in memory
}

// Fix: null the reference
function removeNodeFixed() {
    document.body.removeChild(detachedNode);
    detachedNode = null;  // Allow GC
}
```

#### 4. Timers and Intervals

**JavaScript:**
```javascript
// Leak: interval never cleared
function startPolling() {
    const data = fetchData();
    
    setInterval(() => {
        console.log(data);  // Closure keeps data alive forever
    }, 1000);
}

// Fix: clear interval and avoid unnecessary closures
function startPollingFixed() {
    const data = fetchData();
    
    const intervalId = setInterval(() => {
        console.log(data);
    }, 1000);
    
    // Clear when done
    setTimeout(() => {
        clearInterval(intervalId);
    }, 60000);
}
```

#### 5. Caches Without Eviction

**Python:**
```python
# Leak: cache grows unbounded
_cache = {}

def get_data(key):
    if key not in _cache:
        _cache[key] = expensive_operation(key)
    return _cache[key]

# Better: use LRU cache with size limit
from functools import lru_cache

@lru_cache(maxsize=100)  # Keeps only 100 most recent
def get_data(key):
    return expensive_operation(key)
```

**Java:**
```java
// Leak: HashMap grows forever
public class DataCache {
    private static Map<String, byte[]> cache = new HashMap<>();
    
    public static void cache(String key, byte[] data) {
        cache.put(key, data);  // Never removed!
    }
}

// Better: use WeakHashMap or cache with eviction
import java.util.WeakHashMap;

public class DataCacheFix {
    // Keys can be GC'd if no other references
    private static Map<String, byte[]> cache = new WeakHashMap<>();
    
    // Or use a proper cache library with LRU eviction
    // e.g., Guava's Cache, Caffeine
}
```

#### 6. Closures Capturing Large Data

**JavaScript:**
```javascript
function processData() {
    const hugeArray = new Array(1000000).fill("x");
    
    // Leak: unnecessary capture
    setTimeout(() => {
        console.log("Done");  // Doesn't use hugeArray
        // But hugeArray is still captured and kept alive
    }, 1000);
}

// Fix: don't capture unnecessary variables
function processDataFixed() {
    const hugeArray = new Array(1000000).fill("x");
    doSomething(hugeArray);
    // hugeArray goes out of scope
    
    setTimeout(() => {
        console.log("Done");  // No capture
    }, 1000);
}
```

### Detecting Memory Leaks

**JavaScript/Node.js:**
```javascript
// Browser: Chrome DevTools Memory Profiler
// 1. Take heap snapshot
// 2. Perform action
// 3. Take another snapshot
// 4. Compare to find retained objects

// Node.js: --inspect flag
// node --inspect --expose-gc app.js

// Manual GC and heap check
if (global.gc) {
    global.gc();
    const used = process.memoryUsage().heapUsed / 1024 / 1024;
    console.log(`Memory: ${used.toFixed(2)} MB`);
}
```

**Python:**
```python
import gc
import sys

# Find objects
gc.collect()  # Force collection
print(f"Garbage: {len(gc.garbage)}")

# Find references to object
obj = MyClass()
print(sys.getrefcount(obj))  # How many references?

# Find what references an object
import objgraph
objgraph.show_refs([obj], filename='refs.png')
```

**Java:**
```bash
# Heap dump
jmap -dump:live,format=b,file=heap.bin <pid>

# Analyze with Eclipse Memory Analyzer (MAT)
# or VisualVM
```

### Prevention Strategies

1. **Nullify references** when done
2. **Remove event listeners** explicitly
3. **Use weak references** when appropriate
4. **Implement cache eviction** policies
5. **Profile regularly** during development
6. **Avoid global state** when possible
7. **Clear timers/intervals** when done


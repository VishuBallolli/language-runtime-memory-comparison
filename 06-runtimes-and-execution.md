# Part 6: Runtimes and Execution Models

## 14. Runtime Comparison

### JavaScript: V8 Engine

**V8** is Google's open-source JavaScript engine (used in Chrome and Node.js).

**Components:**
1. **Parser**: Converts source code to AST (Abstract Syntax Tree)
2. **Ignition**: Bytecode interpreter
3. **TurboFan**: Optimizing JIT compiler
4. **Garbage Collector**: Memory management
5. **Built-in functions**: Core JS APIs (Array, Object, Math, etc.)

**What V8 does:**
- Parses JavaScript code
- Generates bytecode
- Executes bytecode in interpreter
- Profiles code execution
- Compiles hot code paths to optimized machine code
- Manages memory (heap, garbage collection)

**V8 execution flow:**
```
JavaScript Source
    ↓
Parser → AST
    ↓
Ignition (Interpreter) → Bytecode
    ↓                      ↓
Execution             Profiling (detects hot functions)
    ↓                      ↓
TurboFan (Optimizer) → Optimized Machine Code
    ↓
Fast Execution
```

### Node.js: V8 + Node.js Runtime + libuv

**Node.js = V8 + Additional Components**

**Components:**
1. **V8**: JavaScript execution
2. **libuv**: Cross-platform asynchronous I/O library
3. **Node.js bindings**: C++ bindings connecting JS to system APIs
4. **Node.js API**: fs, http, crypto, etc.

**What Node.js adds:**
- File system access (`fs`)
- Networking (`http`, `net`, `dgram`)
- Process management (`child_process`, `cluster`)
- Operating system utilities (`os`, `path`)
- Event loop (via libuv)
- Streams and buffers

**Architecture:**
```
┌─────────────────────────────┐
│   JavaScript Application    │
├─────────────────────────────┤
│     Node.js API (JS)        │
├─────────────────────────────┤
│   Node.js Bindings (C++)    │
├─────────────┬───────────────┤
│     V8      │    libuv      │
│  (JS Engine)│  (Event Loop) │
├─────────────┴───────────────┤
│    Operating System         │
└─────────────────────────────┘
```

**libuv responsibilities:**
- Event loop
- Thread pool for blocking operations (file I/O, DNS, etc.)
- Network I/O (sockets)
- Timers
- Signals
- Child processes

### Python: CPython Runtime

**CPython** is the reference implementation of Python (written in C).

**Components:**
1. **Parser**: Converts source to AST
2. **Compiler**: Converts AST to bytecode
3. **Python Virtual Machine (PVM)**: Executes bytecode
4. **Memory Manager**: Reference counting + cyclic GC
5. **Standard Library**: Written in Python and C
6. **C API**: For extending Python with C modules

**What CPython does:**
- Parses Python code
- Compiles to bytecode (.pyc files)
- Executes bytecode in the virtual machine
- Manages memory
- Provides standard library

**CPython execution flow:**
```
Python Source (.py)
    ↓
Parser → AST
    ↓
Compiler → Bytecode (.pyc cached)
    ↓
Python Virtual Machine (PVM)
    ↓
Execution (interpreter loop)
```

**Bytecode example:**
```python
def add(a, b):
    return a + b

import dis
dis.dis(add)
```

Output:
```
  2           0 LOAD_FAST                0 (a)
              2 LOAD_FAST                1 (b)
              4 BINARY_ADD
              6 RETURN_VALUE
```

**Key characteristics:**
- Interpreted bytecode (not machine code)
- Global Interpreter Lock (GIL): Only one thread executes Python bytecode at a time
- Dynamic typing: Type checking at runtime
- Everything is an object

### Java: Java Virtual Machine (JVM)

**JVM** executes Java bytecode (can also run Kotlin, Scala, Groovy, etc.).

**Components:**
1. **Class Loader**: Loads .class files
2. **Bytecode Verifier**: Ensures bytecode is valid and safe
3. **Interpreter**: Executes bytecode
4. **JIT Compiler**: Compiles hot bytecode to machine code
5. **Garbage Collector**: Memory management
6. **Runtime Data Areas**: Heap, stack, method area, etc.

**What the JVM does:**
- Loads compiled bytecode (.class files)
- Verifies bytecode security
- Executes bytecode (interpreted initially)
- Profiles execution
- Compiles hot code to native machine code (JIT)
- Manages memory

**Java execution flow:**
```
Java Source (.java)
    ↓
javac (Compiler) → Bytecode (.class)
    ↓
JVM Class Loader
    ↓
Bytecode Verifier
    ↓
Interpreter → Execution
    ↓           ↓
Profiling   Detect Hot Spots
    ↓
JIT Compiler (C1/C2) → Native Machine Code
    ↓
Fast Execution
```

**JIT compilation tiers:**
1. **Interpreter**: Initial execution, fast startup
2. **C1 (Client Compiler)**: Quick compilation, some optimizations
3. **C2 (Server Compiler)**: Aggressive optimizations, slower compilation

**Bytecode example:**
```java
public class Example {
    public int add(int a, int b) {
        return a + b;
    }
}
```

Bytecode (via `javap -c Example.class`):
```
public int add(int, int);
    Code:
       0: iload_1        // Load first parameter
       1: iload_2        // Load second parameter
       2: iadd           // Add integers
       3: ireturn        // Return result
```

### Comparison Table

| Feature | JavaScript (V8) | Node.js | Python (CPython) | Java (JVM) |
|---------|----------------|---------|------------------|------------|
| **Core Engine** | V8 | V8 + libuv | CPython VM | JVM |
| **Execution** | Bytecode → JIT | Bytecode → JIT | Bytecode (interpreted) | Bytecode → JIT |
| **Compilation** | Runtime (JIT) | Runtime (JIT) | To bytecode | To bytecode (javac) |
| **Optimization** | TurboFan (JIT) | TurboFan (JIT) | Limited | C1/C2 JIT |
| **Type Checking** | Runtime | Runtime | Runtime | Compile time + runtime |
| **Threading Model** | Single-threaded + event loop | Single-threaded + event loop | GIL (limited threading) | Native threads |
| **I/O Model** | Event loop (browser) | Event loop (libuv) | Blocking (default) or asyncio | Blocking (default) or NIO |
| **Startup Time** | Fast | Fast | Very fast | Slow (JVM warmup) |
| **Peak Performance** | Very fast | Very fast | Slower | Very fast (after warmup) |
| **Memory Management** | GC (generational) | GC (generational) | Refcount + cyclic GC | GC (varies) |

---

## 15. Compilation / Interpretation / JIT

### The Spectrum of Execution

The simple "compiled vs interpreted" dichotomy is outdated. Modern languages use a mix:

```
Pure Compilation         Mixed               Pure Interpretation
(C, C++, Rust)     (Java, C#, JavaScript)    (early BASIC, sh)
      ↓                    ↓                        ↓
Source → Machine     Source → Bytecode          Source → Execute
  Code directly      → JIT → Machine Code       line by line
```

### JavaScript / V8: Multi-Tier JIT

**Modern V8 pipeline:**

```
JavaScript Source
    ↓
Parse → AST
    ↓
Ignition (Interpreter)
    ↓
Bytecode Execution (with profiling)
    ↓
[Hot code detected]
    ↓
TurboFan (Optimizing Compiler)
    ↓
Optimized Machine Code
    ↓
[Deoptimization if assumptions invalid]
    ↓
Back to Interpreter
```

**Example:**
```javascript
function add(a, b) {
    return a + b;
}

// First calls: interpreted
add(5, 10);
add(3, 7);

// After many calls: TurboFan optimizes
// Assumes: a and b are always integers
// Generates specialized machine code for integer addition

for (let i = 0; i < 100000; i++) {
    add(i, i + 1);  // Uses optimized code
}

// Deoptimization trigger:
add("hello", "world");  // String addition!
// TurboFan's assumption broken → deoptimize back to bytecode
```

**Optimization examples:**
- **Inline caching**: Cache property access locations
- **Hidden classes**: Optimize object property access
- **Inline expansion**: Copy function body into caller
- **Dead code elimination**: Remove unused code
- **Loop unrolling**: Reduce loop overhead

### CPython: Bytecode Interpretation (No JIT by default)

**CPython execution:**

```
Python Source (.py)
    ↓
Compile → Bytecode (.pyc)
    ↓
Interpret bytecode in loop
(No JIT compilation to machine code)
```

**Why no JIT in CPython?**
- Simplicity and portability
- Dynamic nature makes optimization harder
- GIL already limits performance
- Alternative implementations (PyPy) have JIT

**Bytecode interpretation loop (simplified):**
```c
// CPython's eval loop (simplified concept)
while (true) {
    opcode = next_instruction();
    
    switch (opcode) {
        case LOAD_FAST:
            push(locals[arg]);
            break;
        case BINARY_ADD:
            right = pop();
            left = pop();
            push(left + right);  // Calls PyNumber_Add
            break;
        case RETURN_VALUE:
            return pop();
        // ... hundreds more opcodes
    }
}
```

**Performance implications:**
```python
# Each operation is a bytecode instruction
def compute():
    total = 0
    for i in range(1000000):
        total += i  # BINARY_ADD bytecode each iteration
    return total

# Compare to NumPy (C implementation):
import numpy as np
arr = np.arange(1000000)
total = np.sum(arr)  # Runs in compiled C code
```

**Alternative Python implementations:**
- **PyPy**: JIT compiler, 5-10x faster for many workloads
- **Jython**: Runs on JVM, uses JVM's JIT
- **IronPython**: Runs on .NET CLR
- **Cython**: Compiles Python to C

### Java: Bytecode + Multi-Tier JIT

**Java execution model:**

```
Java Source (.java)
    ↓
javac (Ahead-of-Time Compiler)
    ↓
Java Bytecode (.class) [portable]
    ↓
JVM Interpreter
    ↓
Profile execution
    ↓
C1 (Client) Compiler [fast compilation]
    ↓
[More profiling]
    ↓
C2 (Server) Compiler [aggressive optimization]
    ↓
Highly Optimized Machine Code
```

**Tiered compilation:**

```java
public class Example {
    public static void compute(int n) {
        int sum = 0;
        for (int i = 0; i < n; i++) {
            sum += i;
        }
        return sum;
    }
    
    public static void main(String[] args) {
        // First call: interpreted
        compute(100);
        
        // After threshold: C1 compiled
        for (int i = 0; i < 1000; i++) {
            compute(100);
        }
        
        // After more calls: C2 compiled with aggressive opts
        for (int i = 0; i < 100000; i++) {
            compute(100);
        }
    }
}
```

**JVM optimizations:**
- **Method inlining**: Inline small methods
- **Loop optimizations**: Unrolling, hoisting
- **Escape analysis**: Allocate objects on stack if they don't escape
- **Dead code elimination**: Remove unused code
- **Branch prediction**: Optimize for likely branches
- **Intrinsics**: Replace method calls with optimized CPU instructions

**JVM warmup:**
```java
// First run: slow (interpreted)
long start = System.nanoTime();
compute(1000000);
long end = System.nanoTime();
System.out.println("Time: " + (end - start) + "ns");  // ~500,000ns

// After warmup: fast (C2 optimized)
for (int i = 0; i < 10000; i++) {
    compute(1000000);  // Trigger optimization
}

start = System.nanoTime();
compute(1000000);
end = System.nanoTime();
System.out.println("Time: " + (end - start) + "ns");  // ~50,000ns
```

### Why "Compiled vs Interpreted" is Too Simplistic

**Reality:**
- **JavaScript**: Interpreted + JIT compiled
- **Python**: Compiled to bytecode + interpreted (no JIT in CPython)
- **Java**: Compiled to bytecode + interpreted + JIT compiled

**Modern execution models:**
1. **Parse** source code
2. **Compile** to intermediate representation (bytecode/IR)
3. **Interpret** or **JIT compile** to machine code
4. **Profile** and **reoptimize** dynamically

**Trade-offs:**

| Approach | Startup | Peak Performance | Memory | Complexity |
|----------|---------|------------------|--------|------------|
| Pure Interpretation | Fastest | Slowest | Low | Simple |
| AOT Compilation | Slow | Fast | Medium | Simple |
| JIT Compilation | Medium | Fastest | High | Complex |
| Tiered JIT | Fast | Fastest | Highest | Very Complex |

**The sweet spot:**
- Fast startup (interpreter)
- Optimize hot paths (JIT)
- Adapt to runtime behavior (profiling)

This is why JavaScript, Java, and C# all use JIT compilation.


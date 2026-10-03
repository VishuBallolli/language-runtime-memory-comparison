# Part 10: What Happens Internally?

## 20. Simple Statement, Complex Reality: `result = a + b`

Let's examine what actually happens at multiple levels when executing this seemingly simple statement.

---

## Python: `result = a + b`

### High-Level Code
```python
a = 10
b = 20
result = a + b
```

### What CPython Does

#### Step 1: Compilation to Bytecode
```python
import dis

def add():
    a = 10
    b = 20
    result = a + b

dis.dis(add)
```

**Output:**
```
  2           0 LOAD_CONST               1 (10)
              2 STORE_FAST               0 (a)

  3           4 LOAD_CONST               2 (20)
              6 STORE_FAST               1 (b)

  4           8 LOAD_FAST                0 (a)
             10 LOAD_FAST                1 (b)
             12 BINARY_ADD
             14 STORE_FAST               2 (result)
             16 LOAD_CONST               0 (None)
             18 RETURN_VALUE
```

#### Step 2: Bytecode Execution

**LOAD_FAST 0 (a):**
```c
// CPython interpreter (simplified)
case LOAD_FAST: {
    PyObject *value = GETLOCAL(oparg);  // Get local variable
    if (value == NULL) {
        // Error: local variable not defined
    }
    Py_INCREF(value);  // Increment reference count
    PUSH(value);       // Push onto stack
    break;
}
```

**Memory state:**
```
Stack before:  []
               
Action: LOAD_FAST 0 (a)
- Lookup local variable 0 (a) → PyObject* pointing to int(10)
- Increment refcount of int(10)
- Push reference onto stack

Stack after:   [PyObject* → int(10)]
```

**LOAD_FAST 1 (b):**
```
Stack before:  [PyObject* → int(10)]

Action: LOAD_FAST 1 (b)
- Lookup local variable 1 (b) → PyObject* pointing to int(20)
- Increment refcount
- Push reference onto stack

Stack after:   [PyObject* → int(10), PyObject* → int(20)]
```

**BINARY_ADD:**
```c
case BINARY_ADD: {
    PyObject *right = POP();   // Pop int(20)
    PyObject *left = TOP();    // Peek int(10)
    PyObject *sum;
    
    // Call PyNumber_Add (generic add)
    sum = PyNumber_Add(left, right);
    
    // PyNumber_Add checks:
    // 1. Does left have __add__ method?
    // 2. Call left.__add__(right)
    // 3. If that returns NotImplemented, try right.__radd__(left)
    
    // For integers, calls int.__add__
    // Which creates NEW int object (ints are immutable)
    
    SET_TOP(sum);      // Replace top with result
    Py_DECREF(left);   // Decrement refcount of operands
    Py_DECREF(right);
    break;
}
```

**Under the hood of `int.__add__`:**
```c
// Simplified int addition
PyObject* int_add(PyObject *left, PyObject *right) {
    long a = PyLong_AsLong(left);   // Extract C long
    long b = PyLong_AsLong(right);  // Extract C long
    long result = a + b;             // C integer addition
    return PyLong_FromLong(result);  // Create new Python int object
}
```

**Memory state:**
```
Stack before:  [PyObject* → int(10), PyObject* → int(20)]

Action: BINARY_ADD
1. Pop int(20)
2. Peek int(10)
3. Call int(10).__add__(int(20))
   - Extracts C values: 10, 20
   - Performs C addition: 30
   - Creates new PyObject for int(30) on heap
   - Returns PyObject* → int(30)
4. Decrement refcounts of int(10) and int(20)
5. Push int(30)

Stack after:   [PyObject* → int(30)]

Heap:
[int object: value=30, refcount=1]
```

**STORE_FAST 2 (result):**
```c
case STORE_FAST: {
    PyObject *value = POP();
    SETLOCAL(oparg, value);  // Store in local variable
    break;
}
```

```
Stack before:  [PyObject* → int(30)]

Action: STORE_FAST 2
- Pop int(30)
- Store reference in local variable 2 (result)

Stack after:   []

Local variables:
0: a      → int(10)
1: b      → int(20)
2: result → int(30)
```

### Complete Flow Diagram

```
Python Source: result = a + b

↓ [Parser]

AST Node: BinOp(left=Name('a'), op=Add(), right=Name('b'))

↓ [Compiler]

Bytecode:
  LOAD_FAST    0 (a)
  LOAD_FAST    1 (b)
  BINARY_ADD
  STORE_FAST   2 (result)

↓ [Interpreter Loop]

1. LOAD_FAST 0:   Stack [int(10)]
2. LOAD_FAST 1:   Stack [int(10), int(20)]
3. BINARY_ADD:    Stack [int(30)]  ← New object created
4. STORE_FAST 2:  Stack []         ← result = int(30)

↓ [Memory]

Heap:
  int(10)  refcount=1  (referenced by 'a')
  int(20)  refcount=1  (referenced by 'b')
  int(30)  refcount=1  (referenced by 'result')
```

---

## JavaScript: `let result = a + b`

### High-Level Code
```javascript
let a = 10;
let b = 20;
let result = a + b;
```

### What V8 Does

#### Step 1: Parsing to AST
```javascript
// Source tokens: let result = a + b
// 
// AST (Abstract Syntax Tree):
// VariableDeclaration {
//   kind: "let",
//   declarations: [
//     VariableDeclarator {
//       id: Identifier { name: "result" },
//       init: BinaryExpression {
//         left: Identifier { name: "a" },
//         operator: "+",
//         right: Identifier { name: "b" }
//       }
//     }
//   ]
// }
```

#### Step 2: Ignition Bytecode Generation

**Bytecode (simplified):**
```
LdaGlobal [0]    // Load 'a' into accumulator
Star r0          // Store accumulator to register r0
LdaGlobal [1]    // Load 'b' into accumulator
Add r0           // Add r0 to accumulator
Star r1          // Store result in register r1
```

#### Step 3: Bytecode Execution (Ignition Interpreter)

**LdaGlobal [0] - Load 'a':**
```cpp
// Ignition interpreter handler (simplified)
IGNITION_HANDLER(LdaGlobal, InterpreterAssembler) {
    // Lookup variable 'a' in scope
    // Could be in local scope, closure scope, or global
    Node* value = LoadVariable(/* variable index */);
    SetAccumulator(value);
}
```

**Memory state:**
```
Before: Accumulator: undefined
        Registers:   [empty]

Action: Load 'a'
- Lookup variable 'a' → tagged value 10
  (SMI - Small Integer, stored as tagged pointer)
  
After:  Accumulator: SMI(10)
        Registers:   [empty]
```

**Add r0 - Addition:**
```cpp
IGNITION_HANDLER(Add, InterpreterAssembler) {
    Node* left = GetAccumulator();   // 10
    Node* right = LoadRegister(r0);  // 20
    
    // Type check: are both SMIs?
    Label smi_case, generic_case;
    Branch(BothAreSmis(left, right), &smi_case, &generic_case);
    
    // SMI fast path (no allocation)
    Bind(&smi_case);
    Node* result = SmiAdd(left, right);  // Tagged pointer arithmetic
    SetAccumulator(result);
    
    // Generic path (strings, objects, etc.)
    Bind(&generic_case);
    Node* result = CallRuntime(Runtime::kAdd, left, right);
    SetAccumulator(result);
}
```

**SMI (Small Integer) representation:**
```
JavaScript number: 10

V8 internal representation (64-bit):
┌──────────────────────────────────┬──┐
│         Value: 10                │01│  ← Last bit = 1 means SMI
└──────────────────────────────────┴──┘

Benefits:
- No heap allocation
- Fast arithmetic (just pointer arithmetic with tag handling)
- Immediate GC-safe (no object to collect)
```

**If numbers are large (heap numbers):**
```
JavaScript number: 9007199254740992  (> 32-bit)

V8 internal representation:
Stack/Register: [Pointer] ───→ Heap:
                                 ┌────────────────┐
                                 │ HeapNumber     │
                                 │ Map*           │
                                 │ Value: double  │
                                 └────────────────┘
```

#### Step 4: TurboFan Optimization (after hot code detection)

**After many executions, TurboFan profiles and optimizes:**

```
Original bytecode:
  LdaGlobal [0]
  Star r0
  LdaGlobal [1]
  Add r0

Optimized machine code (x86-64, conceptual):
  mov rax, [a_addr]     ; Load 'a' into register
  mov rbx, [b_addr]     ; Load 'b' into register
  add rax, rbx          ; Direct CPU integer addition
  mov [result_addr], rax ; Store result
```

**Assumptions made by TurboFan:**
- `a` and `b` are always integers
- No overflow
- Variables are local (not global with getters)

**If assumptions violated (deoptimization):**
```javascript
// Optimized for integers
for (let i = 0; i < 10000; i++) {
    result = a + b;  // Fast machine code
}

// Deopt trigger
a = "hello";
result = a + b;  // "hello20" - assumptions broken!
                 // Deoptimize back to interpreter
```

### Complete Flow Diagram

```
JavaScript Source: let result = a + b

↓ [Parser]

AST: BinaryExpression(+)

↓ [Ignition Compiler]

Bytecode:
  LdaGlobal [0]  ; Load 'a'
  Star r0        ; Save to register
  LdaGlobal [1]  ; Load 'b'
  Add r0         ; Add

↓ [Ignition Interpreter]

Execution:
1. Load 'a': Accumulator = SMI(10)
2. Store r0: Register[0] = SMI(10)
3. Load 'b': Accumulator = SMI(20)
4. Add:      Accumulator = SMI(30)  ← Pointer arithmetic, no allocation

↓ [If hot code detected]

TurboFan Optimized Machine Code:
  mov rax, 10
  add rax, 20
  mov result, rax

↓ [Memory]

For small integers: No heap allocation (SMI)
For large numbers: HeapNumber objects created
```

---

## Java: `int result = a + b`

### High-Level Code
```java
int a = 10;
int b = 20;
int result = a + b;
```

### What the JVM Does

#### Step 1: Compilation to Bytecode (javac)

**Java bytecode:**
```
public void add();
  Code:
     0: bipush        10        // Push 10 onto stack
     2: istore_1               // Store in local variable 1 (a)
     3: bipush        20        // Push 20 onto stack
     5: istore_2               // Store in local variable 2 (b)
     6: iload_1                // Load local variable 1 (a)
     7: iload_2                // Load local variable 2 (b)
     8: iadd                   // Integer add
     9: istore_3               // Store in local variable 3 (result)
```

#### Step 2: JVM Interpreter Execution

**bipush 10:**
```
Before: Operand Stack: []

Action: Push constant 10
- Load immediate value 10
- Push onto operand stack

After:  Operand Stack: [10]
```

**istore_1:**
```
Before: Operand Stack: [10]
        Local Vars:    []

Action: Store in local variable 1
- Pop value from stack
- Store in local variable slot 1

After:  Operand Stack: []
        Local Vars:    [?, 10, ?, ?]  (slot 1 = a)
```

**iload_1 and iload_2:**
```
Before: Operand Stack: []
        Local Vars:    [?, 10, 20, ?]

Action: Load a and b
- Push local variable 1 (10)
- Push local variable 2 (20)

After:  Operand Stack: [10, 20]
        Local Vars:    [?, 10, 20, ?]
```

**iadd:**
```
Before: Operand Stack: [10, 20]

Action: Integer addition
- Pop 20
- Pop 10
- Perform CPU integer addition: 10 + 20 = 30
- Push result 30

After:  Operand Stack: [30]
```

**istore_3:**
```
Before: Operand Stack: [30]
        Local Vars:    [?, 10, 20, ?]

Action: Store result
- Pop 30
- Store in local variable slot 3

After:  Operand Stack: []
        Local Vars:    [?, 10, 20, 30]
```

#### Step 3: JIT Compilation (C1/C2)

**After threshold of executions, C1 compiles:**

```
Bytecode:
  iload_1
  iload_2
  iadd
  istore_3

↓ [C1 Compiler - Quick optimization]

Native code (conceptual):
  mov eax, [rbp-4]   ; Load 'a' from stack frame
  mov ebx, [rbp-8]   ; Load 'b' from stack frame
  add eax, ebx       ; CPU add instruction
  mov [rbp-12], eax  ; Store result
```

**After many more executions, C2 further optimizes:**

```
↓ [C2 Compiler - Aggressive optimization]

Analysis:
- a = 10 (constant)
- b = 20 (constant)
- result = a + b (constant propagation)

Optimized code:
  mov eax, 30        ; Computed at compile time!
  mov [rbp-12], eax
  
Or even:
  (removed entirely if result unused)
```

**Optimizations applied:**
- **Constant propagation**: Detect compile-time constants
- **Constant folding**: Compute at compile time
- **Dead code elimination**: Remove unused results
- **Register allocation**: Keep values in CPU registers
- **Inlining**: If in method, inline into caller

#### Step 4: Memory Layout

**Stack frame during execution:**
```
┌─────────────────────────┐
│ Method: add()           │
├─────────────────────────┤
│ Local Variable Table:   │
│  Slot 0: this (if instance method) │
│  Slot 1: a = 10         │  ← int stored directly (4 bytes)
│  Slot 2: b = 20         │  ← int stored directly
│  Slot 3: result = 30    │  ← int stored directly
├─────────────────────────┤
│ Operand Stack:          │
│  (used during execution)│
└─────────────────────────┘

Note: Primitive ints stored DIRECTLY on stack
      No heap allocation, no objects, no GC overhead
```

**If using Integer objects instead:**
```java
Integer a = 10;
Integer b = 20;
Integer result = a + b;  // Autoboxing/unboxing
```

```
Stack frame:
┌─────────────────────────┐
│ Local Variable Table:   │
│  Slot 1: a → reference  │ ──→ Heap: Integer object (value=10)
│  Slot 2: b → reference  │ ──→ Heap: Integer object (value=20)
│  Slot 3: result → ref   │ ──→ Heap: Integer object (value=30)
└─────────────────────────┘

Process:
1. Unbox a: call Integer.intValue() → 10
2. Unbox b: call Integer.intValue() → 20
3. Add: 10 + 20 = 30
4. Box result: Integer.valueOf(30) → new Integer(30)
   (or cached if -128 to 127)

Much more overhead than primitive addition!
```

### Complete Flow Diagram

```
Java Source: int result = a + b;

↓ [javac Compiler]

Bytecode:
  bipush 10
  istore_1    ; a = 10
  bipush 20
  istore_2    ; b = 20
  iload_1     ; Load a
  iload_2     ; Load b
  iadd        ; Add (CPU instruction eventually)
  istore_3    ; result = 30

↓ [JVM Interpreter]

Stack machine execution:
1. Push 10:       Stack [10]
2. Store a:       Stack [], Locals [?, 10, ?, ?]
3. Push 20:       Stack [20]
4. Store b:       Stack [], Locals [?, 10, 20, ?]
5. Load a:        Stack [10]
6. Load b:        Stack [10, 20]
7. Add:           Stack [30]  ← CPU add instruction
8. Store result:  Stack [], Locals [?, 10, 20, 30]

↓ [After warmup - C2 JIT]

Optimized Machine Code:
  mov DWORD PTR [rbp-12], 30  ; Constant folding!
  
Or even eliminated entirely if unused.

↓ [Memory]

Stack: Primitives stored directly (4 bytes per int)
Heap:  No heap allocation for primitive int
GC:    No garbage collection needed
```

---

## Comparison Summary

| Aspect | Python | JavaScript | Java |
|--------|--------|-----------|------|
| **Intermediate Form** | Bytecode (.pyc) | Bytecode (Ignition) | Bytecode (.class) |
| **Execution Model** | Interpreted VM loop | Interpreter + JIT | Interpreter + JIT |
| **Addition Operation** | `PyNumber_Add` (dynamic) | SMI fast path or generic | `iadd` (primitive CPU instruction) |
| **Memory for Integers** | Heap object (28+ bytes) | SMI (tagged pointer, no allocation) or HeapNumber | Stack (4 bytes) |
| **Type Checking** | Runtime (per operation) | Runtime (with inline caching) | Compile time + runtime bounds |
| **Optimization** | None (CPython) | TurboFan JIT (runtime) | C2 JIT (runtime) |
| **Performance (relative)** | 1x (baseline) | 20-50x faster | 50-100x faster (after warmup) |


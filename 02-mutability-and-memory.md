# Part 2: Mutability and Memory Management

## 5. Mutable vs. Immutable

### JavaScript

**Immutable (Primitives):**
```javascript
let str = "hello";
str[0] = "H";  // No effect - strings are immutable
console.log(str);  // "hello"

// String methods return NEW strings
let upper = str.toUpperCase();  // returns "HELLO"
console.log(str);  // still "hello"

let num = 42;
num++;  // Creates new number 43, assigns to num
```

**Mutable (Objects):**
```javascript
let arr = [1, 2, 3];
arr[0] = 99;  // Modifies the array
arr.push(4);  // Modifies the array
console.log(arr);  // [99, 2, 3, 4]

let obj = { name: "Alice" };
obj.name = "Bob";  // Modifies the object
obj.age = 30;      // Adds property
console.log(obj);  // { name: "Bob", age: 30 }
```

### Python

**Immutable Types:**
```python
# int, float, bool, str, tuple, frozenset
num = 42
# num[0] = 5  # TypeError - int is not subscriptable

string = "hello"
# string[0] = "H"  # TypeError - str doesn't support item assignment

tup = (1, 2, 3)
# tup[0] = 99  # TypeError - tuple doesn't support item assignment

# Operations create NEW objects
s = "hello"
s2 = s.upper()  # creates new string "HELLO"
print(s)   # "hello" - original unchanged
print(s2)  # "HELLO"
```

**Mutable Types:**
```python
# list, dict, set, bytearray
lst = [1, 2, 3]
lst[0] = 99         # Modifies the list
lst.append(4)       # Modifies the list
print(lst)  # [99, 2, 3, 4]

dictionary = {"name": "Alice"}
dictionary["name"] = "Bob"  # Modifies the dict
dictionary["age"] = 30      # Adds key
print(dictionary)  # {"name": "Bob", "age": 30}
```

### Java

**Immutable Types:**
```java
// String, wrapper classes (Integer, Double, etc.)
String str = "hello";
// str.charAt(0) = 'H';  // Error - no such method

// String methods return NEW strings
String upper = str.toUpperCase();
System.out.println(str);    // "hello"
System.out.println(upper);  // "HELLO"

Integer num = 42;
// Can't modify Integer internally - immutable
```

**Mutable Types:**
```java
// Arrays, ArrayList, HashMap, StringBuilder, etc.
int[] arr = {1, 2, 3};
arr[0] = 99;  // Modifies the array

StringBuilder sb = new StringBuilder("hello");
sb.append(" world");  // Modifies StringBuilder
System.out.println(sb);  // "hello world"

ArrayList<String> list = new ArrayList<>();
list.add("Alice");
list.set(0, "Bob");  // Modifies the list
```

### Real-World Example: The String Problem

**JavaScript:**
```javascript
let result = "";
for (let i = 0; i < 1000; i++) {
    result += "x";  // Creates 1000 new strings! Inefficient!
}

// Better: use array
let parts = [];
for (let i = 0; i < 1000; i++) {
    parts.push("x");
}
let result = parts.join("");  // One concatenation
```

**Python:**
```python
# Inefficient - creates many intermediate strings
result = ""
for i in range(1000):
    result += "x"  # Each += creates a new string

# Better: use list and join
parts = []
for i in range(1000):
    parts.append("x")
result = "".join(parts)

# Or even better: use string multiplication
result = "x" * 1000
```

**Java:**
```java
// Inefficient
String result = "";
for (int i = 0; i < 1000; i++) {
    result += "x";  // Creates 1000 new String objects
}

// Efficient: use StringBuilder
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1000; i++) {
    sb.append("x");  // Modifies same object
}
String result = sb.toString();
```

---

## 6. Memory: Stack vs. Heap

### What is the Stack?

The **call stack** stores:
- Function call frames
- Local variables (primitives or references)
- Function parameters
- Return addresses

**Characteristics:**
- Fast allocation/deallocation (LIFO - Last In, First Out)
- Limited size (stack overflow if too deep recursion)
- Automatically managed
- Thread-local (each thread has its own stack)

### What is the Heap?

The **heap** stores:
- Objects
- Arrays
- Dynamic data structures

**Characteristics:**
- Slower allocation/deallocation than stack
- Much larger than stack
- Managed by garbage collector
- Shared across threads (with synchronization)

### Why "Variables = Stack, Objects = Heap" is Incomplete

**JavaScript Example:**
```javascript
function example() {
    let num = 42;           // num stored on stack
    let str = "hello";      // In V8, small strings may be on stack or heap
    let obj = { x: 1 };     // obj reference on stack, object on heap
    
    // Stack frame for example():
    // - num: 42
    // - str: reference to string (or value if optimized)
    // - obj: reference to object
    
    // Heap:
    // - { x: 1 } object
}
```

**Python Example:**
```python
def example():
    num = 42          # num (name) on stack, int object on heap
    text = "hello"    # text on stack, str object on heap (or interned)
    data = [1, 2]     # data on stack, list object on heap
    
    # Everything in Python is an object, but:
    # - Small integers (-5 to 256) are pre-allocated
    # - Strings may be interned
    # - Stack holds references to heap objects
```

**Java Example:**
```java
void example() {
    int num = 42;           // num value directly on stack (primitive)
    String str = "hello";   // str reference on stack, String on heap
    int[] arr = {1, 2};     // arr reference on stack, array on heap
    
    // Stack frame:
    // - num: 42 (actual value)
    // - str: 0x1A2B (reference)
    // - arr: 0x3C4D (reference)
    
    // Heap:
    // - String object "hello" at 0x1A2B
    // - int[] array at 0x3C4D
}
```

### The Complete Picture

**Stack stores:**
- Primitive values (Java)
- References/pointers to heap objects (all three languages)
- Function call metadata

**Heap stores:**
- All objects (JavaScript, Python)
- Objects and arrays (Java)

**Optimizations blur the line:**
- **Escape analysis**: JVM may allocate objects on stack if they don't "escape" the method
- **String interning**: Reused strings may be in special memory area
- **Small object optimization**: V8 and other engines optimize small objects
- **Integer caching**: Python caches small integers, Java caches -128 to 127

---

## 7. Function or Method Memory

### Function Call Process

When a function is called:

1. **Create stack frame** (activation record)
2. **Push onto call stack**
3. **Allocate local variables**
4. **Execute function body**
5. **Return value**
6. **Pop frame from stack**

### JavaScript Example

```javascript
function calculate(a, b) {
    let sum = a + b;
    let product = a * b;
    return { sum, product };
}

let result = calculate(5, 10);

// Call stack progression:
// 1. Global context pushed
// 2. calculate(5, 10) frame pushed:
//    - a: 5
//    - b: 10
//    - sum: 15
//    - product: 50
//    - return address
// 3. Object { sum: 15, product: 50 } created on heap
// 4. calculate frame popped
// 5. result references heap object
```

### Python Example

```python
def calculate(a, b):
    sum_val = a + b
    product = a * b
    return {"sum": sum_val, "product": product}

result = calculate(5, 10)

# Call stack:
# 1. Module level frame
# 2. calculate(5, 10) frame pushed:
#    - Local namespace with references to:
#      - a -> int(5)
#      - b -> int(10)
#      - sum_val -> int(15)
#      - product -> int(50)
# 3. Dict created on heap
# 4. Frame popped, but heap object persists
# 5. result references the dict
```

### Java Example

```java
class Calculator {
    public int[] calculate(int a, int b) {
        int sum = a + b;
        int product = a * b;
        return new int[]{sum, product};
    }
    
    public static void main(String[] args) {
        Calculator calc = new Calculator();
        int[] result = calc.calculate(5, 10);
    }
}

// Stack frames:
// 1. main() frame:
//    - calc: reference to Calculator object
//    - result: reference to int[] array
//
// 2. calculate(5, 10) frame (while executing):
//    - this: reference to Calculator
//    - a: 5 (actual value)
//    - b: 10 (actual value)
//    - sum: 15
//    - product: 50
//    - (temporary) reference to new array
//
// Heap:
// - Calculator object
// - int[] {15, 50} array
```

### Visualizing the Call Stack

```javascript
function first() {
    console.log("first start");
    second();
    console.log("first end");
}

function second() {
    console.log("second start");
    third();
    console.log("second end");
}

function third() {
    console.log("third");
}

first();

// Call stack over time:
// 
// first()
// ↓
// first() → second()
// ↓
// first() → second() → third()
// ↓
// first() → second()  (third popped)
// ↓
// first()  (second popped)
// ↓
// (empty)  (first popped)
```

### Stack Overflow

```javascript
function recursive() {
    recursive();  // No base case!
}

recursive();  // RangeError: Maximum call stack size exceeded
```

```python
def recursive():
    recursive()

recursive()  # RecursionError: maximum recursion depth exceeded
```

```java
void recursive() {
    recursive();
}

recursive();  // StackOverflowError
```

The stack has limited size. Too many nested function calls exhaust it.


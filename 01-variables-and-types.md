# Part 1: Variables and Type Systems

## 1. Variable Declaration

### JavaScript
```javascript
// let - block-scoped, can be reassigned
let count = 10;
count = 20; // ✓

// const - block-scoped, cannot be reassigned
const name = "Alice";
// name = "Bob"; // ✗ Error

// var - function-scoped (legacy, avoid in modern code)
var legacy = "old";
```

### Python
```python
# Dynamic variable names - no declaration keyword needed
count = 10
name = "Alice"

# Variables are just names that reference objects
# Type can change at runtime
x = 42        # x references an int
x = "hello"   # now x references a string
```

### Java
```java
// Explicitly typed declarations
int count = 10;
String name = "Alice";
final int MAX = 100;  // final = cannot be reassigned

// Type inference (Java 10+)
var message = "Hello";  // compiler infers String
```

**Key Differences:**
- **JavaScript**: Declaration keywords (`let`, `const`, `var`) with dynamic typing
- **Python**: No declaration keywords, pure dynamic typing
- **Java**: Explicit types required (or inferred with `var`), static typing

---

## 2. Static vs. Dynamic Typing

### Statically Typed: Java

```java
String name = "Alice";
// name = 42;  // ✗ Compile-time error: incompatible types

int age = 30;
// age = "thirty";  // ✗ Compile-time error
```

**Type checking happens at compile time.** The compiler verifies that types match before the code ever runs.

### Dynamically Typed: JavaScript and Python

**JavaScript:**
```javascript
let value = "Alice";  // value is a string
value = 42;           // now value is a number
value = true;         // now value is a boolean
// ✓ All valid - type checking at runtime
```

**Python:**
```python
value = "Alice"  # value references a str object
value = 42       # value references an int object
value = True     # value references a bool object
# ✓ All valid - type checking at runtime
```

**Runtime type checking example:**
```javascript
function add(a, b) {
    return a + b;
}

add(5, 10);      // 15
add("5", "10");  // "510" - string concatenation
add(5, "10");    // "510" - type coercion
```

```python
def add(a, b):
    return a + b

add(5, 10)      # 15
add("5", "10")  # "510"
# add(5, "10")  # TypeError at runtime
```

**When Type Errors Occur:**
- **Java**: Errors caught before running (compile time)
- **JavaScript/Python**: Errors caught while running (runtime)

---

## 3. Primitive vs. Reference or Object Types

### JavaScript: Primitives vs. Objects

**Primitives** (immutable, passed by value):
```javascript
let num = 42;          // number
let str = "hello";     // string
let bool = true;       // boolean
let nothing = null;    // null
let undef = undefined; // undefined
let sym = Symbol();    // symbol
let big = 123n;        // bigint
```

**Objects** (mutable, passed by reference):
```javascript
let obj = { name: "Alice" };     // object
let arr = [1, 2, 3];             // array
let func = function() {};        // function
let date = new Date();           // date
```

### Python: Everything is an Object

```python
# Even "primitives" are objects with methods
x = 42
print(type(x))           # <class 'int'>
print(x.__add__(10))     # 52 - calling method on int

s = "hello"
print(type(s))           # <class 'str'>
print(s.upper())         # "HELLO" - string methods

# But some are immutable, some mutable
immutable = (42, "hello", True, (1, 2))  # int, str, bool, tuple
mutable = [1, 2], {"key": "value"}       # list, dict
```

### Java: Primitives vs. References

**Primitives** (not objects, stored by value):
```java
int num = 42;
double pi = 3.14;
boolean flag = true;
char letter = 'A';
```

**Reference Types** (objects, stored by reference):
```java
String name = "Alice";           // String object
Integer boxed = 42;              // wrapper for int
int[] array = {1, 2, 3};         // array
ArrayList<String> list = new ArrayList<>();  // object
```

**Key Insight:**
- **JavaScript**: Clear primitive/object divide
- **Python**: Everything is an object, but behavior differs (mutable vs immutable)
- **Java**: Primitives are not objects (performance optimization), everything else is

---

## 4. Variable → Object → Reference: What Does `a == b` Really Mean?

### Assignment vs. Equality

**JavaScript:**
```javascript
// Primitives: comparison by value
let a = 42;
let b = 42;
console.log(a == b);   // true - same value
console.log(a === b);  // true - same value and type

// Objects: comparison by reference
let obj1 = { x: 1 };
let obj2 = { x: 1 };
let obj3 = obj1;

console.log(obj1 == obj2);   // false - different objects
console.log(obj1 === obj2);  // false - different objects
console.log(obj1 === obj3);  // true - same reference
```

**Python:**
```python
# Immutable objects: behavior similar to primitives
a = 42
b = 42
print(a == b)   # True - value comparison
print(a is b)   # True (usually) - CPython caches small integers

# Mutable objects: reference comparison with 'is'
list1 = [1, 2, 3]
list2 = [1, 2, 3]
list3 = list1

print(list1 == list2)   # True - value comparison
print(list1 is list2)   # False - different objects
print(list1 is list3)   # True - same reference
```

**Java:**
```java
// Primitives: always by value
int a = 42;
int b = 42;
System.out.println(a == b);  // true

// Objects: == compares references
String str1 = new String("hello");
String str2 = new String("hello");
String str3 = str1;

System.out.println(str1 == str2);      // false - different objects
System.out.println(str1 == str3);      // true - same reference
System.out.println(str1.equals(str2)); // true - value comparison
```

### Assignment Behavior

**JavaScript:**
```javascript
// Primitives: copy value
let x = 10;
let y = x;  // y gets a copy of 10
x = 20;
console.log(y);  // 10 - y unchanged

// Objects: copy reference
let arr1 = [1, 2, 3];
let arr2 = arr1;  // arr2 points to same array
arr1.push(4);
console.log(arr2);  // [1, 2, 3, 4] - arr2 sees the change
```

**Python:**
```python
# Immutable: appears to copy value
x = 10
y = x
x = 20  # x now references a new int object
print(y)  # 10

# Mutable: copy reference
list1 = [1, 2, 3]
list2 = list1  # list2 references same list
list1.append(4)
print(list2)  # [1, 2, 3, 4]
```

**Java:**
```java
// Primitives: copy value
int x = 10;
int y = x;
x = 20;
System.out.println(y);  // 10

// Objects: copy reference
int[] arr1 = {1, 2, 3};
int[] arr2 = arr1;  // arr2 points to same array
arr1[0] = 99;
System.out.println(arr2[0]);  // 99
```

**The Mental Model:**
- **Primitives** (or immutable objects): Variables hold the actual value
- **Objects** (mutable): Variables hold a reference/pointer to the object in memory
- `==` in most languages compares what the variable holds (value for primitives, reference for objects)
- Java's `.equals()` and Python's `==` provide value comparison for objects


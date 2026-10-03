# Part 3: Functions and Parameter Passing

## 8. Functions Across the Four Languages

### JavaScript / Node.js: First-Class Functions

Functions are objects that can be:
- Assigned to variables
- Passed as arguments
- Returned from functions
- Stored in data structures

```javascript
// Function declaration
function greet(name) {
    return `Hello, ${name}!`;
}

// Function expression
const greet2 = function(name) {
    return `Hello, ${name}!`;
};

// Arrow function
const greet3 = (name) => `Hello, ${name}!`;

// Functions as arguments (callbacks)
function processUser(name, callback) {
    const greeting = callback(name);
    console.log(greeting);
}

processUser("Alice", greet);

// Functions as return values (higher-order functions)
function createMultiplier(factor) {
    return function(number) {
        return number * factor;
    };
}

const double = createMultiplier(2);
console.log(double(5));  // 10

// Functions in arrays
const operations = [
    (x) => x + 1,
    (x) => x * 2,
    (x) => x - 3
];

let result = 10;
operations.forEach(op => {
    result = op(result);
});
console.log(result);  // (10 + 1) * 2 - 3 = 19
```

### Python: First-Class Functions

```python
# Function definition
def greet(name):
    return f"Hello, {name}!"

# Functions as arguments
def process_user(name, callback):
    greeting = callback(name)
    print(greeting)

process_user("Alice", greet)

# Functions as return values
def create_multiplier(factor):
    def multiply(number):
        return number * factor
    return multiply

double = create_multiplier(2)
print(double(5))  # 10

# Lambda functions
square = lambda x: x ** 2
print(square(4))  # 16

# Functions in lists
operations = [
    lambda x: x + 1,
    lambda x: x * 2,
    lambda x: x - 3
]

result = 10
for op in operations:
    result = op(result)
print(result)  # 19

# Functions as object attributes
class Calculator:
    def __init__(self):
        self.operation = lambda x, y: x + y
    
    def set_operation(self, func):
        self.operation = func
    
    def calculate(self, a, b):
        return self.operation(a, b)

calc = Calculator()
print(calc.calculate(5, 3))  # 8

calc.set_operation(lambda x, y: x * y)
print(calc.calculate(5, 3))  # 15
```

### Java: Methods + Lambdas / Functional Interfaces

Traditional Java uses methods (not first-class), but Java 8+ added lambdas and functional interfaces:

```java
// Traditional method
public class MathUtils {
    public static int add(int a, int b) {
        return a + b;
    }
}

// Functional interface
@FunctionalInterface
interface Operation {
    int apply(int a, int b);
}

// Lambda expressions (Java 8+)
public class FunctionExample {
    public static void main(String[] args) {
        // Lambda as variable
        Operation add = (a, b) -> a + b;
        Operation multiply = (a, b) -> a * b;
        
        System.out.println(add.apply(5, 3));       // 8
        System.out.println(multiply.apply(5, 3));  // 15
        
        // Pass function as argument
        processOperation(10, 20, add);        // 30
        processOperation(10, 20, multiply);   // 200
        
        // Method reference
        Operation subtract = MathUtils::subtract;
        
        // Built-in functional interfaces
        java.util.function.Function<String, Integer> stringLength = s -> s.length();
        System.out.println(stringLength.apply("Hello"));  // 5
        
        // Return function
        Operation createAdder = getOperation("add");
        System.out.println(createAdder.apply(7, 3));  // 10
    }
    
    public static void processOperation(int a, int b, Operation op) {
        int result = op.apply(a, b);
        System.out.println("Result: " + result);
    }
    
    public static Operation getOperation(String type) {
        if (type.equals("add")) {
            return (a, b) -> a + b;
        } else {
            return (a, b) -> a * b;
        }
    }
}
```

**Java's Approach:**
- Methods belong to classes (not standalone)
- Lambdas compile to instances of functional interfaces
- Standard functional interfaces: `Function`, `Predicate`, `Consumer`, `Supplier`, etc.
- Not as flexible as JS/Python, but more type-safe

---

## 9. Pass-by-Value or Pass-by-Reference

### The Truth: All Three Languages Pass by Value

**But the "value" may be a reference!**

### JavaScript: Pass-by-Value (of References for Objects)

```javascript
// Primitives: pass by value
function modifyPrimitive(x) {
    x = 100;  // Modifies local copy
    console.log("Inside:", x);  // 100
}

let num = 10;
modifyPrimitive(num);
console.log("Outside:", num);  // 10 - unchanged

// Objects: pass by value of reference
function modifyObject(obj) {
    obj.name = "Bob";  // Modifies the object (we have reference)
    console.log("Inside:", obj);  // { name: "Bob" }
}

let person = { name: "Alice" };
modifyObject(person);
console.log("Outside:", person);  // { name: "Bob" } - changed!

// Reassignment doesn't affect original
function reassignObject(obj) {
    obj = { name: "Charlie" };  // Changes local reference only
    console.log("Inside:", obj);  // { name: "Charlie" }
}

let person2 = { name: "Alice" };
reassignObject(person2);
console.log("Outside:", person2);  // { name: "Alice" } - unchanged
```

**Visual Explanation:**
```
Before call:
person variable → [Object: { name: "Alice" }] (heap)

During modifyObject(person):
person variable → [Object: { name: "Alice" }] (heap)
                      ↑
obj parameter ────────┘  (copy of reference)

obj.name = "Bob" modifies the heap object through the reference.

During reassignObject(person):
person variable → [Object: { name: "Alice" }] (heap)

obj parameter → [Object: { name: "Charlie" }] (new heap object)

Reassignment changes where obj points, not where person points.
```

### Python: Pass-by-Value (of References)

```python
# Immutable: appears like pass-by-value
def modify_number(x):
    x = 100  # Rebinds local x to new int object
    print("Inside:", x)  # 100

num = 10
modify_number(num)
print("Outside:", num)  # 10 - unchanged

# Mutable: modifications visible outside
def modify_list(lst):
    lst.append(4)  # Modifies the list object
    print("Inside:", lst)  # [1, 2, 3, 4]

my_list = [1, 2, 3]
modify_list(my_list)
print("Outside:", my_list)  # [1, 2, 3, 4] - changed!

# Reassignment doesn't affect original
def reassign_list(lst):
    lst = [99, 99]  # Rebinds local lst to new list
    print("Inside:", lst)  # [99, 99]

my_list2 = [1, 2, 3]
reassign_list(my_list2)
print("Outside:", my_list2)  # [1, 2, 3] - unchanged
```

### Java: Pass-by-Value (Always)

```java
// Primitives: pass by value
void modifyPrimitive(int x) {
    x = 100;
    System.out.println("Inside: " + x);  // 100
}

int num = 10;
modifyPrimitive(num);
System.out.println("Outside: " + num);  // 10 - unchanged

// Objects: pass by value of reference
class Person {
    String name;
    Person(String name) { this.name = name; }
}

void modifyObject(Person p) {
    p.name = "Bob";  // Modifies the object
    System.out.println("Inside: " + p.name);  // Bob
}

Person person = new Person("Alice");
modifyObject(person);
System.out.println("Outside: " + person.name);  // Bob - changed!

// Reassignment doesn't affect original
void reassignObject(Person p) {
    p = new Person("Charlie");  // Changes local reference only
    System.out.println("Inside: " + p.name);  // Charlie
}

Person person2 = new Person("Alice");
reassignObject(person2);
System.out.println("Outside: " + person2.name);  // Alice - unchanged
```

### The Array/List Example

**JavaScript:**
```javascript
function modifyArray(arr) {
    arr.push(4);        // Modifies the array ✓
    arr[0] = 99;        // Modifies the array ✓
    arr = [1, 1, 1];    // Reassigns local reference (no effect outside) ✗
}

let numbers = [1, 2, 3];
modifyArray(numbers);
console.log(numbers);  // [99, 2, 3, 4]
```

**Python:**
```python
def modify_list(lst):
    lst.append(4)      # Modifies the list ✓
    lst[0] = 99        # Modifies the list ✓
    lst = [1, 1, 1]    # Reassigns local reference (no effect outside) ✗

numbers = [1, 2, 3]
modify_list(numbers)
print(numbers)  # [99, 2, 3, 4]
```

**Java:**
```java
void modifyArray(int[] arr) {
    arr[0] = 99;           // Modifies the array ✓
    arr = new int[]{1, 1}; // Reassigns local reference (no effect outside) ✗
}

int[] numbers = {1, 2, 3};
modifyArray(numbers);
System.out.println(Arrays.toString(numbers));  // [99, 2, 3]
```

### Summary Table

| Language   | Primitives    | Objects/References | Can Modify Object? | Can Reassign Outside? |
|------------|---------------|-------------------|-------------------|----------------------|
| JavaScript | Pass-by-value | Pass-by-value (of reference) | ✓ Yes | ✗ No |
| Python     | Pass-by-value (of reference) | Pass-by-value (of reference) | ✓ Yes (if mutable) | ✗ No |
| Java       | Pass-by-value | Pass-by-value (of reference) | ✓ Yes | ✗ No |

**Key Insight:**
- You can **modify** an object through a passed reference
- You **cannot** make the caller's variable point to a different object by reassignment


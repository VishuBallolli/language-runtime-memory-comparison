# Part 4: Closures and Lexical Scope

## 10. Closures

A **closure** is a function that retains access to variables from its outer (enclosing) scope, even after the outer function has finished executing.

### JavaScript Closures

```javascript
function createCounter() {
    let count = 0;  // Local variable in outer function
    
    return function() {
        count++;    // Inner function "closes over" count
        return count;
    };
}

const counter1 = createCounter();
console.log(counter1());  // 1
console.log(counter1());  // 2
console.log(counter1());  // 3

const counter2 = createCounter();
console.log(counter2());  // 1 - separate closure
console.log(counter1());  // 4 - counter1 unaffected
```

**Why it works:**
- The inner function maintains a reference to `count`
- Even though `createCounter()` has returned, `count` stays alive
- Each call to `createCounter()` creates a new closure with its own `count`

### More JavaScript Closure Examples

**Private variables:**
```javascript
function createBankAccount(initialBalance) {
    let balance = initialBalance;  // Private variable
    
    return {
        deposit: function(amount) {
            balance += amount;
            return balance;
        },
        withdraw: function(amount) {
            if (amount <= balance) {
                balance -= amount;
                return balance;
            }
            return "Insufficient funds";
        },
        getBalance: function() {
            return balance;
        }
    };
}

const account = createBankAccount(100);
console.log(account.deposit(50));    // 150
console.log(account.withdraw(30));   // 120
console.log(account.getBalance());   // 120
// console.log(account.balance);     // undefined - truly private!
```

**Event handlers:**
```javascript
function setupButtons() {
    for (let i = 0; i < 3; i++) {
        const button = document.createElement('button');
        button.textContent = `Button ${i}`;
        
        // Each click handler closes over its own i
        button.addEventListener('click', function() {
            console.log(`Button ${i} clicked`);
        });
        
        document.body.appendChild(button);
    }
}

// With var (common mistake):
function setupButtonsBroken() {
    for (var i = 0; i < 3; i++) {  // var is function-scoped
        const button = document.createElement('button');
        button.textContent = `Button ${i}`;
        
        // All handlers share same i (which becomes 3)
        button.addEventListener('click', function() {
            console.log(`Button ${i} clicked`);  // Always logs 3!
        });
        
        document.body.appendChild(button);
    }
}
```

### Python Closures

```python
def create_counter():
    count = 0  # Enclosed variable
    
    def increment():
        nonlocal count  # Needed to modify enclosed variable
        count += 1
        return count
    
    return increment

counter1 = create_counter()
print(counter1())  # 1
print(counter1())  # 2
print(counter1())  # 3

counter2 = create_counter()
print(counter2())  # 1 - separate closure
```

**Private variables:**
```python
def create_bank_account(initial_balance):
    balance = initial_balance
    
    def deposit(amount):
        nonlocal balance
        balance += amount
        return balance
    
    def withdraw(amount):
        nonlocal balance
        if amount <= balance:
            balance -= amount
            return balance
        return "Insufficient funds"
    
    def get_balance():
        return balance
    
    return {
        'deposit': deposit,
        'withdraw': withdraw,
        'get_balance': get_balance
    }

account = create_bank_account(100)
print(account['deposit'](50))     # 150
print(account['withdraw'](30))    # 120
print(account['get_balance']())   # 120
```

**Decorator example (advanced closure):**
```python
def create_logger(prefix):
    def decorator(func):
        def wrapper(*args, **kwargs):
            print(f"{prefix}: calling {func.__name__}")
            result = func(*args, **kwargs)
            print(f"{prefix}: {func.__name__} returned {result}")
            return result
        return wrapper
    return decorator

@create_logger("DEBUG")
def add(a, b):
    return a + b

result = add(5, 3)
# Output:
# DEBUG: calling add
# DEBUG: add returned 8
```

### Java Lambda Capture (Closures)

Java lambdas can capture variables from enclosing scope, but with restrictions:

```java
import java.util.function.Supplier;

public class ClosureExample {
    public static Supplier<Integer> createCounter() {
        // Local variable for capturing
        final int[] count = {0};  // Array to allow modification
        
        return () -> {
            count[0]++;  // Captures and modifies
            return count[0];
        };
    }
    
    public static void main(String[] args) {
        Supplier<Integer> counter1 = createCounter();
        System.out.println(counter1.get());  // 1
        System.out.println(counter1.get());  // 2
        System.out.println(counter1.get());  // 3
        
        Supplier<Integer> counter2 = createCounter();
        System.out.println(counter2.get());  // 1 - separate closure
    }
}
```

**Effectively final restriction:**
```java
public void example() {
    int x = 10;
    
    Runnable r = () -> {
        System.out.println(x);  // OK - x is effectively final
    };
    
    // x = 20;  // Error! x must be effectively final to be captured
    
    r.run();
}
```

**Workaround for modifiable capture:**
```java
import java.util.function.Function;

public class ClosureWorkaround {
    public static Function<Integer, Integer> createMultiplier() {
        final int[] factor = {2};  // Wrap in array
        
        return (num) -> {
            int result = num * factor[0];
            factor[0]++;  // Can modify array contents
            return result;
        };
    }
    
    public static void main(String[] args) {
        Function<Integer, Integer> mult = createMultiplier();
        System.out.println(mult.apply(5));   // 10 (5 * 2)
        System.out.println(mult.apply(5));   // 15 (5 * 3)
        System.out.println(mult.apply(5));   // 20 (5 * 4)
    }
}
```

### Why Captured Variables Outlive the Original Function Call

**Memory perspective:**

```javascript
function outer() {
    let data = "important";  // Allocated in outer's stack frame
    
    return function inner() {
        console.log(data);   // References data
    };
}

const fn = outer();
// outer() has returned - its stack frame is gone!
// But data is still accessible...

fn();  // Logs "important"
```

**What happens:**

1. **JavaScript/Python:**
   - Variables closed over are moved to the heap (or kept in a closure object)
   - The inner function keeps a reference to this closure environment
   - Garbage collector keeps the data alive as long as inner function exists

2. **Java:**
   - Captured variables are copied into the lambda object
   - The lambda object is created on the heap
   - For effectively final variables, a copy is sufficient

**Visualization:**

```
After outer() returns:

Stack:                    Heap:
fn (reference) ───────→  [Closure Object]
                          - inner function code
                          - data: "important"

As long as fn exists, the closure object (and data) stay alive.
```

### Practical Closure Use Cases

**1. Data Privacy / Encapsulation**
```javascript
function createUser(name, age) {
    // Private data
    let _name = name;
    let _age = age;
    
    // Public interface
    return {
        getName: () => _name,
        getAge: () => _age,
        haveBirthday: () => ++_age
    };
}

const user = createUser("Alice", 25);
user.haveBirthday();
console.log(user.getAge());  // 26
// No direct access to _name or _age
```

**2. Function Factories**
```javascript
function createFormatter(prefix, suffix) {
    return function(text) {
        return `${prefix}${text}${suffix}`;
    };
}

const bold = createFormatter('<b>', '</b>');
const italic = createFormatter('<i>', '</i>');

console.log(bold("Hello"));    // <b>Hello</b>
console.log(italic("World"));  // <i>World</i>
```

**3. Partial Application**
```python
def multiply(a, b):
    return a * b

def create_multiplier(factor):
    def mult(number):
        return multiply(factor, number)
    return mult

double = create_multiplier(2)
triple = create_multiplier(3)

print(double(5))  # 10
print(triple(5))  # 15
```

**4. Memoization**
```javascript
function createMemoized(fn) {
    const cache = {};  // Closed over by returned function
    
    return function(arg) {
        if (arg in cache) {
            console.log("From cache");
            return cache[arg];
        }
        
        console.log("Computing");
        const result = fn(arg);
        cache[arg] = result;
        return result;
    };
}

const expensiveFunction = (n) => {
    // Simulate expensive computation
    let result = 0;
    for (let i = 0; i < 1000000; i++) {
        result += n;
    }
    return result;
};

const memoized = createMemoized(expensiveFunction);
console.log(memoized(5));  // Computing, returns result
console.log(memoized(5));  // From cache, returns result
```

### Common Closure Pitfalls

**Loop variable capture (JavaScript):**
```javascript
// Problem with var
const functions = [];
for (var i = 0; i < 3; i++) {
    functions.push(function() {
        console.log(i);
    });
}

functions[0]();  // 3 (not 0!)
functions[1]();  // 3 (not 1!)
functions[2]();  // 3 (not 2!)

// Solution 1: Use let
const functions2 = [];
for (let i = 0; i < 3; i++) {  // let creates new binding each iteration
    functions2.push(function() {
        console.log(i);
    });
}

functions2[0]();  // 0 ✓
functions2[1]();  // 1 ✓
functions2[2]();  // 2 ✓

// Solution 2: IIFE
const functions3 = [];
for (var i = 0; i < 3; i++) {
    (function(j) {  // Create new scope with j
        functions3.push(function() {
            console.log(j);
        });
    })(i);
}
```

**Memory leaks from closures:**
```javascript
function setupHandler() {
    const bigData = new Array(1000000).fill("x");  // Large data
    
    document.getElementById("button").addEventListener("click", function() {
        console.log("Clicked");
        // Even though we don't use bigData here, it's kept alive!
    });
}

// Better: don't capture unnecessarily
function setupHandlerBetter() {
    const bigData = new Array(1000000).fill("x");
    processData(bigData);  // Use it
    // bigData goes out of scope
    
    document.getElementById("button").addEventListener("click", function() {
        console.log("Clicked");
        // bigData not captured, can be GC'd
    });
}
```


# Part 8: Real-World Request Lifecycle

## 17. HTTP Request Journey: Variables → Objects → Response

Let's trace a complete HTTP request through JavaScript, Python, and Java, showing what happens at each step with memory, functions, and runtime behavior.

### Scenario: Simple User API

**Endpoint:** `GET /users/:id`
- Receives HTTP request
- Queries database
- Processes data
- Returns JSON response

---

## JavaScript / Node.js Example

```javascript
const express = require('express');
const app = express();

// Simulated database
const db = {
    query: async (sql, params) => {
        // Simulate async database call
        return new Promise((resolve) => {
            setTimeout(() => {
                resolve({
                    id: params[0],
                    name: 'Alice',
                    email: 'alice@example.com',
                    age: 30
                });
            }, 50);
        });
    }
};

// Route handler
app.get('/users/:id', async (req, res) => {
    try {
        // 1. Extract parameter
        const userId = parseInt(req.params.id);
        
        // 2. Query database
        const user = await db.query(
            'SELECT * FROM users WHERE id = ?',
            [userId]
        );
        
        // 3. Process data
        const response = {
            success: true,
            data: {
                id: user.id,
                name: user.name,
                email: user.email
            }
        };
        
        // 4. Send response
        res.json(response);
    } catch (error) {
        res.status(500).json({ success: false, error: error.message });
    }
});

app.listen(3000, () => {
    console.log('Server running on port 3000');
});
```

### Memory and Execution Flow (JavaScript)

**Request arrives: `GET /users/123`**

```
1. HTTP Request received by Node.js
   ↓
2. libuv event loop picks up connection
   ↓
3. Express routing matches /users/:id
   ↓
4. STACK FRAME CREATED for route handler
   
   Stack (route handler frame):
   ┌──────────────────────────────┐
   │ req: reference               │ ──→ [Request Object on Heap]
   │ res: reference               │ ──→ [Response Object on Heap]
   │ userId: 123                  │     (primitive, stored on stack)
   └──────────────────────────────┘
   
   Heap:
   ┌──────────────────────────────┐
   │ Request Object               │
   │  - params: { id: "123" }     │
   │  - headers: {...}            │
   │  - method: "GET"             │
   └──────────────────────────────┘
   
   ┌──────────────────────────────┐
   │ Response Object              │
   │  - statusCode: 200           │
   │  - send(), json()            │
   └──────────────────────────────┘

5. Database query (async)
   - Promise created on heap
   - Callback registered
   - Stack frame SUSPENDED (not popped)
   - Event loop continues, handles other requests
   
6. Database responds (50ms later)
   - Promise resolves
   - Callback queued in event loop
   
7. Callback executed
   Stack frame RESUMED:
   ┌──────────────────────────────┐
   │ user: reference              │ ──→ [User Object on Heap]
   │ response: reference          │ ──→ [Response Data on Heap]
   └──────────────────────────────┘
   
   Heap (new objects):
   ┌──────────────────────────────┐
   │ User Object                  │
   │  - id: 123                   │
   │  - name: "Alice"             │
   │  - email: "alice@..."        │
   │  - age: 30                   │
   └──────────────────────────────┘
   
   ┌──────────────────────────────┐
   │ Response Data                │
   │  - success: true             │
   │  - data: {...}               │
   └──────────────────────────────┘

8. res.json(response) called
   - Serializes response to JSON string
   - Sends HTTP response
   
9. Stack frame POPPED
   - Local variables (userId, user, response) removed from stack
   - Heap objects eligible for GC if no other references
   
10. Response sent, connection may close
    - req and res objects eligible for GC
```

**Key Points:**
- **Async I/O**: Database call doesn't block event loop
- **Single thread**: One stack, but many suspended contexts
- **Heap objects**: Request, response, user data all on heap
- **GC**: Objects collected after response sent

---

## Python Example (Flask)

```python
from flask import Flask, jsonify
import time

app = Flask(__name__)

# Simulated database
class Database:
    def query(self, sql, params):
        # Simulate database latency
        time.sleep(0.05)
        return {
            'id': params[0],
            'name': 'Alice',
            'email': 'alice@example.com',
            'age': 30
        }

db = Database()

@app.route('/users/<int:user_id>')
def get_user(user_id):
    try:
        # 1. Parameter extracted by Flask (user_id)
        
        # 2. Query database
        user = db.query(
            'SELECT * FROM users WHERE id = ?',
            [user_id]
        )
        
        # 3. Process data
        response = {
            'success': True,
            'data': {
                'id': user['id'],
                'name': user['name'],
                'email': user['email']
            }
        }
        
        # 4. Return JSON response
        return jsonify(response)
    
    except Exception as e:
        return jsonify({'success': False, 'error': str(e)}), 500

if __name__ == '__main__':
    app.run(port=3000)
```

### Memory and Execution Flow (Python)

**Request arrives: `GET /users/123`**

```
1. HTTP Request received by Flask dev server
   ↓
2. Flask routing matches /users/<int:user_id>
   ↓
3. STACK FRAME CREATED for get_user()
   
   Stack (get_user frame):
   ┌──────────────────────────────┐
   │ user_id: reference           │ ──→ [int object: 123 on Heap]
   │                              │     (or cached int -5 to 256)
   └──────────────────────────────┘
   
   Note: In CPython, even integers are objects on heap
   (but small integers are pre-allocated/cached)

4. Database query (BLOCKING in this example)
   - Thread blocked for 50ms
   - Cannot handle other requests during this time
   - This is why WSGI servers use multiple processes/threads
   
5. Database returns
   Stack frame updated:
   ┌──────────────────────────────┐
   │ user: reference              │ ──→ [dict object on Heap]
   │ response: reference          │ ──→ [dict object on Heap]
   └──────────────────────────────┘
   
   Heap:
   ┌──────────────────────────────┐
   │ User dict                    │
   │  'id': int(123)              │
   │  'name': str("Alice")        │
   │  'email': str("alice@...")   │
   │  'age': int(30)              │
   └──────────────────────────────┘
   
   ┌──────────────────────────────┐
   │ Response dict                │
   │  'success': bool(True)       │
   │  'data': dict(...)           │
   └──────────────────────────────┘

6. jsonify(response) called
   - Creates Flask Response object
   - Serializes dict to JSON string
   
7. Stack frame POPPED
   - Local names (user_id, user, response) removed
   - Reference counts decremented
   - If refcount = 0, objects destroyed immediately
   
8. Response sent, connection may close
```

**Key Points:**
- **Blocking I/O**: Database call blocks the thread
- **Multiple processes**: Production uses Gunicorn/uWSGI with multiple workers
- **Reference counting**: Objects may be destroyed immediately when refcount = 0
- **Everything is an object**: Even `user_id` integer is a heap object

**Async version (Python asyncio):**
```python
from aiohttp import web

async def get_user(request):
    user_id = int(request.match_info['id'])
    
    # Async database query (doesn't block event loop)
    user = await db.query_async(
        'SELECT * FROM users WHERE id = ?',
        [user_id]
    )
    
    response = {
        'success': True,
        'data': {
            'id': user['id'],
            'name': user['name'],
            'email': user['email']
        }
    }
    
    return web.json_response(response)

app = web.Application()
app.router.add_get('/users/{id}', get_user)
web.run_app(app, port=3000)
```

---

## Java Example (Spring Boot)

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.*;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.CompletableFuture;

@SpringBootApplication
@RestController
public class UserController {
    
    // Simulated database
    static class Database {
        public Map<String, Object> query(String sql, int userId) {
            // Simulate database latency
            try { Thread.sleep(50); } catch (InterruptedException e) {}
            
            Map<String, Object> user = new HashMap<>();
            user.put("id", userId);
            user.put("name", "Alice");
            user.put("email", "alice@example.com");
            user.put("age", 30);
            return user;
        }
    }
    
    private Database db = new Database();
    
    @GetMapping("/users/{id}")
    public Map<String, Object> getUser(@PathVariable int id) {
        try {
            // 1. Parameter extracted (id)
            
            // 2. Query database
            Map<String, Object> user = db.query(
                "SELECT * FROM users WHERE id = ?",
                id
            );
            
            // 3. Process data
            Map<String, Object> userData = new HashMap<>();
            userData.put("id", user.get("id"));
            userData.put("name", user.get("name"));
            userData.put("email", user.get("email"));
            
            Map<String, Object> response = new HashMap<>();
            response.put("success", true);
            response.put("data", userData);
            
            // 4. Return (Spring converts to JSON)
            return response;
            
        } catch (Exception e) {
            Map<String, Object> error = new HashMap<>();
            error.put("success", false);
            error.put("error", e.getMessage());
            return error;
        }
    }
    
    public static void main(String[] args) {
        SpringApplication.run(UserController.class, args);
    }
}
```

### Memory and Execution Flow (Java)

**Request arrives: `GET /users/123`**

```
1. HTTP Request received by embedded Tomcat server
   - Request handled by thread from thread pool
   ↓
2. Spring MVC routing matches /users/{id}
   ↓
3. STACK FRAME CREATED for getUser() on thread's stack
   
   Thread Stack (getUser frame):
   ┌──────────────────────────────┐
   │ this: reference              │ ──→ [UserController instance]
   │ id: 123                      │     (primitive int, on stack)
   └──────────────────────────────┘

4. Database query (BLOCKING)
   - Thread sleeps for 50ms
   - Thread occupied, cannot handle other requests
   - Other threads in pool handle other requests
   
5. Database returns
   Stack frame updated:
   ┌──────────────────────────────┐
   │ user: reference              │ ──→ [HashMap on Heap]
   │ userData: reference          │ ──→ [HashMap on Heap]
   │ response: reference          │ ──→ [HashMap on Heap]
   └──────────────────────────────┘
   
   Heap:
   ┌──────────────────────────────┐
   │ User HashMap                 │
   │  "id" → Integer(123)         │
   │  "name" → String("Alice")    │
   │  "email" → String("alice@")  │
   │  "age" → Integer(30)         │
   └──────────────────────────────┘
   
   ┌──────────────────────────────┐
   │ Response HashMap             │
   │  "success" → Boolean(true)   │
   │  "data" → HashMap(userData)  │
   └──────────────────────────────┘

6. Spring converts HashMap to JSON
   - Uses Jackson library
   - Serializes to JSON string
   
7. Stack frame POPPED
   - Local variables removed from stack
   - Heap objects eligible for GC (no longer referenced)
   - Thread returns to pool
   
8. Response sent, connection may close
```

**Key Points:**
- **Thread per request**: Each request handled by a thread from pool
- **Blocking I/O**: Thread blocks during database call
- **True parallelism**: Multiple threads handle multiple requests simultaneously
- **Heap objects**: HashMaps, Strings, wrapper objects on heap
- **Primitives on stack**: `id` stored as primitive int on stack

**Async version (Spring WebFlux):**
```java
@GetMapping("/users/{id}")
public Mono<Map<String, Object>> getUserAsync(@PathVariable int id) {
    return Mono.fromCallable(() -> {
        // Database call runs on separate thread pool
        return db.query("SELECT * FROM users WHERE id = ?", id);
    })
    .subscribeOn(Schedulers.boundedElastic())
    .map(user -> {
        // Process result
        Map<String, Object> userData = new HashMap<>();
        userData.put("id", user.get("id"));
        userData.put("name", user.get("name"));
        userData.put("email", user.get("email"));
        
        Map<String, Object> response = new HashMap<>();
        response.put("success", true);
        response.put("data", userData);
        return response;
    });
}
```

---

## Summary Comparison

### Memory Layout

| Language | Primitives | Objects | Parameters | Return Values |
|----------|-----------|---------|------------|---------------|
| **JavaScript** | Stack (for locals) | Heap | By value (refs for objects) | Heap objects |
| **Python** | Heap (everything) | Heap | By value of reference | Heap objects |
| **Java** | Stack | Heap | By value (primitives), by value of ref (objects) | Stack (primitives) or Heap (objects) |

### Concurrency During Request

| Language | Model | I/O Handling | Blocking Behavior |
|----------|-------|--------------|-------------------|
| **JavaScript** | Event loop | Non-blocking (async/await) | Suspended, other requests handled |
| **Python (Flask)** | Multi-process/thread | Blocking (sync) | Thread blocked, others handle requests |
| **Python (asyncio)** | Event loop | Non-blocking (async/await) | Suspended, other requests handled |
| **Java (Servlet)** | Thread per request | Blocking | Thread blocked, others handle requests |
| **Java (WebFlux)** | Reactive | Non-blocking | Suspended, other requests handled |

### Performance Characteristics

**For I/O-bound workload (typical web API):**

| Language/Framework | Concurrency | Memory per Request | Throughput |
|--------------------|-------------|-------------------|------------|
| **Node.js** | Very High (10k+) | Low (~10KB) | Very High |
| **Python Flask** | Medium (10s-100s) | Medium (~1MB per process) | Medium |
| **Python asyncio** | Very High (10k+) | Low (~10KB) | Very High |
| **Java Servlet** | Medium (100s) | High (~1MB per thread) | High |
| **Java WebFlux** | Very High (10k+) | Medium (~100KB) | Very High |

**Why these differences?**
- **Event loop** (Node.js, asyncio, WebFlux): Handles many concurrent I/O operations efficiently
- **Thread per request** (Flask, Servlet): Limited by thread overhead and context switching
- **True parallelism** (Java threads): Better for CPU-bound work than event loop


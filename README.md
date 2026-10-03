# Language Runtime & Memory Comparison

## JavaScript vs Node.js vs Python vs Java

A comprehensive deep-dive guide exploring how JavaScript, Node.js, Python, and Java handle variables, memory, functions, runtime execution, and performance. This guide goes beyond simple comparisons to show what actually happens under the hood.

## 📚 Guide Structure

### [Part 1: Variables and Type Systems](./01-variables-and-types.md)
- Variable declaration across languages
- Static vs. dynamic typing
- Primitives vs. reference types
- Assignment and equality semantics

### [Part 2: Mutability and Memory Management](./02-mutability-and-memory.md)
- Mutable vs. immutable types
- Stack vs. heap memory
- Function call memory (call stack)
- Why "variables = stack, objects = heap" is incomplete

### [Part 3: Functions and Parameter Passing](./03-functions-and-parameters.md)
- First-class functions (JavaScript, Python)
- Methods and lambdas (Java)
- Pass-by-value vs. pass-by-reference
- What really happens when objects are passed

### [Part 4: Closures and Lexical Scope](./04-closures.md)
- Closures in JavaScript, Python, and Java
- Why captured variables outlive function calls
- Practical use cases and common pitfalls

### [Part 5: Garbage Collection](./05-garbage-collection.md)
- Why garbage collection exists
- When objects become eligible for collection
- GC algorithms: V8 (JavaScript), CPython, JVM
- Memory leaks despite GC

### [Part 6: Runtimes and Execution Models](./06-runtimes-and-execution.md)
- V8 engine architecture
- Node.js runtime (V8 + libuv)
- CPython virtual machine
- JVM and bytecode execution
- Compilation, interpretation, and JIT

### [Part 7: Concurrency Models](./07-concurrency-models.md)
- Event loop (Node.js, Python asyncio)
- Native threading (Java)
- I/O-bound vs. CPU-bound workloads
- When to use which model

### [Part 8: Real-World HTTP Request Lifecycle](./08-real-world-example.md)
- Complete request flow in each language
- Memory and stack frame visualization
- Async vs. blocking execution
- Production deployment considerations

### [Part 9: Performance and Memory Lifetime](./09-performance-considerations.md)
- Why "which language is fastest?" is the wrong question
- Workload types and appropriate languages
- JIT warmup vs. peak performance
- When variables and objects exist and die

### [Part 10: Internal Deep Dive](./10-internals-deep-dive.md)
- What actually happens with `result = a + b`
- Bytecode, AST, and machine code
- Python interpreter, V8 execution, JVM compilation
- Memory layout and optimization

## 🎯 Who This Guide Is For

- **Developers switching between languages** who want to understand fundamental differences
- **Performance-conscious engineers** optimizing real-world applications
- **System designers** choosing appropriate languages for workloads
- **Computer science students** learning runtime and memory management
- **Curious programmers** who want to know what happens under the hood

## 🔑 Key Takeaways

1. **All three languages pass by value** — but for objects, the "value" is a reference
2. **Memory management is automatic** — but memory leaks can still happen
3. **"Compiled vs. interpreted" is outdated** — modern runtimes use bytecode + JIT
4. **Event loops excel at I/O-bound** work, threads at CPU-bound
5. **Architecture matters more than language** for performance
6. **Each language trades off** between simplicity, performance, and type safety

## 🚀 Quick Start

Start with [Part 1](./01-variables-and-types.md) for fundamentals, or jump to specific topics:
- Want to understand async/concurrency? → [Part 7](./07-concurrency-models.md)
- Debugging memory leaks? → [Part 5](./05-garbage-collection.md)
- Choosing a language for a project? → [Part 9](./09-performance-considerations.md)
- Curious about internals? → [Part 10](./10-internals-deep-dive.md)

## 📖 Reading Tips

- Code examples are runnable and tested
- Visual diagrams show memory state over time
- "Key Insight" boxes highlight critical concepts
- Comparison tables summarize differences
- Real-world examples show practical implications

## 🤝 Contributing

Found an error or have a suggestion? Contributions welcome!

---

**Start reading:** [Part 1 - Variables and Type Systems →](./01-variables-and-types.md)

---
layout: cover
background: /img/header-bg.svg
---

# Structured Concurrency

---

## Structured Programming

It is a programming paradigm defined by Dijkstra, Hoare, Dahl, Böhm, and Jacopini

1. **Abstractions as Building Blocks**
   * Variables - abstract over "the current value" while maintaining a specific type
   * Functions - abstract over "what it does" while discarding "how it does it" 
2. **Recursive Decomposition**
   * Problems can be solved by breaking them down into smaller, more manageable subproblems, which can be solved independently and then combined to form a solution to the original problem.  
3. **Local Reasoning and Nested Scopes**
    * The ability to analyse and verify a unit of code in isolation
    * Nested scopes - lexical scopes isolate variables and concerns, allowing for better modularity and maintainability
---

## Structured Programming

4. **Single Entry and Single Exit**
   * Each control structure (like loops, conditionals, and functions) should have one entry point and one exit point, which simplifies the flow of control and makes it easier to understand and maintain the code.
   * No GOTO statements, which can lead to spaghetti code and make it difficult to follow the program's flow.
5. **Soundness and Completeness**
   * The structured program theorem (Böhm and Jacopini, 1966) proved that these principles are sufficient to solve all programming problems. It established that all programs can be written using three basic control structures: sequence, selection (if/then), and repetition (loops)

---

## Crisis of C++ Asynchrony

* **Raw Threads**
  * OS-heavy, unstructured, and difficult to safely join without deadlocks
* **Futures and Promises**
  * Inefficient and lack of composability and cancellation support
* **Callbacks**
  * Lead to "callback hell" and make error handling difficult

---

## Unstructured Concurrency

* Traditional concurrency models often rely on low-level primitives like threads, locks, and condition variables, which can lead to complex and error-prone code
* This is like using **GOTO statements** in structured programming: 
  * it can lead to spaghetti code that is difficult to understand, maintain, and debug
    

---
class: white-slide
---

##

<img src="/img/StructuredConcurrency.png" class="img-lg" />

---

## Structured Concurrency

* A programming paradigm that applies the principles of structured programming — such as abstractions, recursive decomposition, and local reasoning — to concurrent work

* It shifts the focus from low-level primitives like threads and mutexes to the control flow and relationships between tasks

---

## Structured Concurrency

1. **Nested Lifetimes**
    * The lifetime of a child operation must be **entirely nested** within the lifetime of its parent
    * A parent operation cannot complete until all its child tasks have finished
2. **Single Entry, Single Exit**
   * Concurrent "computations" (**work units**) are designed with one entry point (starting the work) and one exit point, which can result in success, an error, or cancellation
3. **Cooperative Cancellation**
   * If a parent task is cancelled, all its children are automatically and **cooperatively cancelled**
4. **Error Propagation**
   * If a child task fails, the **error is propagated to the parent**, which can then trigger the cancellation of other siblings to ensure no orphaned tasks are left running

---

## Benefits of Structured Concurrency

* **Improved Readability and Maintainability**: By structuring concurrent code in a way that mirrors the logical flow of the program, it becomes easier to understand and maintain.
* **Better Resource Management**: Nested lifetimes ensure that resources are properly managed and released, preventing leaks and ensuring that tasks do not outlive their intended scope.
* **Enhanced Error Handling**: With cooperative cancellation and error propagation, structured concurrency allows for more robust error handling, ensuring that failures are handled gracefully and do not lead to inconsistent states.
* **Easier Debugging**: The clear structure of concurrent tasks makes it easier to identify and debug issues, as the relationships between tasks are explicit and well-defined.

---

## Structured Concurrency in C++

* C++20 introduced **Coroutines**, which provide a powerful mechanism for writing asynchronous code in a structured way
* C++26 introduces **Senders and Receivers**, which will further enhance the ability to write structured concurrent code by providing a more flexible and composable model for asynchronous operations
# Structured Concurrency in C++

## Structured Programming

Structured programming is a programming paradigm that revolutionised the software industry by teaching engineers how to structure code to better fit human mental abilities and providing a fundamental step in the development of high-level languages. It shifted development from ad hoc methods to a focus on control flow and formal structure.

The paradigm is defined by five core principles derived from foundational texts by Dijkstra, Hoare, Dahl, Böhm, and Jacopini, which are as follows:

### 1. Abstractions as Building Blocks

Abstractions allow developers to focus on the essential parts of a system while ignoring non-essential details.

* **Variables**: Abstract over "the current value" while maintaining a specific type.
* **Functions**: Abstract "what it does" (the signature and name) while discarding "how it does it" (the implementation).
* **Named Abstractions**: Functions are considered particularly valuable because they allow engineers to differentiate and name complex logic, representing anything from a small loop to an entire program (e.g., the main() function).

### 2. Recursive Decomposition
Also known as a **top-down approach** or **Divide Et Impera** (Divide and Conquer), this is a method for creating and analysing programs.
* Problems are broken into smaller, more manageable sub-problems.
* Engineers make one local decision at a time, ensuring that later decisions generally do not invalidate earlier ones.
* This transforms what could be a quadratic process (where every part of a system is coupled to every other part) into a **linear process**.

### 3. Local Reasoning and Nested Scopes
Local reasoning is the ability to analyse and verify a unit of code in isolation without needing to understand the entire system or every context in which the code is used.
* **Nested Scopes**: Lexical scopes isolate local variables and concerns, preventing "spooky action at a distance".
* **Encapsulation**: Code blocks are encapsulated so that they do not interact with the world outside the block unless explicitly intended.

### 4. Single Entry, Single Exit Points
To ensure that code blocks behave like simple instructions, they should have exactly one entry point and one exit point.

* This allows for **enumerative reasoning**, where the preconditions of one instruction depend directly on the postconditions of the previous one.
* This principle led to the discouragement of the **GOTO statement**, which was argued to be "harmful" because it created multiple entry and exit points that made the human mind struggle to follow the execution flow.

### 5. Soundness and Completeness
The structured program theorem (Böhm and Jacopini, 1966) proved that these principles are sufficient to solve all programming problems.
* It established that all programs can be written using three basic control structures: **sequence, selection (if/then), and repetition (loops)**.

## Structured Concurrency

Structured concurrency is a programming paradigm that applies the principles of structured programming—such as abstractions, recursive decomposition, and local reasoning—to concurrent work. 

It shifts the focus from **low-level primitives** like threads and mutexes to the control flow and relationships between **tasks**.

Core principles of structured concurrency include:

### 1. Nested Lifetimes
The primary rule of structured concurrency is that the lifetime of a child operation must be entirely nested within the lifetime of its parent. A parent operation cannot complete until all its child tasks have finished.

### 2. Single Entry, Single Exit
Concurrent "computations" (work units) are designed with one entry point (starting the work) and one exit point, which can result in success, an error, or cancellation.

### 3. Cooperative Cancellation
If a parent task is cancelled, all its children are automatically and cooperatively cancelled.


### 4. Error Propagation
If a child task fails, the error is propagated to the parent, which can then trigger the cancellation of other siblings to ensure no orphaned tasks are left running.

## Structured Concurrency in C++

C++ has evolved to support structured concurrency through various libraries and language features. 

### Coroutines

The C++20 standard introduced **coroutines**, which provide a powerful mechanism for writing asynchronous code in a structured manner.

Coroutines allow developers to write asynchronous code that looks and behaves like synchronous code, making it easier to read and maintain. They enable the use of `co_await`, `co_yield`, and `co_return` to manage asynchronous operations without blocking threads.

`Task<T>` is a common abstraction used in C++ for representing asynchronous operations that return a value of type `T`. It represents a **unit of work** that can be awaited, and it encapsulates the state and logic needed to manage the asynchronous operation.

### Senders and Receivers

The **senders and receivers** model is another approach to structured concurrency in C++. It provides a way to represent asynchronous computations as a series of operations that can be composed together.

In this model, a **sender** represents an asynchronous operation that can produce a value or an error, while a **receiver** is an entity that can consume the result of a sender. This model allows for flexible composition of asynchronous operations and can be used to build complex workflows while maintaining structured concurrency principles.

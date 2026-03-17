# Senders & Receivers

Senders and receivers are the core components of `std::execution`, a standardized asynchronous programming framework introduced in the C++26 standard.

Often compared to how iterators revolutionized synchronous algorithms by separating data from operations, senders and receivers provide a generic "async equivalent" to bridge the gap between asynchronous task graphs and execution resources like thread pools or GPUs

### The Core Concepts

This model for asynchrony is based around four key abstractions: 

* **Scheduler** - handle to an execution context (such as a thread pool, GPU, or I/O loop)
* **Sender** - object that describe asynchronous computations. Senders are **lazy** - represent work that has not yet started, and they are **composable** - they can be combined together to form larger computations.
* **Receiver** - asynchronous notification handler or callback. It receives the results of a sender's work once it completes.
* **Operation State**: When a sender and a receiver are connected, they produce an operation state
. This object encapsulates the actual work to be done and its internal state. It must remain at a stable memory address (it is non-movable) for the duration of the task.

### The Lifecycle of an Async Operation

An asynchronous operation follows a strict three-step process:

1. **Connect** - a sender is joined with a receiver using the connect() function, yielding an operation state.
2. **Start** - The work only begins when start() is called on that operation state.
3. **Complete** - Once the work finishes, the operation state notifies the receiver through one of three completion channels:
   * `set_value`: Successful completion, potentially passing multiple results
   * `set_error`: Failure, passing an exception or error code
   * `set_stopped`: Cooperative cancellation, indicating the work was abandoned without error or success

This structured lifecycle ensures that asynchronous operations are predictable, composable, and can be efficiently managed across various execution contexts.

### Why Use Senders and Receivers?

* **Composability**: Senders can be chained using adapters (like then, let_value, or when_all) to build complex, non-linear task graphs. This allows different libraries (e.g., networking and GPU processing) to interoperate without custom "glue code".
* **Efficiency**: The model is designed for zero abstraction overhead. Because the structure of the computation is known at compile time, the framework can often avoid dynamic memory allocations and unnecessary synchronization.
* **Structured Concurrency**: Senders and receivers naturally enforce structured concurrency. By explicitly defining dependencies and ensuring that parent tasks wait for child tasks, the framework eliminates common bugs like data races, deadlocks, and orphaned "fire-and-forget" tasks.
* **Generic Execution**: You can write an algorithm once and change where it runs (e.g., from a single CPU thread to a massive GPU pool) simply by swapping the scheduler.

## Schedulers

### Execution resource

A **scheduler** is a lightweight handle that represents a compute resource or a strategy for scheduling work onto an execution resource. It acts as the bridge between abstract asynchronous code and concrete hardware or system resources like thread pools, GPUs, I/O loops, or system-wide execution contexts.

Schedulers provide a uniform interface for diverse execution environments, including:
* **Thread Pools**: Managing multiple OS threads for CPU-bound tasks.
* **Hardware Interrupts**: In embedded systems, schedulers can hook into timer or priority-based interrupts.
* **Accelerators**: Offloading work to heterogeneous resources like GPUs or clusters.
* **Inline/Single Thread**: Simpler contexts like the current thread or a dedicated background thread.


### Scheduler Concept

A **scheduler** is a lightweight handle that represents a strategy for scheduling work onto an execution resource.

Since execution resources don’t necessarily manifest in C++ code, it’s not possible to program directly against their API. A scheduler is a solution to that problem: the scheduler concept is defined by a single sender algorithm, `schedule()`. This function does not perform work immediately; instead, it returns a sender. When this sender is eventually started, it completes (sends an "impulse") on an execution agent belonging to the scheduler's associated resource


```cpp
stdexec::scheduler auto sch = thread_pool.scheduler();
stdexec::sender auto snd = stdexec::schedule(sch);

// snd is a sender (see below) describing the creation of a new execution resource
// on the execution resource associated with sch

```

### Managing Work Migration

Schedulers allow developers to explicitly specify where and when work happens. By using adaptors like `starts_on` and `continues_on`, a program can move a computation from one resource (e.g., an I/O thread) to another (e.g., a GPU thread pool).

```cpp
stdexec::scheduler auto io_scheduler = ...;
stdexec::scheduler auto gpu_scheduler = ...;

stdexec::sender auto snd = stdexec::schedule(io_scheduler)
                         | stdexec::then([]{ /* do some I/O work */ })
                         | stdexec::continues_on(gpu_scheduler)
                         | stdexec::then([]{ /* do some GPU work */ });
```

## Senders

### Core nature of senders

#### Asynchronous Analog of Functions

A **sender** is an object that **describes an asynchronous computation**. It is fundamentally a **lazy value**, representing work that is defined but not yet executed until a consumer explicitly demands the result.

Senders are often described as the **async equivalent of functions**. Just as a `std::function` represents a unit of work that must be invoked to run, a sender describes a task graph that must be started to perform its work.

#### Lazy excecution

Defining or piping senders together does not trigger any background work.

Execution only begins when a sender is **connected to a receiver** (producing an **operation state**) and that state is explicitly **started**.

A sender is said to send some values if a receiver connected (`execution::connect`) to that sender will eventually receive said values.

#### Composability

 Senders are designed to be **highly composable**, allowing developers to build complex, non-linear task graphs using algorithms `like then`, `let_value`, and `when_all`.
 This composition often resembles an "onion" where each sender adaptor wraps around the previous one

#### Generic Basis

Much like iterators act as the glue between containers and algorithms, senders serve as the generic glue between asynchronous task graphs and execution resources like thread pools, GPUs, or I/O loops.

### Connecting senders to receivers

#### Completion and Signaling

Similar to a traditional function, the work represented by a sender has one entry point and one exit point, usually called **completion** (or completion signal).

A typical function has two channels for completion: a **value channel** for successful results and an **error channel** for exceptions. Senders extend this model by adding a third channel for cooperative cancellation, called the **stopped channel**.

Every sender promises to eventually notify its recipient through exactly one of **three completion channels**:

1. **Value Channel - `set_value(auto... vals)`**: Indicates successful completion and can transmit zero or more result values.
2. **Error Channel - `set_error(auto err)`**: Signals a failure, transmitting a single error datum such as an exception pointer or error code.
3. **Stopped Channel - `set_stopped()`**: Represents cooperative cancellation, indicating the work was abandoned because its result was no longer needed.

More precisely, a sender can support any combination of completion channels. Some senders may complete with different set of value types, while other might complete with different types of errors.

For example, we can have a sender that has following completion signatures:
* `set_value(int)` - completes with an int value
* `set_value(std::string)` - completes with a string value
* `set_error(std::exception_ptr)` - completes with an exception pointer on error
* `set_error(std::error_code)` - completes with an error code on error
* `set_stopped()` - completes with stopped if the operation was cancelled

#### The connect() function

In the C++ `std::execution` framework, `connect()` is a core customization point object (CPO) that acts as the functional glue between the synchronous and asynchronous domains. Its primary role is to join a **sender** (the description of work) with a **receiver** (the handler for the results), producing an **operation state**.

The connect operation follows a specific protocol to prepare work for execution:

* **Creating the Operation State**: When you call `connect(sender, receiver)`, it returns an object known as the **operation state**. While a sender is a "lazy value" that only describes a task, the operation state is a concrete object that encapsulates the actual work to be done.

* **The "Onion" Model (Recursive Connection)**: In a chain of **sender adaptors** (often called a "sender onion"), calling connect on the outermost adaptor triggers a recursive process. The outer sender creates its own internal receiver and calls connect on the inner sender it wraps. This continues until the entire chain is "materialized" into a nested hierarchy of operation states that mirrors the structure of the senders.

* **Runtime vs. Compile-time**: While the structure of the task graph is often known at compile-time through template types, `connect()` itself is a runtime function. It typically copies or moves state (like function arguments or durations) from the sender into the operation state.

* **Type Compatibility**: The sender and receiver passed to `connect()` must be compatible. Specifically, the receiver must provide handlers for every completion signature (value, error, and stopped) that the sender might produce

* **Async Lifetime**:  Calling connect does not start the work. It merely prepares the "latent execution". The operation only begins when the `start()` function is explicitly called on the resulting operation state. Once `connect()` is called, the resulting operation state must be kept alive until the asynchronous operation completes. This is because the operation state encapsulates the work and its internal state, and it is non-movable to ensure stability during execution.


End-users rarely call connect directly. Instead, it is invoked "under the hood" by sender consumers. For example:
* `sync_wait(snd)`: Internally creates a receiver with a mutex and condition variable, calls connect to get the operation state, and then calls start while blocking the current thread.
* `start_detached(snd)`: Connects a sender to a receiver that manages its own lifetime and triggers execution in the background.
* Coroutines: When a coroutine `co_await`s a sender, the connect operation is used to hook the sender up to the coroutine's internal machinery so it can be resumed when the work finishes.

The way user code is expected to interact with senders is by using **sender algorithms**: 

```cpp
execution::scheduler auto sch = thread_pool.scheduler();

execution::sender auto snd = execution::schedule(sch);

execution::sender auto continuation = execution::then(snd, []{
    std::fstream file{ "result.txt" };
    file << compute_result;
});

this_thread::sync_wait(continuation); // calls connect() and start internally
// at this point, continuation has completed execution
```

### Multi-shot vs. Single-shot senders

A **single-shot sender** can only be connected to a receiver at most once. Its implementation of `execution::connect()` only has overloads for an rvalue-qualified sender. Callers must pass the sender as an rvalue to the call to `execution::connect()`, indicating that the call consumes the sender.

A **multi-shot sender** can be connected to multiple receivers and can be launched multiple times. 
Multi-shot senders customise `execution::connect()` to accept an lvalue reference to the sender. Callers can indicate that they want the sender to remain valid after the call to `execution::connect()` by passing an lvalue reference to the sender to call these overloads. Multi-shot senders should also define overloads of `execution::connect()` that accept rvalue-qualified senders to allow the sender to be also used in places where only a single-shot sender is required.

### Sender algorithms

With senders, users can describe generic execution pipelines and graphs, and then run them on and across a variety of different schedulers. 

`std::execution` framework incudes a rich set of **sender algorithms** that that can be used as primitives for buiding more complex asynchronous computations. These algorithms can be categorized into three main categories:

* **Sender Factories**: They produce senders without requiring any other senders. For example, `execution::schedule(scheduler)` creates a sender that represents the scheduling of work on a given scheduler.
  
* **Sender Adaptors**: Given one or more senders, they return senders based on the provided senders. For example, `execution::then(sender, func)` returns a sender that represents the execution of `func` after the completion of `sender`.

* **Sender Consumers**, They consume senders but don’t produce any senders. For example, `this_thread::sync_wait(sender)` starts the work represented by `sender` and blocks the current thread until it completes, returning the result.

### Execution resource transitions

During asynchronous computations, it is often necessary to transition work from one execution resource to another. For example, you might want to perform some I/O work on an I/O thread pool and then transition to a GPU thread pool for compute-intensive work.

In `std::execution` all execution resource transitions must be explicit. Running user code anywhere but where they defined it to run must be considered a bug.

The `execution::continues_on(snd, sch)` sender adaptor performs a transition from one execution resource to another:

```cpp
execution::scheduler auto sch1 = ...;
execution::scheduler auto sch2 = ...;

execution::sender auto snd1 = execution::schedule(sch1);
execution::sender auto then1 = execution::then(snd1, []{
    std::cout << "I am running on sch1!\n";
});

// explicit transition from sch1 to sch2
execution::sender auto snd2 = execution::continues_on(then1, sch2);
execution::sender auto then2 = execution::then(snd2, []{
    std::cout << "I am running on sch2!\n";
});

this_thread::sync_wait(then2);
```

### Forking a sender

Any non-trivial program will eventually want to fork a chain of senders into independent streams of work, regardless of whether they are single-shot or multi-shot. For instance, an incoming event to a middleware system may be required to trigger events on more than one downstream system. This requires that we provide well defined mechanisms for making sure that connecting a sender multiple times is possible and correct.

The `split` sender adaptor facilitates connecting to a sender multiple times, regardless of whether it is single-shot or multi-shot:

```cpp
auto some_algorithm(execution::sender auto&& input) 
{
    execution::sender auto multi_shot = split(input);
    // "multi_shot" is guaranteed to be multi-shot,
    // regardless of whether "input" was multi-shot or not

    return when_all(
      then(multi_shot, [] { std::cout << "First continuation\n"; }),
      then(multi_shot, [] { std::cout << "Second continuation\n"; })
    );
}
```

### Joining senders

`when_all(input_sndrs...)` is a sender adaptor that returns a sender that completes when the last of the input senders completes. It sends a pack of values, where the elements of said pack are the values sent by the input senders, in order. `when_all` returns a sender that also does not have an associated scheduler.

`transfer_when_all` accepts an additional scheduler argument. It returns a sender whose value completion scheduler is the scheduler provided as an argument, but otherwise behaves the same as `when_all`. You can think of it as a composition of `transfer(when_all(inputs...), scheduler)`, but one that allows for better efficiency through customization.

### Support for cancellation

Senders are often used in scenarios where the application may be concurrently executing multiple strategies for achieving some program goal. When one of these strategies succeeds (or fails) it may not make sense to continue pursuing the other strategies as their results are no longer useful.

For example, we may want to try to simultaneously connect to multiple network servers and use whichever server responds first. Once the first server responds we no longer need to continue trying to connect to the other servers.

Ideally, in these scenarios, we would somehow be able to request that those other strategies stop executing promptly so that their resources (e.g. cpu, memory, I/O bandwidth) can be released and used for other work.

The ability to be able to cancel in-flight operations is fundamental to supporting some kinds of generic concurrency algorithms.

For example:

* a `when_all(ops...)` algorithm should cancel other operations as soon as one operation fails

* a `first_successful(ops...)` algorithm should cancel the other operations as soon as one operation completes successfully

* a generic `timeout(src, duration)` algorithm needs to be able to cancel the src operation after the timeout duration has elapsed.

* a `stop_when(src, trigger)` algorithm should cancel src if trigger completes first and cancel trigger if src completes first


For example, we can compose the algorithms mentioned above so that child operations are cancelled when any one of the multiple cancellation conditions occurs:

```cpp
sender auto composed_cancellation_example(auto query) 
{
  return stop_when(
    timeout(
      when_all(
        first_successful(
          query_server_a(query),
          query_server_b(query)),
        load_file("some_file.jpg")),
      5s),
    cancelButton.on_click());
}
```

In this example, if we take the operation returned by `query_server_b(query)`, this operation will receive a stop-request when any of the following happens:

* `first_successful` algorithm will send a stop-request if `query_server_a(query)` completes successfully

* `when_all` algorithm will send a stop-request if the `load_file("some_file.jpg")` operation completes with an error or stopped result.

* `timeout` algorithm will send a stop-request if the operation does not complete within 5 seconds.

* `stop_when` algorithm will send a stop-request if the user clicks on the "Cancel" button in the user-interface.

* the parent operation consuming the `composed_cancellation_example()` sends a stop-request

Note that within this code there is no explicit mention of cancellation, stop-tokens, callbacks, etc. yet the example fully supports and responds to the various cancellation sources.

### Pipes

To facilitate an intuitive syntax for composition, most sender adaptors are *pipeable*; they can be composed (*piped*) together with `operator|`. This mechanism is similar to the `operator|` composition that C++ range adaptors support and draws inspiration from piping in *nix shells. Pipeable sender adaptors take a sender as their first parameter and have no other sender parameters.

`a | b` will pass the sender `a` as the first argument to the pipeable sender adaptor `b`. Pipeable sender adaptors support partial application of the parameters after the first. For example, all of the following are equivalent:

```cpp
execution::bulk(snd, N, [] (std::size_t i, auto d) {});
execution::bulk(N, [] (std::size_t i, auto d) {})(snd);
snd | execution::bulk(N, [] (std::size_t i, auto d) {});
```

Exmaple

`````{tab-set}
````{tab-item} Function call (nested)

```cpp
auto snd = execution::then(
             execution::transfer(
               execution::then(
                 execution::transfer(
                   execution::then(
                     execution::schedule(thread_pool.scheduler())
                     []{ return 123; }),
                   cuda::new_stream_scheduler()),
                 [](int i){ return 123 * 5; }),
               thread_pool.scheduler()),
             [](int i){ return i - 5; });
auto [result] = this_thread::sync_wait(snd).value();
// result == 610

```

````

````{tab-item} Function call (named temporaries)

```cpp
auto snd0 = execution::schedule(thread_pool.scheduler());
auto snd1 = execution::then(snd0, []{ return 123; });
auto snd2 = execution::transfer(snd1, cuda::new_stream_scheduler());
auto snd3 = execution::then(snd2, [](int i){ return 123 * 5; })
auto snd4 = execution::transfer(snd3, thread_pool.scheduler())
auto snd5 = execution::then(snd4, [](int i){ return i - 5; });
auto [result] = *this_thread::sync_wait(snd4);
// result == 610
```

````

````{tab-item} Pipe


```cpp
auto snd = execution::schedule(thread_pool.scheduler())
         | execution::then([]{ return 123; })
         | execution::transfer(cuda::new_stream_scheduler())
         | execution::then([](int i){ return 123 * 5; })
         | execution::transfer(thread_pool.scheduler())
         | execution::then([](int i){ return i - 5; });
auto [result] = this_thread::sync_wait(snd).value();
// result == 610
```

````

`````

Certain sender adaptors are not pipeable, because using the pipeline syntax can result in confusion of the semantics of the adaptors involved. Specifically, the following sender adaptors are not pipeable.

* `execution::when_all` and `execution::when_all_with_variant`: Since this sender adaptor takes a variadic pack of senders, a partially applied form would be ambiguous with a non partially applied form with an arity of one less.

* `execution::on`: This sender adaptor changes how the sender passed to it is executed, not what happens to its result, but allowing it in a pipeline makes it read as if it performed a function more similar to transfer.

### Range of senders - async sequence of data

Senders represent a single unit of asynchronous work. In many cases though, what is being modelled is a sequence of data arriving asynchronously, and you want computation to happen on demand, when each element arrives. This requires nothing more than what is in this paper and the range support in C++20. A range of senders would allow you to model such input as keystrokes, mouse movements, sensor readings, or network requests.

Given some expression R that is a range of senders, consider the following in a coroutine that returns an async generator type:

```cpp
for (auto snd : R) {
  if (auto opt = co_await execution::stopped_as_optional(std::move(snd)))
    co_yield fn(*std::move(opt));
  else
    break;
}
```

### All awaitable are senders

All generally awaitable types automatically model the sender concept. The adaptation is transparent and happens in the sender customization points, which are aware of awaitables. (By "generally awaitable" we mean types that don’t require custom `await_transform` trickery from a promise type to make them awaitable.)

For an example, imagine a coroutine type called `Task<T>` that knows nothing about senders. It doesn’t implement any of the sender customization points. Despite that fact, and despite the fact that the `this_thread::sync_wait` algorithm is constrained with the sender concept, the following would compile and do what the user wants:

```cpp
Task<int> doSomeAsyncWork();

int main() {
  // OK, awaitable types satisfy the requirements for senders:
  auto o = this_thread::sync_wait(doSomeAsyncWork());
}
```

Since awaitables are senders, writing a sender-based asynchronous algorithm is trivial if you have a coroutine task type: implement the algorithm as a coroutine. If you are not bothered by the possibility of allocations and indirections as a result of using coroutines, then there is no need to ever write a sender, a receiver, or an operation state.

### Senders can be awaitable

If you choose to implement your sender-based algorithms as coroutines, you’ll run into the issue of how to retrieve results from a passed-in sender. This is not a problem. If the coroutine type opts in to sender support -- trivial with the `execution::with_awaitable_senders` utility -- then a large class of senders are transparently awaitable from within the coroutine.

```cpp
template<class S>
  requires single-sender<S&> // see [exec.as.awaitable]
task<single-sender-value-type<S>> retry(S s) {
  for (;;) {
    try {
      co_return co_await s;
    } catch(...) {
    }
  }
}
```

## Receivers

A receiver is a callback that supports more than one channel. In fact, it supports three of them:

* `set_value`, which is the moral equivalent of an operator() or a function call, which signals successful completion of the operation its execution depends on;

* `set_error`, which signals that an error has happened during scheduling of the current work, executing the current work, or at some earlier point in the sender chain; and

* `set_stopped`, which signals that the operation completed without succeeding (`set_value`) and without failing (`set_error`). This result is often used to indicate that the operation stopped early, typically because it was asked to do so because the result is no longer needed.

Once an async operation has been started exactly one of these functions must be invoked on a receiver before it is destroyed.

```{note}
While the receiver interface may look novel, it is in fact very similar to the interface of std::promise, which provides the first two signals as set_value and set_exception, and it’s possible to emulate the third channel with lifetime management of the promise.
```

Receivers are what is passed as the second argument to `execution::connect`.

## Operation states

An operation state is an object that represents work. Unlike senders, it is not a chaining mechanism; instead, it is a concrete object that packages the work described by a full sender chain, ready to be executed. An operation state is neither movable nor copyable, and its interface consists of a single algorithm: `start`, which serves as the submission point of the work represented by a given operation state.

Operation states are not a part of the user-facing API of this proposal; they are necessary for implementing sender consumers like `execution::ensure_started` and `this_thread::sync_wait`, and the knowledge of them is necessary to implement senders, so the only users who will interact with operation states directly are authors of senders and authors of sender algorithms.

The return value of `execution::connect` must satisfy the operation state concept.

## execution::connect

`execution::connect` is a customization point which **connects senders with receivers**, resulting in an operation state that will ensure that if start is called that one of the completion operations will be called on the receiver passed to connect.

```cpp
execution::sender auto snd = some input sender;
execution::receiver auto rcv = some receiver;
execution::operation_state auto state = execution::connect(snd, rcv);

execution::start(state);
// at this point, it is guaranteed that the work represented by state has been submitted
// to an execution resource, and that execution resource will eventually call one of the
// completion operations on rcv

// operation states are not movable, and therefore this operation state object must be
// kept alive until the operation finishes
```
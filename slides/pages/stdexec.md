---
layout: cover
background: ./img/header-bg.svg
---

# Senders/Receivers (stdexec)

---

## Overview

- The **Senders/Receivers** model is a new asynchronous programming model introduced in C++26
- It provides a way to represent and compose asynchronous operations in a more structured and type-safe manner

---

## The Foundations of Structured Programming

- Abstractions as building blocks
  - Programs are constructed using abstractions
- Recursive decomposition
  - Problems can be broken down into smaller, similar sub-problems, which are then solved recursively
- Local reasoning
  - It should be possible to understand and verify parts of a program in isolation, without needing to know the entire program's context
- Single entry, single exit points
  - Control flow within any program segment should have only one entry point and one exit point

---

## Senders

- Senders **describe computations** - chunks of work that has **single entry point and single exit point**
- Senders are designed to be easily combined to build more complex asynchronous workflows - one can express concurrency as a graph, abstracting away explicit threads and synchronization mechanisms
- Senders are executed lazily; nothing happens until `sync_wait()` is invoked
- A sender may start on one thread and end on a different thread
- Avoiding synchronization - using senders the need for explicit synchronization (like mutexes) is often eliminated, which helps avoid common concurrency problems like races and performance bottlenecks
- Senders enable **structured concurrency**

---

## Senders

- A sender is said to send some values if a receiver connected to that sender will eventually receive said values
- Senders describe work and eventually produce a result
- This result can be of three types, corresponding to the three "exit points" of a sender:
  1. **A value**: The computation completes successfully with a value (or void)
  2. **An error**: The computation completes with an error
  3. **A stop signal**: The computation is stopped or cancelled

---

## Schedulers

- Senders operate with **schedulers** - abstractions that define where and how the work described by a sender is executed
- Schedulers are handles to **execution contexts**, such as threads, thread pools, or I/O services
- They dictate **where** particular work needs to be executed

---

## Hello World with Senders/Receivers

```cpp
static_thread_pool thread_pool{8};

execution::scheduler auto thd_pool = thread_pool.get_scheduler();

execution::sender auto begin = execution::schedule(thd_pool);

execution::sender auto hi = execution::then(begin, [] {
    std::cout << "Hello, World! Have an int." << std::endl;
    return 13;
});

execution::sender auto add_42 = execution::then(hi, [](int arg) {
    std::cout << "Adding 42 to " << arg << std::endl;
    return arg + 42;
});

auto [result] = execution::sync_wait(add_42).value();
```

<!-- Creating a thread pool with 8 threads -->
<!-- Getting a scheduler from the thread pool - handle to the execution context where work will be executed -->
<!-- Creating a sender that schedules work on the thread pool -->
<!-- Creating a sender that prints "Hello, World!" and returns an integer -->
<!-- Creating a sender that takes the integer from the previous sender, adds 42 to it, and returns the result -->
<!-- Executing the composed senders and waiting for the result synchronously - `sync_wait` blocks until the entire chain of senders completes -->

---

## Hello World using pipes

- Senders allow to chain asynchronous operations using pipes (`|` operator)

```cpp
static_thread_pool thread_pool{8};

execution::scheduler auto thd_pool = thread_pool.get_scheduler();

execution::sender auto work = execution::schedule(thd_pool)
    | execution::then([] {
        std::cout << "Hello, World! Have an int." << std::endl;
        return 13;
    })
    | execution::then([](int value) {
        std::cout << "Received: " << value << std::endl;
        return value + 42;
    });

auto [result] = execution::sync_wait(work).value();
```

---

## Managing execution contexts

- Senders can be used to manage and switch between different execution contexts seamlessly
  - `schedule(sch)` starts work in a given execution context
  - `starts_on(sch, sender)` ensures that the work described by `sender` starts executing in the context defined by `sch`
  - `continues_on(sch)` switches the execution to a different context - it can move the executions between threads or thread pools
  - `on(sch, sender)` is a combination of `starts_on` and `continues_on` - `starts_on(sch, sender) | continues_on(back_to_previous_context)`

---

## Managing execution contexts

<div class="text-code-07">

```cpp
exec::single_thread_context thread_context_1;
exec::single_thread_context thread_context_2;

execution::scheduler auto scheduler_1 = thread_context_1.get_scheduler();
execution::scheduler auto scheduler_2 = thread_context_2.get_scheduler();

auto step_1 = [](int i) {
    std::cout << "Work#1 on thread " << std::this_thread::get_id() << std::endl;
    return i * 2;
};

auto step_2 = [](int i) {
    std::cout << "Work#2 on thread " << std::this_thread::get_id() << std::endl;
    return i + 1;
};

execution::sender auto work =
    execution::starts_on(scheduler_1, execution::just(42))
        | execution::then(step_1)
        | execution::continues_on(scheduler_2)
        | execution::then(step_2);


auto [result] = execution::sync_wait(std::move(work)).value();
```

</div>

<!-- Defining workflow that starts on `scheduler_1`, performs `step_1`, switches to `scheduler_2`, and then performs `step_2` -->

---

## Waiting for multiple senders

<div class="text-code-08">

- Senders can be combined to wait for multiple asynchronous operations to complete
- `when_all(sender1, sender2, ...)` creates a new sender that completes when all the input senders have completed

```cpp
static_thread_pool thread_pool{3};

auto scheduler = thread_pool.get_scheduler();

auto func = [](int i) -> int {
    return i * i;
};

auto work = when_all(
    starts_on(scheduler, just(1) | then(func)),
    starts_on(scheduler, just(2) | then(func)),
    starts_on(scheduler, just(3) | then(func))
);

auto [i, j, k] = sync_wait(std::move(work)).value();
```

</div>

<!-- Creating a sender that waits for three computations to complete -->

---

## Splitting workflow

- Senders can be **multi-shot** or **single-shot**
  - Single-shot senders can be connected to a receiver only once (implementation of `connect()` accepts rvalue only)
  - Multi-shot senders can be connected to multiple receivers
- `split(sender)` creates a new sender that can be connected to multiple receivers, effectively allowing the workflow to be forked

---

## Splitting workflow

<div class="text-code-08">

```cpp
static_thread_pool thread_pool{8};

auto to_upper = [](const std::string& text) -> std::string {
    //...
};

auto to_lower = [](const std::string& text) -> std::string {
    //...
};

scheduler auto sch = thread_pool.get_scheduler();

sender auto common = just("Hello World!"s) | split();

sender auto pipe_1 = starts_on(sch, common) | then(to_upper);
sender auto pipe_2 = starts_on(sch, common) | then(to_lower);

sender auto results = when_all(pipe_1, pipe_2);

auto [upper, lower] = sync_wait(std::move(results)).value();
```
</div>

<!-- Splitting a sender to create two independent workflows -->
<!-- Creating two separate pipelines that process the same input concurrently -->
<!-- Gathering results from both pipelines -->
<!-- Starting the execution and waiting for completion -->

---

## Executing in bulk

- When we want to perform the same operation on multiple items concurrently, we can use `bulk(sender, n, invocable)`
- It creates a new sender that, when started, will invoke the provided invocable `n` times, each time passing a different `index` (from `0` to `n-1`) to the `invocable(index, ...)`

---

## Executing in bulk

```cpp
static_thread_pool thread_pool{8};

std::vector<int> data_1 = {1, 2, 3, 4, 5};
std::vector<int> data_2 = {10, 20, 30, 40, 50};
std::vector<int> results(data_1.size());

scheduler auto sch = thread_pool.get_scheduler();

sender auto work =
    schedule(sch)
    | bulk(data_1.size(), [&](size_t i) {
        results[i] = data_1[i] + data_2[i];
    });

sync_wait(std::move(work));
```

---

## Shape of Sender

- The work represented by a sender has one entry point and one exit point, usually called **completion** (or **completion signal**)
- A sender can complete in one of three ways:
  1. `set_value(auto...)` - the operation is completed successfully and produces output values (or void)
  2. `set_error(auto err)` - the operation encountered an error and produces an error value (`std::exception_ptr`, `std::error_code`, or any user-defined error type)
  3. `set_stopped()` - the operation was stopped or cancelled before completion
- Sender is a generalization of a function

---

## Examples of Senders

- A sender can encapsulate a concurrent sort algorithm (which may run on the GPU or on the CPU) - an example of using senders to speed up programs
- A sender can encapsulate the processing of an image; the processing can be done on a single thread, on multiple threads, or on GPUs - an example showing that concurrency concerns are hidden
- A sender can encapsulate a sleep operation; the sender completes when the sleep period ends but doesn't keep any thread busy - an example of asynchrony
- A sender can encapsulate the wait for the results of a remote procedure call over the network, while not keeping the local threads busy - another example of asynchrony

---

## Sender algorithms

- The C++26 standard defines several sender algorithms to be used as primitives for building more complex senders
- These algorithms can be grouped into three categories:
  - **Sender factories**
  - **Sender adaptors**
  - **Sender consumers**

---

## Sender factories

- They produce senders without requiring any other senders
- Algorithms in the standard:
  - `schedule()`
  - `just()`
  - `just_error()`
  - `just_stopped()`
  - `read_env()`

---

## Sender adaptors

<div class="text-code-09">

- Given one or more senders, they return senders based on the provided senders:
  - `starts_on(scheduler auto sch, sender auto snd)`
  - `continues_on(sender auto input, scheduler auto sch)`
  - `on(scheduler auto sch, sender auto snd)`
  - `then(sender auto input, invocable<values-sent-by-input> auto func)`
  - `upon_error(sender auto input, invocable<errors-sent-by-input> auto func)`, `upon_stopped(sender auto input, invocable auto func)`
  - `let_value()`, `let_error()`, `let_stopped()`
  - `bulk(sender auto input, integral auto n, invocable<size_t, values-sent-by-input> auto func)`
  - `split(sender auto snd)`
  - `when_all(sender auto... snds)`
  - `into_variant(sender auto snd)`
  - `stopped_as_optional(sender auto snd)`, `stopped_as_error(sender auto snd, error_type err)`

</div>

---

## Sender consumers

- They consume senders but don't produce any senders
  - `sync_wait(sender auto snd) -> optional<tuple<values-sent-by(snd)>>`
  - `sync_wait_with_variant()`

---

## sync_wait()

- `sync_wait()` takes a sender and performs the following steps:
  - submits the work described by the given sender
  - blocks the current thread until the sender's work is finished
  - returns the result of the sender's work in the appropriate form to the caller:
    - returns an optional tuple of values - those that the given sender completes with - if the sender completes with set_value()
    - throws the received error if the sender completes with set_error()
    - returns an empty optional if the given sender completes with a stopped signal

---

## let_* adaptors

<div class="text-code-09">

- The `let_value()` algorithm is similar to the `then()` algorithm, but the given functor is expected to return a sender
- This is the *monadic bind* operation for senders, i.e., a fundamental building block for senders
  - it is similar to the `optional<T>::and_then()` function

```cpp
auto process_image_sender(const Image& img) -> sender auto {
    // ...
}

Image load();
void save(const Image& img);

sender auto workflow =
    just()
    | then(load)
    | let_value([](Image img) { return process_image_sender(img); })
    | then(save);
```
</div>
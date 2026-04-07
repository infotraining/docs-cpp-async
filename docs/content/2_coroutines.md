# Coroutines

> "Subroutines are special cases of ... coroutines." 
> – Donald Knuth

## Introduction To Coroutines

### Functions and the Call Stack

When a regular function is called, the system allocates a **stack frame** on the call stack to hold its local variables and execution state. When the function returns, the stack frame is reclaimed and control transfers back to the caller.

```{code-block} cpp
int compute_something(int x, int y) // parameters stored in the stack frame
{
    int result = x + y + 665; // local variable stored in the stack frame
    return result;
}

int main()
{
    int a = 10;
    int b = 20;
    int c = compute_something(a, b); // call creates a stack frame
}
```

This is called the **run-to-completion** model: once a function starts executing it runs until it returns, with no interruption possible.


### What Is a Coroutine?

A **coroutine** is a generalisation of a function that can **suspend** its execution and **resume** later from exactly where it left off.

The concept was first described by Melvin Conway in 1958 and is supported natively in C++20.

When a coroutine **suspends**:
- Its local variables are preserved.
- The instruction pointer (the position in the code) is saved.
- Control returns to the caller or to some other piece of code.

When a coroutine **resumes**:
- Local variables are restored to their saved values.
- Execution continues from the suspension point.

### Coroutine Frame

To support suspension and resumption a coroutine must keep its state alive across multiple calls. This is done via a **coroutine frame**:

- Unlike a function's stack frame, the coroutine frame is allocated **on the heap** so it persists across suspensions.
- It stores local variables, function parameters (copies), and the current *instruction pointer*.
- The frame is created when the coroutine is first invoked and destroyed when the coroutine finishes or is explicitly destroyed.

## Motivation: From Callbacks to Coroutines

### Synchronous blocking code

The simplest approach to writing server logic is to call I/O operations synchronously. Each call blocks the thread until the operation completes.

```{code-block} cpp
void handle_request(Connection& conn)
{
    std::string request = conn.read_request();      // blocking
    auto parsed_request = parse_request(request);   // CPU-bound
    auto data = database.query(parsed_request.id);  // blocking
    auto response = compute_response(data);         // CPU-bound
    conn.write(response);                           // blocking
}
```

This is easy to read but wastes a whole thread while I/O is in progress.

### Callback-based asynchronous code

To avoid blocking, developers traditionally used callbacks. The downside is deeply nested "callback hell" that is hard to read and reason about.

```{code-block} cpp
void handle_request(Connection& conn)
{
    conn.async_read([&conn](std::string request) {
        auto parsed = parse_request(request);
        database.async_query(parsed.id, [&conn](auto data) {
            auto response = compute_response(data);
            conn.async_write(response, [&conn]() {
                // request complete
            });
        });
    });
}
```

### The coroutine solution

Coroutines let you write asynchronous code that *looks* like synchronous code, without blocking a thread.

```{code-block} cpp
Task<void> handle_request(Connection& conn)
{
    std::string request = co_await conn.async_read();              // non-blocking
    auto parsed_request = parse_request(request);                  // CPU-bound
    auto data = co_await database.async_query(parsed_request.id);  // non-blocking
    auto response = compute_response(data);                        // CPU-bound
    co_await conn.async_write(response);                           // non-blocking
}
```

Readability and maintainability are preserved while the thread is free to do other work during each `co_await`.

## Coroutines in C++20

A function is a coroutine in C++20 when its body contains at least one of the following keywords:

| Keyword | Purpose |
|---------|---------|
| `co_return` | Finishes the coroutine, optionally returning a value |
| `co_await` | Suspends until an asynchronous operation completes |
| `co_yield` | Produces a value and suspends, then can be resumed for the next value |

C++20 coroutines are **stackless**:
- The coroutine state lives in a separate heap-allocated frame, not on the thread stack.
- This makes it practical to have **more than one million** concurrent coroutines.

### co_await

`co_await` suspends the coroutine until the awaited operation completes. The coroutine saves its state, yields control, and resumes automatically when the result is ready.

```{code-block} cpp
Task<std::string> download_page(std::string url)
{
    auto response = co_await http_get(url); // suspend here; resume when download finishes
    return response.body;
}
```

### co_yield

`co_yield` produces a value and suspends the coroutine. This pattern creates *generators* — coroutines that produce a sequence of values on demand.

```{code-block} cpp
Generator<int> count_up_to(int max)
{
    for (int i = 1; i <= max; ++i)
        co_yield i; // produce i, then suspend until next value is requested
}
```

### co_return

`co_return` completes the coroutine and optionally delivers a final result. Unlike a plain `return`, it interacts with the coroutine machinery to properly finalise the coroutine's state.

```{code-block} cpp
Task<int> compute(int a, int b)
{
    int result = 42 + a + b;
    co_return result; // complete the coroutine, deliver result
}
```

### Awaitables and Awaiters

When we want to suspend a coroutine using `co_await expr`, the expression following `co_await` must be an **awaitable** - something that knows how to suspend and resume our coroutine. The awaitable produces an **awaiter** object that implements three methods:

- `await_ready()` - returns `true` if the result is already available (no suspension needed)
- `await_suspend(coroutine_handle)` - called if suspension is needed; receives a handle to the coroutine so it can schedule resumption
- `await_resume()` - called when the coroutine resumes; returns the result of the `co_await` expression

## Mechanics of Coroutines

### Code transformation

Code that uses coroutines is transformed by the compiler into a state machine that manages the coroutine's execution and suspension points.

`````{tab-set}
````{tab-item} Coroutine code

```cpp
ReturnType coroutine_f(Params)
{
    function_body();
}
```

````

````{tab-item} Transformed code

```{code-block} cpp
---
lineno-start: 1
emphasize-lines: 3,9,14,20
---
ReturnType coroutine_f(Params)
{
    using promise_type = std::coroutine_traits<ReturnType, Params>::promise_type // [1]

    promise_type promise(promise_constructor_arguments);

    try
    {
      co_await promise.initial_suspend(); // [2] initial suspend point
      function_body();  
    }
    catch(...)
    {
        if (!initial_await_resume_called) // [3]
            throw;
        promise.unhandled_exception();
    }

final-suspend: // final suspend point
    co_await promise.final_suspend(); // [4]
}
```
````
`````

where:

* [1] `promise_type` denotes the **promise** type
* [2] the `co_await` expression that calls `promise.initial_suspend()` is the **initial suspend point**
* [3] the `initial_await_resume_called` flag is initially `false` and set to `true` just before evaluating the `await_resume` expression at the *initial suspend point*
* [4] the `co_await` expression that calls `promise.final_suspend()` is the **final suspend point**

### Coroutines - lifetime

#### Coroutine call

Calling a coroutine forces the compiler to construct a *coroutine stack frame* that contains:

* formal parameters
* coroutine local variables
* execution state when the coroutine is suspended (registers, instruction pointers, etc.)
* the *promise* object, which controls coroutine behavior and is used to return value(s) to the caller

In general, the coroutine stack frame must be dynamically allocated on the heap. Heap allocation prevents loss of local state when the coroutine suspends.

Allocation uses `operator new` by default.

In some cases, the compiler can optimize away dynamic allocation.

```{note}
The coroutine frame is created before the coroutine starts executing.

The compiler passes a handle (pointer) to the coroutine frame back to the caller.
```

### Destruction of coroutine state

Coroutine state is destroyed when:

* the coroutine exits - control flow leaves the coroutine
* or when `destroy()` is called on a coroutine handle that refers to a running coroutine

```{important}
* The return object is created before `initial_suspend()` runs, so it is available even if the coroutine suspends immediately

* `final_suspend()` determines whether the coroutine frame persists after completion 
  * if it returns `suspend_always`, you must manually destroy the coroutine; 
  * if it returns `suspend_never`, the frame is destroyed automatically

* If `destroy()` is called for a coroutine that is not in the suspended state, the program has *undefined behavior (UB)*.
```

### Coroutine handle

A coroutine handle is a lightweight object that wraps a pointer to a coroutine frame allocated on the heap. It lets you manipulate a coroutine from outside the coroutine itself. Using a handle, you can resume a suspended coroutine with `resume()` or destroy it with `destroy()`.

There are two common handle cases:

1. Handle for coroutines that return `void` - *untyped handle*.
2. Handle for coroutines that return a value - *typed handle*.

#### Untyped handle - std::coroutine_handle<>

The most basic handle is `std::coroutine_handle<>` (equivalent to `std::coroutine_handle<void>`), which provides no access to the *promise* object. It is a simple wrapper around a pointer to the coroutine frame.

```cpp
template <typename Promise = void>
struct coroutine_handle;

template <>
struct coroutine_handle<void> // no promise access
{ 
    coroutine_handle() noexcept; // it allows to construct empty coroutine_handle
    coroutine_handle(nullptr_t) noexcept;
    coroutine_handle& operator=(nullptr_t) noexcept;
    explicit operator bool() const noexcept; // conversion - if handle is empty
        
    static coroutine_handle from_address(void* _Addr) noexcept; // conversion function - allows to convert a pointer to handle
    void* address() const noexcept; // returns pointer to coroutine handle
  
    void operator()() const; // it allows to resume the execution
    void resume() const; // the same as operator()() - resumes the execution
  
    void destroy(); // allows to manually destroy the coroutine
        
    bool done() const; // tests whether the coroutine has completed execution
private:
    void* ptr_; // a pointer to a coroutine stack frame
};
```

#### Typed handle - std::coroutine_handle\<Promise\>

2. Handle for coroutines that return a value. In this case a specialization of `coroutine_handle` is provided:

```cpp
template <typename Promise>
struct coroutine_handle : coroutine_handle<void>
{
    Promise& promise() const noexcept; // gets the promise back

    static coroutine_handle from_promise(Promise&) noexcept; // returns a handle from the promise object 
};
```

```{note}
Because the *promise* object is constructed in the coroutine frame:

* given a `promise<T>` object, we can obtain the corresponding `coroutine_handle<T>`
* and conversely, given a coroutine handle we can obtain a reference to `promise<T>`
```

#### Getting a coroutine handle (access to the coroutine frame)

You can obtain a `coroutine_handle` in two ways:

* It is passed to the awaiter `await_suspend(std::coroutine_handle<Promise>)` method during a `co_await` expression - after the coroutine is suspended (the passed handle can be treated as a continuation-passing handle: [continuation-passing](https://en.wikipedia.org/wiki/Continuation-passing_style))
* Given a reference to the coroutine *promise* object, you can reconstruct its handle using `coroutine_handle<Promise>::from_promise(*this)`. Usually it is done inside the *promise* object itself, for example in `get_return_object()`

```{note}
* `std::coroutine_handle<>` is NOT an RAII object. You must call `destroy()` manually to release resources (the coroutine frame). 

* You can treat a handle as the equivalent of a `void*`. 

* This design is motivated by performance. RAII would be too expensive (for example, reference counting).
```

### Promise object

The **promise** object provides control over the coroutine. It is a customization point responsible for:

* returning the coroutine interface object at the start
* deciding whether to suspend immediately after start
* deciding whether to suspend at the end
* handling exceptions during coroutine execution
* handling values returned from the coroutine

This object is allocated in the coroutine frame.

The *promise* interface includes:

- constructor/destructor (RAII)
- `get_return_object()` - used to initialize the object returned to the coroutine caller
- `get_return_object_on_allocation_failure()`
- functions that implement result/exception transfer from the coroutine to the caller
  - `co_return`: 
    - `return_value()` when the coroutine returns a value 
    - `return_void()` when the coroutine returns `void`
  - `co_yield`
    - `yield_value()` when the coroutine yields a value

## co_await

The unary `co_await` operator suspends the coroutine and returns control to the caller.

The `co_await` expression is transformed (expanded) by the compiler into the following code:

`````{tab-set}
````{tab-item} Expression
```cpp
auto r = co_await expr;
```
````
````{tab-item} Transformed code
```{code-block} cpp
---
lineno-start: 1
emphasize-lines: 4,7,12
---
auto r = {
  auto&& awaiter = expr;

  if (!awaiter.await_ready())
  {
    __builtin_coro_save(); // frame->suspend_index = n;
    awaiter.await_suspend(<coroutine_handle>);
    __builtin_coro_suspend(); // jmp epilog
  }

resume_label_n:
  awaiter.await_resume();
};
```
````
`````

The expression following `co_await` must be an *awaitable* (a type that stisfies the `awaitable` concept). The compiler produces an *awaiter* object from the awaitable expression and calls its `await_ready()`, `await_suspend()`, and `await_resume()` methods to manage suspension and resumption.

If a coroutine is suspended at a `co_await` expression and later resumed (via `resume()`), the resume point is directly before the call to `awaiter.await_resume()`.

### Awaitable object

To be awaited, a type must satisfy the `awaitable` concept:

```cpp
// simplified awaitable concept - for demonstration purposes only

template <typename T>
concept awaitable = requires(T obj) {
  { obj.await_ready(); } -> std::convertible_to<bool>;
  { obj.await_suspend(coroutine_handle<>); };
  { obj.await_resume(); }
};
```

When the *awaiter* object is created, `awaiter.await_ready()` is called.

* If it returns `true`, there is no need to wait or suspend the coroutine (the awaited result is already available) and execution continues synchronously.

* If it returns `false`, the coroutine is suspended and `awaiter.await_suspend(coroutine_handle<P>)` is called.

Inside `await_suspend(coroutine_handle<P>)`, the suspended coroutine state is accessible through the coroutine handle passed as an argument. The primary responsibility of this function is to schedule the coroutine for resumption (via `resume()`) on a chosen scheduler (*executor*), or to destroy it.

If `await_suspend()`:

* returns `void` - execution returns immediately to the caller/resumer of the current coroutine and the current coroutine is suspended

* returns `bool`:
  * when `true` execution returns immediately to the caller/resumer of the current coroutine
  * when `false` the current coroutine is resumed

* returns a `coroutine_handle<P>` to another coroutine, that coroutine is resumed (via `handle.resume()`) - this may lead to chained resumption of the current coroutine. It allows you to implement *continuation-passing* style of asynchronous programming, where the current coroutine is resumed by another coroutine that is scheduled to run when the awaited operation completes. See [continuation-passing](https://en.wikipedia.org/wiki/Continuation-passing_style) for more details.

* throws an exception, the exception is caught, the current coroutine is resumed, and the exception is rethrown

Finally, `awaiter.await_resume()` is called and its result is the value of the entire `co_await expr` expression.

```{note}
Because the coroutine is fully suspended before entering `awaiter.await_suspend()`, the function can safely transfer the coroutine handle between threads without additional synchronization. For example, you can define a callback that is scheduled on a thread pool when an asynchronous I/O operation completes. In that case, because the current coroutine may have been resumed and destroyed, the `await_suspend()` implementation should treat `*this` as destroyed and avoid accessing it after transferring the coroutine handle to other threads.
```

#### Simple awaitable types

##### suspend_always

```cpp
struct suspend_always
{
  bool await_ready() noexcept
  {
    return false; // I don't have a value yet
  }

  void await_suspend(coroutine_handle<>) noexcept { }

  void await_resume() noexcept { }
};
```

##### suspend_never

```cpp
struct suspend_never
{
  bool await_ready() noexcept
  {
    return true;
  }

  void await_suspend(coroutine_handle<>) noexcept { }

  void await_resume() noexcept { }
};
```

## co_return

To return a value from a coroutine, use `co_return`.

When a coroutine reaches `co_return`, it performs the following steps:
* calls `promise.return_void()` for: 
  - `co_return;`
  - `co_return expr;` when `expr` has type `void`
* or calls `promise.return_value(expr)` for `co_return expr;` where `expr` is not `void`
* destroys all automatic (local) variables in reverse order of initialization
* calls `promise.final_suspend()` and awaits its result using `co_await`

```{attention}
Exiting a coroutine without executing `co_return`, when the *promise* type does not provide `Promise::return_void()`, results in *undefined behavior*.
```

## co_yield

The *yield* expression has the following form:

`````{tab-set}
````{tab-item} Expression
```cpp
co_yield expression;
```
````
````{tab-item} Expanded code
```cpp
co_await p.yield_value(expression);
```
````
`````

## Customization points for a coroutine

Coroutines in C++20 are highly customizable. The behavior of a coroutine is determined by the *promise* type, which is a user-defined type that the compiler uses to manage the coroutine's state and interactions.

The programmer can customize:

* how to allocate the coroutine frame
* what happens when the coroutine is called
* what happens when the coroutine returns (either by normal return or via unhandled exception)
* behavior of any `co_await` or `co_yield` expression within the coroutine body

### Allocating a coroutine frame

The compiler needs to allocate memory for a **coroutine frame**.

#### Default allocation

By default, the compiler uses `operator new` to allocate memory for the coroutine frame on the heap. The frame size depends on:

* the number and size of local variables in the coroutine body
* parameters which are copied into the frame
* the size of the *promise* object - which is also stored in the frame
* compiler-generated state for managing suspension and resumption

#### Heap Allocation eLision Optimization (HALO)

Compilers can sometimes eliminate coroutine frame allocation entirely through HALO (Heap Allocation eLision Optimization).

When the compiler can prove that:

* The coroutine’s lifetime is contained within the caller’s lifetime
* The frame size is known at compile time

…​it may allocate the frame on the caller’s stack instead of the heap.


HALO is most effective when:

* Coroutines are awaited immediately after creation
* Optimization is enabled (-O2 or higher)

```{code-block} cpp
// HALO might apply here because the task is awaited immediately
co_await compute_something();

// HALO cannot apply here because the task escapes
auto task = compute_something();
store_for_later(std::move(task));
```

#### Custom allocators

Promise type can also customize allocation by defining `operator new` and `operator delete`. This allows you to use custom memory pools or other allocation strategies for coroutine frames.

```{code-block} cpp
struct promise_type
{
    // Custom allocation
    static void* operator new(std::size_t size)
    {
        return my_allocator.allocate(size);
    }

    static void operator delete(void* ptr, std::size_t size)
    {
        my_allocator.deallocate(ptr, size);
    }

    // ... rest of promise type
};
```

The promise’s `operator new` receives only the frame size. To access allocator arguments passed to the coroutine, use the leading allocator convention with `std::allocator_arg_t` as the first parameter.


### Initial suspend point

When a coroutine is called, the compiler first constructs the *promise* object and then executes the `co_await promise.initial_suspend();` statement.   

This allows the author of the `promise_type` to control whether the coroutine should suspend before executing the coroutine body that appears in the source code or start executing the coroutine body immediately.

If the coroutine suspends at the initial suspend point then it can be later resumed or destroyed at a time of your choosing by calling `resume()` or `destroy()` on the coroutine’s coroutine_handle.

The result of the `co_await promise.initial_suspend()` expression is discarded so implementations should generally return void from the `await_resume()` method of the awaiter.

For many types of coroutines, the `initial_suspend()` method either returns:

* `std::suspend_always` (if the operation is lazily started) or
* `std::suspend_never` (if the operation is eagerly started)

### Returning to the caller

When the coroutine function reaches its first `<return-to-caller-or-resumer>` point (or if no such point is reached then when execution of the coroutine runs to completion) then the return-object returned from the `get_return_object()` call is returned to the caller of the coroutine.

The type of the `return-object` doesn’t need to be the same type as the `ReturnType` of the coroutine function. An implicit conversion from the `return-object` to the `ReturnType` of the coroutine is performed if necessary.

### Returning from the coroutine using `co_return`

[See co_return behavior](co-return)

### Handling exceptions for the coroutine body

If an exception propagates out of the coroutine body then the exception is caught and the `promise.unhandled_exception()` method is called inside the `catch` block.

Implementations of this method typically call `std::current_exception()` to capture a copy of the exception to store it away to be later re-thrown in a different context.

### The final-suspend point

Once execution exits the user-defined part of the coroutine body and the result has been captured via a call to `return_void()`, `return_value()` or `unhandled_exception()` and any local variables have been destructed, the coroutine has an opportunity to execute some additional logic before execution is returned back to the caller/resumer.

The coroutine executes the `co_await promise.final_suspend();` statement.

This allows the coroutine to execute some logic, such as publishing a result, signalling completion or resuming a continuation. It also allows the coroutine to optionally suspend immediately before execution of the coroutine runs to completion and the coroutine frame is destroyed.

```{note}
It is undefined behavior to `resume()` a coroutine that is suspended at the `final_suspend` point. The only thing you can do with a coroutine suspended here is `destroy()` it.
```

```{note}
It is recommended that you structure your coroutines so that they do suspend at `final_suspend` where possible. 

This is because this forces you to call `.destroy()` on the coroutine from outside of the coroutine (typically from some RAII object destructor) and this makes it much easier for the compiler to determine when the scope of the lifetime of the coroutine-frame is nested inside the caller. This in turn makes it much more likely that the compiler can elide the memory allocation of the coroutine frame.
```

### Customizing the behavior of co_await

C++ coroutines provide two major customization points that control how an expression is awaited: `await_transform` and `operator co_await`. Although both influence the behavior of `co_await`, they operate at different levels and serve different purposes. Understanding their relationship is essential for designing coroutine frameworks and awaitable types.

#### 1. The Awaiting Pipeline

When the compiler encounters:

```cpp
co_await expr;
```

it expands this through a fixed sequence of steps:

1. **`promise.await_transform(expr)`**  
   If defined, this rewrites the awaited expression.

2. `operator co_await` **lookup**
   Applied to the transformed expression.

3. **Awaiter protocol**  
   The resulting object must provide:
   - `await_ready()`
   - `await_suspend()`
   - `await_resume()`

This pipeline is strict: **`await_transform` always runs before `operator co_await`**.

#### 2. The Coroutine-Level Hook - `await_transform`

`await_transform` is a method on the coroutine’s **promise type**. It allows the coroutine author to intercept and rewrite every `co_await` inside the coroutine body.

- Affects **all** `co_await` expressions in the coroutine.
- Can wrap, replace, or forbid awaiting specific expressions.
- Enables powerful behaviors such as:
  - scheduling
  - thread switching
  - logging
  - injecting custom awaitable wrappers

#### Example

```cpp
auto await_transform(auto&& awaitable) {
    return schedule_on_pool(std::forward<decltype(awaitable)>(awaitable));
}
```

This forces every `co_await` to run on a thread pool.

---

#### 3. The Awaitable-Level Hook - `operator co_await`

`operator co_await` is defined on the **awaited type** itself. It allows the type author to specify how that object should behave when awaited.

- Affects only that specific type.
- Returns an awaiter object.
- Defines the mechanics of suspension for that type.

#### Example

```cpp
struct MyAwaitable {
    auto operator co_await() const {
        return MyAwaiter{...};
    }
};
```

This gives the type full control over its await behavior.

#### 4. Which One Matters More?

Both are important, but in different ways:

| Aspect | `await_transform` | `operator co_await` |
|--------|--------------------|----------------------|
| Defined by | Coroutine author (promise type) | Awaitable type author |
| Scope | All awaits in the coroutine | Only this type |
| Power | Can rewrite the entire await expression | Only adapts its own type |
| Priority | Runs **first** | Runs after `await_transform` |
| Typical use | Schedulers, executors, async frameworks | Custom awaitables |

Shortly:

- **`await_transform` is more powerful** because it can override everything else.
- **`operator co_await` is more common** because most awaitable types implement it.

Think of the two hooks like this:

- **`await_transform`** is the coroutine’s *interceptor*.  
  It can rewrite or wrap the awaited expression before anything else happens.

- **`operator co_await`** is the awaitable’s *adapter*.  
  It defines how that specific type behaves when awaited.

If both exist, the flow is:

```
co_await expr
     ↓
promise.await_transform(expr)
     ↓
operator co_await on the result
     ↓
awaiter protocol
```

### Customizing the behavior of co_yield

The final thing you can customize through the promise type is the behavior of the `co_yield` keyword.

If the `co_yield` keyword appears in a coroutine then the compiler translates the expression `co_yield <expr>` into the expression `co_await promise.yield_value(<expr>)`. 

The promise type can therefore customize the behavior of the `co_yield` keyword by defining one or more `yield_value()` methods on the promise object.

## Exception handling

Exceptions in coroutines require special handling because a coroutine can suspend and resume across different call stacks.

### The Exception Flow

When an exception is thrown inside a coroutine and not caught:

* The exception is caught by an implicit `try-catch` surrounding the coroutine body

* `promise.unhandled_exception()` is called while the exception is active

* After `unhandled_exception()` returns, `co_await promise.final_suspend()` executes

* The coroutine completes (suspended or destroyed, depending on `final_suspend`)

### Options for handling exceptions

The `promise.unhandled_exception()` method can be implemented in various ways, depending on the desired behavior:

#### Terminate the program

```cpp
void unhandled_exception()
{
    std::terminate();
}
``` 

#### Store the exception for later retrieval

```cpp
std::exception_ptr exception_;          

void unhandled_exception()
{
    exception_ = std::current_exception();
}
```

#### Rethrow the exception immediately

```cpp
void unhandled_exception()
{
    throw;  // propagates to whoever resumed the coroutine
}
```

#### Swallow the exception

```cpp
void unhandled_exception()
{
    // silently ignored - almost always a mistake
}
```

### Initialization Exceptions

Exceptions thrown before the first suspension point (before `initial_suspend` completes) propagate directly to the caller without going through `unhandled_exception()`. 

If `initial_suspend()` returns `suspend_always`, the coroutine suspends before any user code runs, avoiding this edge case.

## Symetric Transfer

When a coroutine completes or awaits another coroutine, control must transfer somewhere. The naive approach by simply calling `coro_handle.resume()` encounters a problem: each nested coroutine adds a frame to the call stack. With deep nesting, stack overflow may occure.

**Symmetric transfer** solves this by returning a coroutine handle from `await_suspend()`. Instead of resuming the target coroutine via a function call, the compiler generates a tail call that transfers control without growing the stack.

### Stack Accumulation

Let's consider a chain of coroutines where each awaits the next:

```{code-block} cpp
task<> a() { co_await b(); }
task<> b() { co_await c(); }
task<> c() { co_return; }
```
Without symmetric transfer, when `a` awaits `b`:

1. `a` calls into the awaiter's `await_suspend()`
2. `await_suspend()` calls `b.coro_handle.resume()`
3. `b` runs, calls into its awaiter's `await_suspend()`
4. That calls `c.coro_handle.resume`
5. The stack now has frames for `a`'s suspension, `b`'s suspension and `c`'s suspension

### The Solution: Symmetric Transfer

`await_suspend()` can return a `coroutine_handle` to the next coroutine to resume. The compiler generates a tail call to that handle, transferring control directly without adding a new stack frame.

```{code-block} cpp
std::coroutine_handle<> await_suspend(std::coroutine_handle<> current_hndl)
{
    continuation_hndl_ = current_hndl; // store continuation for later resumption
    
    return next_coroutine_hndl_;  // return handle to resume (instead of calling resume())
}
```

When `await_suspend()` returns a handle, the compiler generates code that performs a tail call to that handle, effectively transferring control to the next coroutine without growing the stack. The generated code looks like this:

```{code-block} cpp
auto next_hndl = awaiter.await_suspend(current_hndl);
if (next_hndl != std::noop_coroutine())
{
    next_hndl.resume(); // tail call - no new stack frame
}
```

```{important}
Returning a handle from `await_suspend()` is a powerful optimization but requires careful design to avoid issues like:
- Ensuring the returned handle is valid and properly scheduled
- Avoiding returning a handle that could lead to resuming a destroyed coroutine
```
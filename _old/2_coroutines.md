# Coroutines

> "Subroutines are special cases of ... coroutines." –Donald Knuth"

## Function vs. Coroutine

A coroutine is a generalization of a function (*subroutine*):

**Function / Subroutine**:

* can be called by the caller
* can return control to the caller using a `return` statement

**Coroutine** has both of the above properties, and also:

* can suspend its execution and transfer control to the caller
* once suspended, can resume execution


![](img/function-coroutine.png)

In C++20, a coroutine is a function whose body contains at least one of the following:

* `co_return`

```cpp
Lib::Lazy<int> f() 
{
    co_return 7;
}
```

* or `co_await`

```cpp
Lib::Task<> tcp_echo_server(socket_t& socket) 
{
    using buffer_t = std::array<uint8_t, 1024>;
    buffer_t buffer{};
    for (;;) 
    {
    size_t bytes_read = co_await socket.async_read_some(buffer.data());
    co_await async_write(socket, buffer.data(), bytes_read));
    }
}
```

* or `co_yield`

```cpp
Lib::Generator<int> iota(int n = 0) 
{
while(true)
    co_yield n++;
}
```

### Code transformation

The compiler transforms coroutine code into a standard-defined state machine:

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

```cpp
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

## Coroutines - lifetime

### Function call

When a function is called, the compiler constructs a *stack frame* that contains:

* function call arguments
* local variables
* return value storage
* saved CPU register state

### Coroutine call

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

```{note}
If `destroy()` is called for a coroutine that is not in the suspended state, the program has *undefined behavior (UB)*.
```

## Coroutine handle

A coroutine handle is an object that wraps a pointer to a coroutine frame allocated on the heap. It lets you manipulate a coroutine from outside the coroutine itself. Using a handle, you can resume a suspended coroutine with `resume()` or destroy it with `destroy()`.

There are two common handle cases:

1. Handle for coroutines that return `void`:

```cpp
template <typename _PromiseT = void>
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

### Getting a coroutine handle (access to the coroutine frame)

You can obtain a `coroutine_handle` in two ways:

* it is passed to the awaiter `await_suspend()` method during a `co_await` expression - after the coroutine is suspended (the passed handle can be treated as a continuation-passing handle: [continuation-passing](https://en.wikipedia.org/wiki/Continuation-passing_style))
* given a reference to the coroutine *promise* object, you can reconstruct its handle using `coroutine_handle<Promise>::from_promise()`

```{note}
`std::coroutine_handle` is NOT an RAII object. You must call `destroy()` manually to release resources (the coroutine frame). 

You can treat a handle as the equivalent of a `void*`. 

This design is motivated by performance. RAII would be too expensive (for example, reference counting).
```

## Promise object

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
  - `co_return`: `return_value()`, `return_void()`
  - `co_yield`: `yield_value()`

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
```cpp
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

The expression following `co_await` is converted into an *awaitable* object.

If a coroutine is suspended at a `co_await` expression and later resumed (via `resume()`), the resume point is directly before the call to `awaiter.await_resume()`.

### Awaitable object

To be awaited, a type must satisfy the `awaitable` concept:

```cpp
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

* returns a `coroutine_handle<P>` to another coroutine, that coroutine is resumed (via `handle.resume()`) - this may lead to chained resumption of the current coroutine

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

The interface of the **promise** object specifies methods for customizing the behavior of the coroutine. The library writer can customize:

* what happens when the coroutine is called
* what happens when the coroutine returns (either by normal return or via unhandled exception)
* behavior of any `co_await` or `co_yield` expression within the coroutine body

### Allocating a coroutine frame

The compiler needs to allocate memory for a coroutine frame. To achieve this it generates a call to `operator new`.
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

The promise type can optionally customize the behavior of every `co_await` expression that appears in the body of the coroutine.

By simply defining a method named `await_transform()` on the promise type, the compiler will then transform every `co_await <expr>` appearing in the body of the coroutine into `co_await promise.await_transform(<expr>)`.

This has a number of important and powerful uses:
* It lets you enable awaiting types that would not normally be awaitable.
* It lets you disallow awaiting on certain types by declaring `await_transform` overloads as ` = delete`.

  ```cpp
  template<typename T>
  class generator_promise
  {
    ...
  
    // Disable any use of co_await within this type of coroutine.
    template<typename U>
    std::suspend_never await_transform(U&&) = delete;
  
  };
  ```

* It lets you adapt and change the behavior of normally awaitable values

### Customizing the behavior of co_yield

The final thing you can customize through the promise type is the behavior of the `co_yield` keyword.

If the `co_yield` keyword appears in a coroutine then the compiler translates the expression `co_yield <expr>` into the expression `co_await promise.yield_value(<expr>)`. 

The promise type can therefore customize the behavior of the `co_yield` keyword by defining one or more `yield_value()` methods on the promise object.

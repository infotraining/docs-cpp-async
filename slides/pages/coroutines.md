---
layout: cover
background: /img/header-bg.svg
---

# Coroutines

> "Subroutines are special cases of ... coroutines." – Donald Knuth.

---

## Coroutine

- A function whose execution may be suspended and later resumed
- The concept of coroutine as a subroutine was defined by Melvin Conway in 1958

---
class: white-slide
---

## Function vs Coroutine

<img src="/img/coroutines-concept.png" class="img-lg" />

---

## Function and the Call Stack

* When a function is called, the system allocates space on the **call stack** for the function's local variables and execution state
* This stack space is called a **stack frame**
* When the function returns, the stack space is automatically reclaimed and the execution state is restored to the caller
* The function's state exists only during the call

---

## Function and the Call Stack

```cpp {none|11|1|3|4,11|11|all}{at:1}
int compute_something(int x, int y) // function parameters are stored in the stack frame
{
	int result = x + y + 665; // local variable stored in the stack frame	
	return result;
}

int main()
{
	int a = 10;
	int b = 20;
	int c = compute_something(a, b); // calling the function
}
```

<v-switch>
  <template #1> A stack frame is created for the function </template>
  <template #2> The parameters <code>x</code> and <code>y</code> are stored in the frame </template>
  <template #3> The local variable <code>result</code> is stored in the frame </template>
  <template #4> The function executes and returns the result to the caller </template>
  <template #5> The stack frame is deallocated </template>
</v-switch>

<div v-click="6">
This is <strong>run-to-completion</strong> execution model: once a function starts executing, it runs until it returns, without any interruption.
</div>

---

## What Is a Coroutine?

* A **coroutine** is a function 
  * that can **suspend** its execution 
  * and **resume** later from exactly where it left off.
* When a coroutine **suspends**:
  * Its local variables are preserved
  * The instruction pointer (where you are in the code) is saved
  * Control returns to the caller or some other code
* When a coroutine **resumes**:
  * Local variables are restored to their previous values
  * Execution continues from the suspension point

---
class: white-slide
---

## Coroutine

<iframe src="https://link.excalidraw.com/p/readonly/CmjbtTBcyvMVJiTqcX9y" width="100%" height="100%" style="border: none;"></iframe>

---

## Coroutine Frame

* To support suspension and resumption, a coroutine needs to maintain its state across suspensions
* This is implemented via a **coroutine frame**:
  * It is allocated on the heap (unlike a function's stack frame) to ensure it persists across suspensions
  * It is a data structure that holds the state of the coroutine, including local variables and the instruction pointer
  * The coroutine frame is created when the coroutine is invoked and exists until the coroutine finishes execution or is destroyed

---

## Motivation for Coroutines

- Synchronous & blocking code

```cpp
void handle_request(Connection& conn)
{
    std::string request = conn.read_request();     // blocking call	
    auto parsed_request = parse_request(request);  // CPU-bound work
    auto data = database.query(parsed_request.id); // blocking call
    auto response = compute_response(data);        // CPU-bound work
    conn.write(response);                          // blocking call
}
```

---

## Motivation for Coroutines

* Callback-based asynchronous code

```cpp
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
<div class="hint" v-click="1" >

Callback-based code can lead to "callback hell" with deeply nested callbacks, making it hard to read and maintain

</div>

---

## The Coroutine Solution

* Coroutines allow us to write asynchronous code in a synchronous style
```cpp
void handle_request(Connection& conn)
{
	std::string request = co_await conn.async_read();             // non-blocking call	
	auto parsed_request = parse_request(request);                 // CPU-bound work
	auto data = co_await database.async_query(parsed_request.id); // non-blocking call
	auto response = compute_response(data);                       // CPU-bound work
	co_await conn.async_write(response);                          // non-blocking call
}
```
<div class="hint" v-click="1" >

Coroutines improve readability and maintainability !!!

</div>

---

## Beyond Asynchronous Programming

* Coroutines are not limited to asynchronous programming; they can also be used for:
  * **Generators** (producing a sequence of values on demand)
  * **Cooperative multitasking** (allowing multiple tasks to run in the same thread by yielding control)
  * **State machines**, and more


---

## Coroutines in C++20

- A coroutine in C++20 is a function whose body contains at least one of the statements:
  - `co_return`
  - `co_await`
  - `co_yield`
- C++20 coroutines are implemented as stackless
  - efficient handling of a large number of coroutines (`>1'000'000`)
  - high scalability

---

## co_await

```cpp {all| 3}{at:1}
Task<std::string> download_page(std::string url)
{
	auto response = co_await http_get(url);
	return response.body;
}
```

<v-clicks>

* The `co_await` keyword *suspends the execution of a coroutine* until some operation completes
* When awaiting, the coroutine saves its state, pauses execution, and potentially allows other code to run
* When the awaited operation completes, the coroutine can be resumed, restoring its state and continuing execution from the point of suspension

</v-clicks> 

---

## co_yield

```cpp {all| 3}{at:1}
Generator<int> count_up_to(int max)
{
	for (int i = 1; i <= max; ++i)
	{
		co_yield i; // produce value, suspend, resume when next value requested
	}
}
```

* The `co_yield` keyword produces a value and *suspends the coroutine*. 
* This pattern creates generators—functions that produce sequences of values one at a time. 
* After yielding a value, the coroutine pauses until someone asks for the next value.

---

## co_return

```cpp
Task<int> compute(int a, int b)
{
	int result = 42 + a + b; // some computation
	co_return result; // return value and end the coroutine
}
```

* The `co_return` keyword *completes the coroutine* and optionally provides a final result. 
* Unlike a regular return statement, `co_return` interacts with the coroutine machinery to properly finalize the coroutine’s state.

---

## Coroutine Restrictions

* Functions with a variable number of arguments using `varargs` can’t be coroutines (a variadic function template can be a coroutine)
* A class constructor or destructor cannot be a coroutine
* The `constexpr` and `consteval` functions cannot be coroutines
* A function returning `auto` cannot be a coroutine but `auto` with a trailing return type can be
* The `main()` function cannot be a coroutine
* Lambdas can be coroutines

---

## The simplest coroutine

```cpp
TaskResumer simple_coroutine(std::string name)
{
	std::cout << "Starting " << name << "..." <<std::endl;
    
	co_await std::suspend_always(); // suspension point

	std::cout << name << " has been resumed..." << std::endl;

	co_await std::suspend_always(); // suspension point

	std::cout << name << " has been resumed again..." << std::endl;
	std::cout << name << " is finishing..." << std::endl;
	
	co_return; // end of the coroutine
}
```

---

## The simplest coroutine - client code

```cpp
#include <coroutine>

int main()
{
	TaskResumer coro_task = simple_coroutine("Coroutine#1");

	std::cout << "Coroutine created..." << std::endl;

	do {
		std::cout << "Resuming coroutine from main..." << std::endl;
	} while (coro_task.resume());

	std::cout << "Coroutine has finished execution." << std::endl;
}
```

---

## The simplest coroutine - output

```console
Coroutine created...
Resuming coroutine from main...
Starting Coroutine#1...
Resuming coroutine from main...
Coroutine#1 has been resumed...
Resuming coroutine from main...
Coroutine#1 has been resumed again...
Coroutine#1 is finishing...
Resuming coroutine from main...
Coroutine has finished execution.
```

---

## The simplest coroutine - Behind the courtain

<div class="text-code-07">

```cpp
class TaskResumer { 
public:
	struct promise_type { /* details on next slide */ }

	TaskResumer(std::coroutine_handle<promise_type> handle) : handle_(handle)
	{}

	TaskResumer(const TaskResumer&) = delete;
	TaskResumer& operator=(const TaskResumer&) = delete;

	~TaskResumer() {
		if (handle_)
			handle_.destroy();
	}

	bool resume() {
		if (!handle_ || handle_.done())
			return false;

		handle_.resume();
		return !handle_.done();
	}

private:
	std::coroutine_handle<promise_type> handle_;
};
```
</div>

---

## The simplest coroutine - promise_type

* The `promise_type` is a customization point that defines how the coroutine behaves at various stages of its lifecycle (start, suspension, resumption, completion, exception handling)

```cpp
struct promise_type
{
	TaskResumer get_return_object();

	std::suspend_always initial_suspend() { return {}; }

	std::suspend_always final_suspend() noexcept { return {}; }

	void return_void() {}

	void unhandled_exception() { std::terminate(); }
};
```

---

## Coroutine mechanics in C++20

* Coroutine frame
* Return object (coroutine interface)
* Promise type
* Coroutine Handle
* Suspend ans Resume machanism

---
class: white-slide
---

## Coroutine - scheme

<img src="/img/Coroutine-1.png" class="img-lg" />

---

## Coroutine Frame

- When a coroutine is invoked (started):
  - a *coroutine frame* is created
	- it is usually heap allocated (compilers may optimize and allocate the frame on the stack)
  - all call parameters are copied into the frame
	- for references, references are copied, not the referred values
  - local variables are stored in the frame
  - a *promise* object is created in the frame; it is responsible for coroutine state and behavior

---

## Coroutine Frame - destruction

- Destruction of the coroutine state (frame object) occurs when:
  - control flow leaves the coroutine
  - or `destroy()` is called on a coroutine handle that refers to a running coroutine
	- `std::coroutine_handle<T>::destroy()`
- `destroy()` must be called while the coroutine is suspended!!! (otherwise UB)


---

## Return Object - coroutine interface

- The *Return Object* is the type that the coroutine function returns (e.g., `TaskResumer` in the example)
- Defines the API to manage the coroutine from the outside
  - the caller interacts with the coroutine through this interface to resume, check status, retrieve results, etc.
  - usually defines an internal `promise_type` as a customization point
  - holds a `coroutine_handle` to manage the coroutine's execution

---

## Promise type
    
- The coroutine customization point responsible for:
  - returning the Return Object from the start point
  - deciding whether to suspend the coroutine immediately after start
  - deciding whether to suspend the coroutine at the end
  - handling exceptions thrown inside the coroutine
  - handling values returned from the coroutine

---

## Coroutine handle - `std::coroutine_handle<T>`

- A wrapper object around *a pointer to the coroutine frame* (heap allocated)
- Two versions of the handle are available:
  - `std::coroutine_handle<>` - no access to the `promise` object (type-erased)
  - `std::coroutine_handle<promise_type>` - provides access to the `promise` object

---

## `std::coroutine_handle<>`

* Type-erased handle that does not provide access to the `promise` object

<div class="text-code-07">
```cpp
template <typename _PromiseT = void>
struct coroutine_handle;

template <>
struct coroutine_handle<void> { // no access to promise
  coroutine_handle() noexcept; // it allows to construct empty coroutine_handle
  coroutine_handle(nullptr_t) noexcept;
  coroutine_handle& operator=(nullptr_t) noexcept;
  explicit operator bool() const noexcept; // conversion - if handle is empty
      
  static coroutine_handle from_address(void* _ptr) noexcept; // conversion
  void* address() const noexcept; // returns pointer to coroutine handle

  void operator()() const; // it allows to resume the execution
  void resume() const; // the same as operator()() - resumes the execution
  
  void destroy(); // allows to manually destroy the coroutine
      
  bool done() const; // tests whether the coroutine has completed execution

private:
  void* ptr_; // a pointer to a coroutine stack frame
};
```
</div>

---

## `std::coroutine_handle<promise_type>`

* Provides access to the `promise` object of the coroutine frame

```cpp
template <typename Promise>
struct coroutine_handle : coroutine_handle<void>
{
	Promise& promise() const noexcept; // gets the promise back

	// returns a handle from the promise object 
	static coroutine_handle from_promise(Promise& promise) noexcept; 
};
```

---

## How to get a coroutine handle?

* The coroutine can be obtained from the `promise_type` via `get_return_object()` method
	
```cpp {all|3|5-6|8-14|all}{at:1}
class TaskResumer // Return Object
{
	std::coroutine_handle<promise_type> handle_;
public:
	TaskResumer(std::coroutine_handle<promise_type> handle) : handle_(handle)
	{ }

	struct promise_type
	{
		TaskResumer get_return_object()
		{
			return TaskResumer{std::coroutine_handle<promise_type>::from_promise(*this)};
		}
	};
	// ...
};
```

---
layout: two-cols-header
---

## Coroutine - Code Transformation

::left::

```cpp
ReturnObject coroutine_f(Params)
{
    function_body();
}
```

::right::

<div class="text-code-07">
```cpp{all|3-4|6|10|11|13-18|20-21|all}{at:1}
ReturnObject coroutine_f(Params)
{
    using promise_type = 
		std::coroutine_traits<ReturnObject, Params>::promise_type // [1]

    promise_type promise(promise_constructor_arguments);

    try
    {
      co_await promise.initial_suspend(); // [2]
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
</div>

<style>
.two-cols-header {
  column-gap: 20px; /* Adjust the gap size as needed */
}
</style>

---
layout: cover
background: /img/bg-blue-2.jpg
---

## Example 1 - The Simplest Coroutine

### TaskResumer

---

### TaskResumer - coroutine

```cpp
TaskResumer coro(int max)
{
	//std::cout << std::format("  coro({:>5} / {})\n", "START", max);
	std::cout << "..coro(START, " << max << ")\n";
	for (int value = 1; value <= max; ++value)
	{
		//std::cout << std::format("  coro({:>5} / {})\n", value, max);
		std::cout << "..coro(" << value << ", " << max << ")\n";
		co_await std::suspend_always{}; // SUSPENSION POINT
	}
	//std::cout << std::format("  coro({:>5} / {})\n", "END", max);
	std::cout << "..coro(END, " << max << ")\n";
}
```

---

### TaskResumer - client code

```cpp
TaskResumer coro_task = coro(5);

std::cout << "coro(5) called\n";

while (coro_task.resume())
{
	std::cout << "coro() suspended...\n";
}

std::cout << "coro() done!\n";
```

---

### TaskResumer - coroutine interface

<div class="text-code-07">
```cpp
struct [[nodiscard]] TaskResumer {
	struct promise_type;
	using CoroutineHandle_t = std::coroutine_handle<promise_type>;

	struct promise_type { /*...*/ };

	bool resume() const {
		if (!coro_handle_ || coro_handle_.done())	
			return false;
		coro_handle_.resume();
		return !coro_handle_.done();
	}
    
	TaskResumer(auto handle) : coro_handle_{handle}
	{ }
    
	~TaskResumer() noexcept {
		if (coro_handle_)
			coro_handle_.destroy();
	}
    
	// no-copyable - delete copy constructor and copy assignment operator
    
private:
	CoroutineHandle_t coro_handle_;
};
```
</div>

---

### TaskResumer::promise_type

<div class="text-code-08">
```cpp
struct [[nodiscard]] TaskResumer {
	
	struct promise_type {
		TaskResumer get_return_object() {
			return TaskResumer{CoroutineHandle::from_promise(*this)};
		}

		std::suspend_always initial_suspend() { return {}; }
        
		std::suspend_always final_suspend() noexcept { return {}; }
        
		void unhandled_exception() {
			std::terminate();
		}
        
		void return_void() { }
	};
}
```
</div>

---

## co_return

- Returning values from a coroutine is only possible via `co_return`
- In a coroutine:
  - `co_return;` invokes `promise.return_void()`
  - `co_return expr;`
	- if `expr` evaluates to `void` it calls `promise.return_void()`
	- if `expr` is non-void it calls `promise.return_value(expr)`
- Exiting a coroutine without an explicit return
  - calls `promise.return_void()`
  - if `promise_type` does not have `return_void()` in that case, UB occurs

---

## co_return

- After calling `co_return` or after a silent exit from the coroutine
  - all local variables are destroyed (in reverse order of their definition)
  - `co_await promise.final_suspend()` is invoked

---
layout: cover
background: /img/bg-blue-2.jpg
---

## Example 2
### ResultTask

---

### ResultTask - coroutine

```cpp
ResultTask<double> average(std::ranges::view auto data)
{
	double sum = 0;

	for (const auto& value : data)
	{
		std::cout << "   process " << value << "\n";
		sum += value;
		co_await std::suspend_always{};
	}

	co_return sum / std::ranges::ssize(data);
}
```

---

### ResultTask - client code

```cpp
std::vector data = {4, 34, 52, 665, 24, 34};

ResultTask<double> task = average(std::views::all(data));

while (task.resume())
{
	std::cout << "task resumed...\n";
}

std::cout << "result - average: " << task.get_result() << "\n";
```

---

### ResultTask

<div class="text-code-07">
```cpp {all|14-22|all}{at:1}
template <typename T>
struct [[nodiscard]] ResultTask {
	struct promise_type { /*...*/ };
    using CoroutineHandle_t = std::coroutine_handle<promise_type>;

	ResultTask(auto handle) : handle_{handle} { }
	// no-copyable - delete copy constructor and copy assignment operator

	~ResultTask() noexcept {
		if (handle_)
			handle_.destroy();
	}

	bool resume() const {
		if (!handle_ || handle_.done())
			return false;

		handle_.resume();
		return !handle_.done();
	}

	T get_result() const { return handle_.promise().result; }

private:
	CoroutineHandle_t handle_;
};
```
</div>

---

### ResultTask<>::promise_type

<div class="text-code-08">
```cpp
struct promise_type
{
	T result;

	void return_value(const auto& value)
	{
		result = value;
	}

	ResultTask<T> get_return_object()
	{
		return {std::coroutine_handle<promise_type>::from_promise(*this)};
	}

	std::suspend_always initial_suspend() { return {}; }
    
	std::suspend_always final_suspend() noexcept { return {}; }
    
	void unhandled_exception() { std::terminate(); }
};
```
</div>

---

## co_await

- A new unary operator in C++20
- Suspends the coroutine and returns control to the caller of the coroutine while waiting for the operation represented by the expression to complete

---

## co_await

```cpp
auto r = co_await expr;
```

It is expanded by the compiler to:

```cpp {all|2|4|6-8|11-12|all}{at:1}
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

---

## Awaitables & Awaiters

- To be usable in a `co_await` expression the operand must implement the *Awaitable* interface
- An *awaiter* is an object that implements the `Awaitable` interface and is returned by the `operator co_await()` of an `Awaitable` type
- The standard library does not provide a concept, but its definition (simplified) looks like:

```cpp
template <typename T>
concept Awaitable = requires(T obj) {
	{ obj.await_ready(); } -> std::convertible_to<bool>;
	{ obj.await_suspend(coroutine_handle<>) };
	{ obj.await_resume(); }
};
```

---

## Awaitable

<div class="text-08">

- `bool awaiter.await_ready()`
  - if the call returns `false`, the coroutine is suspended
  - a return value of `true` means there is no need to wait and a synchronous return to the coroutine may occur

- `[void | bool | std::coroutine_handle<>] awaiter.await_suspend(std::coroutine_handle<>)`
  - `void` - the coroutine is suspended and control returns to the caller
  - `bool`
	- `true` - returns control to the caller and suspends the coroutine
	- `false` - resumes the current coroutine
  - `std::coroutine_handle<>` - if a handle to another coroutine is returned, that coroutine is resumed via `handle.resume()`

- `TResult awaiter.await_resume()`
  - the result of `await_resume()` becomes the result of the whole `co_await expr;` expression
</div>

---

## std::suspend_always

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

---

## std::suspend_never

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

---

## Custom awaiter

```cpp
DetachedTask coro_on_many_threads(int id)
{
	int step = 1;
	std::thread::id thd_id = std::this_thread::get_id();
	sync_out() << "Coro#" << id << " - Step#" << step << " - THD#" << thd_id << "\n";

	thd_id = co_await resume_on_new_thread();

	++step;
	sync_out() << "Coro#" << id << " - Step#" << step << " - THD#" << thd_id << "\n";
    

	thd_id = co_await resume_on_new_thread();

	++step;
	sync_out() << "Coro#" << id << " - Step#" << step << " - THD#" << thd_id << "\n";
}
```

---

## resume_on_new_thread()

```cpp
auto resume_on_new_thread()
{
	struct ResumeOnNewThreadAwaiter : std::suspend_always
	{
		void await_suspend(std::coroutine_handle<> coroutine_hndl)
		{
			std::thread([coroutine_hndl] { coroutine_hndl.resume(); }).detach();
		}

		std::thread::id await_resume() const noexcept
		{
			return std::this_thread::get_id();
		}
	};

	return ResumeOnNewThreadAwaiter{};
}
```

---

## co_yield

- The `co_yield` expression returns a value to the caller of the coroutine and suspends it
- The basic building block for creating generators

---

## co_yield

```cpp
co_yield expr;
```

- it is translated by the compiler to:

```cpp
co_await promise.yield_value(expr);
```

---
layout: cover
background: /img/bg-blue-2.jpg
---

## Exercise - Implementing a Generator

---
class: white-slide
---

## Generator - scheme

<img src="/img/Coro-Generator-1.png" class="img-lg" style="max-width: 70%;" />

---
class: white-slide
---

## Generator - scheme

<img src="/img/Coro-Generator-2.png" class="img-lg" />

---

## Code - Client code

```cpp
Generator<int> count_up_to(int max)
{
	for (int i = 1; i <= max; ++i)
	{
		co_yield i; // produce value, suspend, resume when next value requested
	}
}
```

```cpp
Generator<int> gen = count_up_to(5);

for(auto value : gen)
{
	std::cout << value << "\n";
}
```

---

## Code - Generator implementation

```cpp
template <typename T>
class Generator
{
	// TODO - exercise: implement the Generator class based on the scheme from previous slides
};
```

---

## Allocation of the coroutine frame

* The compiler needs to allocate memory for a coroutine frame.
* By default, the compiler uses `operator new` to allocate memory for the coroutine frame on the heap. 
* The frame size depends on the local variables, the promise object, and other bookkeeping information required by the coroutine.

---

## Heap Allocation eLision Optimization (HALO)

* Compilers can sometimes eliminate coroutine frame allocation entirely through HALO (Heap Allocation eLision Optimization).
* HALO allocation occurs on stack, instead of heap when:
  * The coroutine’s lifetime is contained within the caller’s lifetime
  * The frame size is known at compile time
* HALO is most effective when:
  * Coroutines are awaited immediately after creation
  * Optimization is enabled (-O2 or higher)

---

## HALO - example

```cpp
// HALO might apply here because the task is awaited immediately
co_await compute_something();

// HALO cannot apply here because the task escapes
auto task = compute_something();
store_for_later(std::move(task));
```

---

## Custom allocators

* Promise type can also customize allocation by defining operator new and operator delete. 
* This allows to use custom memory pools or other allocation strategies for coroutine frames.

<div class="text-code-08">
```cpp
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
</div>

---

## Exception handling in coroutines

* Exceptions in coroutines require special handling because a coroutine can suspend and resume across different call stacks.
* When an exception is thrown inside a coroutine and not caught:
  * The exception is caught by an implicit `try-catch` surrounding the coroutine body
  * `promise.unhandled_exception()` is called while the exception is active
  *	After `unhandled_exception()` returns, `co_await promise.final_suspend()` executes
  * The coroutine completes (suspended or destroyed, depending on `final_suspend`'s behavior)

---

## Options for handling exceptions - 1

* Terminate the program (e.g., via `std::terminate()`)

```cpp
void unhandled_exception()
{
    std::terminate();
}
```

* Store the exception for later retrieval by the caller

```cpp
std::exception_ptr exception_;          

void unhandled_exception()
{
    exception_ = std::current_exception();
}
```
---

## Options for handling exceptions - 2

* Rethrow the exception immediately

```cpp
void unhandled_exception()
{
	throw; // propagate the exception to whoever resumed the coroutine
}
```

* Swallow the exception (not recommended)

```cpp
void unhandled_exception()
{
	// do nothing - swallow the exception (not recommended)
}
```


---

## Symetric transfer

* When a coroutine completes or awaits another coroutine, control must transfer somewhere.
* The naive approach by simply calling `coro_handle.resume()` encounters a problem: each nested coroutine adds a frame to the call stack. 
* With deep nesting, stack overflow may occur

---

## Symmetric transfer

```cpp
task<> a() { co_await b(); }
task<> b() { co_await c(); }
task<> c() { co_return; }
```

* Without symmetric transfer, when `a` awaits `b`:
  * `a` calls into the awaiter’s `await_suspend()`
  * `await_suspend()` calls `b.coro_handle.resume()`
  * `b` runs, calls into its awaiter’s `await_suspend()`
  * That calls `c.coro_handle.resume()`
  * The stack now has frames for `a`’s suspension, `b`’s suspension and `c`’s suspension
  
---

## Symmetric transfer

* The solution for growing stack is *Symmetric Transfer* 
* Instead of resuming the awaited coroutine, the current coroutine transfers control directly to the awaited coroutine without returning to the caller
* This is achieved by returning the awaited coroutine’s handle from `await_suspend()`, which causes the runtime to switch to that coroutine immediately, without adding a new frame to the call stack

```cpp
std::coroutine_handle<> await_suspend(std::coroutine_handle<> current_hndl)
{
    continuation_hndl_ = current_hndl; // store continuation for later resumption
    
    return next_coroutine_hndl_;  // return handle to resume (instead of calling resume())
}
```
---

## Symmetric transfer

* When `await_suspend()` returns a handle, the compiler generates code that performs a *tail call* to that handle, effectively transferring control to the next coroutine without growing the stack.
* The generated code looks like this:

```cpp
auto next_hndl = awaiter.await_suspend(current_hndl);
if (next_hndl != std::noop_coroutine())
{
    next_hndl.resume(); // tail call - no new stack frame
}
```

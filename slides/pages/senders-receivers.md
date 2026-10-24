---
layout: cover
background: /img/bg/bg2.jpg
---

# Sender Sub-Language

---

# Theoretical Foundations

* The Sender Sub‑Language is built on **continuation-passing style**, **monads**, and **algebraic effects**.  
Understanding these foundations helps developers reason about async composition.

* Senders are **lazy**, **composable**, and **zero‑allocation** until started.

> “The Sender Sub-Language is continuation-passing style expressed as composable value types.”

---

# Continuations & CPS

* **Continuation-passing Style (CPS)**: every function receives an explicit continuation representing "what happens next."

* Instead of returning a value to its caller, a function passes its result forward into the continuation

* Senders are CPS graphs: each stage is a continuation.

<div class="span-v-2"/>

```
Direct Style (Call / Return):
  Caller ──call──> [ Function ] ──return──> Caller

CPS / Senders & Receivers Style:
  [ Sender ] ──set_value(val)──> [ Receiver (Continuation) ]
```

---

# Senders & CPS


* `connect(sndr, rcvr)` - reifies the continuation - it binds a **sender** to its **receiver**, producing an **operation state** that holds the entire work graph as a concrete type.

* `start(op)` - triggers CPS evaluation of the entire compile-time work graph without dynamic allocations or virtual dispatch.

<div class="span-v-2"/>

```cpp
auto s = ex::just(10)
  | ex::then([](int v) { return v * 2; });

auto op = ex::connect(s, receiver); // Reification of continuation (CPS)

ex::start(op); // Concrete CPS evaluation
```

---

#  Monads

Senders map directly to monadic structure

- `just(x)` → **monadic return**

- `let_value(f)` → **monadic bind**

- `then(f)` → **functor lift / fmap**

<div class="span-v-2"/>

```cpp
auto s = ex::just(5)
     | ex::let_value([](int v) {
           return ex::just(v * 3);
       });
```

---

# Algebraic Effects

* Senders have **three channels**:

  - `set_value`

  - `set_error`

  - `set_stopped`

* This makes cancellation a **first-class effect**.

```cpp
auto s = ex::just_stopped()
    | ex::upon_stopped([] { return 0; });
```

---

# Theoretical Mapping Table

* Sender operations correspond to well-known PL constructs.

<div class="span-v-2"/>

| Sender Concept                        | Theory                                          |
| ------------------------------------- | ----------------------------------------------- |
| `just(x)`                             | monadic return                                  |
| `let_value(f)`                        | monadic bind                                    |
| `then(f)`                             | functor fmap                                    |
| `set_value / set_error / set_stopped` | Algebraic effect channels / Coproduct injection |
| `connect(sndr, rcvr)`                 | CPS continuation reification                    |
| `start(op)`                           | CPS forced evaluation                           |

---

# Theoretical Mapping Table

<div class="span-v-2"/>

| Sender Concept             | Theory                           |
| -------------------------- | -------------------------------- |
| `read_env / write_env`     | Reader monad (`ask` / `local`)   |
| `when_all`                 | parallel composition             |
| `split`                    | linear logic                     |
| Async scopes (`associate`) | Region-based lifetime boundaries |
| `bulk` family              | Data-parallel map primitive      |


---

# Value Lifting

* Value lifting introduces data into the sender world.

* The simplest sender algorithms place a value into the pipeline and let it carry forward.

*  Nothing is computed, nothing is fetched. The value is already known, and these algorithms are purely about lifting it into the sender context.


---

## just — Lift a Value

* `just(vs...)` creates a sender that completes immediately with the given values on the value channel

* Put values into the sender pipeline without performing any computation.


```cpp
auto sndr = just(42, std::string("test"));

// Plain C++ equivalent:
// int a = 42;
// std::string b = "test";
```

---

## just_error — Lift an Error

* Lifts an error payload directly onto the `set_error` channel without throwing CPU exceptions.

* Injects failure into the effect algebra as a first-class value.

* Propagates downstream until intercepted by `upon_error` or `let_error`.


```cpp
auto sndr = ex::just_error(
    std::make_error_code(std::errc::invalid_argument)
);

// Plain C++ equivalent:
// throw std::system_error(make_error_code(std::errc::invalid_argument));
```

---

## just_stopped — Lift Cancellation

* Initiates cancellation on the dedicated `set_stopped` channel.

* Cancellation is not an error — it is a separate semantic path.

* Represents clean, non-error cooperative cancellation as a first-class citizen.


```cpp
auto sndr = ex::just_stopped();

// Plain C++ equivalent:
// Regular C++ has no dedicated cancellation channel
```

---

# sync_wait

* The bridge between the Sender Sub-Language and regular C++
  
  * The boundary consumer connecting asynchronous senders to synchronous C++.
  * Blocks the calling thread until the graph completes.
  * Single point where lazy descriptions are driven to evaluation.
  * Returns `std::optional<std::tuple<...>>`.


```cpp
auto [result] = ex::sync_wait(ex::just(42)).value();

assert(result == 42);

// Plain C++ equivalent:
// int result = 42;
```

---

# Transformation

* Transformations apply functions to pipeline results.


---

## then — Transform Values

* Applies a pure function $f$ to values arriving on `set_value`.

* Transforms values without reshaping the pipeline topology.

* Exceptions thrown inside $f$ are automatically caught and routed to `set_error`.


```cpp
auto sndr = ex::just(3, 4)
  | ex::then([](int a, int b) {
        return a * a + b * b;
    });
// completes with set_value(25)

// Plain C++ equivalent:
// int result = 3 * 3 + 4 * 4;
```

---

## upon_error — Transform Errors

* Transforms an error from `set_error` into a fallback value on `set_value`.

* Cross-channel recovery: converts failure directly into success.

* Normal values on `set_value` pass through unaffected.


```cpp
auto sndr = just_error(
  make_error_code(errc::connection_reset))
  | upon_error([](error_code ec) {
      return -1; // Recovery value
    });
// completes with set_value(-1)

// Plain C++ equivalent:
// try { throw ... } catch (...) { return -1; }
```

---

## upon_stopped — Transform Cancellation

* Catches cancellation from `set_stopped` and emits a recovery value on `set_value`.

* Allows a pipeline to provide a default value when upstream operations are cancelled.

* Eliminates the `set_stopped` channel, routing execution back onto `set_value`.


```cpp
auto sndr = 
    just_stopped()
    | upon_stopped([] {
          return 0; // Fallback value on cancellation
      });
// completes with set_value(0)

// Plain C++ equivalent:
// Regular C++ has no dedicated cancellation channel
```

---

# Monadic Composition

* Sometimes the next step in a computation is itself a whole new computation.

* When the next step is itself a sender, use monadic composition - `let_*`.




---

## let_value — Bind Values

* Accepts a function returning a **fresh sender** and chains it based on upstream results.

* Enables dynamic, multi-stage pipelines where subsequent work depends on earlier values.

* The result is a sender whose completion type is determined by the inner sender, not by the function's return type directly


```cpp
auto sndr = 
    ex::just(std::string("hello"))
    | ex::let_value([](std::string s) {
        return ex::just(s + " world")
        | ex::then([](std::string r) {
            return r.size();
        });
    });
// completes with set_value(11)
```

---

## let_error — Bind Errors

* Performs error recovery via an asynchronous fallback pipeline returning a fresh sender.

* Enables complex asynchronous failover routes (e.g., query alternate replica or disk cache).

* Seamlessly branches execution upon encountering errors.


```cpp
auto sndr = 
    ex::just_error(make_error_code(errc::timed_out))
    | ex::let_error([](error_code ec) {
          return ex::just(ec.message());
      });

// completes with set_value("timed out")
```

---

## let_stopped — Bind Cancellation

* Replaces a stopped signal by instantiating and executing a brand new sender.

* Invoked when upstream cancellation occurs; receives no parameters and returns a sender.


```cpp
auto sndr = just_stopped()
  | let_stopped([] {
      // Launch an asynchronous fallback/compensation pipeline
      return just(std::string("recovered"));
    });
// completes with set_value("recovered")

// Plain C++ equivalent:
// Regular C++ has no dedicated cancellation channel
```

---

# Execution Contexts

* Execution context algorithms control **where** work runs.

* They decide where -  which thread, which I/O loop, which GPU stream owns the next step of the computation.


---

## schedule — Gateway into Execution Contexts 

* Produces an empty sender completing on the execution context of the specified scheduler.

* Schedulers represent compute resources (OS threads, I/O runloops, GPU streams).

* Serves as the standard entry point into an execution context.


```cpp
auto sndr = 
    ex::schedule(pool.get_scheduler())
    | ex::then([]() {
          return handle_request();
      });

// Plain C++ equivalent:
// pool.post([&] { handle_request(); });
```

---

## starts_on — Initiating Pipelines on Schedulers

* Arranges for an existing sender pipeline to begin execution on the specified scheduler.

* Ensures that the initial step and evaluation start on the chosen context.

```cpp
auto sndr = 
    ex::starts_on(
        pool.get_scheduler(),
        just(request) | then([](auto req) {
            return handle(req);
        })
    );

// Plain C++ equivalent:
// pool.post([&] { auto res = handle(request); });
```

---

## continues_on — Switch Scheduler

* Transfers downstream execution to another scheduler after the predecessor completes.

* Standard pattern: perform I/O on runloop, transfer compute to worker pool.

```cpp
auto sndr =
    ex::starts_on(
        io_ctx.get_scheduler(),
        async_read(socket, buf))
    | ex::continues_on(pool.get_scheduler())
    | then([](auto data) {
        return parse(data); // runs on worker pool!
    });
```

---

## on — Round Trip

* Executes a subcomputation on a target scheduler, then returns execution to the caller's context.

* Brackets computation with a context switch and its inverse.

```cpp
auto sndr = ex::on(
    pool.get_scheduler(),
    just(request) | then([](auto req) {
        return handle(req);
    })
);

// Result automatically returns to caller's original scheduler
```

---

## affine — Scheduler Affinity  

* Ensures completion on the receiver's start scheduler (`get_start_scheduler`).
* **Critical Optimization:** Skips rescheduling overhead if sender already completes on the target context.
* Foundation of coroutine resumption affinity in C++26 `task<T>`.


```cpp
auto sndr1 = ex::affine(async_read(fd));

auto sndr2 = some_operation() | ex::affine;
```

---

## schedule_from — Force Completion Context

* Three-Channel Scheduler Dispatch.

* Low-level primitive guaranteeing that **all three channels** (value, error, stopped) arrive on the scheduler.

* Deeper than `continues_on` (which transitions only the value channel).

* Essential tool for engine and driver implementers.

```cpp
auto sndr = ex::schedule_from(pool.get_scheduler(), async_write(fd, buf));
```

---

# Environment  

* The pipeline asks its context a question.

* Senders can read and modify the execution environment.


---

## read_env — Ask the Environment

<div class="text-08">

* Used for querying ambient properties (schedulers, allocators, stop tokens) from the receiver's environment.

* `read_env(query)` creates a sender that reads the value associated with query from the receiver's environment and completes with it on the value channel.

* The **sender reads it from the receiver at connection time**, when the environment is known. 

* Enables context-dependent pipelines that can behave differently depending on which scheduler, allocator, or stop token the enclosing context provides.

</div>

```cpp
auto s = 
    ex::read_env(get_scheduler)
    | ex::let_value([](auto sch) { return ex::schedule(sch); });

// Plain C++ equivalent:
// auto config = load_config();
```

---

## write_env — Modify Environment

* `write_env(env, sndr)` wraps sndr so that downstream receivers see the additional or overridden entries in `env` - allocator, scheduler, stop token, etc.

*  The wrapped sender and all of its children see the modified environment.

* It provides a custom allocator, overrides a scheduler, or injects application-specific context without pushing parameters through every function signature  

```cpp
auto s = ex::write_env(make_env(get_allocator, pool_alloc),
                       ex::process_file(path));

// Plain C++ equivalent:
// auto result = process_file(path);
```

---

## unstoppable — Mask Cancellation

* Shields a sender from ambient cancellation signals.

* Ensures that critical operations run to completion regardless of parent stop tokens.

* **Type System Impact:** The sender's completion signatures lose `set_stopped_t()`.

* **Use Cases:** Transaction commits, resource cleanups, hardware finalization routines.


```cpp
auto s = ex::unstoppable(commit_transaction(db, txn));
```

---

# Structured Concurrency

* Run multiple senders concurrently with lifetime guarantees.

* Ensures that all spawned senders complete before the scope ends, preventing resource leaks and dangling operations.


---

## when_all — Concurrency Join

* Starts all child senders concurrently and joins values by concatenating them in order.

* **Structural Guarantee:** No child outlives the join point.

* If any child emits an error or stop, all siblings are cancelled immediately.

```cpp
auto sndr = 
    ex::when_all(
        fetch_user(user_id),
        fetch_orders(user_id),
        fetch_preferences(user_id)
    )
    | ex::then([](auto user, auto orders, auto prefs) {
        return build_profile(user, orders, prefs);
    });
```

---

## when_all_with_variant — Heterogeneous Join  

* Concurrent join accommodating children with heterogeneous or multiple value signatures.

* Each child output is wrapped into `std::variant<std::tuple<...>>`.

* Relaxes single-signature constraints across parallel branches.


```cpp
auto sndr = 
    ex::when_all_with_variant(
        fetch_json(url_a),
        fetch_binary(url_b)
    ) 
    | ex::then([](auto json_var, auto bin_var) {
        return combine(json_var, bin_var);
    });
```

---

## split — Share Results

* Converts a single-shot (linear) sender into a multi-shot (sharable) sender.

* The sender runs once; everyone gets a copy of the result.

* Allocates a reference-counted state that caches and distributes the result.

```cpp
auto shared = ex::split(fetch_data(url));

auto sndr = ex::when_all(
    shared | ex::then([](auto data) { return analyze(data); }),
    shared | ex::then([](auto data) { return archive(data); })
);

// Plain C++ equivalent:
// auto data = fetch_data(url);
// auto analysis = analyze(data);
// auto archived = archive(data);
```

---

# Signal Adaptation

* Adapt completion signatures to unify or reshape results.
  
    * `into_variant`,
    * `stopped_as_optional`,
    *  and `stopped_as_error`.

---

## into_variant — Unifying Value Signatures

* Folds multiple distinct value completion signatures into a single `std::variant`.

* Enables passing multi-signature senders into algorithms expecting uniform signatures like `when_all`.

```cpp
auto sndr = 
    query_database(stmt)
    | ex::into_variant;

// completes with:
// set_value(variant<tuple<Row>, tuple<int>>)
```

---

## stopped_as_optional — Cancellation → Optional

* Collapses `set_stopped` into `std::nullopt` on the value channel; values become `std::optional<T>`.

* Merges cancellation into simple monadic `std::optional` branch checks.

```cpp
auto sndr = dequeue(work_queue)
    | ex::stopped_as_optional;

// value(item) -> set_value(optional(item))
// stopped()   -> set_value(optional())
```

---

## stopped_as_error — Cancellation → Error

* Reclassifies cancellation (`set_stopped`) as an explicit error on `set_error`.

* Replaces the stopped channel with an error code, allowing standard error combinators (`upon_error`, `let_error`) to handle timeouts uniformly.

```cpp
auto sndr = 
    timed_operation(deadline)
    | ex::stopped_as_error(
          make_error_code(std::errc::timed_out)
      );

// Plain C++ equivalent:
// auto result = timed_operation(deadline);
// if (!result) throw system_error(make_error_code(errc::timed_out));
```


---

# Data Parallelism

*  The `bulk` family is a parallel for-loop expressed as a sender - one function, applied uniformly across an index space.


---

## bulk — Parallel Map

* A parallel for-loop over an index space, where each iteration can be executed concurrently according to the specified policy.
  
* `bulk(policy, shape, f)`
  * invokes `f(i, args...)` for each index `i` in range `[0, shape)` 
  * `policy` defines the execution strategy (e.g., `std::par`, std::seq`).
  * `shape` is a number of invokations (i.e., the size of the index space)
  * `args...` are values produced by the predecessor sender.
* Forwards value arguments to all iterations without cloning.

---

## bulk — Parallel Map

* `bulk` example:

```cpp
auto sndr = 
    ex::just(records)
    | ex::bulk(std::par, records.size(),
          [](size_t i, auto& recs) {
              recs[i].score = compute_score(recs[i]);
          }
      );
```

* Equivalent program

```cpp
std::for_each(std::execution::par, records.begin(), records.end(),
              [](auto& rec) { rec.score = compute_score(rec); });
```

---

## bulk_chunked — Chunked Parallelism

<div class="text-09">

* Partitions the index space into cache-friendly chunks for vectorized / SIMD execution.

* Exposes chunk index ranges directly to the callable.

* `bulk_chunked` gives the implementation control over how the index space is divided. 
  * the `parallel_scheduler` uses this to distribute work across threads in chunks sized for the hardware.
  * the callable receives a range of indices rather than a single index, enabling vectorization and cache-friendly access patterns within each chunk.

</div>

```cpp
auto sndr = 
    ex::just(data)
    | ex::bulk_chunked(std::par, data.size(),
          [](auto chunk, auto& d) {
              for (auto i : chunk)
                  d[i] = transform(d[i]);
          }
      );
```


---

## bulk_unchunked — Per‑Index Parallelism

<div class="text-09">

* Element-Wise Parallel Iteration

* Invokes the callable once per index with no chunking guarantees:

* Simplest mental model for parallel iteration.

* `bulk` dispatches by default to `bulk_chunked`, which the underlying scheduler may specialize for target hardware topologies.

</div>

```cpp
auto sndr = 
    ex::just(pixels)
    | ex::bulk_unchunked(std::par, pixels.size(),
          [](size_t i, auto& px) {
              px[i] = apply_filter(px[i]);
          }
      );
```

---

# Async Scopes

* Tie dynamically spawned work to a lifetime scope.

* Spawns background work into a scope while returning a sender handle to its future result.

* The structured, safe equivalent of `std::async`.

---

## associate — Structured Spawn

* Ties an independently spawned sender's lifetime to an enclosing `counting_scope`.

* Guarantees zero orphaned background tasks during server shutdown

```cpp
ex::counting_scope scope;
auto token = scope.get_token();

for (auto& conn : accepted_connections) {
    auto sndr = ex::associate(token,
        ex::starts_on(pool.get_scheduler(), handle_connection(conn)));
}
scope.join() // asynchronously drains all active conn

// Plain C++ equivalent:
// for (auto& conn : accepted_connections)    
//    pool.post([&] { handle_connection(conn); }); 
// pool.join();

```


---

## spawn_future — Spawn + Return Value

* Structured Async Futures

* Spawns background work into a scope while returning a sender handle to its future result.

* The structured, safe equivalent of `std::async`.

```cpp
auto sndr = ex::spawn_future(token,
    ex::starts_on(pool.get_scheduler(), fetch_data(key)))
    | ex::then([](auto data) {
          return process(data);
      }
    );

// Plain C++ equivalent:
// auto future = std::async(std::launch::async, [&] { return fetch_data(key); }); 
// auto result = process(future.get());
```

---

# task\<T\> - Coroutine Type 

* Coroutines integrate seamlessly with senders.

* The `task<T>` is the bridge between coroutine-style `co_await` and the sender pipeline model. 

* A task can `co_await` any sender, and a task can be used as a `sender` in any pipeline.

---

## task\<T\> - Overview

* **Lazy Evaluation:** Coroutine body does not execute until connected to a receiver and started.

* **Dual Nature:**
  * Can `co_await` any sender inside `task<T>`.
  * Is itself a sender (composable via `|`, `when_all`, etc.).

* **Scheduler Affinity:** Automatically injects `affine` around `co_await` expressions to resume execution on the task's originating context.

* **Typed Errors:** `co_yield with_error(e)` dispatches directly to `set_error` without throwing.


---

## task\<T\> - Basics

* `task<T>` declares a coroutine that produces a value of type `T` on the value channel when it completes.

```cpp
ex::task<int> the_answer() 
{ 
    co_return 42; 
} 

int main() 
{ 
    auto [v] = ex::sync_wait(the_answer()).value(); // v == 42
}
```

---

## Co‑awaiting Senders

* Inside a task, any sender can be co_awaited. 

* The sender is connected to an internal receiver, started, and the result is delivered as the value of the `co_await` expression.

```cpp
ex::task<int> add_one() 
{
    int v = co_await ex::just(41);
    co_return v + 1;
}
```

---

## Co‑awaiting Senders

* When the sender completes with `set_value(args...)`, the `co_await` expression produces the values. 
  * A single value is returned directly. 
  * Multiple values are returned as a `tuple<...>`.
  * A sender with no value arguments produces `void`. 

* If the sender completes with `set_error(err)`, the `err` is thrown as an exception. 

* If it completes with `set_stopped()`, the coroutine is cancelled without resuming.
  

---

##  Coroutines in C++26: Code Example

```cpp
ex::task<sensor_data> read_sensors(sensor_array const& sensors) 
{
    // Concurrent acquisition across 3 distinct hardware schedulers:
    auto [lidar, radar, camera] = 
        co_await when_all(
            starts_on(sensors.lidar_ctx(),
                      read_lidar()),
            starts_on(sensors.radar_ctx(),
                      read_radar()),
            starts_on(sensors.camera_ctx(),
                      read_camera())
        );

    co_return fuse(lidar, radar, camera);
}
```

---

## Scheduler Affinity

* The `task`'s promise type injects affine around every `co_await`ed sender via `await_transform`. 
  
* The effect: after each `co_await`, execution resumes on the task's scheduler regardless of where the sender completed.

```cpp
task<void> affine_demo() {    
    // running on pool's scheduler    
    co_await async_read(socket, buf);    
    // still on pool's scheduler    
    auto result = parse(buf);    
    co_await async_write(socket, result);    
    // still on pool's scheduler 
}
```    

---

## task_scheduler — Affinity

* Typed erased scheduler used by `task` for scheduler affinity. 

* When a `task` is connected to a receiver, the scheduler is obtained from the receiver's environment via `get_start_scheduler(get_env(rcvr))` and stored in the `task_scheduler`.

```cpp
ex::task<void> work() {    
    co_await async_read(socket, buf);
    // resumes on the task's scheduler
    process(buf); 
} 

ex::sync_wait(ex::starts_on(pool.get_scheduler(), work()));
```

---

## inline_scheduler — Disable Affinity

* `inline_scheduler` completes immediately on the calling thread. 
 
* Using it as the task's scheduler type disables scheduler affinity.

```cpp
struct no_affinity {
    using start_scheduler_type = ex::inline_scheduler; 
}; 

ex::task<void, no_affinity> fast_path() {
    co_await async_op();    
    // no rescheduling - resumes wherever    
    //  the sender completed 
}
```
---

## co_yield with_error(e)

* `co_yield with_error(e)` reports an error on the error channel without throwing an exception.

* Delivers a typed error to the sender composition algebra without relying on exceptions.

* The downstream pipeline handles the error through `upon_error` or `let_error`.

```cpp
ex::task<void> validate(request const& req) { 
    if (!req.valid()) 
        co_yield ex::with_error(
            std::make_error_code(errc::invalid_argument));
     co_return; 
}
```

---

## Cancellation

* `co_await just_stopped()` cancels the coroutine. 
  
* The coroutine completes with `set_stopped()`.

*  The coroutine is never resumed - local variables are destroyed and the task completes on the stopped channel

```cpp
ex::task<void> check_cancel() 
{ 
    auto token = co_await ex::read_env(ex::get_stop_token);
    if (token.stop_requested()) 
        co_await ex::just_stopped();
    co_return; 
}
```

---

## Error Channel Mapping

* When a task `co_awaits` a sender that completes with `set_error(ec)`, the error is delivered as a thrown exception

```cpp
ex::task<void> error_mapping_demo() 
{    
    try 
    {        
        co_await async_connect(socket, endpoint);    
    } 
    catch (std::system_error const& e) 
    {        
        // set_error(ec) became throw        
        log("connect failed: {}", e.code().message());    
    } 
}
```

---

## Compound Results - IO operations

* When an I/O operation returns both a status code and a byte count, the compound result can stay on the value channel. 

* Both values arrive at the `co_await` expression.

```cpp
ex::task<void> read_with_status() 
{    
    auto [ec, n] = co_await async_read(socket, buf);
    
    if (!ec) 
        process(buf, n);
    else if (ec == errc::connection_reset)
        cleanup(n);   
}
```
---

## Pipelines Inside Coroutines

* A sender pipeline can be `co_await`ed as a single expression inside a task.

```cpp
ex::task<string> fetch_and_transform(url u) 
{
    auto result = co_await (        
        async_fetch(u)
        | ex::then([](auto resp) { return resp.body(); })
        | ex::then([](auto body) { return compress(body); })
    );
    
    co_return result; 
}
```

---

# Real-World Example: Autonomous Vehicle System

* 4-Tier Real-Time Architecture (P4014R2 Section 13)

```
[Layer 1: safety_controller] (Pipeline)
   │  • associate in counting_scope
   │  • upon_error() / upon_stopped() catch-all failover
   ▼
[Layer 2: actuate_brakes] (Task)
   │  • co_await collision_pipeline
   │  • dispatch to realtime_ctx()
   ▼
[Layer 3: collision_pipeline] (Pipeline)
   │  • then() obstacle detection
   │  • let_value() braking decision & trajectory
   ▼
[Layer 4: read_sensors] (Task)
      • co_await when_all() across 3 heterogeneous I/O contexts
```

---

## Sensor Fusion

* A task that `co_await`s a structured-concurrency pipeline to read three sensors in parallel and fuse the results

```cpp
ex::task<sensor_data> read_sensors(sensors) 
{
    auto [lidar, radar, camera] = co_await when_all(
        ex::starts_on(sensors.lidar_ctx(), async_read(sensors.lidar()))
        | ex::then(decode_lidar),
        ex::starts_on(sensors.radar_ctx(), async_read(sensors.radar()))
        | ex::then(decode_radar),
        ex::starts_on(sensors.camera_ctx(), async_read(sensors.camera()))
        | ex::then(decode_camera)
    );
  
    co_return fuse(lidar, radar, camera);
}
```

---

## Collision Detection

* The task from previous slide is used as a sender inside a pipeline that evaluates whether immediate braking is required.

```cpp
auto collision_pipeline(sensors, state) {
    return read_sensors(sensors)
        | ex::then([&](auto data){ return detect_obstacles(data, state); })
        | ex::let_value([](auto obs) {
              if (obs.imminent())
                  return ex::just(brake_cmd{obs.severity()});
              return ex::just(brake_cmd::none());
            }
        );
}
```

---

## Actuator Command

* A task that `co_await`s the collision pipeline and sends brake commands on a real-time scheduler.

```cpp
ex::task<actuator_result> actuate_brakes(
        sensor_array const& sensors,
        vehicle_state const& state,
        brake_controller& brakes) 
{
    auto cmd = co_await collision_pipeline(sensors, state);
    
    if (cmd.severity() > 0) 
    {
        co_await ex::starts_on(brakes.realtime_ctx(), async_write(brakes.channel(), cmd));
        co_return;
    }
}
```

---

## Failover

<div class="text-09">

```cpp {all}{maxHeight: '400px'}
auto safety_controller(
    sensor_array const& sensors,
    vehicle_state const& state,
    brake_controller& brakes,
    ex::counting_scope& scope) 
{    
    return ex::associate(scope.get_token(),
        actuate_brakes(sensors, state, brakes)
        | ex::let_error(
            [&](error_code ec) {
                return 
                    ex::starts_on(
                        brakes.realtime_ctx(),
                        async_write(brakes.channel(), brake_cmd::emergency()))
                    | ex::then([ec](auto) {
                        log("sensor failure: {}, emergency brake applied", ec);
                        return actuator_result::emergency();
                    });
            })
        | ex::upon_stopped([&] {
            log("controller stopped");
            return actuator_result::none();
        })
    );
}
```
</div>

---

## Equivalent Program - Coroutine-only Version

<div class="text-09">

```cpp {all}{maxHeight: '400px'}
ex::task<actuator_result> safety_controller(        
    sensor_array const& sensors,
    vehicle_state const& state,
    brake_controller& brakes)
{
    try 
    {
        auto data = co_await read_all_sensors(sensors);
        auto obs = detect_obstacles(data, state);
        if (obs.imminent()) 
        {
            auto cmd = brake_cmd{obs.severity()};
            co_await brakes.apply(cmd); 
            co_return actuator_result::applied(cmd); 
        }
        
        co_return actuator_result::none();    
    } 
    catch (...) 
    {
        co_await brakes.emergency_stop();
        co_return actuator_result::emergency();    
    }
}
```
</div>
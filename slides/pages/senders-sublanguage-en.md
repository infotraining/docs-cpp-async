---
layout: cover
background: /img/bg/bg1.jpg
---

# C++26 Senders/Receivers

## Algorithms and Theory

<div class="pt-12 text-sm opacity-80">
Lambda Calculus • Continuation-Passing Style • Category Theory & Monads • Algebraic Effects • 30 Adaptors
</div>

---

# Agenda (1/3): Foundations & Core Model

1. **Theoretical Foundations:**
   * Lambda Calculus and Continuation-Passing Style (CPS)
   * Category Theory: Monads, Functors, and Kleisli Arrows
   * Delimited Continuations, Algebraic Effects, and Three-Channel Coproducts
   * Linear Logic, Process Calculi, and Region Discipline
2. **Formal Theoretical Mapping:**
   * Core Operations & Compositional Morphisms
   * Concurrency, Effects, and Region Boundaries
3. **Execution Model & Semantics:**
   * Zero-Allocation Control Flow Reification
   * The Tripartite Completion Algebra (`set_value`, `set_error`, `set_stopped`)

---

# Agenda (2/3): Algorithms & Architecture

4. **The 30 Algorithms & Adaptors:**
   * Value Lifting & Consumption (`just`, `just_error`, `just_stopped`, `sync_wait`)
   * Transformation (`then`, `upon_error`, `upon_stopped`)
   * Monadic Composition (`let_value`, `let_error`, `let_stopped`)
   * Execution Contexts (`schedule`, `starts_on`, `continues_on`, `on`, `affine`, `schedule_from`)
   * Environment (`read_env`, `write_env`, `unstoppable`)
   * Structured Concurrency & Contraction (`when_all`, `when_all_with_variant`, `split`)
   * Signal Adaptation (`into_variant`, `stopped_as_optional`, `stopped_as_error`)
   * Data Parallelism (`bulk`, `bulk_chunked`, `bulk_unchunked`)
   * Async Scopes (`associate`, `spawn_future`)
  
---

# Agenda (3/3)


5. **Coroutines & Real-World Systems:**
   * `std::execution::task<T>` (P3552R3)
   * Custom Adaptor Implementation (`retry`)
   * Multilayer Autonomous Vehicle System

---

# Lambda Calculus & CPS: Foundations

### Continuation-Passing Style (Steele & Sussman, 1975–1980)
* **The Lambda Papers:** Functions do not return values directly to callers; instead, they pass results forward into an explicit continuation $k$ (*"what happens next"*).
* **Explicit Evaluation Order (Plotkin, 1975):** In call-by-value lambda calculus, CPS exposes the order of evaluation directly in the term structure.
* **Compiler Intermediate Representation:** CPS is widely used in optimizing functional compilers (SML/NJ, GHC STG) because control flow is represented as simple function calls.

---

# CPS in Senders & Receivers: Zero Allocation

### Control Flow Reification in C++26

```
Direct Style (Call / Return):
  Caller ──call──> [ Function ] ──return──> Caller

CPS / Senders & Receivers Style:
  [ Sender ] ──set_value(val)──> [ Receiver (Continuation) ]
```

* **`connect(sndr, rcvr)`:** Reifies the continuation, constructing an unallocated, concrete operation state type.
* **`start(op)`:** Triggers CPS evaluation of the entire compile-time work graph without dynamic allocations or virtual dispatch.

```cpp
// Reification of continuation (CPS):
auto op = std::execution::connect(sndr, rcvr);

// Concrete CPS evaluation:
std::execution::start(op);
```

---

# Category Theory: Monads & Kleisli Arrows

### The Monad $(T, \eta, \mu)$ (Moggi, 1991; Wadler, 1992)
* **Computational Effects:** Monads formalize pure computations carrying side effects.
* **Unit ($\eta : 	ext{Id} 	o T$):** Lifts a pure value into the computational context.
  * In C++26: **`just(x)`**.
* **Multiplication ($\mu : T(T(A)) 	o T(A)$):** Flattens nested computations.
* **Kleisli Arrows ($A 	o T(B)$):** Functions producing fresh asynchronous computations.
* **Monadic Bind ($\gg=$):**
  $$(m \gg= f) = \mu(T(f)(m))$$
  * In C++26: **`let_value(f)`**.

---

# Category Theory: Functors & Reader Monad

### Endofunctor Action ($fmap$)
* Every monad is an endofunctor equipped with arrow mapping:
  $$fmap : (A 	o B) 	o (T(A) 	o T(B))$$
* **`then(f)` in C++26:** Transforms the pure value payload without reshaping the pipeline topology.

### The Reader Monad (Wadler, 1992)
* Models computations dependent on an immutable ambient environment $E 	o A$.
* **`read_env`:** Corresponds to `ask` (querying properties).
* **`write_env`:** Corresponds to `local` (scoped environmental overrides).

---

# Delimited Continuations & Algebraic Effects

### Three-Channel Coproduct as an Effect Algebra (Danvy & Filinski, 1990)
* Rather than a single return point, C++26 standardizes a **three-way coproduct completion**:
  $$	ext{Completion} \cong 	ext{Value} + 	ext{Error} + 	ext{Stopped}$$
* Delimited control operators (`shift`/`reset`) allow intercepting, handling, and masking specific effect channels.
* **Curry-Howard for Control (Griffin, 1990):** Control operators correspond to classical logic typing.
* **Completion Signatures:** Form type-level declarations specifying allowed completion channels and argument types.

---

# Linear Logic & Region Discipline

### Linear Logic (Girard, 1987)
* Senders are **linear by default** — consumed upon a single call to `connect`.
* The **`split`** adaptor applies the contraction structural rule:
  $$\Delta : A 	o A \otimes A$$
* Because duplication is not free, `split` introduces a reference-counted state holding cached results.

### Region Discipline (Tofte & Talpin, 1994)
* **Async Scopes:** Impose lexical and dynamic lifetime boundaries so that background work cannot leak.
* Ensures structured cleanup and prevents dangling detached operations.

---

# Theoretical Mapping: Core & Composition

| Sender Concept | Theoretical Origin | Seminal Reference |
| :--- | :--- | :--- |
| `just(x)` | Monadic return / unit ($\eta$) | Moggi (1991) |
| `then(f)` | Functorial morphism mapping ($fmap$) | Moggi (1991), Milewski (2019) |
| `let_value(f)` | Monadic bind ($\gg=$) / Kleisli extension | Moggi (1991), Wadler (1992) |
| `set_value / set_error / set_stopped` | Algebraic effect channels / Coproduct injection | Danvy & Filinski (1990) |
| `connect(sndr, rcvr)` | CPS continuation reification | Steele & Sussman (1975–1980) |
| `start(op)` | CPS forced evaluation | Plotkin (1975) |

---

# Theoretical Mapping: Concurrency, Effects & Scopes

| Sender Concept | Theoretical Origin | Seminal Reference |
| :--- | :--- | :--- |
| Completion signatures | Type-level signature sets | Griffin (1990) |
| `read_env / write_env` | Reader monad (`ask` / `local`) | Wadler (1992) |
| `when_all` | N-ary tensor product in symmetric monoidal category | Milner (1989, CCS) |
| `split` | Contraction rule in linear logic ($\Delta: A 	o A \otimes A$) | Girard (1987) |
| Async scopes (`associate`) | Region-based lifetime boundaries | Tofte & Talpin (1994) |
| `bulk` family | Data-parallel map primitive | Hillis & Steele (1986) |

---

# 1. Value Lifting: `just`

### Monadic Unit ($\eta$)
Lifts pure values onto the `set_value` channel.
* Completes synchronously, inline, with zero allocations and no scheduling overhead.
* The foundational building block for constructing pure pipelines.

```cpp
auto sndr = just(42, std::string("test"));

// Plain C++ equivalent:
// int a = 42;
// std::string b = "test";
```

---

# 1. Value Lifting: `just_error`

### Algebraic Failure Injection
Lifts an error payload directly onto the `set_error` channel without throwing CPU exceptions.
* Injects failure into the effect algebra as a first-class value.
* Propagates downstream until intercepted by `upon_error` or `let_error`.

```cpp
auto sndr = just_error(
  std::make_error_code(std::errc::invalid_argument)
);

// Plain C++ equivalent:
// throw std::system_error(make_error_code(std::errc::invalid_argument));
```

---

# 1. Value Lifting: `just_stopped`

### First-Class Cooperative Cancellation
Initiates cancellation on the dedicated `set_stopped` channel.
* The third path in the completion coproduct.
* Represents clean, non-error cooperative cancellation as a first-class citizen.

```cpp
auto sndr = just_stopped();

// Plain C++ equivalent:
// Regular C++ has no dedicated cancellation channel
```

---

# 1. Value Lifting: `sync_wait`

### Boundary Consumer & Counit
The boundary consumer connecting asynchronous senders to synchronous C++.
* Blocks the calling thread until the graph completes.
* Single point where lazy descriptions are driven to evaluation.
* Returns `std::optional<std::tuple<...>>`.

```cpp
auto opt = sync_wait(just(42));
if (opt) {
  auto [v] = *opt; // v == 42
}

// Plain C++ equivalent:
// int result = 42;
```

---

# 2. Transformation: `then`

### Functorial $fmap$ on Values
Applies a pure function $f: A 	o B$ to values arriving on `set_value`.
* Transforms values without reshaping the pipeline topology.
* Exceptions thrown inside $f$ are automatically caught and routed to `set_error`.

```cpp
auto sndr = just(3, 4)
  | then([](int a, int b) {
      return a * a + b * b;
    });
// completes with set_value(25)

// Plain C++ equivalent:
// int result = 3 * 3 + 4 * 4;
```

---

# 2. Transformation: `upon_error`

### Cross-Channel Error Recovery
Transforms an error from `set_error` into a fallback value on `set_value`.
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

# 2. Transformation: `upon_stopped`

### Cancellation Recovery to Value
Catches cancellation from `set_stopped` and emits a recovery value on `set_value`.
* Allows a pipeline to provide a default value when upstream operations are cancelled.
* Eliminates the `set_stopped` channel, routing execution back onto `set_value`.

```cpp
auto sndr = just_stopped()
  | upon_stopped([] {
      return 0; // Fallback value on cancellation
    });
// completes with set_value(0)

// Plain C++ equivalent:
// Regular C++ has no dedicated cancellation channel
```

---

# 3. Monadic Composition: `let_value`

### Monadic Bind ($\gg=$) / Kleisli Extension
Accepts a function returning a **fresh sender** and chains it based on upstream results.
* Enables dynamic, multi-stage pipelines where subsequent work depends on earlier values.
* Flattens nested senders ($T(T(A)) 	o T(A)$).

```cpp
auto sndr = just(std::string("hello"))
  | let_value([](std::string s) {
      return just(s + " world")
        | then([](std::string r) {
            return r.size();
          });
    });
// completes with set_value(11)
```

---

# 3. Monadic Composition: `let_error`

### Asynchronous Error Recovery Pipelines
Performs error recovery via an asynchronous fallback pipeline returning a fresh sender.
* Enables complex asynchronous failover routes (e.g., query alternate replica or disk cache).
* Seamlessly branches execution upon encountering errors.

```cpp
auto sndr = just_error(
  make_error_code(errc::timed_out))
  | let_error([](error_code ec) {
      return just(ec.message());
    });
// completes with set_value("timed out")
```

---

# 3. Monadic Composition: `let_stopped`

### Monadic Cancellation Handling
Replaces a stopped signal by instantiating and executing a brand new sender.
* Invoked when upstream cancellation occurs; receives no parameters and returns a sender.
* The Kleisli extension applied specifically to the stopped summand of the coproduct.

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

# 4. Execution Contexts: `schedule`

### Gateway into Execution Contexts
Produces an empty sender completing on the execution context of the specified scheduler.
* Schedulers represent compute resources (OS threads, I/O runloops, GPU streams).
* Serves as the standard entry point into an execution context.

```cpp
auto sndr = schedule(pool.get_scheduler())
  | then([] {
      return handle_request();
    });

// Plain C++ equivalent:
// pool.post([&] { handle_request(); });
```

---

# 4. Execution Contexts: `starts_on`

### Initiating Pipelines on Schedulers
Arranges for an existing sender pipeline to begin execution on the specified scheduler.
* Ensures that the initial step and evaluation start on the chosen context.

```cpp
auto sndr = starts_on(
  pool.get_scheduler(),
  just(request) | then([](auto req) {
    return handle(req);
  })
);

// Plain C++ equivalent:
// pool.post([&] { auto res = handle(request); });
```

---

# 4. Execution Contexts: `continues_on`

### Downstream Context Transfer (Handoff)
Transfers downstream execution to another scheduler after the predecessor completes.
* Standard pattern: perform I/O on runloop, transfer compute to worker pool.

```cpp
auto sndr =
  starts_on(io_ctx.get_scheduler(),
            async_read(socket, buf))
  | continues_on(pool.get_scheduler())
  | then([](auto data) {
      return parse(data); // runs on worker pool!
    });
```

---

# 4. Execution Contexts: `on`

### Round-Trip Context Swapping
Executes a subcomputation on a target scheduler, then returns execution to the caller's context.
* Brackets computation with a context switch and its inverse.

```cpp
auto sndr = on(
  pool.get_scheduler(),
  just(request) | then([](auto req) {
    return handle(req);
  })
);
// Result automatically returns to caller's original scheduler
```

---

# 4. Execution Contexts: `affine`

### Scheduler Affinity Optimization (P4151R1 / P3941R4)
Ensures completion on the receiver's start scheduler (`get_start_scheduler`).
* **Critical Optimization:** Skips rescheduling overhead if sender already completes on the target context.
* Foundation of coroutine resumption affinity in C++26 `task<T>`.

```cpp
auto sndr = some_operation()
  | affine;
```

---

# 4. Execution Contexts: `schedule_from`

### Three-Channel Scheduler Dispatch
Low-level primitive guaranteeing that **all three channels** (value, error, stopped) arrive on the scheduler.
* Deeper than `continues_on` (which transitions only the value channel).
* Essential tool for engine and driver implementers.

```cpp
auto sndr = schedule_from(
  pool.get_scheduler(),
  async_read(socket, buf)
);
```

---

# 5. Environment: `read_env`

### Reader Monad: Querying Environment (`ask`)
Queries ambient properties (schedulers, allocators, stop tokens) from the receiver's environment.
* Enables context-dependent pipelines parameterized by caller environment.

```cpp
auto sndr = read_env(get_scheduler)
  | let_value([](auto sch) {
      return schedule(sch)
        | then([] { return load_config(); });
    });
```

---

# 5. Environment: `write_env`

### Reader Monad: Injecting Properties (`local`)
Overlays properties onto downstream receivers.
* Injects custom PMR allocators or stop tokens down the tree without parameter clutter.

```cpp
auto sndr = write_env(
  make_env(get_allocator, pool_alloc),
  process_file(path)
);
```

---

# 5. Environment: `unstoppable`

### Algebraic Effect Masking
Shields a sender from ambient cancellation signals:
* **Semantics:** Ensures that critical operations run to completion regardless of parent stop tokens.
* **Type System Impact:** The sender's completion signatures lose `set_stopped_t()`.
* **Use Cases:** Transaction commits, resource cleanups, hardware finalization routines.

```cpp
auto sndr = unstoppable(
  commit_transaction(db, txn)
);

// Plain C++ equivalent:
// commit_transaction(db, txn);
```

---

# 6. Structured Concurrency: `when_all`

### N-ary Concurrency Join
Starts all child senders concurrently and joins values by concatenating them in order.
* **Structural Guarantee:** No child outlives the join point.
* If any child emits an error or stop, all siblings are cancelled immediately.

```cpp
auto sndr = when_all(
  fetch_user(user_id),
  fetch_orders(user_id),
  fetch_preferences(user_id)
) | then([](auto user, auto orders, auto prefs) {
    return build_profile(user, orders, prefs);
  });
```

---

# 6. Structured Concurrency: `when_all_with_variant`

### Heterogeneous Concurrency Join
Concurrent join accommodating children with heterogeneous or multiple value signatures.
* Each child output is wrapped into `std::variant<std::tuple<...>>`.
* Relaxes single-signature constraints across parallel branches.

```cpp
auto sndr = when_all_with_variant(
  fetch_json(url_a),
  fetch_binary(url_b)
) | then([](auto json_var, auto bin_var) {
    return combine(json_var, bin_var);
  });
```

---

# 6. Linear Logic: `split`

### Contraction Rule ($\Delta : A 	o A \otimes A$)
Converts a single-shot (linear) sender into a multi-shot (sharable) sender.
* Senders are consumed by connect. Duplicating work requires explicit contraction.
* Allocates a reference-counted state that caches and distributes the result.

```cpp
auto shared = split(fetch_data(url));

auto sndr = when_all(
  shared | then([](auto data) { return analyze(data); }),
  shared | then([](auto data) { return archive(data); })
);

// Plain C++ equivalent:
// auto data = fetch_data(url);
// auto analysis = analyze(data);
// auto archived = archive(data);
```

---

# 7. Signal Adaptation: `into_variant`

### Unifying Value Signatures
Folds multiple distinct value completion signatures into a single `std::variant`.
* Enables passing multi-signature senders into algorithms expecting uniform signatures like `when_all`.

```cpp
auto sndr = query_database(stmt)
  | into_variant;

// completes with:
// set_value(variant<tuple<Row>, tuple<int>>)
```

---

# 7. Signal Adaptation: `stopped_as_optional`

### Mapping Cancellation to Optional
Collapses `set_stopped` into `std::nullopt` on the value channel; values become `std::optional<T>`.
* Merges cancellation into simple monadic `std::optional` branch checks.

```cpp
auto sndr = dequeue(work_queue)
  | stopped_as_optional;

// value(item) -> set_value(optional(item))
// stopped()   -> set_value(optional())
```

---

# 7. Signal Adaptation: `stopped_as_error`

### Converting Cancellation to Error
Reclassifies cancellation (`set_stopped`) as an explicit error on `set_error`.
* Replaces the stopped channel with an error code, allowing standard error combinators (`upon_error`, `let_error`) to handle timeouts uniformly.

```cpp
auto sndr = timed_operation(deadline)
  | stopped_as_error(
      make_error_code(std::errc::timed_out)
    );

// Plain C++ equivalent:
// auto result = timed_operation(deadline);
// if (!result) throw system_error(make_error_code(errc::timed_out));
```

---

# 8. Data Parallelism: `bulk`

### Data-Parallel Map Primitive (Hillis & Steele, 1986)
Invokes $f(i, 	ext{args}\dots)$ for each index in $[0, 	ext{shape})$ under an execution policy.
* Single algorithm spanning sequential loops, multicore parallel maps, or GPU kernels.
* Forwards value arguments to all iterations without cloning.

```cpp
auto sndr = just(records)
  | bulk(std::par, records.size(),
      [](size_t i, auto& recs) {
        recs[i].score = compute_score(recs[i]);
      });
```

---

# 8. Data Parallelism: `bulk_chunked`

### Hardware-Aligned Chunked Parallelism
Partitions the index space into cache-friendly chunks for vectorized / SIMD execution.
* Exposes chunk index ranges directly to the callable.

```cpp
auto sndr = just(data)
  | bulk_chunked(std::par, data.size(),
      [](auto chunk, auto& d) {
        for (auto i : chunk)
          d[i] = transform(d[i]);
      });
```

---

# 8. Data Parallelism: `bulk_unchunked`

### Element-Wise Parallel Iteration
Invokes the callable once per index with no chunking guarantees:
* Simplest mental model for parallel iteration.
* `bulk` dispatches by default to `bulk_chunked`, which the underlying scheduler may specialize for target hardware topologies.

```cpp
auto sndr = just(pixels)
  | bulk_unchunked(std::par, pixels.size(),
      [](size_t i, auto& px) {
        px[i] = apply_filter(px[i]);
      });

// Plain C++ equivalent:
// std::for_each(std::execution::par, pixels.begin(), pixels.end(),
//   [](auto& p) { p = apply_filter(p); });
```

---

# 9. Async Scopes: `associate`

### Region Discipline for Detached Senders (Tofte & Talpin, 1994)
Ties an independently spawned sender's lifetime to an enclosing `counting_scope`.
* Guarantees zero orphaned background tasks during server shutdown.

```cpp
counting_scope scope;
auto token = scope.get_token();

for (auto& conn : accepted_connections) {
  auto sndr = associate(token,
    starts_on(pool.get_scheduler(),
              handle_connection(conn)));
}
// scope.join() asynchronously drains all active connections
```

---

# 9. Async Scopes: `spawn_future`

### Structured Async Futures
Spawns background work into a scope while returning a sender handle to its future result.
* The structured, safe equivalent of `std::async`.

```cpp
auto sndr = spawn_future(token,
  starts_on(pool.get_scheduler(),
            fetch_data(key)))
  | then([](auto data) {
      return process(data);
    });

// Plain C++ equivalent:
// auto future = std::async(launch::async, [&] { return fetch(); });
```

---

# Coroutines in C++26: `execution::task<T>`

### Unifying `co_await` with Sender Graphs (P3552R3)
* **Lazy Evaluation:** Coroutine body does not execute until connected to a receiver and started.
* **Dual Nature:**
  * Can `co_await` any sender inside `task<T>`.
  * Is itself a sender (composable via `|`, `when_all`, etc.).
* **Scheduler Affinity:** Automatically injects `affine` around `co_await` expressions to resume execution on the task's originating context.
* **Typed Errors:** `co_yield with_error(e)` dispatches directly to `set_error` without throwing.

---

# Coroutines in C++26: Code Example

### Heterogeneous Sensor Fusion
```cpp
task<sensor_data> read_sensors(sensor_array const& sensors) {
  // Concurrent acquisition across 3 distinct hardware schedulers:
  auto [lidar, radar, camera] = co_await
    when_all(
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
* Three distinct execution contexts, unified structured join, zero boilerplate.

---

# Custom Adaptors: The `retry` Concept

### P4014R2 Reference Adaptor Implementation
To demonstrate how custom algorithms integrate into the sender sub-language, P4014R2 details the implementation of a `retry` adaptor.

* **Architecture:**
  * A **Sender** modeling the algorithm contract and completion signatures.
  * A **Receiver** intercepting `set_error` from the child operation.
  * An **Operation State** that holds the receiver and re-connects upon error.
* **Algebraic Absorption:** Absorbs the error channel, retrying until success (`set_value`) or explicit stop (`set_stopped`).

---

# Custom Adaptors: `retry` Implementation

```cpp
template <class S>
struct _retry_sender {
  using sender_concept = stdexec::sender_t;
  S s_;

  template <class R>
  auto connect(R r) && -> _op<S, R> {
    return { static_cast<S&&>(s_), static_cast<R&&>(r) };
  }
};

// Operation state reconstructs and restarts upon set_error:
void _retry() noexcept {
  STDEXEC_TRY {
    o_.emplace(_connect());
    stdexec::start(*o_);
  } STDEXEC_CATCH_ALL {
    stdexec::set_error(static_cast<R&&>(r_), std::current_exception());
  }
}
```

---

# Multilayer Autonomous Vehicle System

### 4-Tier Real-Time Architecture (P4014R2 Section 13)

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

# Layered System: Separation of Concerns

### Coroutines vs. Pipelines
* **Pipelines (Senders):**
  * Excel at concurrent joins (`when_all`), channel mapping (`upon_error`), and declarative graph construction.
  * Static type composition enables aggressive inlining and zero dynamic allocation.
* **Tasks (Coroutines):**
  * Excel at sequential imperative business logic, complex branch conditions, and local variable scoping.
  * `co_await` integration bridges seamlessly with pipeline outputs.

---
layout: end
---

# Thank You!
### Questions & Discussion

**References:**
* ISO/IEC JTC1/SC22/WG21 P4014R2 (Vinnie Falco, Mungo Gill, May 2026)
* P3552R3: *Add a Coroutine Task Type* (Dietmar Kühl, Maikel Nadolski)
* P4151R1 / P3941R4: *Scheduler Affinity and affine redesign* (Robert Leahy, Dietmar Kühl)

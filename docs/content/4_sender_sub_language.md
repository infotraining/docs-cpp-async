---
title: The Sender Sub-Language
---

# The Sender Sub-Language

The previous chapter introduced senders, receivers, operation states, and schedulers. This chapter looks at how sender algorithms fit together to describe larger asynchronous computations.

The examples use `ex::` as a short, illustrative namespace for an execution library. The exact namespace and available algorithms depend on the C++ standard library or third-party implementation in use. In particular, facilities such as `task`, asynchronous scopes, and the `bulk` variants are not available in every implementation.

## A compositional model

A sender is a lazy description of work and its possible completion signals. Sender adaptors transform or combine those descriptions. Connecting the final sender to a receiver creates an operation state; starting that state initiates the computation.

This model has a useful connection to continuation-passing style (CPS). In CPS, a function receives an explicit continuation describing what should happen next. A receiver plays a similar role: it is the continuation that accepts a sender's eventual completion.

```{code-block} cpp
auto work = ex::just(10)
    | ex::then([](int value) { return value * 2; });

auto operation = ex::connect(std::move(work), receiver);
ex::start(operation);
```

`connect` makes the relationship between the sender and its continuation concrete in an operation state. `start` begins the work. The graph can be composed before it is connected and started; constructing the description alone does not normally perform the asynchronous work.

This is an analogy, not a claim that every sender is implemented as a literal continuation or that all sender operations have a formal monadic or effect-system interpretation. Those connections are useful ways to reason about composition:

| Sender idea | Related idea |
| --- | --- |
| `just(value)` | Introduce a value into a computation |
| `then(function)` | Map a function over successful values |
| `let_value(function)` | Bind a value to a computation that returns another sender |
| `set_value`, `set_error`, `set_stopped` | Distinct completion channels |
| `connect(sender, receiver)` | Associate work with its continuation |
| `when_all` | Compose independent work concurrently |

## Completion channels

A sender can complete through one of three channels:

- **Value** — `set_value(values...)` reports successful completion. A sender may produce zero, one, or several values.
- **Error** — `set_error(error)` reports failure. The error type is determined by the sender's completion signatures.
- **Stopped** — `set_stopped()` reports cooperative cancellation, distinct from both success and failure.

An adaptor can observe or transform one channel while allowing the others to pass through. Thinking in terms of these channels helps make recovery and cancellation behavior explicit.

## Starting and observing work

### `just`, `just_error`, and `just_stopped`

These sender factories introduce an already-known completion into a pipeline:

```{code-block} cpp
auto value = ex::just(42, std::string("answer"));

auto failure = ex::just_error(
    std::make_error_code(std::errc::invalid_argument));

auto cancelled = ex::just_stopped();
```

`just` completes with its values. `just_error` and `just_stopped` exercise the corresponding non-value channels. They bring values into the pipeline immediately, allowing you to test or handle specific completion scenarios without performing actual asynchronous work.

### `sync_wait`

`sync_wait` is a boundary from sender-based asynchronous composition to blocking, synchronous code. It waits for completion on the calling thread and returns a representation of the value completion, commonly an optional tuple. A stopped completion may be represented by an empty optional; error behavior depends on the library.

```{code-block} cpp
auto result = ex::sync_wait(ex::just(42));

if (result) {
    auto [value] = *result;
    assert(value == 42);
}
```

Because it blocks, `sync_wait` is generally best used at a program boundary, in tests, or in small examples—not inside an event loop or a thread that must remain responsive.

We can use structured bindings to unpack the values returned by `sync_wait`, making it convenient to work with multiple return values.

```{code-block} cpp
auto [a, b, c] = ex::sync_wait(ex::just(1, 2, 3)).value();
```

## Transforming completions

### `then`: transform values

`then` invokes a function when its predecessor completes with values. The function's result becomes the next value completion. An exception thrown by the function is normally represented as an error completion according to the sender library's rules.

```{code-block} cpp
auto area_squared = ex::just(3, 4)
    | ex::then([](int width, int height) {
          return width * height;
      });
```

Use `then` when the next step is an ordinary computation. If the next step itself is asynchronous and returns a sender, use `let_value`.

### `upon_error` and `upon_stopped`: recover in place

`upon_error` handles an error by invoking a function and turning its result into a value completion. `upon_stopped` does the same for cancellation. These are convenient when recovery is synchronous:

```{code-block} cpp
auto recovered = fetch_record(id)
    | ex::upon_error([](std::error_code) {
          return Record{};
      })
    | ex::upon_stopped([] {
          return Record{};
      });
```

The handlers change the meaning of the handled completion: downstream code sees a value instead of that error or stopped signal. Use a fallback only when it is valid for the application to treat that outcome as a value.

## Composing computations

Sometimes the next operation depends on a value from the previous operation and produces another sender. The `let_*` adaptors flatten that sender into the pipeline rather than producing a sender as an ordinary value.

### `let_value`

```{code-block} cpp
auto length = ex::just(std::string("hello"))
    | ex::let_value([](std::string text) {
          return ex::just(text + " world")
              | ex::then([](const std::string& result) {
                    return result.size();
                });
      });
```

The function passed to `let_value` receives the upstream values and returns a new sender. That sender determines the downstream completion types. This is useful for dependent operations such as looking up a record and then fetching its details.

### `let_error` and `let_stopped`

The corresponding adaptors start a new sender when the input completes with an error or stopped signal:

```{code-block} cpp
auto with_fallback = primary_lookup(key)
    | ex::let_error([&](std::error_code) {
          return cache_lookup(key);
      })
    | ex::let_stopped([] {
          return ex::just(Result::cancelled());
      });
```

Unlike `upon_error` and `upon_stopped`, these adaptors allow the recovery path to perform asynchronous work. A `let_*` handler can also propagate another error or a stopped completion from its returned sender.

## Choosing an execution context

A scheduler represents a way to submit work to an execution context, such as a thread pool or event loop. Scheduling algorithms describe where work should begin or continue; they do not themselves guarantee that every implementation has the same set of scheduler types or context-switching behavior.

### `schedule` and `starts_on`

`schedule(scheduler)` creates a sender that completes on the scheduler's execution context. `starts_on(scheduler, sender)` arranges for a sender computation to begin there.

```{code-block} cpp
auto request = ex::starts_on(
    pool.get_scheduler(),
    ex::just(request_id)
        | ex::let_value([](auto id) { return load_request(id); }));
```

Use these operations when the place where work starts is significant, for example when dispatching CPU-bound work to a worker pool.

### `continues_on` and `on`

`continues_on` expresses a transition to another scheduler for subsequent work. A common pattern is to perform I/O using an I/O context and move CPU-heavy processing to a worker pool:

```{code-block} cpp
auto parsed = async_read(socket, buffer)
    | ex::continues_on(pool.get_scheduler())
    | ex::then([](auto data) {
          return parse(data);
      });
```

`on(scheduler, sender)` is useful for expressing a subcomputation on a chosen scheduler while preserving the surrounding execution context according to that algorithm's contract. Consult the implementation's documentation for exact completion-context guarantees, especially for errors and cancellation.

Some libraries also provide affinity adaptors such as `affine`, which can avoid an unnecessary reschedule when a sender already completes on the desired context.

## Environments and context

Sender environments carry ambient information needed by an operation, for example a scheduler, allocator, or stop token. Environments allow context to be supplied without adding another explicit parameter to every function.

### `read_env`

`read_env(query)` reads a property from the receiver's environment when the sender is connected:

```{code-block} cpp
auto on_current_scheduler = ex::read_env(ex::get_scheduler)
    | ex::let_value([](auto scheduler) {
          return ex::schedule(scheduler);
      });
```

This makes a sender depend on the context in which it is eventually used.

### `write_env`

`write_env(environment, sender)` wraps a sender so that it and its downstream work observe the supplied or overridden environment:

```{code-block} cpp
auto configured = ex::write_env(
    make_env(ex::get_allocator, pool_allocator),
    process_file(path));
```

The exact queries and environment-building APIs vary by library. Environments are most useful for cross-cutting execution details; ordinary application data is usually clearer as an explicit sender value or function argument.

### Cancellation control

Cancellation is cooperative: a stop request asks participating operations to stop, but cannot forcibly interrupt arbitrary work. An adaptor such as `unstoppable`, where provided, can prevent ambient cancellation from reaching a critical operation. This should be used sparingly—for example, to protect a cleanup step that must finish—and does not make an operation immune to all possible failures.

## Structured concurrency

Structured concurrency ties child work to a parent operation's lifetime. A parent should not complete while child operations it owns are still accessing its resources.

### `when_all`

`when_all` starts multiple senders and completes after their results are collected:

```{code-block} cpp
auto profile = ex::when_all(
        fetch_user(user_id),
        fetch_orders(user_id),
        fetch_preferences(user_id))
    | ex::then([](auto user, auto orders, auto preferences) {
          return build_profile(user, orders, preferences);
      });
```

The input senders are independent and can make progress concurrently, subject to their schedulers and implementations. If a child fails or is stopped, the join may request cancellation of siblings. That request is cooperative; the join still has to respect the lifetime of its child operations.

Some libraries provide `when_all_with_variant` for children whose value completions have different signatures. Such algorithms typically represent alternatives with a `std::variant`-based result. Check the library's type and error rules before composing heterogeneous senders.

### Sharing a result with `split`

Many senders represent single-shot work. An adaptor such as `split` can turn one execution into a shared result for multiple consumers:

```{code-block} cpp
auto shared_data = ex::split(fetch_data(url));

auto analyses = ex::when_all(
    shared_data | ex::then([](auto data) { return analyze(data); }),
    shared_data | ex::then([](auto data) { return archive(data); }));
```

Sharing usually requires state to retain and distribute the completion. Consider the storage, synchronization, and value-copying behavior documented by the implementation; sharing is not necessarily free.

## Adapting completion signatures

Adapters can make different completion paths easier to compose, but they also change what downstream code must handle.

### `into_variant`

`into_variant` represents alternative value signatures using a variant, which can be helpful when a sender has more than one possible value shape:

```{code-block} cpp
auto result = query_database(statement)
    | ex::into_variant;
```

The exact variant alternatives depend on the sender's completion signatures and the library's representation.

### Mapping stopped to an ordinary result

`stopped_as_optional` converts cancellation into an empty optional on the value channel. Successful values are represented as present values:

```{code-block} cpp
auto maybe_item = dequeue(queue)
    | ex::stopped_as_optional;
```

This can be convenient when cancellation naturally means “no result.” It also removes the distinction between cancellation and an empty result for downstream code.

`stopped_as_error` instead maps cancellation to an error completion, such as a timeout error:

```{code-block} cpp
auto timed = timed_operation(deadline)
    | ex::stopped_as_error(
        std::make_error_code(std::errc::timed_out));
```

Choose a mapping that preserves the meaning the caller needs. Avoid converting cancellation into success or failure merely to simplify the type if callers need to distinguish it.

## Data-parallel work

The `bulk` family expresses work over an index range as part of a sender computation. A policy and scheduler determine how that work is executed; parallel execution requires that iterations meet the relevant independence and synchronization requirements.

For example, a point cloud can be filtered after it has arrived as a sender value. The sender's value arguments are passed to the bulk function for each index:

```{code-block} cpp
auto filtered = ex::just(point_cloud)
    | ex::bulk(std::execution::par, point_cloud.size(),
          [](std::size_t index, auto& points) {
              points[index] = filter_point(points[index]);
          });
```

The shape (`point_cloud.size()` here) defines the index range `[0, shape)`. Each invocation should operate safely with respect to other indices. Writing to distinct elements is often suitable; updating shared state without synchronization can introduce a data race.

Some libraries offer chunked and unchunked variants. A chunked algorithm passes a range of indices to each invocation, allowing the implementation to group work for locality or vectorization:

```{code-block} cpp
auto filtered = ex::just(point_cloud)
    | ex::bulk_chunked(std::execution::par, point_cloud.size(),
          [](auto chunk, auto& points) {
              for (auto index : chunk) {
                  points[index] = filter_point(points[index]);
              }
          });
```

An unchunked algorithm exposes one index per invocation:

```{code-block} cpp
auto filtered = ex::just(point_cloud)
    | ex::bulk_unchunked(std::execution::par, point_cloud.size(),
          [](std::size_t index, auto& points) {
              points[index] = filter_point(points[index]);
          });
```

The names, signatures, policies, and availability of `bulk_chunked` and `bulk_unchunked` are library- or proposal-specific; they are not universally portable sender algorithms. Choose chunking based on the execution library and workload rather than assuming that one variant is always faster.

## Asynchronous scopes

When work is spawned dynamically—for example, once per accepted connection—an asynchronous scope can associate child lifetimes with a parent scope. The scope must be joined or otherwise drained before the resources used by its children are destroyed.

```{code-block} cpp
ex::counting_scope scope;
auto token = scope.get_token();

for (auto& connection : accepted_connections) {
    auto child = ex::associate(
        token,
        ex::starts_on(pool.get_scheduler(),
                      handle_connection(connection)));
    spawn(child); // Illustrative: spawning API is library-specific.
}

co_await scope.join();
```

Scope and spawn APIs differ between libraries. The important design rule is to make the ownership and shutdown boundary explicit rather than launching detached work that may outlive the objects it uses.

## Coroutines and senders

Sender pipelines and coroutines are complementary ways to express asynchronous control flow. A coroutine task can often be awaited by other senders, and a sender can often be awaited within a coroutine, when the library provides the appropriate integration.

### A task that is also a sender

Some execution libraries provide a lazy `task<T>` type. A task's coroutine body begins when the task is connected and started, and its result is delivered through the sender completion channels:

```{code-block} cpp
ex::task<int> answer() {
    co_return 42;
}

auto result = ex::sync_wait(answer());
```

This `task<T>` is library-specific; it is not a general coroutine type supplied by the C++ language itself.

### Awaiting a sender

With sender-aware coroutine support, a coroutine can await a sender and continue with its value:

```{code-block} cpp
ex::task<int> add_one() {
    int value = co_await ex::just(41);
    co_return value + 1;
}
```

The mapping from completion channels into coroutine behavior depends on the task implementation. A common convention is that a single value from `set_value` becomes the `co_await` result, multiple values become a tuple, and a completion with no values becomes `void`. Errors may be translated into exceptions, while `set_stopped` may cancel the coroutine without resuming it. Check those rules rather than assuming that every sender can be awaited in the same way.

For example, an I/O sender that returns both an error code and a byte count can expose both as values for the coroutine to inspect:

```{code-block} cpp
ex::task<void> read_request(Socket& socket, Buffer& buffer) {
    auto [error, bytes_read] = co_await async_read(socket, buffer);

    if (!error) {
        process(buffer, bytes_read);
    } else if (error == std::errc::connection_reset) {
        cleanup_partial_request(bytes_read);
    }
}
```

Some task types also provide a typed way to complete on the error channel without throwing. In the following illustrative API, `with_error` is yielded as an error completion and can be handled by sender adaptors such as `let_error`:

```{code-block} cpp
ex::task<void> validate(const Request& request) {
    if (!request.valid()) {
        co_yield ex::with_error(
            std::make_error_code(std::errc::invalid_argument));
    }
    co_return;
}
```

This mechanism is specific to task implementations that support it. Where an awaited sender's error is instead translated to an exception, ordinary `try`/`catch` provides the coroutine-side bridge:

```{code-block} cpp
ex::task<void> connect_and_log(Socket& socket, Endpoint endpoint) {
    try {
        co_await async_connect(socket, endpoint);
    } catch (const std::system_error& error) {
        log("connect failed: {}", error.code().message());
    }
}
```

Likewise, cancellation behavior belongs to the task's contract. In an implementation where awaiting a stopped sender propagates cancellation, a stop request can end the coroutine through the stopped completion path; it is not necessarily an exception or an ordinary return value.

### Awaiting a pipeline

An entire pipeline can be awaited as one expression:

```{code-block} cpp
ex::task<std::string> fetch_and_transform(Url url) {
    auto body = co_await (
        async_fetch(url)
        | ex::then([](auto response) { return response.body(); })
        | ex::then([](auto text) { return compress(text); }));

    co_return body;
}
```

If a task implementation provides scheduler affinity, it can arrange for the coroutine to resume on its originating scheduler after an awaited sender completes elsewhere. For example, a task started on a worker scheduler may resume there after an I/O sender completes:

```{code-block} cpp
ex::task<void> parse_after_read(Socket& socket, Buffer& buffer) {
    co_await async_read(socket, buffer);
    auto request = parse(buffer); // Resumes on the task scheduler if affinity is provided.
    co_await handle_request(request);
}

auto operation = ex::starts_on(pool.get_scheduler(),
                               parse_after_read(socket, buffer));
```

Some task implementations expose the start scheduler through the receiver environment and apply an affinity adaptor around awaited senders. They may also allow affinity to be disabled by selecting an inline scheduler, in which case the coroutine resumes wherever the awaited sender completes:

```{code-block} cpp
struct no_affinity {
    using start_scheduler_type = ex::inline_scheduler;
};

ex::task<void, no_affinity> inline_task() {
    co_await async_operation();
    // No task-level scheduler rescheduling is requested here.
}
```

These scheduler and task customization points are implementation-specific. They are behavior of the task type, not automatic properties of all C++ coroutines.

## Putting the pieces together

The following example follows an autonomous vehicle's control path from sensor acquisition to a brake command. The domain types, I/O operations, and execution-library APIs are illustrative; the sequence demonstrates how sender pipelines and coroutine tasks can be composed.

### Sensor fusion

Each sensor read starts on its associated execution context. The task awaits a structured join, then fuses the three results:

```{code-block} cpp
ex::task<SensorData> read_sensors(const Sensors& sensors) {
    auto [lidar, radar, camera] = co_await ex::when_all(
        ex::starts_on(sensors.lidar_scheduler(),
                      async_read_lidar(sensors.lidar()))
            | ex::then(decode_lidar),
        ex::starts_on(sensors.radar_scheduler(),
                      async_read_radar(sensors.radar()))
            | ex::then(decode_radar),
        ex::starts_on(sensors.camera_scheduler(),
                      async_read_camera(sensors.camera()))
            | ex::then(decode_camera));

    co_return fuse(lidar, radar, camera);
}
```

### Collision detection and decision

The task itself can be used as a sender. The pipeline detects obstacles, then uses `let_value` because choosing a command produces a sender:

```{code-block} cpp
auto collision_pipeline(const Sensors& sensors,
                        const VehicleState& state) {
    return read_sensors(sensors)
        | ex::then([&state](auto data) {
              return detect_obstacles(data, state);
          })
        | ex::let_value([](auto obstacles) {
              if (obstacles.imminent()) {
                  return ex::just(
                      BrakeCommand{obstacles.severity()});
              }
              return ex::just(BrakeCommand::none());
          });
}
```

### Actuator dispatch

The actuator task awaits the collision pipeline and schedules a write on the brake controller's real-time context when braking is required:

```{code-block} cpp
ex::task<ActuatorResult> actuate_brakes(
    const Sensors& sensors,
    const VehicleState& state,
    BrakeController& brakes) {
    auto command = co_await collision_pipeline(sensors, state);

    if (command.requires_braking()) {
        co_await ex::starts_on(
            brakes.realtime_scheduler(),
            async_write(brakes.channel(), command));
        co_return ActuatorResult::applied(command);
    }

    co_return ActuatorResult::no_action();
}
```

### Structured lifetime and failover

An asynchronous scope ties controller work to shutdown. The sender pipeline handles sensor or actuator errors with an asynchronous emergency-brake path and maps cancellation to a non-error result:

```{code-block} cpp
auto safety_controller(
    const Sensors& sensors,
    const VehicleState& state,
    BrakeController& brakes,
    ex::counting_scope& scope) {
    return ex::associate(
        scope.get_token(),
        actuate_brakes(sensors, state, brakes)
            | ex::let_error([&](std::error_code error) {
                  return ex::starts_on(
                             brakes.realtime_scheduler(),
                             async_write(
                                 brakes.channel(),
                                 BrakeCommand::emergency()))
                      | ex::then([error](auto) {
                            log("sensor failure: {}",
                                error.message());
                            return ActuatorResult::emergency();
                        });
              })
            | ex::upon_stopped([] {
                  return ActuatorResult::no_action();
              }));
}

// During shutdown, await the scope's join operation before
// destroying resources used by its associated work.
```

The exact scope-spawning and joining API differs between libraries. The important lifetime rule is to drain associated work before destroying the sensors, controller, or other objects it accesses.

### Coroutine-only alternative

The same high-level control flow can be written with coroutine-style error handling. This alternative emphasizes sequential code, but its catch-all handler deliberately loses error detail and does not itself show the scope ownership from the sender version:

```{code-block} cpp
ex::task<ActuatorResult> safety_controller_task(
    const Sensors& sensors,
    const VehicleState& state,
    BrakeController& brakes) {
    try {
        auto data = co_await read_sensors(sensors);
        auto obstacles = detect_obstacles(data, state);

        if (!obstacles.imminent()) {
            co_return ActuatorResult::no_action();
        }

        auto command = BrakeCommand{obstacles.severity()};
        co_await ex::starts_on(
            brakes.realtime_scheduler(),
            async_write(brakes.channel(), command));
        co_return ActuatorResult::applied(command);
    } catch (const std::system_error& error) {
        log("sensor failure: {}", error.what());
        co_await ex::starts_on(
            brakes.realtime_scheduler(),
            async_write(
                brakes.channel(),
                BrakeCommand::emergency()));
        co_return ActuatorResult::emergency();
    }
}
```

The two styles are complementary: sender composition makes branching, completion channels, and scope association explicit; the coroutine version presents a sequential control flow and uses the task's documented error and cancellation mapping. A production controller must additionally define deadlines, cancellation policy, failure handling for the emergency action, and resource lifetimes.

Sender composition is most useful when these relationships are explicit. Use value transformations for ordinary computation, `let_*` when a branch starts another asynchronous operation, scheduler adaptors when placement matters, and structured joins or scopes when child lifetimes must be bounded.

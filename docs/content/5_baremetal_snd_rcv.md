# Intel Baremetal Senders/Receivers — Full Technical Walkthrough

This document is a practical, hands‑on walkthrough for using Intel’s Baremetal Senders/Receivers (S/R) library on microcontroller firmware. It is organized in three parts:

1. **Simple example** — a minimal, end‑to‑end pipeline: periodic sampling → processing → UART send.  
2. **More complex example** — composing multiple senders with `when_all`, error handling, and resource considerations.  
3. **Binding to specific interrupts** — using `incite_on` (or `incite_on`‑style API) to run parts of the pipeline on particular hardware interrupts, with ISR safety and priority notes.

Each part contains code, explanation of the runtime behavior, lifetime and memory notes, and practical tips for embedded constraints (no heap, no exceptions, interrupt-driven concurrency).

---

## Conventions and assumptions

- API names follow the Intel Baremetal S/R style: `async::periodic`, `async::let_value`, `async::just_value`, `async::when_all`, `async::start_detached`, `async::trigger_scheduler`, `async::incite_on` (or equivalent). Replace with exact names from your SDK if they differ.  
- Target: baremetal microcontroller (e.g., ARM Cortex‑M). No OS, no threads, concurrency via interrupts.  
- No dynamic allocation and no exceptions. All objects are static or stack‑allocated with deterministic lifetimes.  
- `trigger_scheduler` is the interrupt‑driven scheduler primitive that the library uses to wake senders.  
- `start_detached` registers a sender pipeline with the runtime and returns immediately.

---

### Quick glossary

- **sender**: a description of asynchronous work that will eventually produce a value, error, or stopped signal.  
- **receiver**: the consumer of a sender’s completion signals (value/error/stopped).  
- **scheduler / trigger_scheduler**: the primitive that wakes senders; on baremetal this is typically an ISR or a deferred handler.  
- **incite_on**: an operator that moves downstream execution to a specified scheduler/interrupt context.  
- **when_all**: an operator that waits for multiple senders to complete and yields their results together.

---

# Part 1 — Simple example: periodic sampling → processing → UART send

### Goal

Sample an ADC every 10 ms, apply a simple filter, and send the processed value over UART. The pipeline must be interrupt‑driven, deterministic, and use no heap.

### Key building blocks

- `async::trigger_scheduler irq_sched{}` — scheduler bound to a hardware timer interrupt.  
- `async::periodic(irq_sched, 10ms)` — sender that emits a tick every 10 ms when the scheduler is triggered.  
- `async::let_value` — transforms a value produced by a sender.  
- `async::just_value(v)` — a sender that completes immediately with `v`.  
- `async::start_detached(pipeline)` — register pipeline to run when scheduler fires.

### Minimal code

```cpp
#include <async/periodic.hpp>
#include <async/let_value.hpp>
#include <async/start_detached.hpp>
#include <async/trigger_scheduler.hpp>
#include <chrono>

using namespace std::chrono_literals;

async::trigger_scheduler irq_sched{}; // bound to timer ISR

int read_adc();            // platform ADC read (synchronous, fast)
int lowpass_filter(int);   // simple filter
void uart_send(int);       // synchronous UART send (non-blocking or short)

void setup_pipeline() {
    auto pipeline =
        async::periodic(irq_sched, 10ms)
        | async::let_value([]() {
            return read_adc();
        })
        | async::let_value([](int raw) {
            return lowpass_filter(raw);
        })
        | async::let_value([](int processed) {
            uart_send(processed);
            return async::just_value(processed);
        });

    async::start_detached(pipeline);
}

// Timer ISR (platform-specific) should call irq_sched.trigger() or similar
extern "C" void TIM_IRQHandler() {
    // clear timer interrupt flag
    irq_sched.trigger(); // wake the scheduler; library runs one pipeline step
}
```

### Runtime behavior

1. Timer ISR fires every 10 ms and calls `irq_sched.trigger()`.  
2. The scheduler wakes the `periodic` sender; it emits a tick.  
3. `let_value` chain executes synchronously in the ISR context (or in the scheduler's context) and calls `read_adc()`, `lowpass_filter()`, and `uart_send()` in sequence.  
4. The pipeline completes and suspends until the next tick.

### Important notes

- **ISR vs deferred execution**: If `read_adc()` or `uart_send()` are long, prefer to make them non‑blocking or move heavy work to a lower‑priority context. The library executes the chain in the scheduler context; that context is typically the ISR or a short deferred handler.  
- **Stack usage**: The pipeline is executed inline; ensure stack usage in ISR context is bounded and tested.  
- **No heap**: All sender state must be static or in preallocated buffers.  
- **Error handling**: This minimal pipeline assumes success; see Part 2 for error paths.

---

# Part 2 — More complex example: parallel sampling + `when_all`, error handling, and resource management

### Scenario

Two sensors (ADC1 and ADC2) must be sampled at the same 10 ms tick. After both samples are available, compute a fused value and send it. If either sensor read fails, report an error via a telemetry sender.

### Goals

- Use `when_all` to wait for both sensor senders to complete for a single tick.  
- Keep everything ISR‑safe and allocation‑free.  
- Provide deterministic error handling and backpressure considerations.

### API primitives used

- `async::when_all(s1, s2)` — completes when both `s1` and `s2` complete; yields a tuple of results.  
- `async::let_value`, `async::let_error` — transform success and error paths.  
- `async::start_detached` — register pipeline.

### Code (detailed)

```cpp
#include <async/periodic.hpp>
#include <async/let_value.hpp>
#include <async/let_error.hpp>
#include <async/when_all.hpp>
#include <async/start_detached.hpp>
#include <async/trigger_scheduler.hpp>

using namespace std::chrono_literals;

async::trigger_scheduler irq_sched{};

struct SensorResult { bool ok; int value; };

SensorResult read_adc1();
SensorResult read_adc2();
int fuse_values(int a, int b);
void telemetry_report_error(int code);
void uart_send(int);

void setup_pipeline() {
    // Each sensor is read on each tick; wrap each read in a sender
    auto s1 = async::periodic(irq_sched, 10ms)
              | async::let_value([](){ return read_adc1(); });

    auto s2 = async::periodic(irq_sched, 10ms)
              | async::let_value([](){ return read_adc2(); });

    // Combine both sensor results for the same tick
    auto combined = async::when_all(s1, s2)
        | async::let_value([](std::tuple<SensorResult, SensorResult> tup) {
            auto [r1, r2] = tup;
            if (!r1.ok) return async::just_value(std::variant<int, int>{1}); // error code 1
            if (!r2.ok) return async::just_value(std::variant<int, int>{2}); // error code 2
            int fused = fuse_values(r1.value, r2.value);
            return async::just_value(std::variant<int, int>{fused});
        })
        | async::let_value([](std::variant<int, int> v) {
            if (std::holds_alternative<int>(v)) {
                int fused = std::get<int>(v);
                uart_send(fused);
            } else {
                int err = std::get<int>(v);
                telemetry_report_error(err);
            }
            return async::just_value(0);
        });

    async::start_detached(combined);
}
```

### Explanation and pitfalls

- **`when_all` semantics**: On each tick, both `s1` and `s2` will be scheduled. `when_all` completes only when both have produced values for that tick. Implementation detail: the library must correlate the two emissions to the same tick; Intel’s baremetal S/R typically ties them to the same scheduler wake or uses sequence counters.  
- **Error representation**: The example uses `std::variant` to represent either an error code or a fused value. In a real baremetal environment, prefer a compact tagged union or an integer status to avoid heavy C++ runtime features.  
- **Atomicity and ordering**: If `s1` and `s2` are both `periodic` on the same scheduler, they will be emitted in some deterministic order; `when_all` must buffer the first until the second arrives. Ensure the buffering is bounded.  
- **Memory footprint**: `when_all` needs temporary storage for the two results per tick. Allocate this statically or in a ring buffer sized for the maximum number of concurrent ticks you allow (usually 1 for strict real‑time pipelines).  
- **Backpressure**: If downstream `uart_send` is slow, you must ensure the pipeline either drops samples, signals backpressure, or uses a bounded queue. On baremetal, prefer dropping or sampling at a lower rate rather than unbounded buffering.  
- **Deterministic failure handling**: Use `let_error` or explicit error branches to handle sensor failures without exceptions.

### Example: bounded buffering for `when_all`

If `when_all` needs to buffer results, implement a small ring buffer with capacity `N` (often `N=1` for synchronous tick pipelines). Keep the buffer in static memory and ensure ISR-safe access (disable interrupts briefly or use atomic flags).

---

# Part 3 — Binding pipeline stages to specific interrupts using `incite_on`

### Motivation

On microcontrollers, different hardware events have different priorities and contexts. You may want:

- ADC sampling to run in a high‑priority timer ISR (fast, deterministic),  
- Heavy processing to run in a lower‑priority deferred interrupt or background handler,  
- UART transmission to run in a DMA completion ISR or a low‑priority context.

`incite_on` (or similarly named API) lets you attach a sender or a stage to a specific scheduler/interrupt so that that stage executes in the chosen context.

### Key concepts

- **Scheduler per interrupt**: Create `trigger_scheduler` instances bound to different ISRs (e.g., `timer_sched`, `processing_sched`, `uart_sched`).  
- **`incite_on(sched, sender)`**: returns a sender that, when the upstream emits, will schedule the downstream work on `sched` rather than executing inline.  
- **ISR safety**: `incite_on` must be implemented without dynamic allocation and must be safe to call from ISR context. It typically enqueues a small descriptor into a lock‑free ring buffer or sets a flag that the target ISR polls.

### Example architecture

- `timer_sched` — high priority, triggers ADC sampling every 10 ms.  
- `proc_sched` — lower priority, used for heavier processing.  
- `uart_sched` — lowest priority, used for UART send operations.

### Code example

```cpp
#include <async/periodic.hpp>
#include <async/let_value.hpp>
#include <async/incite_on.hpp>
#include <async/start_detached.hpp>
#include <async/trigger_scheduler.hpp>

using namespace std::chrono_literals;

async::trigger_scheduler timer_sched{}; // high priority timer ISR
async::trigger_scheduler proc_sched{};  // lower priority deferred handler
async::trigger_scheduler uart_sched{};  // UART handler or DMA completion

int read_adc();
int heavy_process(int);
void uart_send_dma(int);

void setup_pipeline() {
    auto pipeline =
        async::periodic(timer_sched, 10ms)
        | async::let_value([](){ return read_adc(); })
        // move heavy processing to proc_sched
        | async::incite_on(proc_sched)
        | async::let_value([](int raw){ return heavy_process(raw); })
        // move UART send to uart_sched (e.g., to use DMA or low-priority ISR)
        | async::incite_on(uart_sched)
        | async::let_value([](int processed){
            uart_send_dma(processed);
            return async::just_value(processed);
        });

    async::start_detached(pipeline);
}

// Timer ISR
extern "C" void TIM_IRQHandler() {
    timer_sched.trigger();
}

// proc_sched handler (lower priority) — platform-specific
extern "C" void PROC_IRQHandler() {
    proc_sched.trigger();
}

// uart_sched handler
extern "C" void UART_IRQHandler() {
    uart_sched.trigger();
}
```

### How `incite_on` typically works internally

1. **Upstream emits**: When the upstream sender produces a value, `incite_on` does not execute the downstream inline.  
2. **Enqueue descriptor**: `incite_on` places a small descriptor (pointer/ID and value or index into a preallocated buffer) into a lock‑free ring buffer associated with the target scheduler.  
3. **Trigger target scheduler**: `incite_on` triggers the target scheduler (e.g., sets a pending interrupt or writes to an event register) so the target ISR will run soon.  
4. **Target ISR runs**: The target ISR dequeues descriptors and executes the downstream stages in its context.

This design keeps the upstream ISR short and moves heavier work to a lower‑priority context.

### ISR safety and implementation details

- **Descriptor size**: Keep descriptors small (pointer + small payload index). Avoid copying large payloads in ISR; instead, write the payload into a preallocated slot and pass the slot index.  
- **Preallocated pools**: Use a fixed array of slots for payloads and a lock‑free ring buffer of indices. If the pool is full, decide whether to drop the sample or overwrite oldest data.  
- **Atomic operations**: Use atomic head/tail indices for ring buffers. On Cortex‑M, `__disable_irq()` for a few cycles is acceptable if the critical section is tiny.  
- **Priority inversion**: Ensure that the target ISR priority is lower (numerically higher) than the source ISR to avoid preemption issues.  
- **No blocking**: Target ISR must not block; use DMA for long transfers.

### Example: bounded slot pool for `incite_on`

```cpp
constexpr size_t POOL_SIZE = 8;
struct Slot { int value; bool used; } slots[POOL_SIZE];
std::atomic<uint8_t> ring_head{0}, ring_tail{0};

bool try_enqueue(int v) {
    uint8_t head = ring_head.load(std::memory_order_relaxed);
    uint8_t next = (head + 1) % POOL_SIZE;
    if (next == ring_tail.load(std::memory_order_acquire)) return false; // full
    slots[head].value = v;
    slots[head].used = true;
    ring_head.store(next, std::memory_order_release);
    return true;
}

bool try_dequeue(int &out) {
    uint8_t tail = ring_tail.load(std::memory_order_relaxed);
    if (tail == ring_head.load(std::memory_order_acquire)) return false; // empty
    out = slots[tail].value;
    slots[tail].used = false;
    ring_tail.store((tail + 1) % POOL_SIZE, std::memory_order_release);
    return true;
}
```

`incite_on` would use `try_enqueue` in ISR context and trigger the target scheduler; the target ISR calls `try_dequeue` repeatedly to process available slots.

### Example: moving heavy work out of ISR

- `read_adc()` runs in `timer_sched` (fast, minimal work).  
- `heavy_process()` runs in `proc_sched` (deferred, lower priority).  
- `uart_send_dma()` runs in `uart_sched` and uses DMA to avoid blocking.

This pattern keeps high‑priority ISRs short and deterministic.

---

# Cross‑cutting concerns, best practices, and debugging

### Lifetime and ownership

- Senders and pipelines must outlive the scheduler registration. Use static storage or ensure `start_detached` stores the pipeline state in static memory.  
- Avoid capturing pointers to stack objects in lambdas used by senders unless you guarantee the stack object outlives the pipeline.

### Memory usage

- Keep per‑tick buffers small. For `when_all`, prefer `N=1` buffering unless you explicitly need to queue multiple ticks.  
- Use fixed‑size arrays and ring buffers. Avoid `std::vector` or dynamic containers.

### Error handling

- Use explicit error channels (`let_error`) and compact error codes.  
- Do not use exceptions; the library is designed for `noexcept` environments.

### Testing and validation

- **Unit test**: Run pipeline logic in a host simulator where `trigger_scheduler` can be invoked from a test harness.  
- **Timing test**: Measure ISR latency and worst‑case execution time for each pipeline stage.  
- **Stress test**: Fill the `incite_on` slot pool to ensure the system behaves as expected under overload (drop vs block).

### Debugging tips

- Add lightweight instrumentation counters (atomic counters) for emitted ticks, processed items, and dropped items.  
- Use a logic analyzer to observe ISR timing and UART DMA activity.  
- Keep a small debug UART message path that is only enabled in development builds.

---

# Summary checklist before deployment

- [ ] All sender state is statically allocated or preallocated.  
- [ ] ISR handlers are short; heavy work is deferred via `incite_on` or lower‑priority schedulers.  
- [ ] `when_all` buffering is bounded and sized appropriately.  
- [ ] No exceptions or heap allocations are used in pipeline code.  
- [ ] DMA is used for long transfers (UART, SPI) where possible.  
- [ ] Priorities of ISRs are configured to avoid preemption of critical timers.  
- [ ] Thorough timing and stress tests are performed on target hardware.

# Asynchronous I/O with io_uring

## Introduction to io_uring

`io_uring` is a Linux kernel interface for high-performance asynchronous I/O.
It uses two shared ring buffers between user space and the kernel:

- The **submission queue** (**SQ**) carries work requests into the kernel.
- The **completion queue** (**CQ**) carries results back to user space.

This design reduces syscalls, avoids per-operation allocations, and allows
batching. Instead of calling `read()` or `send()` and waiting, you prepare a
submission entry (SQE), notify the kernel, and later collect completion
entries (CQEs).

### Key concepts

- **SQE (Submission Queue Entry):** describes one operation, such as read,
	write, accept, or connect.
- **CQE (Completion Queue Entry):** result for a completed operation, including
	return value and user data.
- **User data:** an opaque pointer or integer used to match a CQE to its
	request.
- **Submission/completion batching:** submit many SQEs with a single syscall
	and reap many CQEs at once.

### Why it matters

Traditional async I/O on Linux often relies on `epoll` + nonblocking I/O or
`aio` with thread pools. `io_uring` provides a unified interface for file and
network I/O with lower overhead and higher throughput. It also supports
features like fixed buffers, registered files, and linked operations.

### Basic flow (high level)

1. Initialize an `io_uring` instance (rings shared with the kernel).
2. Get an SQE and fill it with an operation.
3. Submit to the kernel.
4. Wait for CQEs and process results.

Below is a small pseudocode sketch using `liburing`:

```cpp
io_uring ring;
io_uring_queue_init(256, &ring, 0);

int fd = open("data.bin", O_RDONLY);
char buf[4096];

io_uring_sqe* sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, fd, buf, sizeof(buf), 0);
sqe->user_data = 1; // track request

io_uring_submit(&ring);

io_uring_cqe* cqe;
io_uring_wait_cqe(&ring, &cqe);
int bytes = cqe->res;
io_uring_cqe_seen(&ring, cqe);
```

This example blocks on `io_uring_wait_cqe()`, but you can also poll, peek, or
integrate with an event loop.

## Asynchronous File I/O with io_uring

`io_uring` supports asynchronous file reads and writes with fewer context
switches than thread-pool based approaches. The most common operations are
`IORING_OP_READ`, `IORING_OP_WRITE`, and their vectorized variants.

### Typical patterns

- **Read-ahead:** submit multiple reads for sequential blocks.
- **Scatter/gather I/O:** use vectored reads/writes to reduce syscalls.
- **Fixed buffers:** register a set of buffers to avoid per-I/O pinning.
- **Registered files:** avoid repeated file descriptor lookups.

### Example: batched reads

The snippet below submits several reads and then reaps all completions. This
illustrates how batching reduces syscalls.

```cpp
constexpr int kBatch = 8;
io_uring ring;
io_uring_queue_init(256, &ring, 0);

int fd = open("data.bin", O_RDONLY);
std::array<std::array<char, 4096>, kBatch> bufs;

for (int i = 0; i < kBatch; ++i) {
	io_uring_sqe* sqe = io_uring_get_sqe(&ring);
	io_uring_prep_read(sqe, fd, bufs[i].data(), bufs[i].size(), i * 4096);
	sqe->user_data = static_cast<unsigned long long>(i);
}

io_uring_submit(&ring);

for (int i = 0; i < kBatch; ++i) {
	io_uring_cqe* cqe;
	io_uring_wait_cqe(&ring, &cqe);
	int bytes = cqe->res; // negative values are -errno
	io_uring_cqe_seen(&ring, cqe);
}
```

### Error handling

When a CQE arrives, `cqe->res` is the return value of the operation. On error
it is `-errno`. This mirrors normal Linux syscalls but without throwing.

### Performance notes

- Avoid submitting tiny reads/writes; batch or use larger buffers.
- Consider `O_DIRECT` if you want to bypass the page cache (tradeoffs apply).
- Register fixed buffers if you reuse them often; it reduces overhead.

### Common file I/O caveats

- Some filesystems may not support all features equally.
- For synchronous semantics (e.g., `fsync`), use `IORING_OP_FSYNC` or
	`IORING_OP_FSYNC` with the appropriate flags.
- Buffer lifetimes must outlive the operation; do not reuse or free memory
	until the CQE has been received.

## Asynchronous Network I/O with io_uring

`io_uring` can replace the classic `epoll` + nonblocking socket pattern. It
supports operations such as `accept`, `connect`, `recv`, `send`, and `recvmsg`.
You submit socket operations the same way as file I/O and handle CQEs when
they complete.

### Server-side pattern

1. Submit an `accept` SQE for each listening socket.
2. When an `accept` completes, submit `recv` or `read` for the new connection.
3. Re-issue another `accept` to keep the server hot.

### Example: async accept + recv

```cpp
io_uring ring;
io_uring_queue_init(256, &ring, 0);

int server_fd = setup_listening_socket(8080);

io_uring_sqe* sqe = io_uring_get_sqe(&ring);
io_uring_prep_accept(sqe, server_fd, nullptr, nullptr, 0);
sqe->user_data = 100; // tag as accept

io_uring_submit(&ring);

while (true) {
	io_uring_cqe* cqe;
	io_uring_wait_cqe(&ring, &cqe);

	if (cqe->user_data == 100) {
		int client_fd = cqe->res;
		io_uring_cqe_seen(&ring, cqe);

		io_uring_sqe* recv_sqe = io_uring_get_sqe(&ring);
		io_uring_prep_recv(recv_sqe, client_fd, buffer, buffer_len, 0);
		recv_sqe->user_data = 200; // tag as recv

		io_uring_sqe* accept_sqe = io_uring_get_sqe(&ring);
		io_uring_prep_accept(accept_sqe, server_fd, nullptr, nullptr, 0);
		accept_sqe->user_data = 100; // re-arm accept

		io_uring_submit(&ring);
	} else if (cqe->user_data == 200) {
		int bytes = cqe->res;
		io_uring_cqe_seen(&ring, cqe);
		// process data and optionally queue a send
	}
}
```

### Advantages over epoll

- Fewer syscalls: submit many socket ops at once, reap many completions at once.
- Unified model: files and sockets use the same API.
- Reduced wakeups: the kernel can complete operations without extra readiness
	notifications.

### Practical considerations

- Always handle `-EAGAIN` or transient errors and re-submit when needed.
- Ensure buffers stay valid until CQE arrives.
- For high-scale servers, consider fixed buffers and registered file
	descriptors for improved throughput.

### Where it fits

`io_uring` shines for I/O-heavy applications such as high-throughput servers,
log ingestion pipelines, or storage systems. For smaller or CPU-bound programs,
the complexity may not pay off. A hybrid model is common: use `io_uring` for
I/O, then hand off to worker threads for CPU-bound tasks.
